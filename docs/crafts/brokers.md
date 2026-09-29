---
icon: lucide/workflow
---

# Message Brokers

Run background tasks from Django with Celery, using Valkey, Redis, RabbitMQ or another broker. You switch brokers by changing a few environment variables.

The full working project is in the repository: [`django_brokers/`](https://github.com/yokwejuste/DjangoCrafts/tree/main/django_brokers).

## How it works

1. A Django view calls `task.delay()`, which puts a message on the **broker**.
2. A **Celery worker** takes the message off the queue and runs the task outside the request/response cycle.
3. The worker saves the task's state and return value in the **result backend**.
4. Django reads the result later with `AsyncResult(task_id)`.

| Service | Broker | Result backend | Django cache |
|---|---|---|---|
| Valkey | Yes | Yes | Yes |
| Redis | Yes | Yes | Yes |
| RabbitMQ | Yes | No (use Valkey/Redis) | No |

## Installation

```bash
pip install "celery>=5.5" "redis>=5.2" "amqp>=5.3" python-dotenv
```

`redis` is the Python client for both Redis and Valkey. `amqp` is only needed for RabbitMQ.

## Celery Setup

### `project/celery.py`

```python
import os

from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')

app = Celery('project')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

### `project/__init__.py`

```python
from .celery import app as celery_app

__all__ = ('celery_app',)
```

### `settings.py`

Every `CELERY_*` setting maps to the matching Celery option, e.g. `CELERY_BROKER_URL` becomes `broker_url`.

```python
import os

CELERY_BROKER_URL = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6380/0')
CELERY_RESULT_BACKEND = os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6380/1')
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 60
CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP = True

CACHE_URL = os.getenv('CACHE_URL')

if CACHE_URL:
    CACHES = {
        'default': {
            'BACKEND': 'django.core.cache.backends.redis.RedisCache',
            'LOCATION': CACHE_URL,
        }
    }
```

### A task

```python
# jobs/tasks.py
import time

from celery import shared_task


@shared_task
def add(x, y):
    return x + y


@shared_task(bind=True)
def build_report(self, rows=5):
    for done in range(1, rows + 1):
        time.sleep(1)
        self.update_state(state='PROGRESS', meta={'done': done, 'total': rows})
    return {'done': rows, 'total': rows}
```

### Calling it from a view

```python
from celery.result import AsyncResult
from django.http import JsonResponse

from jobs.tasks import add


def enqueue(request):
    result = add.delay(2, 3)
    return JsonResponse({'task_id': result.id})


def task_status(request, task_id):
    result = AsyncResult(task_id)
    return JsonResponse({'state': result.state, 'result': result.result if result.successful() else None})
```

## Using Valkey

[Valkey](https://valkey.io/) is the open-source (BSD) fork of Redis maintained by the Linux Foundation. It speaks the Redis protocol, so Celery, `redis-py` and Django's `RedisCache` work with it unchanged: use `redis://` URLs.

1. Start Valkey (port `6380`, so it can run next to Redis):

    ```bash
    docker run -d --name valkey -p 6380:6379 valkey/valkey:8-alpine
    ```

    Or install it locally with `brew install valkey` / `apt install valkey`, then run `valkey-server --port 6380`.

2. Configure `.env`:

    ```bash
    CELERY_BROKER_URL='redis://localhost:6380/0'
    CELERY_RESULT_BACKEND='redis://localhost:6380/1'
    CACHE_URL='redis://localhost:6380/2'
    ```

    The number at the end of each URL is a separate logical database, which keeps task messages, results and cache keys apart.

3. Start a worker and Django:

    ```bash
    celery -A project worker -l info
    python manage.py runserver
    ```

For a password or TLS, use `redis://:password@host:6379/0` or `rediss://:password@host:6380/0`.

For caching, sessions, ACL users, TLS, rate limiting, locks, configuration and migrating from Redis, see the full [Valkey with Django](valkey.md) guide.

!!! tip
    [`valkey-py`](https://github.com/valkey-io/valkey-py) and [`django-valkey`](https://github.com/django-commons/django-valkey) are Valkey-native clients, but they aren't required: the Redis clients work with Valkey.

## Using Redis

Same setup as Valkey, on the default port `6379`:

```bash
docker run -d --name redis -p 6379:6379 redis:8-alpine
```

```bash
CELERY_BROKER_URL='redis://localhost:6379/0'
CELERY_RESULT_BACKEND='redis://localhost:6379/1'
CACHE_URL='redis://localhost:6379/2'
```

## Using RabbitMQ

RabbitMQ is a dedicated message broker with stronger delivery guarantees (acknowledgements, durable quorum queues) and routing features. It doesn't store task results or act as a cache, so pair it with Valkey or Redis.

1. Start RabbitMQ (management UI at `http://localhost:15672`, guest / guest):

    ```bash
    docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:4-management-alpine
    ```

2. Configure `.env`:

    ```bash
    CELERY_BROKER_URL='amqp://guest:guest@localhost:5672//'
    CELERY_RESULT_BACKEND='redis://localhost:6380/1'
    CACHE_URL='redis://localhost:6380/2'
    ```

3. Add the RabbitMQ 4 settings to `settings.py`:

    ```python
    if CELERY_BROKER_URL.startswith(('amqp://', 'amqps://', 'pyamqp://')):
        CELERY_TASK_DEFAULT_QUEUE_TYPE = 'quorum'
        CELERY_BROKER_TRANSPORT_OPTIONS = {'confirm_publish': True}
        CELERY_WORKER_DETECT_QUORUM_QUEUES = True
        CELERY_WORKER_ENABLE_REMOTE_CONTROL = False
    ```

4. Start the worker without mingle and gossip:

    ```bash
    celery -A project worker -l info --without-mingle --without-gossip
    ```

!!! warning "RabbitMQ 4"
    RabbitMQ 4 rejects the transient, non-exclusive queues Celery declares by default (`Feature transient_nonexcl_queues is deprecated`). The settings above switch to quorum queues and disable remote control, and the worker flags stop it from declaring transient reply queues. For the same reason, don't use the `rpc://` result backend with RabbitMQ 4.

## Other Alternatives

| Option | Broker URL | Notes |
|---|---|---|
| KeyDB / Dragonfly | `redis://host:6379/0` | Redis-compatible, same setup as Valkey |
| Amazon SQS | `sqs://` | `pip install "celery[sqs]"`; needs a separate result backend |
| Google Pub/Sub | `gcpubsub://projects/<project-id>` | `pip install "celery[gcpubsub]"` |
| Django Tasks | n/a | Django 6.0+ ships `django.tasks` for simple background work without Celery; production backends come from third-party packages |

## Health Check

Check that Django can reach the broker and cache:

```python
from django.core.cache import cache
from django.http import JsonResponse

from project.celery import app


def health(request):
    checks = {}
    try:
        with app.connection_for_write() as conn:
            conn.ensure_connection(max_retries=1)
        checks['broker'] = 'ok'
    except Exception as exc:
        checks['broker'] = f'error: {exc}'

    try:
        cache.set('health_check', 'ok', timeout=5)
        checks['cache'] = 'ok' if cache.get('health_check') == 'ok' else 'error'
    except Exception as exc:
        checks['cache'] = f'error: {exc}'

    healthy = all(value == 'ok' for value in checks.values())
    return JsonResponse(checks, status=200 if healthy else 503)
```

## Testing

Run tasks in-process with `.apply()` so tests don't need a broker:

```python
from django.test import TestCase

from jobs.tasks import add


class TaskTests(TestCase):
    def test_add(self):
        self.assertEqual(add.apply(args=(2, 3)).get(), 5)
```
