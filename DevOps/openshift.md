# OpenShift — Interview Fundamentals

## What is OpenShift?
Red Hat OpenShift is an enterprise Kubernetes platform (a distribution of Kubernetes) with enhanced developer experience, built-in CI/CD, security hardening, and enterprise support. It runs on-premises, on the major clouds (AWS, Azure, GCP), or as a fully-managed service (ROSA, ARO).

---

## OpenShift vs Vanilla Kubernetes

| Feature | Kubernetes | OpenShift |
|---------|-----------|-----------|
| Installation | Manual | Automated (IPI/UPI) |
| Security | Permissive by default | Hardened (SCCs, restricted) |
| Container Runtime | Any (CRI-O, containerd) | CRI-O |
| Image Registry | External | Built-in (Quay/internal) |
| CI/CD | External | OpenShift Pipelines (Tekton) |
| Routes | Not built-in | Built-in (HAProxy) |
| Web Console | Basic | Rich, developer-focused |
| Operators | Available | OperatorHub built-in |
| Multi-tenancy | Manual RBAC | Projects with quotas |
| CLI | `kubectl` | `oc` (superset of kubectl) |

---

## Architecture

```
OpenShift Cluster
  ├── Control Plane (Master) Nodes
  │     ├── API Server (kube-apiserver)
  │     ├── etcd (cluster state store)
  │     ├── Controller Manager
  │     ├── Scheduler
  │     └── OpenShift-specific controllers
  │
  ├── Worker (Compute) Nodes
  │     ├── kubelet
  │     ├── CRI-O (container runtime)
  │     ├── kube-proxy
  │     └── Pods (application workloads)
  │
  └── Infrastructure Nodes
        ├── Ingress controllers (HAProxy)
        ├── Image Registry
        └── Monitoring stack
```

---

## Core OpenShift Concepts

### Projects (Namespaces)
OpenShift wraps Kubernetes namespaces in a **Project** resource that adds access control and resource quotas.
```bash
oc new-project myapp-prod \
  --display-name="MyApp Production" \
  --description="Production workloads for MyApp"

oc projects                  # list all projects
oc project myapp-prod        # switch project
oc get all -n myapp-prod     # all resources in project
```

### Security Context Constraints (SCCs)
OpenShift's security policy system — more granular than Kubernetes Pod Security Standards.

| SCC | Description |
|-----|-------------|
| `restricted` | Default; no root, drops all capabilities |
| `restricted-v2` | Most restrictive (OCP 4.11+) |
| `anyuid` | Run as any UID (including root) |
| `privileged` | Full access (infrastructure only) |
| `nonroot` | Must not run as root |

```bash
oc get scc
oc describe scc restricted
oc adm policy add-scc-to-serviceaccount anyuid -z myapp-sa -n mynamespace
oc adm policy who-can use scc privileged
```

> In OpenShift, containers run with a **random UID** by default (from the namespace's allocated UID range). Container images must handle this (don't hardcode UID in permissions).

---

## Workload Resources

### Deployment (standard Kubernetes)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trading-api
  namespace: trading-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: trading-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: trading-api
    spec:
      serviceAccountName: trading-sa
      securityContext:
        runAsNonRoot: true
      containers:
        - name: api
          image: registry.example.com/trading/api:v2.1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

### DeploymentConfig (OpenShift-native — legacy)
OpenShift's own deployment resource with additional triggers (ImageStream, config change). In OCP 4.x, prefer standard `Deployment`.
```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
spec:
  triggers:
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames: [api]
        from:
          kind: ImageStreamTag
          name: 'trading-api:latest'
    - type: ConfigChange
  strategy:
    type: Rolling
```

---

## OpenShift-Specific Resources

### Routes (Ingress)
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: trading-api
spec:
  host: trading-api.apps.cluster.example.com
  to:
    kind: Service
    name: trading-api
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: edge               # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect
```

```bash
oc expose svc/trading-api                            # create route from service
oc expose svc/trading-api --hostname=myapp.apps...  # custom hostname
oc get routes
```

### ImageStreams
Internal image registry abstraction. Triggers deployments when image is updated.
```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: trading-api
spec:
  lookupPolicy:
    local: false
```

```bash
oc import-image trading-api:v2.1 \
  --from=registry.example.com/trading/api:v2.1 \
  --confirm
