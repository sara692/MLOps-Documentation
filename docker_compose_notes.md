# Docker Compose — Summary Notes

Reference for defining and running **multi-container** applications with a single YAML file and a single command.

---

## 1. Dockerfile vs. Docker Compose — The Core Difference

These solve two different problems, and beginners often mix them up:

| | **Dockerfile** | **Docker Compose** |
|---|---|---|
| **What it defines** | How to **build one image** — the steps to assemble a container's filesystem (base OS, dependencies, code, startup command) | How to **run one or more containers together** — which images/ports/volumes/env vars each service needs, and how services connect to each other |
| **Scope** | A single container | A whole application — multiple containers working together (e.g. an API + a database + a cache) |
| **Command** | `docker build -t myimage .` | `docker compose up` |
| **Output** | A reusable image | Running containers, networked and configured, based on images (which may themselves come from Dockerfiles) |
| **Analogy** | The **recipe** for one dish | The **meal plan** — which dishes to make, in what order, and how they're served together |

**In practice, they work together, not instead of each other:** a Dockerfile builds your app's image; `docker-compose.yml` then says "run a container from that image, plus a Postgres container, plus a Redis container, connect them, and start it all with one command."

```yaml
services:
  api:
    build: .              # ← builds from the Dockerfile in this directory
    ports: ["8000:8000"]
  db:
    image: postgres:16    # ← uses a pre-built image, no Dockerfile needed
```
- Note above: `api` uses your own `Dockerfile` (via `build: .`), while `db` just pulls a ready-made image from Docker Hub (`image: postgres:16`) — Compose can mix both approaches freely within the same file.

**Rule of thumb:** if you're packaging *your own code* into a container, you need a Dockerfile. If you're running *multiple containers together* (your app + a database + supporting services), you need Docker Compose — and it will usually reference your Dockerfile for the services that are your own code.

---

## 2. A Basic `docker-compose.yml`

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://user:pass@db/mydb
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

---

## 3. Key Directives

```yaml
services:
```
- Top-level key — every container your app needs is defined as a named service under here (`api`, `db`, `redis`, etc.). The service name also becomes its **hostname** on the internal network (see §5).

```yaml
build: .
```
- Builds an image from the Dockerfile in the given directory (`.` = current directory), instead of pulling a pre-made one.
```yaml
build:
  context: .
  dockerfile: Dockerfile.dev
```
- Longer form — lets you point at a differently-named Dockerfile, useful when you keep separate Dockerfiles for dev/prod.

```yaml
image: postgres:16
```
- Uses an existing image (from Docker Hub or another registry) directly — no build step. Common for off-the-shelf services like databases, caches, message queues.

```yaml
ports:
  - "8000:8000"
```
- Maps `<host_port>:<container_port>` — same concept as `docker run -p` (see the Docker notes file). Optional — omit if the service doesn't need to be reachable from outside the Compose network.

```yaml
environment:
  DATABASE_URL: postgresql://user:pass@db/mydb
```
- Sets environment variables inside the container — used here to pass config/secrets without hardcoding them into the image.
```yaml
env_file:
  - .env
```
- Alternative: load environment variables from a separate `.env` file instead of listing them inline — keeps secrets out of the YAML file itself (and out of version control, if `.env` is gitignored).

```yaml
volumes:
  - db_data:/var/lib/postgresql/data
```
- Mounts a **named volume** (`db_data`) to a path inside the container, so data persists even if the container is removed/recreated. Must also be declared at the bottom-level `volumes:` key (see §2).
```yaml
volumes:
  - ./app:/usr/src/app
```
- A **bind mount** instead — maps a local folder (`./app`) directly into the container. Common in development so code changes on your machine reflect inside the container immediately, without rebuilding.

```yaml
depends_on:
  - db
```
- Controls **startup order** — makes sure `db` starts before `api`. Important caveat: this only waits for the container to *start*, not for the database to actually be *ready to accept connections* — for that, add a healthcheck (below) or retry logic in your app.

