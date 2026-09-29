---
icon: lucide/server
---

# Valkey with Django

[Valkey](https://valkey.io/) is an open-source, in-memory key-value store: a community fork of Redis maintained by the Linux Foundation under the BSD 3-Clause license. It speaks the Redis protocol, so Django's built-in Redis cache, `redis-py` and Celery all work with it unchanged.

This page covers using Valkey from Django as a **cache**, **session store**, **Celery broker and result backend**, and directly from Python for **rate limiting**, **locks** and **pub/sub**. For Celery and other brokers see [Message Brokers](brokers.md). The runnable demo project is [`django_brokers/`](https://github.com/yokwejuste/DjangoCrafts/tree/main/django_brokers).

## Why Valkey?

- **Open-source license.** In March 2024 Redis moved from BSD to source-available licenses (RSALv2/SSPLv1). Valkey was forked from Redis 7.2.4 to keep a BSD-licensed version, backed by AWS, Google Cloud, Oracle, Ericsson and others. Redis 8 later added AGPLv3 as an option; Valkey stays BSD.
- **Drop-in compatible.** Same protocol, same commands, same URL format. Valkey can load RDB/AOF data files from Redis 7.2 and earlier.
- **Active development.** Valkey 8 added improved multi-threaded I/O for higher throughput on multi-core machines, plus memory-efficiency improvements.
- **Managed offerings.** AWS ElastiCache and MemoryDB, Google Cloud Memorystore, Aiven, DigitalOcean and others offer managed Valkey, often cheaper than their Redis tiers.

| | Valkey | Redis |
|---|---|---|
| License | BSD 3-Clause | RSALv2 / SSPLv1 / AGPLv3 (Redis 8+) |
| Governance | Linux Foundation | Redis Ltd. |
| Protocol / commands | Redis-compatible | — |
| Django `RedisCache` | Works | Works |
| Celery broker | Works (`redis://` URL) | Works |

## Running Valkey

=== "Docker"

    ```bash
    docker run -d --name valkey -p 6379:6379 valkey/valkey:8-alpine
    ```

=== "Docker Compose"

    ```yaml
    services:
      valkey:
        image: valkey/valkey:8-alpine
        command: valkey-server --appendonly yes --requirepass ${VALKEY_PASSWORD}
        ports:
          - "6379:6379"
        volumes:
          - valkey-data:/data
        healthcheck:
          test: ["CMD", "valkey-cli", "-a", "${VALKEY_PASSWORD}", "--no-auth-warning", "ping"]
          interval: 10s
          timeout: 3s
          retries: 5

    volumes:
      valkey-data:
    ```

=== "macOS"

    ```bash
    brew install valkey
    brew services start valkey
    ```

=== "Ubuntu / Debian"

    ```bash
    sudo apt install valkey-server
    sudo systemctl enable --now valkey-server
    ```

Check it's running:

```bash
valkey-cli ping          # PONG
valkey-cli INFO server | grep valkey_version
```

## Python Clients

Pick one. Both work with Valkey.

| Package | Install | URL schemes | Use when |
|---|---|---|---|
| `redis` (redis-py) | `pip install redis` | `redis://`, `rediss://` | You use Django's built-in `RedisCache` or Celery (both need it) |
| `valkey` (valkey-py) | `pip install valkey` | `valkey://`, `valkeys://`, `redis://` | You want the Valkey-maintained client for direct access |

!!! warning "Celery needs `redis://`"
    Celery's transport layer (Kombu) has no `valkey://` transport. Using `CELERY_BROKER_URL='valkey://...'` fails with `KeyError: 'No such transport: valkey'`. Always use `redis://` (or `rediss://` for TLS) for Celery, even when the server is Valkey.

## Connection URLs

```text
redis://localhost:6379/0                     # no auth, database 0
redis://:password@localhost:6379/0           # password (requirepass)
redis://django:password@localhost:6379/0     # ACL user + password
rediss://:password@valkey.example.com:6380/0 # TLS
```

The number at the end is the logical database (0-15 by default). Use separate databases, or better, separate instances, for the broker, results and cache so they don't interfere.

## Django Cache

### Built-in backend (recommended)

Django 4.0+ ships `RedisCache`, which works with Valkey:

```bash
pip install redis
```

```python
# settings.py
import os

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': os.getenv('CACHE_URL', 'redis://localhost:6379/2'),
        'KEY_PREFIX': 'crafts',
        'TIMEOUT': 300,
    }
}
```

With replicas, list the primary first. Writes go to the first server, reads are spread across the rest:

```python
'LOCATION': [
    'redis://valkey-primary:6379/2',
    'redis://valkey-replica-1:6379/2',
    'redis://valkey-replica-2:6379/2',
],
```

Use it like any Django cache:

```python
from django.core.cache import cache
from django.views.decorators.cache import cache_page

cache.set('greeting', 'hello', timeout=60)
cache.get('greeting')
cache.incr('page_views')

@cache_page(60 * 15)
def article_list(request):
    ...
```

### django-valkey backend

[`django-valkey`](https://github.com/django-commons/django-valkey) is a Valkey-native cache backend from Django Commons. It adds features the built-in backend lacks, such as `ttl()`, `lock()`, `delete_pattern()`, compression and pluggable serializers.

```bash
pip install django-valkey
```

```python
CACHES = {
    'default': {
        'BACKEND': 'django_valkey.cache.ValkeyCache',
        'LOCATION': 'valkey://localhost:6379/2',
    }
}
```

```python
from django.core.cache import cache

cache.set('report:42', data, timeout=600)
cache.ttl('report:42')             # seconds left
cache.delete_pattern('report:*')   # delete matching keys

with cache.lock('nightly-report', timeout=300):
    build_report()
```

## Sessions in Valkey

Store sessions in the cache for fast lookups:

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

Pure cache sessions are lost if Valkey restarts without persistence, or evicts keys under memory pressure. To keep sessions safe, use `cached_db`, which reads from Valkey and writes through to the database:

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cached_db'
```

## Celery Broker and Results

```python
# settings.py
CELERY_BROKER_URL = os.getenv('CELERY_BROKER_URL', 'redis://localhost:6379/0')
CELERY_RESULT_BACKEND = os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6379/1')

# Tasks not acknowledged within this time are redelivered to another worker.
# Set it longer than your longest task (default: 1 hour).
CELERY_BROKER_TRANSPORT_OPTIONS = {'visibility_timeout': 3600}
CELERY_RESULT_EXPIRES = 60 * 60 * 24
```

Full setup, tasks and the worker command are on the [Message Brokers](brokers.md) page.

## Using Valkey Directly

For features beyond caching, talk to Valkey with `valkey-py` (or `redis-py`).

```python
# core/valkey.py
import os

import valkey

client = valkey.from_url(os.getenv('VALKEY_URL', 'valkey://localhost:6379/0'), decode_responses=True)
```

### Rate limiting

A fixed-window limiter: at most `limit` requests per `window` seconds per key.

```python
from django.http import HttpResponse

from core.valkey import client


def allow(key, limit=10, window=60):
    pipe = client.pipeline()
    pipe.incr(key)
    pipe.expire(key, window, nx=True)
    count, _ = pipe.execute()
    return count <= limit


def login_view(request):
    ip = request.META.get('REMOTE_ADDR')
    if not allow(f'ratelimit:login:{ip}', limit=5, window=60):
        return HttpResponse('Too many attempts, try again in a minute.', status=429)
    ...
```

### Distributed lock

Stop two workers or servers from running the same job at once:

```python
from core.valkey import client

with client.lock('lock:nightly-report', timeout=300, blocking_timeout=5):
    build_report()
```

### Pub/Sub

```python
from core.valkey import client

client.publish('notifications', 'order:42:shipped')

pubsub = client.pubsub()
pubsub.subscribe('notifications')
for message in pubsub.listen():
    if message['type'] == 'message':
        print(message['data'])
```

!!! tip "Django Channels"
    `channels_redis` works with Valkey as a channel layer. Point its `hosts` at your Valkey URL.

## Configuration

Key `valkey.conf` settings (or pass them as `valkey-server --option value`):

```conf
# Memory
maxmemory 512mb
maxmemory-policy allkeys-lru     # for a cache instance

# Persistence
appendonly yes                   # AOF: safer, needed for broker/sessions
save 3600 1 300 100 60 10000     # RDB snapshots

# Security
requirepass change-me
bind 0.0.0.0
protected-mode yes
```

!!! warning "Separate cache and broker instances"
    A cache should evict old keys (`allkeys-lru`). A Celery broker must never evict (`noeviction`), or queued tasks silently disappear. Since `maxmemory-policy` is per server, run **two Valkey instances** in production: one for the cache and one for Celery.

### ACL users

Give Django its own user instead of the shared password, and block dangerous commands like `FLUSHALL`:

```bash
valkey-cli ACL SETUSER django on '>strong-password' '~*' '&*' '+@all' '-@dangerous'
```

```python
CACHE_URL = 'redis://django:strong-password@localhost:6379/2'
```

### TLS

Managed Valkey services usually require TLS. Use `rediss://` (or `valkeys://` with valkey-py):

```python
CELERY_BROKER_URL = 'rediss://:password@valkey.example.com:6380/0?ssl_cert_reqs=required'
CELERY_RESULT_BACKEND = 'rediss://:password@valkey.example.com:6380/1?ssl_cert_reqs=required'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'rediss://:password@valkey.example.com:6380/2',
    }
}
```

## Migrating from Redis

1. **Code:** nothing changes. Keep `redis://` URLs and the `redis` package.
2. **Data:** Valkey loads RDB and AOF files from Redis 7.2 and earlier. Stop Redis, copy `dump.rdb` (or the AOF directory) into Valkey's data directory, and start Valkey. Files from Redis 7.4+ use a newer format that Valkey can't read; for those, replicate with `REPLICAOF` or re-populate the data.
3. **Commands:** `redis-cli` works against Valkey, and `valkey-cli` against Redis.
4. **Managed services:** AWS ElastiCache supports in-place upgrades from Redis OSS to Valkey.

For a cache, the simplest migration is to point Django at an empty Valkey and let it warm up.

## Monitoring and Troubleshooting

```bash
valkey-cli INFO memory            # used_memory_human, maxmemory, evicted_keys
valkey-cli INFO stats             # keyspace_hits / keyspace_misses (cache hit rate)
valkey-cli --latency              # live latency
valkey-cli --bigkeys              # largest keys
valkey-cli LLEN celery            # tasks waiting in Celery's default queue
valkey-cli MONITOR                # every command, dev only (slows the server)
```

| Symptom | Likely cause | Fix |
|---|---|---|
| `KeyError: 'No such transport: valkey'` | `valkey://` URL in Celery | Use `redis://` for Celery |
| `NOAUTH Authentication required` | Password missing from URL | `redis://:password@host:6379/0` |
| `NOPERM ... has no permissions` | ACL user lacks the command | Adjust `ACL SETUSER` rules |
| Tasks run twice | Task longer than `visibility_timeout` | Raise `visibility_timeout` |
| Tasks disappear | Broker instance evicting keys | `maxmemory-policy noeviction` on the broker |
| Users logged out after restart | Cache sessions without persistence | Use `cached_db` sessions or enable AOF |

## Testing

Tests shouldn't need a running Valkey. Swap in the in-memory cache:

```python
from django.test import TestCase, override_settings

LOCMEM = {'default': {'BACKEND': 'django.core.cache.backends.locmem.LocMemCache'}}


@override_settings(CACHES=LOCMEM)
class CacheTests(TestCase):
    def test_page_views(self):
        ...
```

## Additional Resources

- [Valkey documentation](https://valkey.io/docs/)
- [Valkey Docker image](https://hub.docker.com/r/valkey/valkey)
- [valkey-py](https://github.com/valkey-io/valkey-py)
- [django-valkey](https://github.com/django-commons/django-valkey)
- [Django cache framework](https://docs.djangoproject.com/en/stable/topics/cache/)
- [Celery with Redis/Valkey](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/redis.html)
