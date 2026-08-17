# WordPress on Kubernetes

WordPress + MariaDB, moved from `docker-compose` to Kubernetes and packaged as a Helm
chart. Runs on minikube on an EC2 instance, with Prometheus and Grafana for monitoring.

## How it works

The original `docker-compose.yml` had two services and two volumes. Each became a
Kubernetes object:

| docker-compose | Kubernetes |
|---|---|
| `wordpress` service | Deployment (2 replicas) + Service + Ingress |
| `db` service | StatefulSet (1 replica) + Service named `db` |
| `wp_data` volume | PersistentVolumeClaim |
| `db_data` volume | `volumeClaimTemplates` in the StatefulSet |
| passwords | Secret |
| `depends_on` | nothing needed — WordPress finds the database by the Service name `db` |

A request travels like this:

```
browser
  -> NGINX Ingress Controller   (host: wordpress.local)
  -> Service wordpress          (ClusterIP :80)
  -> WordPress pods             (2 replicas)
  -> Service db                 (:3306)
  -> pod db-0
  -> PersistentVolumeClaim      (/var/lib/mysql)
```

Images live in a private Amazon ECR repository.

## Repository structure

```
wordpress-k8s/
├── README.md
├── secret.yaml.example          # template for the DB Secret (the real one is gitignored)
├── values-monitoring.yaml       # settings for Prometheus + Grafana
├── grafana-dashboard.json       # exported dashboard, 4 panels
└── wordpress-chart/
    ├── Chart.yaml
    ├── values.yaml              # everything configurable
    └── templates/
        ├── deployment.yaml      # WordPress
        ├── service.yaml
        ├── ingress.yaml
        ├── pvc.yaml
        ├── db-statefulset.yaml  # MariaDB
        └── db-service.yaml
```

## Prerequisites

- EC2 instance running minikube
- `kubectl`, `helm`, AWS CLI configured
- NGINX Ingress Controller installed
- Images in ECR:
  - `992382545251.dkr.ecr.us-east-1.amazonaws.com/rotem-repo:wordpress`
  - `992382545251.dkr.ecr.us-east-1.amazonaws.com/rotem-repo:mariadb`

## Install

**1. Namespace**

```bash
kubectl create namespace wordpress
```

**2. Pull secret** — the ECR repository is private, so the cluster needs credentials:

```bash
kubectl create secret docker-registry ecr-creds \
  --namespace wordpress \
  --docker-server=992382545251.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region us-east-1)"
```

The ECR token is valid for 12 hours. If a pod later fails with `ImagePullBackOff`, delete
this secret, run the command again, and delete the pods so they retry.

**3. Database Secret** — passwords are not in the chart, so copy the example and fill in
real values:

```bash
cp secret.yaml.example secret.yaml
kubectl apply -n wordpress -f secret.yaml
```

**4. Install the chart**

```bash
helm install wordpress ./wordpress-chart -n wordpress
```

## Settings

Everything configurable is in `wordpress-chart/values.yaml`:

| Value | Default | What it does |
|---|---|---|
| `replicaCount` | `2` | How many WordPress pods |
| `image.repository` | ECR URI | Where both images come from |
| `image.wordpressTag` | `wordpress` | WordPress image tag |
| `image.mariadbTag` | `mariadb` | MariaDB image tag |
| `image.pullSecret` | `ecr-creds` | Name of the registry secret |
| `ingress.host` | `wordpress.local` | Hostname the app answers on |
| `storage.wordpress` | `5Gi` | Disk for `/var/www/html` |
| `storage.database` | `5Gi` | Disk for `/var/lib/mysql` |
| `resources.*` | requests 100m CPU / 128–256Mi, limits 500m CPU / 512Mi | CPU and memory |

Change a value for one install only, without editing files:

```bash
helm install wordpress ./wordpress-chart -n staging --set ingress.host=staging.local
```

See the final YAML without touching the cluster:

```bash
helm template ./wordpress-chart
```

## Check it works

```bash
helm list -n wordpress
kubectl get pods,pvc,svc,ingress -n wordpress
curl -I -H "Host: wordpress.local" http://$(minikube ip)
```

A `302` redirect means everything is connected. If WordPress could not reach the database
you would get an error page instead, so the redirect is the proof.

## Open it in a browser

The minikube IP only works from the EC2 instance. From a laptop, add to `/etc/hosts`:

```
127.0.0.1 wordpress.local
127.0.0.1 grafana.local
```