```yaml
healthcheck:
  test: ["CMD", "pg_isready", "-U", "user"]
  interval: 5s
  timeout: 3s
  retries: 5
```
- Defines how Compose checks whether a service is actually healthy/ready, not just running. Combine with `depends_on: { db: { condition: service_healthy } }` to make one service truly wait for another to be ready.

```yaml
restart: unless-stopped
```
- Restart policy if the container crashes or the host reboots. Options: `no` (default), `always`, `on-failure`, `unless-stopped`.

```yaml
networks:
  - backend
```
- Explicitly attaches a service to a named network (declared at the bottom-level `networks:` key) — useful when you want to isolate groups of services from each other. By default, Compose already puts all services in one shared network automatically, so this is optional unless you need multiple isolated networks.

---

## 4. Common Commands

```bash
docker compose up
```
- Builds (if needed) and starts every service defined in `docker-compose.yml`. Streams logs from all containers to your terminal.

```bash
docker compose up -d
```
- Same as above, but **detached** — runs in the background and returns your terminal.

```bash
docker compose up --build
```
- Forces a rebuild of any services with a `build:` directive before starting — needed after changing your Dockerfile or code, since Compose otherwise reuses a cached image.

```bash
docker compose down
```
- Stops and removes all containers (and the default network) created by `up`. Named volumes are kept by default — data isn't lost.

```bash
docker compose down -v
```
- Same as above, but also removes named volumes — use when you actually want to wipe persisted data (e.g. reset a database to a clean state).

```bash
docker compose ps
```
- Lists the status of all services defined in this Compose file (running, stopped, etc.) — the Compose-aware equivalent of `docker ps`.

```bash
docker compose logs
```
- Shows logs from all services. Add a service name to see just one: `docker compose logs api`. Add `-f` to follow logs live (like `tail -f`).

```bash
docker compose exec api sh
```
- Opens an interactive shell inside the running `api` service's container — same idea as `docker exec` (or `docker run -it ... sh`), but targeting a Compose-managed service by name instead of a container ID.

```bash
docker compose build
```
- Builds (or rebuilds) images for services with a `build:` directive, without starting any containers.

```bash
docker compose stop
```
- Stops running containers without removing them (unlike `down`) — can be restarted quickly with `docker compose start`.

---

## 5. How Services Talk to Each Other

Compose automatically creates a shared network for all services in the file, and each service is reachable by **its service name as a hostname** — no manual networking config needed.

```python
# inside the 'api' service's code, connecting to the 'db' service:
DATABASE_URL = "postgresql://user:pass@db/mydb"
#                                      ^^ service name used as hostname, not "localhost"
```
- This is why the earlier example set `DATABASE_URL` pointing at `db`, not `localhost` or `127.0.0.1` — from inside a container, `localhost` refers to *that container itself*, not other Compose services. Referencing another service by its name is how containers find each other.

---

## Quick Reference — Cheat Sheet

| Command | Purpose |
|---|---|
| `docker compose up` | Build (if needed) + start all services, logs in foreground |
| `docker compose up -d` | Same, but detached (background) |
| `docker compose up --build` | Force rebuild before starting |
| `docker compose down` | Stop + remove containers (volumes kept) |
| `docker compose down -v` | Stop + remove containers **and** named volumes |
| `docker compose ps` | List service statuses |
| `docker compose logs -f` | Follow logs from all services |
| `docker compose exec <service> sh` | Shell into a running service |
| `docker compose build` | Build images without starting containers |
| `docker compose stop` / `start` | Stop/restart without removing containers |

| YAML key | Purpose |
|---|---|
| `services:` | Defines each container to run |
| `build:` | Build from a local Dockerfile |
| `image:` | Use an existing pre-built image |
| `ports:` | Map host↔container ports |
| `environment:` / `env_file:` | Set config/secrets inside the container |
| `volumes:` | Persist data or mount local code |
| `depends_on:` | Control startup order |
| `healthcheck:` | Define what "ready" actually means for a service |
| `restart:` | Auto-restart policy on crash/reboot |
| `networks:` | Attach to a specific (non-default) network |
