# Docker & Kubernetes — Learning Checklist

A roadmap of what to cover for each core concept. For each one:
**Understand → Practice (hands-on) → Self-check**.
Tick the boxes as you go. Commands live in `docker-kubernetes-notes.md`.

Order to learn: **Image → Container → Registry → Pod → Deployment → Service.**
(First three are Docker; last three are Kubernetes.)

---

## 1. Image

> The built package: your app + its runtime + dependencies. Read-only blueprint.

**Understand**
- [ ] What an image is and how it differs from a container
- [ ] What a `Dockerfile` is and the common instructions: `FROM`, `WORKDIR`,
      `COPY`, `RUN`, `EXPOSE`, `CMD`/`ENTRYPOINT`
- [ ] Image **layers** & build cache (why order of `COPY`/`RUN` matters)
- [ ] **Multi-stage builds** (build stage vs smaller runtime stage)
- [ ] **Tags** (`myapp:latest`, `myapp:1.0`) and what `latest` really means
- [ ] Base images: `alpine` vs `slim` vs full (size/security trade-offs)

**Practice**
- [ ] Write a Dockerfile and `docker build -t myapp .`
- [ ] List images: `docker images`
- [ ] Tag a second version: `docker tag myapp myapp:1.0`
- [ ] Inspect: `docker history myapp` and `docker inspect myapp`
- [ ] Remove an image: `docker rmi myapp:1.0`

**Self-check**
- [ ] Can I explain why changing one line of code only rebuilds *some* layers?
- [ ] Why is a multi-stage image smaller than a single-stage one?

---

## 2. Container

> A running instance of an image. You can run many containers from one image.

**Understand**
- [ ] Image (static) vs container (running) — the difference
- [ ] Port mapping `-p host:container` and which side is which
- [ ] Detached (`-d`) vs foreground; viewing `logs`
- [ ] Container **lifecycle**: create → start → stop → remove
- [ ] **Environment variables** (`-e KEY=value`) for config
- [ ] **Volumes / bind mounts** (`-v`) for persisting data
- [ ] Containers are **ephemeral** — data is lost on removal unless on a volume

**Practice**
- [ ] Run: `docker run -d -p 8080:80 --name myapp myapp`
- [ ] See it: `docker ps` (and `docker ps -a` for stopped ones)
- [ ] Logs: `docker logs myapp`
- [ ] Shell inside it: `docker exec -it myapp sh`
- [ ] Stop/start/remove: `docker stop`, `docker start`, `docker rm -f`
- [ ] Pass an env var and a volume, and verify they work

**Self-check**
- [ ] If I delete a container and run a new one from the same image, what
      happens to data written inside the old one?
- [ ] Why can't two containers map to the same host port at once?

---

## 3. Registry

> Where images are stored and shared (Docker Hub, GitHub GHCR, Azure ACR…).

**Understand**
- [ ] What a registry/repository is; public vs private
- [ ] Image naming for registries: `registry/user/name:tag`
      (e.g. `ghcr.io/me/myapp:1.0`)
- [ ] `login` → `push` → `pull` flow
- [ ] Why Kubernetes usually pulls from a registry (vs local-only images)
- [ ] `imagePullPolicy` (`Always` / `IfNotPresent` / `Never`) and when each fits

**Practice**
- [ ] `docker login` (Docker Hub or `ghcr.io`)
- [ ] Tag for the registry: `docker tag myapp ghcr.io/<user>/myapp:1.0`
- [ ] Push: `docker push ghcr.io/<user>/myapp:1.0`
- [ ] Pull on a "fresh" spot: `docker pull ghcr.io/<user>/myapp:1.0`
- [ ] (Optional) make a private repo and pull it using a secret

**Self-check**
- [ ] Why does deploying to a real (multi-node) cluster usually require a
      registry instead of a locally built image?
- [ ] When is `imagePullPolicy: IfNotPresent` the right choice?

---

## 4. Pod

> The smallest Kubernetes unit: one (or a few tightly-coupled) containers
> sharing network/storage. You rarely create Pods directly — a Deployment does.

**Understand**
- [ ] What a Pod is and why it's not the same as a container
- [ ] Pod has its own IP; containers in a Pod share it
- [ ] Pods are **disposable** — they get recreated, names/IPs change
- [ ] `containerPort`, basic resource `requests`/`limits`
- [ ] **Liveness/readiness probes** (health checks) — why they matter
- [ ] (Concept) sidecar pattern: a helper container in the same Pod

