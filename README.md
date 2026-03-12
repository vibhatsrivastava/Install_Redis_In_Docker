# Install Redis in Docker

Run a password-protected, persistent Redis instance on Ubuntu using Docker Compose.

---

## Prerequisites

- Ubuntu server (20.04 / 22.04 / 24.04)
- [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) installed
- Docker Compose v2 (`docker compose` — bundled with Docker Engine ≥ 23)

---

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/vibhatsrivastava/Install_Redis_In_Docker.git
cd Install_Redis_In_Docker
```

### 2. Create your local environment file

```bash
cp .env.example .env
```

> **Important:** `.env` is gitignored and will never be committed. It holds your Redis password.

### 3. Set a strong password

```bash
nano .env
```

Replace `change_me_strong_password` with a strong, unique password:

```
REDIS_PASSWORD=MyStr0ng!P@ssw0rd
```

### 4. Start the Redis container

```bash
docker compose up -d
```

### 5. Verify Redis is running

```bash
# Check container status
docker compose ps

# Ping Redis using the password from the container environment
docker exec redis sh -c 'redis-cli -a "$REDIS_PASSWORD" ping'
# Expected output: PONG
```

---

## Configuration

| File | Purpose | Committed? |
|------|---------|------------|
| `docker-compose.yml` | Container definition | Yes |
| `redis.conf` | Redis settings (no secrets) | Yes |
| `.env.example` | Password template | Yes |
| `.env` | Your actual password | **No** (gitignored) |

### Key settings (`redis.conf`)

| Setting | Value | Notes |
|---------|-------|-------|
| `maxmemory` | `256mb` | Adjust to your server RAM |
| `maxmemory-policy` | `allkeys-lru` | Evict least-recently-used keys when full |
| `appendonly` | `yes` | AOF persistence enabled |
| RDB snapshots | 900s/1 key, 300s/10 keys, 60s/10000 keys | Configurable in `redis.conf` |

---

## Data Persistence

Redis data is stored in a named Docker volume (`redis_data`) so it survives container restarts and re-creates.

```bash
# List volumes
docker volume ls

# Inspect the volume
docker volume inspect install_redis_in_docker_redis_data
```

---

## Useful Commands

```bash
# Start
docker compose up -d

# Stop (data is preserved)
docker compose stop

# Stop and remove container (data volume is preserved)
docker compose down

# Stop and remove container AND volume (destructive — deletes all data)
docker compose down -v

# View logs
docker compose logs -f redis

# Open Redis CLI
docker exec -it redis redis-cli -a "$REDIS_PASSWORD"
```

---

## Connecting from a Python Application

This Redis instance is designed to be used as a backend for **agentic AI applications** — for caching LLM responses, storing agent memory/state, managing task queues, and pub/sub messaging between agents.

### Install the Python client

```bash
pip install redis
```

For async applications (e.g. FastAPI, asyncio-based agents):

```bash
pip install redis[asyncio]
```

### Store the connection details securely

Add Redis connection settings to your Python app's own `.env` file (never hardcode credentials):

```
REDIS_HOST=<ubuntu-server-ip>
REDIS_PORT=6379
REDIS_PASSWORD=MyStr0ng!P@ssw0rd
REDIS_DB=0
```

### Basic connection (synchronous)

```python
import os
import redis

client = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=int(os.environ.get("REDIS_PORT", 6379)),
    password=os.environ["REDIS_PASSWORD"],
    db=int(os.environ.get("REDIS_DB", 0)),
    decode_responses=True,   # return strings instead of bytes
)

client.ping()  # raises ConnectionError if unreachable
```

### Async connection (for asyncio / FastAPI agents)

```python
import os
import redis.asyncio as aioredis

client = aioredis.Redis(
    host=os.environ["REDIS_HOST"],
    port=int(os.environ.get("REDIS_PORT", 6379)),
    password=os.environ["REDIS_PASSWORD"],
    db=int(os.environ.get("REDIS_DB", 0)),
    decode_responses=True,
)

await client.ping()
```

### Connection pool (recommended for long-running agents)

Reuse connections across requests instead of opening a new one each time:

```python
import os
import redis

pool = redis.ConnectionPool(
    host=os.environ["REDIS_HOST"],
    port=int(os.environ.get("REDIS_PORT", 6379)),
    password=os.environ["REDIS_PASSWORD"],
    db=int(os.environ.get("REDIS_DB", 0)),
    decode_responses=True,
    max_connections=20,
)

client = redis.Redis(connection_pool=pool)
```

---

### Common Use Cases for Agentic AI Applications

#### 1. Caching LLM responses

Avoid redundant API calls by caching prompt → response pairs:

```python
import hashlib, json

def get_llm_response(prompt: str, call_llm_fn) -> str:
    key = "llm:cache:" + hashlib.sha256(prompt.encode()).hexdigest()
    cached = client.get(key)
    if cached:
        return cached
    response = call_llm_fn(prompt)
    client.setex(key, 3600, response)  # cache for 1 hour
    return response
```

#### 2. Agent memory / session state

Store and retrieve agent working memory across turns:

```python
import json

SESSION_TTL = 86400  # 24 hours

def save_agent_state(session_id: str, state: dict) -> None:
    client.setex(f"agent:state:{session_id}", SESSION_TTL, json.dumps(state))

def load_agent_state(session_id: str) -> dict | None:
    raw = client.get(f"agent:state:{session_id}")
    return json.loads(raw) if raw else None
```

#### 3. Task queue (agent → worker)

Push tasks from an orchestrator agent and pop them in a worker agent:

```python
QUEUE = "agent:tasks"

# Orchestrator: enqueue a task
client.rpush(QUEUE, json.dumps({"task": "summarise", "doc_id": "abc123"}))

# Worker: blocking dequeue (waits up to 30 s for a task)
_, raw = client.blpop(QUEUE, timeout=30)
task = json.loads(raw) if raw else None
```

#### 4. Pub/Sub between agents

Broadcast events from one agent and subscribe in another:

```python
# Publisher agent
client.publish("agent:events", json.dumps({"event": "task_complete", "id": "abc123"}))

# Subscriber agent
pubsub = client.pubsub()
pubsub.subscribe("agent:events")

for message in pubsub.listen():
    if message["type"] == "message":
        event = json.loads(message["data"])
        print(event)
```

#### 5. Distributed lock (prevent duplicate agent runs)

```python
import uuid

def acquire_lock(resource: str, ttl_seconds: int = 30) -> str | None:
    lock_id = str(uuid.uuid4())
    acquired = client.set(f"lock:{resource}", lock_id, nx=True, ex=ttl_seconds)
    return lock_id if acquired else None

def release_lock(resource: str, lock_id: str) -> None:
    if client.get(f"lock:{resource}") == lock_id:
        client.delete(f"lock:{resource}")
```

---

### Recommended Key Naming Convention

Use colon-separated namespaces to keep keys organised:

```
llm:cache:<hash>          — cached LLM responses
agent:state:<session_id>  — per-session agent memory
agent:tasks               — task queue list
agent:events              — pub/sub channel
lock:<resource>           — distributed locks
```

---

## Security Notes

- The Redis password is **never** stored in `redis.conf` or `docker-compose.yml`. It is injected at runtime via the `--requirepass` flag using the `REDIS_PASSWORD` environment variable.
- The `.env` file containing the password is gitignored and must not be committed.
- Redis port `6379` is exposed to the host. If this server is internet-facing, restrict access with a firewall rule (e.g. `ufw allow from <trusted-ip> to any port 6379`).
- `protected-mode yes` is set in `redis.conf` as an additional safeguard.
