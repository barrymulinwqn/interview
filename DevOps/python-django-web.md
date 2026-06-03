# Python, Django, HTML5, CSS & JavaScript — Interview Fundamentals

---

## PYTHON

### Core Language

#### Data Types & Structures
```python
# Immutable: int, float, str, bytes, tuple, frozenset, bool, None
# Mutable:   list, dict, set, bytearray

# List comprehensions
squares = [x**2 for x in range(10) if x % 2 == 0]

# Dict comprehension
word_len = {w: len(w) for w in ["hello", "world"]}

# Set comprehension
unique_evens = {x for x in range(20) if x % 2 == 0}

# Generator expression (lazy evaluation — memory efficient)
total = sum(x**2 for x in range(1_000_000))

# Unpacking
a, *rest, b = [1, 2, 3, 4, 5]   # a=1, rest=[2,3,4], b=5
x, y = y, x                       # swap in one line
```

#### Functions
```python
def process(data: list, *, key: str = "id", reverse: bool = False) -> list:
    """Keyword-only args after *."""
    return sorted(data, key=lambda d: d[key], reverse=reverse)

# *args and **kwargs
def log(*args, **kwargs):
    print(args, kwargs)

# Lambda
transform = lambda x: x * 2

# Closures
def make_multiplier(n):
    def multiply(x):
        return x * n
    return multiply
double = make_multiplier(2)

# Decorators
import functools
def retry(times=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
        return wrapper
    return decorator

@retry(times=3)
def call_api(url): ...
```

#### Classes & OOP
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import ClassVar

class Animal(ABC):
    species_count: ClassVar[int] = 0

    def __init__(self, name: str):
        self.name = name
        Animal.species_count += 1

    @abstractmethod
    def speak(self) -> str: ...

    @classmethod
    def from_dict(cls, data: dict) -> "Animal":
        return cls(data["name"])

    @staticmethod
    def is_alive() -> bool:
        return True

    def __repr__(self) -> str:
        return f"{type(self).__name__}(name={self.name!r})"

class Dog(Animal):
    def speak(self) -> str:
        return "Woof"

# Dataclass (auto __init__, __repr__, __eq__)
@dataclass
class Trade:
    trade_id: str
    symbol: str
    quantity: float
    price: float
    tags: list = field(default_factory=list)

    @property
    def notional(self) -> float:
        return self.quantity * self.price
```

#### Error Handling
```python
class AppError(Exception):
    """Base application exception."""
    def __init__(self, message: str, code: int = 500):
        super().__init__(message)
        self.code = code

try:
    result = risky_operation()
except (ValueError, KeyError) as e:
    logger.error("Validation failed: %s", e)
    raise AppError(str(e), code=400) from e
except Exception:
    logger.exception("Unexpected error")
    raise
else:
    process(result)             # runs if no exception
finally:
    cleanup()                   # always runs

# Context managers
class ManagedResource:
    def __enter__(self):
        return self
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False            # re-raise exceptions

# Using contextlib
from contextlib import contextmanager
@contextmanager
def timer(label: str):
    import time
    start = time.monotonic()
    try:
        yield
    finally:
        print(f"{label}: {time.monotonic() - start:.3f}s")
```

#### Concurrency
```python
# Threading (I/O-bound tasks, GIL)
import threading
thread = threading.Thread(target=worker, args=(data,), daemon=True)
thread.start()
thread.join()

# Multiprocessing (CPU-bound tasks, bypasses GIL)
from multiprocessing import Pool
with Pool(processes=4) as pool:
    results = pool.map(cpu_task, data_chunks)

