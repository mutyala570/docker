# Docker & Kubernetes — Generic Notes

Step-by-step reference for Dockerizing **any** app and deploying it to
Kubernetes. The **commands and concepts below are the same for every stack**
(Node.js, Bun, .NET, Python, Go, Angular/React static sites, …). The **only
part that changes per language is the `Dockerfile`** — see
[§9 Dockerfile per stack](#9-dockerfile-per-stack).

Throughout, replace `myapp` with your own image/app name.

---

## 0. Prerequisites

- **Docker Desktop** installed and running (whale icon 🐳 in the menu bar).
  - On macOS the `docker` CLI needs the Docker Desktop engine running. If it's
    off you get: `failed to connect to the docker API at unix:///var/run/docker.sock`
  - Fix: open Docker Desktop (`open -a Docker`).
- `docker` and `kubectl` CLIs (both ship with Docker Desktop).

```bash
docker info        # should print server version, no "daemon not running" error
```

---

## 1. The mental model (same for all projects)

| Term | What it is |
|------|------------|
| **Dockerfile** | A recipe describing how to package your app. |
| **Image** | The built package (app + runtime + dependencies). Read-only. |
| **Container** | A running instance of an image. |
| **Registry** | Where images are stored/shared (Docker Hub, GHCR, ACR…). |
| **Pod** (K8s) | One or more containers running together; smallest K8s unit. |
| **Deployment** (K8s) | Keeps N copies (pods) of your app running; auto-restarts/scales. |
| **Service** (K8s) | A stable address that load-balances across pods. |

Flow:

```
Dockerfile ──build──► Image ──run──► Container
                        │
                        └── Kubernetes (Deployment + Service) ──► Pods
```

---

## 2. Create a Dockerfile

Put a file named `Dockerfile` (no extension) in your project root. The contents
depend on your stack — jump to [§9](#9-dockerfile-per-stack). Two universal
ideas:

- **Multi-stage build**: a "build" stage compiles/builds, a smaller "runtime"
  stage holds only what's needed to run → smaller, safer images.
- **`EXPOSE <port>`**: documents the port your app listens on *inside* the
  container.

---

## 3. Create .dockerignore

A `.dockerignore` keeps junk out of the build (faster builds, smaller images).
Generic starting point — adjust per stack:

```
.git
.github
node_modules        # Node/Bun (deps get installed inside the image)
dist                # build output (rebuilt inside the image)
bin
obj                 # .NET build output
__pycache__         # Python
*.log
.env
```

---

## 4. Build the image

```bash
cd /path/to/your/project
docker build -t myapp .
```

- `-t myapp` = name (tag) the image.
- The trailing **`.`** = build context (current folder). **It is required.**
- Add a version tag too if you like: `-t myapp:1.0`.

---

## 5. Run the container

```bash
docker run -d -p 8080:80 --name myapp myapp
```

- `-d` = detached (background)
- `-p 8080:80` = map host port **8080** → container port **80**
  - ⚠️ The right-hand number must match the port **your app listens on inside
    the container** (nginx=80, many Node apps=3000, .NET=8080, etc.). Set the
    left-hand number to any free port on your machine.
- `--name myapp` = name the container

Open: <http://localhost:8080>

---

## 6. Inspect images & containers

```bash
docker images        # built images
docker ps            # running containers
docker ps -a         # all containers (incl. stopped)
docker logs myapp    # container output/logs
docker exec -it myapp sh   # open a shell inside the running container
```

Or use the **Images** / **Containers** tabs in Docker Desktop.

---

## 7. Common Docker commands

```bash
docker stop myapp              # stop
docker start myapp             # start again
docker rm -f myapp             # force-remove container
docker rmi myapp               # remove image

# rebuild after code changes, then re-run:
docker rm -f myapp
docker build -t myapp .
docker run -d -p 8080:80 --name myapp myapp
```

---

## 8. Kubernetes (stack-agnostic)

Enable it: **Docker Desktop → Settings → Kubernetes → Enable**.

```bash
kubectl get nodes      # should show "docker-desktop  Ready"
```

### k8s/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          imagePullPolicy: IfNotPresent   # use LOCAL image, don't pull from a registry
          ports:
            - containerPort: 80            # the port your app listens on inside the container
```

> `imagePullPolicy: IfNotPresent` matters when the image is only built locally
> (no registry). If you push to a registry, you can drop this.

### k8s/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  type: LoadBalancer        # Docker Desktop exposes this on localhost
  selector:
    app: myapp
  ports:
    - port: 8081            # the port you open in the browser
      targetPort: 80        # must match containerPort above
```

### Deploy

```bash
cd /path/to/your/project
kubectl apply -f k8s/
kubectl wait --for=condition=available --timeout=90s deployment/myapp
```

Open: <http://localhost:8081>

### Common kubectl commands

```bash
kubectl get pods                 # running pods
kubectl get all                  # deployment + pods + service
kubectl logs -l app=myapp        # logs from all pods
kubectl describe deployment myapp
kubectl scale deployment myapp --replicas=4    # scale up/down
kubectl rollout restart deployment myapp       # pick up a rebuilt image
kubectl delete -f k8s/                          # tear everything down
```

---

## 9. Dockerfile per stack

Each app listens on a port *inside* the container — note it and match it in
`-p host:container`, `EXPOSE`, and the K8s `targetPort`/`containerPort`.

### Node.js (Express/Nest/etc.) — typically listens on 3000

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # skip if there's no build step

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist   # adjust to your output dir
EXPOSE 3000
CMD ["node", "dist/main.js"]         # adjust to your entry file
```
Run: `docker run -d -p 3000:3000 --name myapp myapp`

### Bun — typically listens on 3000

```dockerfile
FROM oven/bun:1 AS build
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build            # skip if no build step

FROM oven/bun:1-slim
WORKDIR /app
COPY --from=build /app .
EXPOSE 3000
CMD ["bun", "run", "start"]  # or: ["bun", "index.ts"]
```
Run: `docker run -d -p 3000:3000 --name myapp myapp`

### .NET (ASP.NET Core) — listens on 8080 in modern images

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApp.dll"]   # replace MyApp.dll
```
Run: `docker run -d -p 8080:8080 --name myapp myapp`

### Python (FastAPI/Flask) — example listens on 8000

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
Run: `docker run -d -p 8000:8000 --name myapp myapp`

### Static frontend (Angular / React / Vue) — served by nginx on 80

```dockerfile
# build stage uses Node, runtime stage uses nginx
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # Angular: add -- --base-href=/ (see Mistake #2)

FROM nginx:alpine
# adjust the source path to your framework's output dir:
#   Angular: dist/<project>/browser   React(CRA): build   Vite: dist
COPY --from=build /app/dist/<project>/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
Run: `docker run -d -p 8080:80 --name myapp myapp`

---

## Mistakes I made (and the fixes)

### Mistake #1 — forgot the `.` at the end of `docker build`

```bash
docker build -t myapp        # ❌ ERROR: 'docker buildx build' requires 1 argument
docker build -t myapp .      # ✅ the trailing dot = build context (current folder)
```

### Mistake #2 — blank page, 404s for JS/CSS (static-frontend specific)

Symptom: `GET http://localhost:8080/some-base/main-XXXX.js  404 (Not Found)`

Cause: the app's **base href** pointed to a sub-path (e.g. Angular
`"baseHref": "/infinity-events/"` in `angular.json`) but nginx served files
from the root `/`.

Fix: build with the base href set to `/` for local serving —
`RUN npm run build -- --base-href=/` (Angular). Keep the real sub-path base
href if you deploy under that path on a server.

> General lesson: **the path/port the app expects must match how it's served.**
> Check base paths, ports, and output directories when a containerized app
> 404s or won't connect.

---

## First worked example: meety-events (Angular)

The original hands-on run used **meety-events** (Angular 21) at
`/Users/mutyalaqwipo/qwipo/infinit-events` — that repo holds the actual
`Dockerfile`, `.dockerignore`, and `k8s/` manifests built while learning. It's
the static-frontend case above (nginx on port 80, browser on 8080; Kubernetes
on 8081).