oc tag trading-api:v2.1 trading-api:latest
oc get imagestream trading-api
oc describe istag trading-api:latest
```

### BuildConfigs
```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: trading-api
spec:
  source:
    type: Git
    git:
      uri: https://bitbucket.org/org/trading-api.git
      ref: main
    contextDir: /
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
  output:
    to:
      kind: ImageStreamTag
      name: 'trading-api:latest'
  triggers:
    - type: GitHub
    - type: ConfigChange
    - type: ImageChange
```

```bash
oc start-build trading-api
oc start-build trading-api --follow       # stream build logs
oc logs bc/trading-api -f
oc cancel-build trading-api-5
```

---

## Configuration & Secrets

```bash
# ConfigMaps
oc create configmap app-config \
  --from-file=config.properties \
  --from-literal=LOG_LEVEL=INFO

# Secrets
oc create secret generic db-creds \
  --from-literal=username=appuser \
  --from-literal=password=secret123

oc create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=robot \
  --docker-password="$(cat token)"

# Mount in pod
oc set env deploy/myapp --from=secret/db-creds
oc set volume deploy/myapp \
  --add --name=config-vol \
  --type=configmap \
  --configmap-name=app-config \
  --mount-path=/etc/config
```

---

## Networking

### Service Types
| Type | Description |
|------|-------------|
| `ClusterIP` | Internal cluster IP (default) |
| `NodePort` | Expose on every node's IP |
| `LoadBalancer` | Cloud LB (use Route in OCP) |
| `ExternalName` | DNS alias |

### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-trading-only
spec:
  podSelector:
    matchLabels:
      app: trading-api
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - port: 3306
```

### OpenShift SDN / OVN-Kubernetes
- Default CNI: **OVN-Kubernetes** (OCP 4.12+)
- Provides: network isolation per namespace, network policies, egress IPs, load balancing

---

## Storage

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp2-csi       # cloud or OCP storage class
```

```bash
oc get pvc
oc get pv
oc get storageclass
oc describe pvc app-data
```

### Storage Classes (examples)
| Class | Provider |
|-------|---------|
| `gp2-csi` | AWS EBS (ReadWriteOnce) |
| `ocs-storagecluster-ceph-rbd` | ODF/Ceph (ReadWriteOnce) |
| `ocs-storagecluster-cephfs` | ODF/CephFS (ReadWriteMany) |

---

## Resource Management & Quotas

### Resource Quotas
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: trading-prod-quota
spec:
  hard:
    pods: "50"
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "20"
```

### LimitRange (default per pod)
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
    - type: Container
      default:
        memory: "512Mi"
        cpu: "500m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "4Gi"
        cpu: "4"
```

---

## RBAC

```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-deployer
  namespace: trading-prod
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: trading-prod
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: cicd
roleRef:
  kind: Role
  name: app-deployer
  apiGroup: rbac.authorization.k8s.io
```

```bash
# OCP shorthand
oc adm policy add-role-to-user edit user@example.com -n trading-prod
oc adm policy add-cluster-role-to-user cluster-admin admin@example.com
oc auth can-i create deployments --as=system:serviceaccount:trading-prod:ci-deployer
```

### Default OCP Roles
| Role | Permissions |
|------|-------------|
| `view` | Read-only access |
| `edit` | Read/write resources, no RBAC |
| `admin` | Full project access including RBAC |
| `cluster-admin` | Full cluster access |

---

## Operators

Operators encode operational knowledge as Kubernetes controllers. They manage the full lifecycle of complex applications.

```bash
oc get operators
oc get csv -n openshift-operators         # ClusterServiceVersions (installed operators)
oc get subscription -n openshift-operators
oc describe clusterserviceversion elasticsearch-operator.v5.8.0
```

### Operator Lifecycle Manager (OLM)
Manages installation, upgrades, and RBAC of operators. Operators are installed via `Subscription` → `InstallPlan` → `ClusterServiceVersion`.

---

## Monitoring & Logging

### Built-in Monitoring (Prometheus + Grafana)
```bash
# Access via OpenShift console: Observe → Metrics / Dashboards
oc get prometheus -n openshift-monitoring
oc get servicemonitor -n openshift-monitoring

