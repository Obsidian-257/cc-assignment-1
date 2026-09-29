# Assignment 1 — Containerise and deploy

**Name:** K. Montarroso
**Campus:** Madrid

## What this is

A small Python/Flask HTTP API that returns a greeting, tracks a request counter persisted to disk, and exposes a health-check endpoint. I containerised it with a multi-stage Docker build, ran it locally with a named volume for persistence, and deployed it to Azure Container Instances with a public IP and a secure environment variable.

## Build and run it locally

Clone the repository and build the image:

```bash
git clone <repo-url> && cd cc-assignment-1
docker build -t cc-demo:1.0 .
```

Create a named volume and run the container:

```bash
docker volume create cc-data
docker run -d --name api -p 8000:8000 -e GREETING="Hello from Docker" -v cc-data:/data cc-demo:1.0
```

Test the endpoints:

```bash
curl http://localhost:8000/
curl http://localhost:8000/count
curl http://localhost:8000/healthz
```

Expected output from `/`:

```json
{"greeting":"Hello from Docker","hostname":"bfefd9057dad"}
```

Expected output from `/count` (increments each call):

```json
{"count":1,"stored_in":"/data/counter.json"}
```

Show the counter surviving a container restart:

```bash
curl http://localhost:8000/count          # returns count: 1
curl http://localhost:8000/count          # returns count: 2
docker rm -f api
docker run -d --name api2 -p 8000:8000 -v cc-data:/data cc-demo:1.0
curl http://localhost:8000/count          # returns count: 3 — data survived
```

Verify the container runs as a non-root user:

```bash
docker exec api2 id
# uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
```

## Configuration

| Variable | Default | What it does |
|:--|:--|:--|
| `GREETING` | `Hello from the container` | The greeting string returned by `GET /` |
| `DATA_DIR` | `/data` | Directory where the counter JSON file is stored |
| `PORT` | `8000` | The port the Flask server listens on |

## Part B — layers and cache

### B1 — image size

| Build | Base | Stages | Size |
|:--|:--|--:|--:|
| A — single-stage, full base | `python:3.12` | 1 | 1.62 GB |
| B — single-stage, slim base | `python:3.12-slim` | 1 | 198 MB |
| C — your `Dockerfile` | `python:3.12-slim` | 2 | 185 MB |

| Step | Saves | What left the image |
|:--|--:|:--|
| A → B | ~1.42 GB | The full Debian toolchain — compilers, development headers, and system libraries. Looking at `docker history` for build A, the largest layers are a 694 MB `apt-get install` of build-essential packages, a 202 MB layer of additional development tools, and a 65 MB system utilities layer. These exist in `python:3.12` so that packages with C extensions can compile from source. The slim image strips all of that out, keeping only what is needed to run Python. |
| B → C | ~13 MB | The pip-installed packages that lived in the final image. In build B, the `RUN pip install` layer is 15.3 MB because pip downloads, unpacks, and installs Flask and its dependencies directly in the final image. In build C, the multi-stage approach installs to a `--prefix=/install` directory in a throwaway builder stage, then copies only the installed files into the final image — that `COPY --from=builder` layer is 5.66 MB. The difference (~10 MB) is pip's metadata and intermediate files that never reach the final stage, plus the builder stage itself is discarded entirely. |

Evidence (from `docker image ls cc-demo` and `docker history`):

```
IMAGE         DISK USAGE   CONTENT SIZE
cc-demo:1.0   185MB         44.9MB
cc-demo:a     1.62GB        416MB
cc-demo:b     198MB         48.2MB
cc-demo:c     185MB         44.9MB
```

The A → B step did far more work. It removed over 1.4 GB of build tooling and system libraries that have nothing to do with running the application. The B → C step only saved about 13 MB because this application has a single, small, pure-Python dependency (Flask) — there is very little build waste to discard.

An application where B → C would save far more is one with heavy compiled dependencies — for example, a data science application that installs numpy, pandas, and scikit-learn. Those packages require a C compiler, Fortran libraries, and development headers at build time. In a single-stage build, all that build tooling stays in the final image. A multi-stage build compiles everything in the builder stage and copies only the compiled wheels into the slim final image, potentially saving hundreds of megabytes. The key difference is that compiled dependencies create a large gap between what is needed to build and what is needed to run.

### B2 — changing one line of source

I changed the default greeting in `app.py` from `"Hello from the container"` to `"Hello from the container!"` and rebuilt with `docker build --progress=plain -t cc-demo:1.1 .`.

```
#6 [builder 2/4] WORKDIR /app
#6 CACHED

#7 [builder 3/4] COPY requirements.txt .
#7 CACHED

#8 [builder 4/4] RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
#8 CACHED

#9 [stage-1 3/5] COPY --from=builder /install /usr/local
#9 CACHED

#10 [stage-1 4/5] COPY . .
#10 DONE 0.0s

#11 [stage-1 5/5] RUN useradd --create-home --uid 10001 appuser ...
#11 DONE 0.4s
```

Steps 6–9 were all CACHED because nothing they depend on changed. `requirements.txt` was unchanged, so the builder stage was fully reused. Step 10 (`COPY . .`) was rebuilt because `app.py` changed and Docker detected the different file checksum. Step 11 (the `useradd` command) also rebuilt even though it has nothing to do with `app.py` — this is because the Docker build cache is a prefix: once any layer is invalidated, every subsequent layer must rebuild too.