**Practice**
- [ ] `kubectl get pods` and `kubectl get pods -o wide` (see node/IP)
- [ ] `kubectl describe pod <name>` — read events at the bottom
- [ ] `kubectl logs <pod>` and `kubectl exec -it <pod> -- sh`
- [ ] Delete a pod managed by a Deployment and watch it get recreated
- [ ] Add a readiness probe to your Deployment and observe `READY` column

**Self-check**
- [ ] If a Pod is deleted, why does my app stay up?
- [ ] Why shouldn't I rely on a Pod's IP address staying the same?

---

## 5. Deployment

> Manages a set of identical Pods: keeps N replicas running, restarts crashed
> ones, and rolls out new versions.

**Understand**
- [ ] `replicas` — desired number of Pods
- [ ] `selector` + `labels` — how a Deployment finds "its" Pods
- [ ] Self-healing: crashed/deleted Pods are recreated
- [ ] **Scaling** up/down
- [ ] **Rolling updates** and **rollback** (`rollout` commands)
- [ ] Difference between Deployment / ReplicaSet / Pod

**Practice**
- [ ] `kubectl apply -f k8s/deployment.yaml`
- [ ] `kubectl get deployment` and `kubectl get all`
- [ ] Scale: `kubectl scale deployment myapp --replicas=4` (watch pods appear)
- [ ] Roll out a new image: `kubectl rollout restart deployment myapp`
- [ ] Check status / history: `kubectl rollout status` / `kubectl rollout history`
- [ ] Roll back: `kubectl rollout undo deployment myapp`

**Self-check**
- [ ] How does a Deployment know which Pods belong to it?
- [ ] What happens during a rolling update so users see no downtime?

---

## 6. Service

> A stable network address that load-balances traffic across the (changing)
> Pods of a Deployment.

**Understand**
- [ ] Why a Service is needed (Pod IPs change; Service IP/DNS is stable)
- [ ] `selector` links a Service to Pods by label
- [ ] `port` (the Service) vs `targetPort` (the container)
- [ ] Service **types**: `ClusterIP` (internal), `NodePort`, `LoadBalancer`
- [ ] In-cluster DNS: reach a service by its name (`http://myapp`)
- [ ] (Concept) **Ingress** — HTTP routing/hostnames in front of Services

**Practice**
- [ ] `kubectl apply -f k8s/service.yaml`
- [ ] `kubectl get service` — note CLUSTER-IP / EXTERNAL-IP / PORT(S)
- [ ] Open the app via the `LoadBalancer` port on `localhost`
- [ ] `kubectl describe service myapp` — see its **Endpoints** (the Pod IPs)
- [ ] Scale the Deployment and confirm the Service load-balances to all pods
- [ ] Try a `ClusterIP` service + reach it from another pod by name

**Self-check**
- [ ] If Pods are constantly replaced, how does traffic keep reaching them?
- [ ] When would I use `ClusterIP` vs `NodePort` vs `LoadBalancer`?

---

## How it all fits together

```
Dockerfile ──build──► IMAGE ──push──► REGISTRY
                        │                 │
                        │            (K8s pulls)
                        ▼                 ▼
                    CONTAINER        DEPLOYMENT ──manages──► PODS (run containers)
                    (plain Docker)        ▲                    ▲
                                          │                    │
                                          └──────  SERVICE ────┘
                                            (stable address, load-balances)
```

- **Docker world:** Dockerfile → Image → (Registry) → Container
- **Kubernetes world:** Deployment → Pods, fronted by a Service

---

## Next topics (after the six above)

- [ ] **docker-compose** — run multi-container apps locally (app + DB)
- [ ] **ConfigMaps & Secrets** — config and credentials in Kubernetes
- [ ] **Volumes / PersistentVolumeClaims** — persistent storage in K8s
- [ ] **Namespaces** — isolating environments
- [ ] **Ingress** — HTTP routing and hostnames
- [ ] **Helm** — packaging/templating Kubernetes manifests
- [ ] **Health probes & resource limits** — production readiness
- [ ] **CI/CD** — build & push images automatically (GitHub Actions)
