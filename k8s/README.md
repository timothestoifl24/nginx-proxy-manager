# Running Nginx Proxy Manager on Kubernetes / k3s

These [Kustomize](https://kustomize.io/) manifests run the regular
`jc21/nginx-proxy-manager` image on Kubernetes. They need nothing beyond
`kubectl` (which has Kustomize built in).

```
k8s/
├── base/                 # SQLite, LoadBalancer service (k3s, MetalLB, cloud)
└── overlays/
    ├── kind/             # NodePort + kind cluster config, for local testing
    └── postgres/         # base + a PostgreSQL StatefulSet
```

## What gets deployed

| Resource | Purpose |
| --- | --- |
| `Deployment/nginx-proxy-manager` | One replica, `Recreate` strategy. The startup probe waits for `GET /api/`, readiness and liveness check nginx on port 80 |
| `PVC/nginx-proxy-manager-data` | `/data`: database, generated nginx configs, custom certs, logs, JWT keys |
| `PVC/nginx-proxy-manager-letsencrypt` | `/etc/letsencrypt`: Let's Encrypt accounts and certificates |
| `Service/nginx-proxy-manager` | `LoadBalancer` on 80/443 with `externalTrafficPolicy: Local`, which preserves client IPs |
| `Service/nginx-proxy-manager-admin` | `ClusterIP` on 81 (admin UI), not exposed publicly |

Things to know:

- **Single replica only.** NPM writes nginx config to its volume and reloads
  its own nginx process, so it cannot be scaled out. The PVCs are
  `ReadWriteOnce`.
- **The container runs as root.** It uses s6-overlay as init and fails to
  start otherwise. Set the `PUID`/`PGID` env vars to have nginx and the
  backend drop privileges after start-up. This means the pod does not pass
  the `restricted` Pod Security Standard. Use `baseline` for the namespace.
- **Upstream hostnames must be fully qualified.** nginx's resolver ignores
  the search domains in `/etc/resolv.conf`. To proxy to a service inside
  the cluster, forward to `<service>.<namespace>.svc.cluster.local` (or its
  ClusterIP), not `<service>`.
- **Streams:** add every TCP/UDP port you configure as a Stream in the UI to
  `Service/nginx-proxy-manager` as well (there is a commented example in
  `base/service.yaml`).
- **IPv6** is disabled via `DISABLE_IPV6=true` because most clusters are
  IPv4 single-stack. Set it to `"false"` on dual-stack clusters.

## Optional: initial admin user

Without this, the first visit to the admin UI shows the setup wizard. To create
the first admin user non-interactively, create this Secret **before** the
first start:

```bash
kubectl create namespace nginx-proxy-manager
kubectl -n nginx-proxy-manager create secret generic nginx-proxy-manager-admin \
  --from-literal=email=you@example.com \
  --from-literal=password='a-strong-password'
```

It is only used while the database has no users. You can delete it afterwards.

## k3s (e.g. on a Raspberry Pi)

Requirements:

- A **64-bit** OS (e.g. Raspberry Pi OS 64-bit or Ubuntu arm64). The image is
  published for `amd64` and `arm64` only.
- The memory cgroup must be enabled, as k3s requires on Raspberry Pi OS:
  append `cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt`
  and reboot.
- **Traefik must not hold ports 80/443.** k3s installs Traefik by default and its
  ServiceLB can only give one service a given port per node. Either install
  k3s without it:

  ```bash
  curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -
  ```

  or, on an existing install, add this to `/etc/rancher/k3s/config.yaml` and
  restart k3s:

  ```yaml
  disable:
    - traefik
  ```

Deploy:

```bash
kubectl apply -k k8s/base
kubectl -n nginx-proxy-manager rollout status deploy/nginx-proxy-manager --timeout=5m
kubectl -n nginx-proxy-manager get svc nginx-proxy-manager   # EXTERNAL-IP = node IP
```

The first start can take a few minutes on a Pi. Ports 80/443 are then served on
the node's IP, so forward them from your router to that IP.

Open the admin UI through a port-forward:

```bash
kubectl -n nginx-proxy-manager port-forward svc/nginx-proxy-manager-admin 8181:81
# then browse to http://localhost:8181
```

## kind (e.g. with Podman Desktop)

kind has no LoadBalancer, so the `kind` overlay uses fixed NodePorts. The
`cluster.yaml` there maps those ports to the host (to ports above 1024, so it
also works with rootless Podman):

| Host | NodePort | Service |
| --- | --- | --- |
| `localhost:8080` | 30080 | HTTP |
| `localhost:8443` | 30443 | HTTPS |

The admin UI stays cluster-internal, as in the base.

Port mappings can only be set when the cluster is created, so create the cluster from
that config with the kind CLI rather than the Podman Desktop wizard:

```bash
KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster --name npm --config k8s/overlays/kind/cluster.yaml
kubectl apply -k k8s/overlays/kind
kubectl -n nginx-proxy-manager rollout status deploy/nginx-proxy-manager --timeout=5m
```

Then open the admin UI with
`kubectl -n nginx-proxy-manager port-forward svc/nginx-proxy-manager-admin 8181:81`
and browse to http://localhost:8181. The cluster shows up in Podman Desktop's
Kubernetes view like any other kind cluster. On an existing kind cluster
without the port mappings, port-forward `svc/nginx-proxy-manager` as well.

Let's Encrypt HTTP validation will not work in a local kind cluster, because it is
not reachable from the internet on port 80. Use DNS challenges or custom
certificates for testing.

## PostgreSQL instead of SQLite

```bash
cp k8s/overlays/postgres/postgres.env.example k8s/overlays/postgres/postgres.env
# edit postgres.env and set POSTGRES_PASSWORD
kubectl apply -k k8s/overlays/postgres
```

`postgres.env` is git-ignored. The overlay adds a `postgres:17` StatefulSet and
a NetworkPolicy that only lets the NPM pod reach it. It wires the credentials
into NPM through the `DB_POSTGRES_*` variables.

To combine it with the kind overlay, create your own overlay that lists the
postgres overlay as a resource and uses a copy of `kind/service-patch.yaml` as a
patch. Kustomize does not allow patch files from outside the overlay's folder.

## Customising

Create an overlay of your own instead of editing `base/`, for example:

```yaml
# my-overlay/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../k8s/base          # adjust the path
images:
  - name: jc21/nginx-proxy-manager
    newTag: "2.15.1"     # pin / upgrade the image here
patches:
  - target:
      kind: PersistentVolumeClaim
      name: nginx-proxy-manager-data
    patch: |-
      - op: add
        path: /spec/storageClassName
        value: longhorn
```

All environment variables documented for the Docker image, such as `TZ`, `PUID`,
`PGID`, `DB_MYSQL_*` and `X_FRAME_OPTIONS`, work unchanged. If you set
`NPM_ADMIN_PORT`, also patch the `admin` containerPort to match. The startup
probe and the admin Service use it. The `VAR__FILE`
convention works with mounted Secret files too, but `valueFrom.secretKeyRef` is
usually simpler.

## Upgrading

Change `newTag` in your overlay (or `base/kustomization.yaml`) and re-apply.
The `Recreate` strategy stops the old pod before starting the new one, so there
is a short downtime while it restarts.

## Uninstalling

> **Warning**
> This deletes the namespace and its PVCs, including the database, all proxy host
> configuration and all certificates. Back up `/data` and `/etc/letsencrypt` first
> if you might need them again.

```bash
kubectl delete -k k8s/base   # or the overlay you applied
```