# asyncio (async I/O — single thread, event loop)
import asyncio
async def fetch(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

async def main():
    tasks = [fetch(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)

asyncio.run(main())
```

#### Useful Standard Library Modules
| Module | Purpose |
|--------|---------|
| `os`, `pathlib` | Filesystem operations |
| `sys` | Interpreter info, argv |
| `subprocess` | Run external commands |
| `logging` | Structured logging |
| `json`, `csv` | Serialization |
| `datetime`, `zoneinfo` | Date/time handling |
| `collections` | Counter, defaultdict, deque, namedtuple |
| `itertools` | chain, groupby, product, combinations |
| `functools` | lru_cache, reduce, partial |
| `contextlib` | contextmanager, suppress |
| `threading`, `multiprocessing`, `asyncio` | Concurrency |
| `typing` | Type hints |
| `re` | Regular expressions |
| `hashlib`, `hmac` | Cryptographic hashing |

---

## DJANGO

### Architecture (MTV)
```
Browser → URL Router → View → Model → Database
                          └──→ Template → HTML Response
```

| Django | MVC Equivalent |
|--------|----------------|
| Model | Model |
| Template | View |
| View | Controller |

### Project Structure
```
myproject/
  manage.py
  myproject/
    settings/
      base.py
      production.py
      development.py
    urls.py
    wsgi.py
    asgi.py
  apps/
    trading/
      models.py
      views.py
      serializers.py   (DRF)
      urls.py
      admin.py
      forms.py
      tests/
      migrations/
```

### Models & ORM
```python
from django.db import models

class Trade(models.Model):
    class Status(models.TextChoices):
        PENDING   = 'P', 'Pending'
        FILLED    = 'F', 'Filled'
        CANCELLED = 'C', 'Cancelled'

    trade_id  = models.CharField(max_length=32, unique=True, db_index=True)
    symbol    = models.CharField(max_length=10)
    quantity  = models.DecimalField(max_digits=18, decimal_places=6)
    price     = models.DecimalField(max_digits=18, decimal_places=6)
    status    = models.CharField(max_length=1, choices=Status.choices, default=Status.PENDING)
    trader    = models.ForeignKey('auth.User', on_delete=models.PROTECT, related_name='trades')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']
        indexes  = [models.Index(fields=['symbol', 'status'])]
        verbose_name_plural = 'trades'

    def __str__(self):
        return f"{self.trade_id} {self.symbol}"

# ORM Queries
Trade.objects.all()
Trade.objects.filter(status='F', symbol='GBPUSD')
Trade.objects.exclude(status='C')
Trade.objects.get(trade_id='T001')                   # raises if not found
Trade.objects.filter(price__gte=1.25, price__lt=1.30)
Trade.objects.filter(created_at__date=date.today())
Trade.objects.filter(symbol__in=['GBPUSD', 'EURUSD'])
Trade.objects.select_related('trader')               # JOIN (ForeignKey)
Trade.objects.prefetch_related('tags')               # prefetch M2M
Trade.objects.values('symbol').annotate(
    count=Count('id'),
    total_notional=Sum(F('quantity') * F('price'))
).order_by('-total_notional')
Trade.objects.filter(...).update(status='C')
Trade.objects.filter(...).delete()
```

### Views
```python
# Function-Based View (FBV)
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods

@require_http_methods(["GET", "POST"])
def trades(request):
    if request.method == "GET":
        data = list(Trade.objects.values())
        return JsonResponse({"trades": data})

# Class-Based View (CBV)
from django.views import View
class TradeView(View):
    def get(self, request, trade_id):
        trade = get_object_or_404(Trade, trade_id=trade_id)
        return JsonResponse({"trade": model_to_dict(trade)})

# Django REST Framework ViewSet
from rest_framework import viewsets, permissions
class TradeViewSet(viewsets.ModelViewSet):
    queryset = Trade.objects.all().select_related('trader')
    serializer_class = TradeSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        qs = super().get_queryset()
        if symbol := self.request.query_params.get('symbol'):
            qs = qs.filter(symbol=symbol)
        return qs
```

### Middleware
```python
# settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'myapp.middleware.RequestLoggingMiddleware',
]
```

### Migrations
```bash
python manage.py makemigrations
python manage.py migrate
python manage.py showmigrations
python manage.py migrate app_name 0005    # migrate to specific version
python manage.py sqlmigrate app_name 0001 # show SQL
python manage.py migrate app_name zero    # rollback all
```

### Security Checklist
```python
DEBUG = False                            # NEVER True in production
ALLOWED_HOSTS = ['myapp.example.com']
SECRET_KEY = os.environ['DJANGO_SECRET_KEY']
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
X_FRAME_OPTIONS = 'DENY'
```

---

## HTML5

### Semantic Elements
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>App</title>
</head>
<body>
    <header><nav>...</nav></header>
    <main>
        <article>
            <section>
                <h1>Heading</h1>
                <p>Content</p>
            </section>
            <aside>Sidebar</aside>
        </article>
    </main>
    <footer>...</footer>
</body>
</html>
```

### Key HTML5 APIs
| API | Use Case |
|-----|---------|
| `<canvas>` | 2D drawing / charts |
| `<video>/<audio>` | Media without plugins |
| Local Storage / Session Storage | Client-side key-value storage |
| IndexedDB | Client-side structured storage |
| WebSockets | Full-duplex real-time communication |
| Geolocation | Location access |
| Web Workers | Background JS threads |
| Service Workers | Offline caching / PWA |
| Fetch API | Modern `XMLHttpRequest` replacement |

---

## CSS

### Box Model
```
margin > border > padding > content
```
`box-sizing: border-box` — width/height include padding and border (recommended default).

### Flexbox
```css
.container {
    display: flex;
    flex-direction: row;           /* row | column | row-reverse */
    justify-content: space-between; /* main axis alignment */
    align-items: center;           /* cross axis alignment */
    flex-wrap: wrap;
    gap: 1rem;
}
.item {
    flex: 1 1 200px;              /* grow shrink basis */
    order: 2;
}
```

### CSS Grid
```css
.grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    grid-template-rows: auto;
    gap: 1rem 2rem;
}
.item {
    grid-column: 1 / 3;           /* span 2 columns */
    grid-row: 1;
}
```

### Responsive Design
```css
/* Mobile-first approach */
.card { width: 100%; }

@media (min-width: 768px)  { .card { width: 50%; } }
@media (min-width: 1024px) { .card { width: 33%; } }

/* Modern: clamp() */
.heading { font-size: clamp(1.5rem, 4vw, 3rem); }
```

---

## JAVASCRIPT

### Modern ES6+ Features
```javascript
// Destructuring
const { name, age = 25, ...rest } = person;
const [first, , third] = array;

// Spread / Rest
const merged = { ...defaults, ...overrides };
const combined = [...arr1, ...arr2];
function sum(...numbers) { return numbers.reduce((a, b) => a + b, 0); }

// Template literals
const msg = `Hello ${name}, you have ${count} message${count !== 1 ? 's' : ''}`;

// Optional chaining & nullish coalescing
const city = user?.address?.city ?? 'Unknown';

// Modules
export const PI = 3.14;
export default class App { ... }
import App, { PI } from './app.js';

// Promises & async/await
async function fetchTrades(symbol) {
    try {
        const response = await fetch(`/api/trades?symbol=${symbol}`);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return await response.json();
    } catch (error) {
        console.error('Fetch failed:', error);
        throw error;
    }
}

// Promise.all (parallel)
const [trades, prices] = await Promise.all([fetchTrades(), fetchPrices()]);
// Promise.allSettled (don't fail on first rejection)
const results = await Promise.allSettled([p1, p2, p3]);
```

### Key Array Methods
```javascript
const nums = [1, 2, 3, 4, 5];

nums.map(x => x * 2)                   // [2, 4, 6, 8, 10]
nums.filter(x => x % 2 === 0)          // [2, 4]
nums.reduce((acc, x) => acc + x, 0)    // 15
nums.find(x => x > 3)                  // 4
nums.findIndex(x => x > 3)             // 3
nums.some(x => x > 4)                  // true
nums.every(x => x > 0)                 // true
nums.flat(Infinity)                     // flatten nested
nums.flatMap(x => [x, x * 2])          // map + flatten
```

### Event Loop & Asynchrony
```
Call Stack → Web APIs → Callback Queue (macrotasks) → Microtask Queue
                                                          (Promises, queueMicrotask)

Execution order:
1. Current synchronous code
2. Microtasks (Promise callbacks) — exhausted before next macrotask
3. Next macrotask (setTimeout, setInterval, I/O)
```

### Common Patterns
```javascript
// Debounce
function debounce(fn, delay) {
    let timer;
    return (...args) => {
        clearTimeout(timer);
        timer = setTimeout(() => fn(...args), delay);
    };
}

// Throttle
function throttle(fn, limit) {
    let inThrottle;
    return (...args) => {
        if (!inThrottle) {
            fn(...args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}
```

---

## Common Interview Questions

### Python

**Q1: GIL — what is it?**
Global Interpreter Lock — a mutex preventing multiple native threads from executing Python bytecode simultaneously. Means CPU-bound multithreading doesn't scale; use multiprocessing or async I/O.

**Q2: Mutable default argument trap?**
```python
# BAD
def append(item, lst=[]):
    lst.append(item); return lst

# GOOD
def append(item, lst=None):
    if lst is None: lst = []
    lst.append(item); return lst
```

**Q3: `__slots__`?**
Defines fixed attributes on a class, replacing the per-instance `__dict__`. Saves memory for classes with many instances.

### Django

**Q4: N+1 query problem in Django?**
Accessing a related object in a loop triggers one query per object. Fix: `select_related()` (JOIN) for ForeignKey, `prefetch_related()` for M2M/reverse FK.

**Q5: What is Django middleware?**
Hook into request/response processing. Runs in order during request, reverse order during response. Used for auth, logging, CORS, CSRF, caching.

**Q6: CSRF protection?**
Django includes a CSRF token in forms and cookies. The `CsrfViewMiddleware` validates the token on POST/PUT/DELETE requests, preventing cross-site request forgery.

### JavaScript

**Q7: `==` vs `===`?**
`==` coerces types before comparing; `===` strict equality, no coercion. Always prefer `===`.

**Q8: `var`, `let`, `const`?**
- `var`: function-scoped, hoisted (initialised as `undefined`)
- `let`: block-scoped, not initialised (temporal dead zone)
- `const`: block-scoped, must be initialised, binding immutable (not value)

**Q9: What is closure?**
A function that retains access to its enclosing scope even after the outer function has returned. Used for data encapsulation, factory functions, memoization.

**Q10: Event delegation?**
Attaching a single event listener to a parent element instead of individual children. Uses event bubbling. More efficient for dynamic lists.
```javascript
document.querySelector('#list').addEventListener('click', (e) => {
    if (e.target.matches('li.item')) handleClick(e.target);
});
```
