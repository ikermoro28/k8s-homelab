# K8s Homelab

Clúster Kubernetes local construido desde cero para aprendizaje práctico de DevOps/SecOps. Desplegado en un portátil con k3d (k3s en Docker), con una stack completa de aplicación, observabilidad y GitOps.

## Stack

| Capa | Tecnología |
|---|---|
| Clúster | [k3d](https://k3d.io) — k3s en Docker (1 server + 2 agents) |
| App web | Nginx 1.27 + PHP-FPM 8.3 (patrón sidecar) |
| Base de datos | MariaDB 11.4 (StatefulSet con PVC) |
| Ingress | Traefik v2 via Helm |
| Monitorización | kube-prometheus-stack (Prometheus + Grafana + node-exporter) |
| GitOps | ArgoCD v3 — sincronización automática desde GitHub |
| Empaquetado | Helm + manifiestos YAML declarativos |

## Arquitectura

```
GitHub (fuente de verdad)
    └── ArgoCD (sync automático cada 3 min)
            └── namespace: homelab
                    ├── Deployment: nginx (Nginx + PHP-FPM sidecar, HPA min:2 max:6)
                    │       └── initContainer: copia index.php al volumen compartido
                    ├── StatefulSet: mariadb (PVC 1Gi)
                    ├── Service: nginx (ClusterIP :80)
                    ├── Service: mariadb (Headless — DNS estático por pod)
                    └── Ingress: homelab.local → nginx
            └── namespace: monitoring
                    ├── Prometheus (retención 7 días)
                    └── Grafana (dashboards K8s)
```

## URLs locales

| Servicio | URL |
|---|---|
| App web | http://homelab.local:8080 |
| Grafana | http://grafana.homelab.local:8080 |
| ArgoCD | http://argocd.homelab.local:8080 |

> Requiere añadir las entradas correspondientes en `/etc/hosts` apuntando al LoadBalancer de k3d.

## Estructura del repositorio

```
manifests/
├── namespace.yaml               # Namespaces: homelab + monitoring
├── nginx/
│   ├── configmap.yaml           # HTML con HOSTNAME_PLACEHOLDER
│   ├── configmap-php.yaml       # Script PHP + config FastCGI para nginx
│   ├── deployment.yaml          # Nginx + PHP-FPM sidecar + initContainer
│   ├── service.yaml             # ClusterIP :80
│   └── hpa.yaml                 # HPA: min 2, max 6, target 50% CPU
├── mariadb/
│   ├── secret.yaml              # Credenciales en base64
│   ├── pvc.yaml                 # PersistentVolumeClaim 1Gi RWO
│   ├── statefulset.yaml         # MariaDB 11.4 con probes y resources
│   └── service.yaml             # Headless service (clusterIP: None)
├── ingress/
│   ├── traefik-values.yaml      # Helm values para Traefik
│   ├── ingress.yaml             # Ingress para homelab.local y grafana
│   └── ingress-argocd.yaml      # Ingress para argocd.homelab.local
├── monitoring/
│   └── prometheus-values.yaml   # Helm values para kube-prometheus-stack
└── argocd-app.yaml              # Application CRD — define el GitOps loop
```

## Conceptos asumidos

- Pod, Deployment, ReplicaSet, StatefulSet, Service, ConfigMap, Secret, PVC, Ingress, HPA
- Patrón initContainer + sidecar (nginx y php-fpm en el mismo pod)
- DNS interno de Kubernetes: `<service>.<namespace>.svc.cluster.local`
- ClusterIP vs Headless Service
- Resources requests/limits y liveness/readiness probes
- Autoscaling horizontal basado en CPU
- GitOps: Git como única fuente de verdad, reconciliación automática con ArgoCD

## Requisitos

- Docker
- kubectl
- k3d
- Helm
