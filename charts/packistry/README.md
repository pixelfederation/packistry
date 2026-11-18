# Packistry Helm chart

Helm chart for [Packistry](https://packistryphp.com/), a self-hosted Composer repository.
It runs the official image `ghcr.io/packistry/packistry`.

## Install

```sh
helm repo add packistry https://pixelfederation.github.io/packistry
helm repo update
helm install packistry packistry/packistry -n packistry --create-namespace -f my-values.yaml
```

## Minimal configuration

Packistry needs an app key and a database. Keep secrets in a Secret and load it with `envFrom`:

```sh
kubectl -n packistry create secret generic packistry-env \
  --from-literal=APP_KEY="base64:$(openssl rand -base64 32)" \
  --from-literal=DB_PASSWORD=change-me
```

```yaml
env:
  APP_URL: https://packistry.example.com
  DB_CONNECTION: mysql
  DB_HOST: mysql.example.internal
  DB_DATABASE: packistry
  DB_USERNAME: packistry

envFrom:
  - secretRef:
      name: packistry-env

ingress:
  internal:
    enabled: true
    className: nginx
    hosts:
      - host: packistry.example.com
    tls:
      - secretName: packistry-tls
        hosts:
          - packistry.example.com
```

The image runs database migrations on every start (`php artisan migrate --force`).

## Key values

| Value | Description |
|---|---|
| `env` | Env variables as a map (`NAME: value`). A value can also be a `valueFrom` object. Set a key to `null` to drop it. A map merges across values files, a list would not. |
| `envFrom` | List of Secret / ConfigMap refs. All their keys become env variables. |
| `ingress.<name>` | Ingresses keyed by name (e.g. `internal`, `external`). Each enabled entry renders `<fullname>-<name>`. `paths` defaults to `/`. |
| `httpRoute` | Gateway API alternative to Ingress. Needs `parentRefs`. Routes `/` to Packistry by default. |
| `supervisordConfig` | Overrides the image's supervisord config so the web server and queue worker log to stdout/stderr (the image logs to files). Set to `""` to keep the image's config. |
| `podDisruptionBudget` | `enabled`, plus `minAvailable` or `maxUnavailable` (default `maxUnavailable: 1`). |
| `hostUsers` | Set to `false` to run the pod in a user namespace (Kubernetes 1.33+, containerd 2.x, kernel 6.3+). |
| `automountServiceAccountToken` | Pod-level override. Packistry does not use the Kubernetes API. |
| `livenessProbe` / `readinessProbe` | Default to `GET /up`. Liveness waits 60s so startup migrations can finish. |

## Useful env variables

| Variable | Notes |
|---|---|
| `TRUSTED_PROXIES` | Needed behind an ingress / reverse proxy. `*` (the default when empty) or comma-separated IPs/CIDRs. Symfony keywords `REMOTE_ADDR` and `PRIVATE_SUBNETS` also work. |
| `FILESYSTEM_DISK`, `AWS_BUCKET`, `AWS_DEFAULT_REGION` | Store archives in S3 (`FILESYSTEM_DISK: s3`). Always set the region. Leave `AWS_ENDPOINT` empty for AWS S3. Without access keys, the AWS SDK uses the default credential chain (e.g. IRSA / Pod Identity). |
| `LOG_CHANNEL`, `LOG_STDERR_FORMATTER` | `stderr` and `Monolog\Formatter\JsonFormatter` for JSON logs on stdout/stderr. |
| `RR_NUM_WORKERS` | RoadRunner PHP workers per pod. Defaults to the node's CPU count, so pin it when you set memory limits. |
| `WORKER_NUMPROCS` | Queue worker processes per pod (default `2`). |

## Security context

The image runs as user `packistry` (UID `100`, GID `101`). `runAsNonRoot` needs a numeric UID:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 100
  runAsGroup: 101
  seccompProfile:
    type: RuntimeDefault
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
```

`readOnlyRootFilesystem` is not supported, because the image writes to its filesystem on startup.

## Scaling

All shared state (cache, sessions, queue, locks) lives in the database by default, and archives live on the configured disk. Multiple replicas work as long as storage is shared (e.g. S3), not the local disk.
