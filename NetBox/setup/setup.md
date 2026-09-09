# NetBox Community Setup on a Local Mac mini

This guide installs the NetBox Community Edition source release directly on a local Mac mini using Homebrew-managed PostgreSQL and Redis. It is suitable for learning, development, and a private lab. It deliberately binds NetBox to `127.0.0.1`; do not expose this development deployment to an untrusted network.

The commands work on both Apple Silicon and Intel Macs because they derive Homebrew paths with `brew --prefix`.

## What will be installed

| Component | Purpose | Version used by this guide |
| --- | --- | --- |
| NetBox Community | Network source-of-truth application | `v4.7.0` |
| Python | Runs NetBox | `3.13` |
| PostgreSQL | Persistent NetBox database | `17` |
| Redis | Cache and background-job queue | Homebrew current release |

NetBox officially requires Python 3.12 through 3.14, PostgreSQL 15 or later, and Redis 6.0 or later. PostgreSQL is the only supported relational database; MySQL is not supported.

## 1. Prerequisites

1. Install [Xcode Command Line Tools](https://developer.apple.com/xcode/resources/) if they are not already present:

	```bash
	xcode-select --install
	```

2. Install [Homebrew](https://brew.sh/) if necessary, then confirm it is available:

	```bash
	brew --version
	```

3. Update Homebrew and install NetBox's local dependencies:

	```bash
	brew update
	brew install git python@3.13 postgresql@17 redis
	```

4. Define paths for this terminal session. Add the `export PATH=...` line to `~/.zshrc` if the installation will be used regularly.

	```bash
	export PYTHON_BIN="$(brew --prefix python@3.13)/bin/python3.13"
	export PATH="$(brew --prefix python@3.13)/bin:$(brew --prefix postgresql@17)/bin:$PATH"

	"$PYTHON_BIN" --version
	psql --version
	redis-server --version
	```

	Expected results include Python 3.13, PostgreSQL 17, and Redis 6.0 or newer.

## 2. Initialize and start PostgreSQL

Homebrew's PostgreSQL service needs a local database cluster. The guarded initialization below is safe to run when the default Homebrew cluster already exists.

```bash
export PG_DATA="$(brew --prefix)/var/postgresql@17"

if [[ ! -f "$PG_DATA/PG_VERSION" ]]; then
  initdb --encoding=UTF8 --locale=C "$PG_DATA"
fi

brew services start postgresql@17
pg_isready
```

`pg_isready` should report that port `5432` is accepting connections. The initial Homebrew PostgreSQL administrator is normally the current macOS user, so no `sudo` is required.

### Create the NetBox database and role

1. Generate and retain a database password. The hexadecimal format is safe to use in the `psql` commands below.

	```bash
	export NETBOX_DB_PASSWORD="$(openssl rand -hex 32)"
	printf '%s\n' "$NETBOX_DB_PASSWORD"
	```

2. Create the `netbox` login role, its database, and schema permission. Run these commands once on a new installation.

	```bash
	psql postgres -v ON_ERROR_STOP=1 \
	  -v netbox_db_password="$NETBOX_DB_PASSWORD" \
	  -c "CREATE USER netbox WITH LOGIN PASSWORD :'netbox_db_password';"

	psql postgres -v ON_ERROR_STOP=1 \
	  -c "CREATE DATABASE netbox OWNER netbox ENCODING 'UTF8';"

	psql netbox -v ON_ERROR_STOP=1 \
	  -c "GRANT CREATE ON SCHEMA public TO netbox;"
	```

3. Confirm that the NetBox role can authenticate, then remove the transient password environment variable when finished.

	```bash
	PGPASSWORD="$NETBOX_DB_PASSWORD" psql \
	  --username netbox --host 127.0.0.1 --dbname netbox \
	  -c '\conninfo'

	unset PGPASSWORD
	```

The connection output should name database `netbox`, user `netbox`, and host `127.0.0.1`. Keep `NETBOX_DB_PASSWORD` available only until it has been placed in NetBox's local configuration in step 5.

## 3. Start and verify Redis

Redis stores NetBox's cache and queues background jobs. Keep it local: anyone able to write to NetBox's Redis task database could cause code to run in a worker process.

```bash
brew services start redis
redis-cli ping
```

The expected response is:

```text
PONG
```

Do not configure Redis to listen on a public interface for this local setup. NetBox will use two isolated logical Redis databases on this same local service: database `0` for tasks and database `1` for caching.

## 4. Download a pinned NetBox Community release

Using a versioned directory and a Git tag makes the installed version explicit and gives upgrades a clear starting point.

```bash
export NETBOX_VERSION="v4.7.0"
export NETBOX_HOME="$HOME/src/netbox-${NETBOX_VERSION#v}"

mkdir -p "$HOME/src"
git clone https://github.com/netbox-community/netbox.git "$NETBOX_HOME"
cd "$NETBOX_HOME"
git checkout "$NETBOX_VERSION"
git status --short --branch
```

The final command should show a detached `HEAD` at the selected release tag and no local changes. To select a newer release later, find its tag on the [NetBox releases page](https://github.com/netbox-community/netbox/releases).

## 5. Configure NetBox

1. Copy the upstream configuration template:

	```bash
	cd "$NETBOX_HOME/netbox/netbox"
	cp configuration_example.py configuration.py
	chmod 600 configuration.py
	```

2. Generate a distinct `SECRET_KEY` and API-token pepper. Save both outputs temporarily; they are needed in the next edit.

	```bash
	export NETBOX_SECRET_KEY="$("$PYTHON_BIN" "$NETBOX_HOME/generate_secret_key.py")"
	export NETBOX_API_TOKEN_PEPPER="$("$PYTHON_BIN" "$NETBOX_HOME/generate_secret_key.py")"

	printf 'SECRET_KEY=%s\nAPI_TOKEN_PEPPER=%s\n' \
	  "$NETBOX_SECRET_KEY" "$NETBOX_API_TOKEN_PEPPER"
	```

3. Open `configuration.py` in an editor. Replace the template values for the following required settings, substituting the values generated above and the `NETBOX_DB_PASSWORD` value from step 2.

	```python
	ALLOWED_HOSTS = ['localhost', '127.0.0.1']

	DATABASES = {
		 'default': {
			  'NAME': 'netbox',
			  'USER': 'netbox',
			  'PASSWORD': '<NETBOX_DB_PASSWORD>',
			  'HOST': '127.0.0.1',
			  'PORT': '',
			  'CONN_MAX_AGE': 300,
		 }
	}

	REDIS = {
		 'tasks': {
			  'HOST': '127.0.0.1',
			  'PORT': 6379,
			  'PASSWORD': '',
			  'DATABASE': 0,
			  'SSL': False,
		 },
		 'caching': {
			  'HOST': '127.0.0.1',
			  'PORT': 6379,
			  'PASSWORD': '',
			  'DATABASE': 1,
			  'SSL': False,
		 }
	}

	SECRET_KEY = '<NETBOX_SECRET_KEY>'

	API_TOKEN_PEPPERS = {
		 1: '<NETBOX_API_TOKEN_PEPPER>',
	}
	```

	Do not commit `configuration.py`, its database password, the secret key, or the API-token pepper to source control. The two Redis definitions must use separate database IDs.

4. Clear the shell secrets after confirming they are correctly stored in the configuration file:

	```bash
	unset NETBOX_DB_PASSWORD NETBOX_SECRET_KEY NETBOX_API_TOKEN_PEPPER
	```

## 6. Install Python dependencies and initialize NetBox

Run NetBox's provided upgrade script from the repository root. On a fresh installation it creates `venv`, installs all Python dependencies, applies PostgreSQL migrations, and collects static assets. No `sudo` is needed because the installation belongs to the current user.

```bash
cd "$NETBOX_HOME"
PYTHON="$PYTHON_BIN" ./upgrade.sh
```

When it completes without errors, activate its virtual environment and create the first administrator account:

```bash
source "$NETBOX_HOME/venv/bin/activate"
cd "$NETBOX_HOME/netbox"

python manage.py check
python manage.py createsuperuser
```

Enter the requested username, optional email address, and a strong password. The `check` command must finish without errors before proceeding.

## 7. Run NetBox locally

Use two Terminal windows. Both commands assume the environment from the previous step is active. The first terminal serves the web application only on the Mac mini; the second processes background jobs from Redis.

**Terminal 1 - NetBox web application**

```bash
source "$NETBOX_HOME/venv/bin/activate"
cd "$NETBOX_HOME/netbox"
python manage.py runserver 127.0.0.1:8000 --insecure
```

**Terminal 2 - NetBox background worker**

```bash
source "$NETBOX_HOME/venv/bin/activate"
cd "$NETBOX_HOME/netbox"
python manage.py rqworker
```

Open <http://127.0.0.1:8000/> and sign in with the superuser created in step 6. `runserver` is Django's development server; it is appropriate for this private local lab only, not for a production deployment.

Press `Ctrl+C` in each terminal to stop NetBox. PostgreSQL and Redis continue to run as Homebrew services.

## 8. Routine operations

### Start services after a restart

Homebrew services normally start automatically after logging in. Check their status with:

```bash
brew services list
pg_isready
redis-cli ping
```

Then start the NetBox web process and worker again using the two commands from step 7.

### Stop local services

```bash
brew services stop redis
brew services stop postgresql@17
```

### Upgrade NetBox

1. Read the [official upgrade guide](https://netbox.readthedocs.io/en/stable/installation/upgrading/) and the target release notes before changing versions.
2. Stop the local web process and worker.
3. Fetch tags, select the target version, and run the supplied upgrade script:

	```bash
	cd "$NETBOX_HOME"
	git fetch --tags
	git checkout vX.Y.Z
	PYTHON="$PYTHON_BIN" ./upgrade.sh
	```

4. Run `python manage.py check` in the virtual environment, then restart the web process and worker.

For larger or production installations, use NetBox's documented Gunicorn and reverse-proxy deployment instead of `runserver`.

## Troubleshooting

| Symptom | Check | Likely resolution |
| --- | --- | --- |
| `psql: command not found` | `echo "$PATH"` | Re-run the exports in step 1 or add them to `~/.zshrc`. |
| `pg_isready` is not accepting connections | `brew services list` | Start `postgresql@17`; if the service immediately stops, confirm `initdb` completed for `$PG_DATA`. |
| NetBox cannot connect to PostgreSQL | `PGPASSWORD='<password>' psql -U netbox -h 127.0.0.1 -d netbox` | Verify the database password and `HOST` in `configuration.py`. |
| Redis connection error | `redis-cli ping` | Start Redis and retain the local host/port values in `REDIS`. |
| `DisallowedHost` in the browser | Review `ALLOWED_HOSTS` | Access `127.0.0.1:8000` or add the exact hostname being used. |
| Background jobs do not complete | Inspect the `rqworker` terminal | Keep `python manage.py rqworker` running and verify Redis returns `PONG`. |

## Upstream references

This guide is adapted for macOS from the NetBox Community project's official source and installation documentation:

- [NetBox Community source repository](https://github.com/netbox-community/netbox)
- [NetBox release tags](https://github.com/netbox-community/netbox/releases)
- [Official installation overview and supported dependency versions](https://netbox.readthedocs.io/en/stable/installation/)
- [Official PostgreSQL setup](https://netbox.readthedocs.io/en/stable/installation/1-postgresql/)
- [Official Redis setup](https://netbox.readthedocs.io/en/stable/installation/2-redis/)
- [Official release/Git installation and configuration](https://netbox.readthedocs.io/en/stable/installation/3-netbox/)
