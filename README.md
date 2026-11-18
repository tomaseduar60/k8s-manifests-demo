# k8s-manifests-demo

[![ci](https://github.com/tomaseduar60/k8s-manifests-demo/actions/workflows/ci.yml/badge.svg)](https://github.com/tomaseduar60/k8s-manifests-demo/actions/workflows/ci.yml)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?logo=argo&logoColor=white)
![Kustomize](https://img.shields.io/badge/Kustomize-326CE5?logo=kubernetes&logoColor=white)
![KEDA](https://img.shields.io/badge/KEDA-0063D1?logoColor=white)
![Crossplane](https://img.shields.io/badge/Crossplane-EF7B4D?logo=crossplane&logoColor=white)
![Kafka](https://img.shields.io/badge/Kafka-231F20?logo=apachekafka&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)

Un microservicio completo desplegado con GitOps: una API y un consumidor de Kafka, con secretos desde GCP Secret Manager, topics pedidos como recurso de plataforma, autoscaling por CPU y por lag, y tres ambientes con Kustomize gestionados por Argo CD.

Es la misma estructura que se repite por cada namespace en una plataforma con muchos equipos: copiar la carpeta de la app, cambiar nombres, listo.

## Estructura

```
.
├── apps/orders-api/
│   ├── base/                  # manifiestos comunes a todos los ambientes
│   └── overlays/
│       ├── develop/           # 1-3 réplicas, topics con prefijo develop.
│       ├── qa/
│       └── production/        # 3-10 réplicas, sync manual
├── platform/                  # lo que instala el equipo de plataforma una vez
│   ├── external-secrets/      # ClusterSecretStore hacia Secret Manager
│   └── crossplane/            # API KafkaTopic (XRD + Composition)
└── argocd/                    # AppProject + ApplicationSet
```

## Qué hay en cada archivo

### Secretos: `ExternalSecret` + `secretKeyRef`

[`externalsecret.yaml`](apps/orders-api/base/externalsecret.yaml) arma **un solo Secret** con todas las keys de la app, leyendo de Secret Manager con la convención `<app>-<key>`. El [`deployment.yaml`](apps/orders-api/base/deployment.yaml) consume cada key con su propio `secretKeyRef`.

Se usa `secretKeyRef` explícito en vez de `envFrom` a propósito: si falta una key, el pod queda en `CreateContainerConfigError` con el nombre exacto de la key faltante, en lugar de arrancar y fallar en runtime. Con muchos namespaces, ese mensaje ahorra mucho tiempo.

El [`ClusterSecretStore`](platform/external-secrets/cluster-secret-store.yaml) se autentica con Workload Identity y solo atiende a namespaces con el label `secrets.example.io/enabled=true`.

### Topics de Kafka: Crossplane

Los topics se piden en el namespace del equipo con un claim [`KafkaTopic`](apps/orders-api/base/kafka-topics.yaml). La [Composition](platform/crossplane/composition.yaml) los crea en Confluent Cloud con las credenciales de plataforma; los equipos nunca tocan la API key del cluster. En develop y qa el overlay agrega el prefijo del ambiente al topic, y lo mismo en el ConfigMap y en KEDA, para que los tres apunten siempre al mismo nombre.

### Autoscaling: HPA para la API, KEDA para el worker

| Workload | Escala con | Por qué |
|---|---|---|
| `orders-api` | [HPA](apps/orders-api/base/hpa.yaml) por CPU y memoria | La carga de una API se ve en CPU. Scale up inmediato, scale down de a un pod cada minuto para no oscilar |
| `orders-worker` | [KEDA](apps/orders-api/base/keda.yaml) por lag del consumer group | La CPU de un consumidor no dice cuánto trabajo pendiente hay; el lag sí. `maxReplicaCount` no pasa la cantidad de particiones, porque más consumidores que particiones quedan ociosos |

### Exposición: `Ingress`

[`ingress.yaml`](apps/orders-api/base/ingress.yaml) usa ingress-nginx con TLS de cert-manager y rate limit por IP. Cada overlay cambia el host.

### Seguridad del pod

Todos los contenedores corren como no-root, con filesystem de solo lectura, sin capabilities y con `seccompProfile: RuntimeDefault`. La [ServiceAccount](apps/orders-api/base/serviceaccount.yaml) está ligada a una GSA con Workload Identity, y un [PDB](apps/orders-api/base/pdb.yaml) mantiene al menos un pod durante los upgrades de nodos.

### GitOps: Argo CD

El [`ApplicationSet`](argocd/applicationset.yaml) genera una Application por ambiente. develop y qa sincronizan solos con `prune` y `selfHeal`; producción queda con sync manual. El [`AppProject`](argocd/project.yaml) limita el equipo a sus namespaces `orders-*`.

## Probarlo

```bash
kubectl kustomize apps/orders-api/overlays/production
```

En un cluster con Argo CD, ESO, KEDA y Crossplane instalados:

```bash
kubectl apply -f platform/external-secrets/ -f platform/crossplane/
kubectl apply -f argocd/
```

## CI

En cada PR se renderizan los tres overlays y se validan con kubeconform (incluyendo los CRDs de ESO, KEDA y Argo CD), además de yamllint sobre todo el repo.

## Licencia

[MIT](LICENSE)