# Port-forward to Prometheus
oc port-forward -n openshift-monitoring svc/prometheus-operated 9090
```

### Custom Metrics (User Workload Monitoring)
```yaml
# Enable in cluster-monitoring-config
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

### Logging (OpenShift Logging Operator → EFK/Loki)
```bash
oc get clusterlogging -n openshift-logging
oc get logforwarder -n openshift-logging
oc logs -f pod/trading-api-7d4f9b-xkp2n
oc logs -f deploy/trading-api --tail=100
oc logs -f bc/trading-api             # build logs
```

---

## `oc` CLI Essentials

```bash
# Cluster info
oc version
oc cluster-info
oc get nodes
oc describe node worker-01

# Workloads
oc get pods -o wide
oc get pods -l app=trading-api
oc describe pod trading-api-abc123
oc exec -it trading-api-abc123 -- bash
oc rsh trading-api-abc123              # OCP shortcut for exec -it bash

# Deployments
oc rollout status deploy/trading-api
oc rollout history deploy/trading-api
oc rollout undo deploy/trading-api
oc rollout pause deploy/trading-api
oc rollout resume deploy/trading-api
oc scale deploy/trading-api --replicas=5
oc set image deploy/trading-api api=registry.example.com/api:v2.2

# Port forwarding (local debugging)
oc port-forward svc/trading-api 8080:8080
oc port-forward pod/trading-api-abc123 8080:8080

# Resource management
oc apply -f manifests/
oc delete -f manifests/
oc get events --sort-by='.lastTimestamp'
oc top pods
oc top nodes

# Debug
oc debug pod/trading-api-abc123              # launch debug container
oc debug node/worker-01                      # debug node (privileged)
oc adm top nodes
oc adm must-gather                           # collect cluster diagnostics
```

---

## Common Interview Questions

### Q1: How does OpenShift differ from Kubernetes?
OpenShift is an enterprise Kubernetes distribution with: built-in Routes (ingress), Source-to-Image (S2I) builds, ImageStreams, enhanced RBAC/SCCs, integrated monitoring/logging, OperatorHub, and tighter security defaults.

### Q2: What are Security Context Constraints (SCCs)?
OpenShift's admission control mechanism for pod security — determines what a pod is allowed to do (run as root, use host network, mount volumes, etc.). Stricter than Kubernetes Pod Security Standards.

### Q3: What is the difference between a Route and a Service?
- **Service**: Internal L4 load balancer across pod IPs (cluster-internal)
- **Route**: External L7 URL exposure via HAProxy ingress controller; adds TLS termination and hostname routing

### Q4: What is an ImageStream?
An abstraction over container images stored in OpenShift's internal registry. Acts as a pointer that can trigger automated deployments when the referenced image is updated.

### Q5: How does rolling deployment work in OpenShift?
New pods are created incrementally while old pods are removed. Controlled by `maxSurge` (extra pods during rollout) and `maxUnavailable` (pods that can be down). Health checks (readiness probes) gate the rollout.

### Q6: How do you debug a CrashLoopBackOff pod?
```bash
oc describe pod <pod>         # check Events section for root cause
oc logs <pod> --previous      # logs from crashed container
oc debug pod/<pod>            # start debug container with same spec
oc exec -it <pod> -- sh       # if pod is running momentarily
```

### Q7: What is a liveness vs readiness probe?
- **Readiness**: Is the pod ready to receive traffic? Failing → removed from Service endpoints
- **Liveness**: Is the pod alive? Failing → pod is restarted

### Q8: How do you perform a zero-downtime deployment?
1. Set `maxUnavailable: 0` in rolling update strategy
2. Configure readiness probe correctly
3. Use `preStop` hook for graceful shutdown
4. Ensure `terminationGracePeriodSeconds` is sufficient
5. Use PodDisruptionBudgets to prevent too many pods being down

### Q9: What is an Operator?
A Kubernetes controller that encodes operational knowledge for managing complex applications. It watches custom resources (CRDs) and reconciles actual state with desired state — handling installation, upgrades, backups, and failure recovery.

### Q10: How do you restrict inter-namespace communication?
Use **NetworkPolicies** to define allowed ingress/egress traffic. By default in OpenShift, namespaces cannot communicate unless permitted by policy. The `default-deny` pattern blocks all traffic, then explicit policies allow specific paths.