This is why the Dockerfile copies `requirements.txt` separately before `COPY . .` — it puts the rarely-changing dependency layer early, so that a code change only invalidates the final layers and pip does not re-run.

### B3 — making the cache worse

I created `Dockerfile.bad` which moves `COPY . .` above the `pip install` in the builder stage:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
RUN useradd --create-home --uid 10001 appuser && \
    mkdir -p /data && \
    chown -R appuser:appuser /data /app
USER appuser
VOLUME ["/data"]
EXPOSE 8000
CMD ["python", "app.py"]
```

After changing one line in `app.py` and rebuilding:

```
#5 [builder 2/4] WORKDIR /app
#5 CACHED

#7 [builder 3/4] COPY . .
#7 DONE 0.1s

#8 [builder 4/4] RUN pip install --no-cache-dir --prefix=/install -r requirements.txt
#8 1.980 Collecting flask==3.1.0 ...
#8 DONE 3.4s
```

Step 7 (`COPY . .`) was invalidated because `app.py` changed, and step 8 (`pip install`) was forced to rebuild too — downloading and installing all packages from scratch — even though `requirements.txt` was unchanged. The rule I broke is: order instructions from least-frequently changing to most-frequently changing. By placing `COPY . .` (which changes on every code edit) before `pip install` (which only needs to change when dependencies change), every code edit forces a full dependency reinstall.

## Part C — Azure

The `az` commands I ran, in order:

```bash
# Register providers
az provider register --namespace Microsoft.ContainerInstance --wait
az provider register --namespace Microsoft.ContainerRegistry --wait
az provider register --namespace Microsoft.Storage --wait

# Create resource group
az group create -n rg-cc-a1-kmontarroso -l swedencentral

# Create container registry
az acr create -g rg-cc-a1-kmontarroso -n crcckmontarroso -l swedencentral --sku Basic

# Log in to registry
az acr login -n crcckmontarroso

# Tag and push image
docker tag cc-demo:1.0 crcckmontarroso.azurecr.io/cc-demo:1.0
docker push crcckmontarroso.azurecr.io/cc-demo:1.0

# Enable admin credentials for ACR pull authentication
az acr update -n crcckmontarroso --admin-enabled true

# Deploy to ACI with secure environment variable
az container create --resource-group rg-cc-a1-kmontarroso --name aci-cc-a1 --location swedencentral --image crcckmontarroso.azurecr.io/cc-demo:1.0 --os-type Linux --cpu 1 --memory 1 --ports 8000 --ip-address Public --restart-policy OnFailure --registry-login-server crcckmontarroso.azurecr.io --registry-username crcckmontarroso --registry-password <retrieved at runtime, not stored> --environment-variables GREETING="Hello Azure" --secure-environment-variables API_KEY="super-secret-123"

# Clean up
az group delete --name rg-cc-a1-kmontarroso --yes --no-wait
```

I passed `API_KEY` as a secure environment variable using `--secure-environment-variables`. Querying the container afterward with `az container show` returns `"value": null` for `API_KEY`, while the regular `GREETING` variable shows its value normally. This proves the secure variable is not readable through the Azure control plane after deployment. The evidence is in `evidence/c4-secure-var.txt`.

I authenticated the ACR pull using admin credentials retrieved at runtime and passed directly to the `az container create` command. The credentials were stored only in PowerShell session variables and were never written to a file or committed to the repository.

The ACI filesystem is ephemeral — there is no `docker volume` equivalent in Azure. The counter will reset on every container restart. The named volume requirement in Part A applies to local runs only.

Evidence: see `evidence/` — `c1-acr.txt` shows the image tag in ACR, `c2-aci.txt` shows the running container with its public IP, `c3-request.txt` shows a successful HTTP request, and `c4-secure-var.txt` shows the secure variable reading null.

## Before this went to production

The first thing I would change is the web server. The application currently runs on Flask's built-in development server (Werkzeug), which is single-threaded and not designed for production traffic. I would put a production WSGI server like Gunicorn in front of it, which handles concurrent requests properly and fails more gracefully under load.

The second change is HTTPS. The container currently serves plain HTTP on a public IP with no encryption. In production I would place it behind a reverse proxy or load balancer (such as Azure Application Gateway) that terminates TLS, so that traffic between users and the application is encrypted.

The third issue is the counter storage. Writing to a local file means the state is lost whenever the container restarts, and it cannot be shared across multiple instances. In production I would replace the file-based counter with a small database or Azure Table Storage, so the data survives independently of the container lifecycle.

## AI use

I used Claude (Anthropic) as a conversational guide throughout the assignment. It explained Docker concepts, helped me understand the assignment requirements, walked me through each command step by step, generated the Dockerfile and the two comparison Dockerfiles, wrote the README analysis based on my actual build outputs, and helped troubleshoot PowerShell-specific issues (execution policy, curl alias). All evidence and build outputs are my own — generated on my machine from my actual builds and deployment.

## Sources

- Course lecture slides: Session 2 (Virtualization) and Session 3 (Docker Lab I)
- Docker documentation: [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- Docker documentation: [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