Open a tunnel:

```bash
ssh -i <key.pem> -L 8080:192.168.49.2:80 ubuntu@<ec2-address>
```

- `http://wordpress.local:8080` — the site
- `http://grafana.local:8080` — Grafana

One tunnel serves both; the Ingress Controller decides where to send each request based on
the hostname.

## Monitoring

Installed separately from the application:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  -f values-monitoring.yaml
```

`values-monitoring.yaml` changes four things:

- Turns off three scrape targets that cannot work on minikube, so Prometheus stops
  reporting them as broken.
- Gives Grafana and Prometheus their own disks, so dashboards and metrics survive a pod
  restart.
- Sets CPU and memory limits.
- Publishes Grafana on `grafana.local`, so no `kubectl port-forward` is needed.

Grafana password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
```

### Dashboard

`grafana-dashboard.json` has four panels:

| Panel | Shows |
|---|---|
| Container Uptime | Green when a container is up, red when it is down |
| Container Restarts | How many times each container has crashed and restarted |
| Memory Usage vs Limit | Actual memory use against the configured ceiling |
| Whole Cluster Pods Health | Pods per state across the whole cluster, and the name of any pod that is not running |

The dashboard is committed here because a dashboard that lives only inside Grafana is lost
when its pod is replaced.

## Why it is built this way

**MariaDB is a StatefulSet, WordPress is a Deployment.**
WordPress pods are interchangeable — any of them can answer any request, and they store
nothing. A database is the opposite: it needs the same name and the same disk every time
it restarts. That is exactly what a StatefulSet gives.

**WordPress uses ClusterIP, not NodePort.**
The Ingress Controller is the single entry point, and it reaches the Service from inside
the cluster. Exposing the Service directly would create a second way in that nothing
manages.

**The Secret is not part of the chart.**
Putting it in the chart would move the passwords into `values.yaml` — the same problem in
a different file. So the Secret is created before installing, and only an example file
with placeholder values is committed.

**A Kubernetes Secret is not encrypted.**
It is base64-encoded, and anyone with read access can decode it with one command. What it
does give: passwords stay out of the workload files, and access can be restricted with
RBAC. Encrypting secrets in Git needs a tool like Sealed Secrets or AWS Secrets Manager.

**Both containers have health probes.**
A *readiness* probe answers "can this pod take traffic yet?" — until it passes, the
Service does not send it requests. A *liveness* probe answers "is this still alive?" — if
it fails, the container is restarted. Without them Kubernetes assumes any running process
is a working one, which is not true: before the database existed, the WordPress pods
reported `Running` while the site was broken.

WordPress is checked with an HTTP request to `/`. MariaDB is checked with a TCP connection
to port 3306, since there is no HTTP endpoint to ask and a real query would need
credentials inside the probe.

**One flat chart instead of sub-charts.**
An umbrella chart with separate `wordpress` and `mariadb` sub-charts is the usual pattern
when each component is independently useful — the way Bitnami ships MariaDB as a chart
that dozens of applications reuse. Here MariaDB exists only to serve this WordPress, so
splitting would add nested values and a dependency lock file without adding flexibility.
Six templates in one chart are easier to read.

## Known limitations

**Two replicas share one `ReadWriteOnce` disk.**
That access mode allows one node to mount the volume. minikube has one node, so both pods
land there and it works. On a real multi-node cluster the second pod could be scheduled
elsewhere and fail to start.

**Image tags are not versions.**
`wordpress` and `mariadb` say which image, not which version, so there is nothing to roll
back to. Real version tags would fix it.

**Only one database pod.**
Adding replicas would create a second empty database, not a copy. Real replication needs
configuration the image does not do by itself.

**The compose file carried a MySQL-only flag.**
`--default-authentication-plugin=mysql_native_password` was migrated as-is, but MariaDB
logs that it ignores it. Kept for fidelity with the original file.

**Resource requests are small on purpose.**
The first version requested 250m CPU per container. The database then would not schedule
at all — `Insufficient cpu`, with the node at 95% allocated, while barely doing any work.
The scheduler reserves whatever `requests` asks for, used or not, so oversized requests
fill a small node with nothing. Requests were lowered to 100m; the 500m limits, which cap
actual use, stayed.

## Possible next steps

- CI/CD: `helm lint` and `helm template` on every pull request, `helm upgrade` on merge
- HTTPS on the Ingress, using a TLS Secret
- Real version tags on the images, so rollbacks are possible
