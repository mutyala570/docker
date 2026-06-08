# Docker & Kubernetes — Learning Notes

My step-by-step notes for Dockerizing the **meety-events** Angular app
(`qwipo/infinit-events`) and deploying it to Kubernetes, including the
mistakes I made and how I fixed them.

---

## 0. Prerequisites

- **Docker Desktop** installed and running (whale icon 🐳 in the menu bar).
  - On macOS the `docker` CLI needs the Docker Desktop engine running in the
    background. If the engine is off you get:
    `failed to connect to the docker API at unix:///var/run/docker.sock`
  - Fix: just open Docker Desktop (`open -a Docker`).
- `docker` and `kubectl` CLIs (both ship with Docker Desktop).

Check everything is ready:

```bash
docker info        # should print server version, no "daemon not running" error
```

---

## 1. The project

- App: **meety-events** (Angular 21), folder `qwipo/infinit-events`
- Build output: `dist/meety-events/browser`

---

## 2. Create the Dockerfile

File: `infinit-events/Dockerfile`

```dockerfile
# --- Stage 1: build the Angular app ---
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build -- --base-href=/      # see Mistake #2 below

# --- Stage 2: serve with nginx ---
FROM nginx:alpine
COPY --from=build /app/dist/meety-events/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

It's a **multi-stage build**: stage 1 builds the Angular app with Node,
stage 2 copies the built files into a tiny nginx image to serve them.

---

## 3. Create .dockerignore

File: `infinit-events/.dockerignore` — keeps the build fast and image small.

```
node_modules
dist
.angular
.git
.github
```

---

## 4. Build the image

```bash
cd /Users/mutyalaqwipo/qwipo/infinit-events
docker build -t meety-events .
```

> `-t meety-events` = name (tag) the image.
> The trailing `.` = build context (current folder). **It is required.**

---

## 5. Run the container

```bash
docker run -d -p 8080:80 --name meety meety-events
```

- `-d` = run in background (detached)
- `-p 8080:80` = map Mac port **8080** → container port **80** (nginx)
- `--name meety` = name the container

Open the app: <http://localhost:8080>

---

## 6. Check images & containers

```bash
docker images        # list built images (look for "meety-events")
docker ps            # running containers
docker ps -a         # all containers, incl. stopped
docker logs meety    # container output/logs
```

Or use the **Images** / **Containers** tabs in the Docker Desktop window.

---

## 7. Useful Docker commands

```bash
docker stop meety              # stop container
docker start meety             # start it again
docker rm -f meety             # force-remove container
docker rmi meety-events        # remove image

# rebuild after code changes, then re-run:
docker rm -f meety
docker build -t meety-events .
docker run -d -p 8080:80 --name meety meety-events
```

---

## Mistakes I made (and the fixes)

### Mistake #1 — forgot the `.` at the end of `docker build`

I ran:

```bash
docker build -t meety-events
```

Error:

```
ERROR: docker: 'docker buildx build' requires 1 argument
```

**Why:** the trailing `.` (the build context / path) was missing, so Docker
had no folder to build from.

**Fix:** add the dot — note the space before it:

```bash
docker build -t meety-events .
```

### Mistake #2 — blank page, 404s for JS/CSS files

After running the container, the browser showed errors like:

```
GET http://localhost:8080/infinity-events/main-XXXX.js  404 (Not Found)
```

**Why:** `angular.json` had `"baseHref": "/infinity-events/"`, so the built
`index.html` looked for all files under `/infinity-events/…`, but nginx
served them from the root `/`. Mismatch → 404 → blank page.

**Fix:** override the base href to `/` at build time in the Dockerfile:

```dockerfile
RUN npm run build -- --base-href=/
```

Then rebuild & re-run, and hard-refresh the browser (Cmd+Shift+R).

> Note: `--base-href=/` is for running locally in Docker. If the app is
> deployed to a real sub-path on a server, keep the original
> `/infinity-events/` base href.

---

## 8. Kubernetes

Enabled Kubernetes in **Docker Desktop → Settings → Kubernetes → Enable**.
Verify the cluster is up:

```bash
kubectl get nodes      # should show "docker-desktop  Ready"
```

### Deployment

File: `infinit-events/k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meety-events
  labels:
    app: meety-events
spec:
  replicas: 2                     # run 2 copies (pods)
  selector:
    matchLabels:
      app: meety-events
  template:
    metadata:
      labels:
        app: meety-events
    spec:
      containers:
        - name: meety-events
          image: meety-events:latest
          imagePullPolicy: IfNotPresent   # use the LOCAL image, don't pull from a registry
          ports:
            - containerPort: 80
```

> `imagePullPolicy: IfNotPresent` is important — the image only exists locally,
> so Kubernetes must NOT try to pull it from a registry.

### Service

File: `infinit-events/k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: meety-events
spec:
  type: LoadBalancer        # Docker Desktop exposes this on localhost
  selector:
    app: meety-events       # routes to pods with this label
  ports:
    - port: 8081            # the port I open in the browser
      targetPort: 80        # the port nginx listens on inside the pod
```

### Deploy to the cluster

```bash
cd /Users/mutyalaqwipo/qwipo/infinit-events
kubectl apply -f k8s/

# wait until it's ready
kubectl wait --for=condition=available --timeout=90s deployment/meety-events
```

Open the app: <http://localhost:8081>

### Useful kubectl commands

```bash
kubectl get pods                    # running pods
kubectl get all                     # deployment + pods + service together
kubectl get service meety-events    # the service + its port
kubectl logs -l app=meety-events    # logs from all pods
kubectl describe deployment meety-events

# scale to 4 copies
kubectl scale deployment meety-events --replicas=4

# after rebuilding the image, roll out the new version
kubectl rollout restart deployment meety-events

# tear everything down
kubectl delete -f k8s/
```

---

## Why two different ports (8080 vs 8081)?

- The app **always** listens on port **80** inside the container (nginx).
- The **external** port is just the "front door" I choose:
  - Plain Docker: `-p 8080:80`  → <http://localhost:8080>
  - Kubernetes:   Service `port: 8081` → <http://localhost:8081>
- They differ on purpose: **two programs can't share the same port on the Mac
  at once.** Docker already used 8080, so Kubernetes got 8081.

| Layer        | External port | Internal port | Chosen in            |
|--------------|---------------|---------------|----------------------|
| Plain Docker | 8080          | 80            | `docker run` command |
| Kubernetes   | 8081          | 80            | `service.yaml`       |

---

## Summary of the whole flow

```
Dockerfile  ──build──►  Image (meety-events)  ──run──►  Container (port 8080)
                                  │
                                  └──Kubernetes (Deployment + Service)──►  2 Pods (port 8081)
```
