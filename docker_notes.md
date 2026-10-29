# Docker Notes — MLOps Bootcamp

Personal reference for building, running, and shipping containers.

---

## 1. The Dockerfile — Instructions to Build an Image

```dockerfile
FROM python:3.8

# set a directory for the app
WORKDIR /usr/src/app

# copy all the files to the container
COPY . .

# install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# tell the port number the container should expose
EXPOSE 5000

# run the command
CMD ["python", "./app.py"]
```

| Line | What it does | Optional? |
|---|---|---|
| `FROM python:3.8` | Sets the **base image** — starts your container from an existing image that already has Python 3.8 installed, instead of building an OS from scratch. Every Dockerfile needs one. | No — every Dockerfile must start with a `FROM`. |
| `WORKDIR /usr/src/app` | Sets the **working directory** inside the container — like doing `cd /usr/src/app`. All following commands (`COPY`, `RUN`, `CMD`) run relative to this path. Creates the folder if it doesn't exist. | Technically optional (defaults to `/`), but strongly recommended — keeps your app isolated from system files. |
| `COPY . .` | Copies files from your **local project folder** (first `.`) into the **container's working directory** (second `.`, which is `/usr/src/app` here because of `WORKDIR`). | No — the container needs your code/files inside it to run. |
| `RUN pip install --no-cache-dir -r requirements.txt` | Executes a command **at build time** — here, installing Python dependencies listed in `requirements.txt`. `--no-cache-dir` skips caching pip's download cache, keeping the image smaller. | The flag `--no-cache-dir` is optional (just an image-size optimization) — the `pip install` itself is needed if you have dependencies. |
| `EXPOSE 5000` | **Documents** that the container listens on port 5000. This is metadata only — it does **not** actually publish the port to your host machine (that's done later with `-p` when you run the container). | Optional — purely informational/documentation, doesn't affect functionality. |
| `CMD ["python", "./app.py"]` | The **default command** that runs when the container starts (not at build time). Only one `CMD` is used per Dockerfile — if you list multiple, only the last one takes effect. | No — without a `CMD` (or `ENTRYPOINT`), the container has nothing to run and exits immediately. |

> **Key distinction:** `RUN` executes during the **build** (baking something into the image, like installing packages). `CMD` executes when the container **starts** (the actual app process).

---

## 2. Docker Basics — Install Check & First Containers

```bash
docker --version
```
- **What it does:** Prints the installed Docker version — a quick sanity check that Docker is installed and the CLI can talk to the Docker daemon.

```bash
docker run hello-world
```
- **What it does:** Downloads (if not already local) and runs a tiny test image that just prints a confirmation message. The standard "is Docker actually working?" test.

```bash
docker pull busybox
```
- **What it does:** Downloads the `busybox` image (a minimal Linux toolkit, popular for testing) from Docker Hub **without running it**. `pull` fetches only.

```bash
docker images
```
- **What it does:** Lists all images currently stored locally on your machine.

```bash
docker run busybox
```
- **What it does:** Creates and starts a container from the `busybox` image. Since `busybox` has no default long-running process, this container starts and **exits immediately** — that's expected.

```bash
docker run busybox echo "hello from busybox mlops"
```
- **What it does:** Same as above, but overrides the default command — runs `echo "hello from busybox mlops"` inside the container, prints it, then the container exits (the job's done).

```bash
docker ps
```
- **What it does:** Lists **currently running** containers only.

```bash
docker ps -a
```
- **What it does:** Lists **all** containers, including ones that have already stopped/exited. `-a` = "all."

```bash
docker run -it busybox sh
```
- **What it does:** Starts a busybox container and drops you into an **interactive shell** inside it. `-i` keeps input open (interactive), `-t` allocates a pseudo-terminal (so it feels like a normal terminal session). Type `exit` to leave.

```bash
docker rm <container_id>
```
- **What it does:** **Deletes** a stopped container (frees up disk space / clears clutter from `docker ps -a`). You need the container's ID or name (get it from `docker ps -a`). Won't work on a still-running container unless you add `-f` to force it.

---

## 3. Running a Real App Container (catgif example)

```bash
docker run -d -P --name catgif manifoldailearning/catgif
```
- **What it does:** Runs a container from the `manifoldailearning/catgif` image (pulled automatically from Docker Hub if not local).
  - `-d` — **detached mode**: runs in the background, gives you your terminal back immediately instead of streaming logs.
  - `-P` (capital) — publishes **all** the ports the image's Dockerfile marked with `EXPOSE`, mapping each to a **random** available port on your host.
  - `--name catgif` — gives the container a friendly name (`catgif`) instead of Docker's auto-generated random name, so it's easier to reference later.

```bash
docker ps
```
- Same as before — confirms `catgif` is running and shows its container ID/status.

```bash
docker port catgif
```
- **What it does:** Shows which **host port** Docker randomly assigned (because of `-P`) to the container's exposed port — tells you where to actually open it in a browser.

```bash
docker stop catgif
```
- **What it does:** Gracefully stops the running `catgif` container (sends a shutdown signal, container stays on disk — use `docker rm` after if you also want it deleted).

```bash
docker run -p 8888:5000 manifoldailearning/catgif
```
- **What it does:** Same image, but this time using **explicit port mapping** instead of random (`-P`). `-p 8888:5000` means: `<host_port>:<container_port>` — so `localhost:8888` on your machine forwards to port `5000` inside the container (matching the `EXPOSE 5000` from the Dockerfile). This is the more predictable, commonly-used option day-to-day.

---

## 4. Building Your Own Image

```bash
docker build -t catgifv2 .
```
- **What it does:** Builds a new image **from the Dockerfile in the current directory** (`.`). `-t catgifv2` **tags** (names) the resulting image `catgifv2` so you can reference it later instead of a random hash ID.

```bash
docker run -p 8888:5000 catgifv2
```
- **What it does:** Runs a container from your freshly built `catgifv2` image, same port-mapping pattern as before (host `8888` → container `5000`).

---

## 5. Pushing Your Image to Docker Hub

```bash
docker login
```
- **What it does:** Authenticates your terminal with Docker Hub (or another registry) using your account credentials — required before you can push images.

```bash
docker build -t manifoldailearning/catgif-devops .
```
- **What it does:** Builds the image again, but this time tags it with the **`<dockerhub_username>/<repo_name>`** format (`manifoldailearning/catgif-devops`). This naming convention is **required** for pushing to Docker Hub — the tag tells Docker Hub which account/repository the image belongs to.

```bash
docker push manifoldailearning/catgif-devops
```
- **What it does:** Uploads the tagged image to Docker Hub, making it publicly (or privately, depending on repo settings) available for others — or yourself, from another machine — to `docker pull`.

---

## Quick Reference — Command Cheat Sheet

| Command | Purpose |
|---|---|
| `docker --version` | Check Docker is installed |
| `docker pull <image>` | Download an image without running it |
| `docker images` | List local images |
| `docker run <image>` | Create + start a container |
| `docker run -it <image> sh` | Start a container with an interactive shell |
| `docker run -d ...` | Run detached (background) |
| `docker run -p host:container ...` | Explicit port mapping |
| `docker run -P ...` | Auto-map all exposed ports to random host ports |
| `docker run --name <name> ...` | Give the container a friendly name |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (incl. stopped) |
| `docker port <name>` | Show a container's host port mapping |
| `docker stop <name/id>` | Stop a running container |
| `docker rm <name/id>` | Delete a stopped container |
| `docker build -t <tag> .` | Build an image from the Dockerfile in the current folder |
| `docker login` | Authenticate with Docker Hub |
| `docker push <user>/<repo>` | Upload an image to Docker Hub |

### Image naming convention for pushing
```
<dockerhub_username>/<repository_name>[:tag]
```
If `:tag` is omitted, Docker defaults to `:latest`.
