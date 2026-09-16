# k8s-apps

Repo de manifiestos de aplicaciones (GitOps), compartido entre `aws-eks-cluster` (EKS) y `azure-aks-cluster` (AKS) — separado a propósito de la infraestructura. Cada cluster corre su propio Argo CD, que vigila este repo y sincroniza ese cluster — no hay `kubectl apply` manual ni pipeline de CD acá.

## Decisión clave: Argo CD se instala en cada cluster, no acá

Este repo **no instala Argo CD**. Lo único que vive acá es el `ApplicationSet` (`bootstrap/applicationset.yaml`) — la config declarativa de *qué* sincronizar. El controller en sí (Helm install, upgrades, RBAC) es infraestructura del cluster, mismo criterio que el ALB Controller o CoreDNS — todo eso va en `aws-eks-cluster` o `azure-aks-cluster`, según el cluster. Separación deliberada: instalar Argo es una decisión de infra (una vez, por cluster), agregar una app nueva es una decisión de este repo (constante) — mezclarlas en un solo repo hace que cada cambio de app dispare revisión de infra sin necesidad.

## Por qué Kustomize, no Helm

Sin lenguaje de templating, `kubectl kustomize` nativo, y el patrón `base` + `overlays` mapea directo a "una app, múltiples ambientes". Helm tendría sentido si estas apps se fueran a *distribuir* a terceros — no es el caso acá.

## Por qué `podinfo` como app de ejemplo

[`stefanprodan/podinfo`](https://github.com/stefanprodan/podinfo) es la app de referencia estándar para probar pipelines de GitOps — construida por el propio mantenedor de Flux, usada en los tests end-to-end de Flux/Flagger. Se usa acá en vez de armar una imagen propia para no tener que mantener un Dockerfile solo para el demo. Health checks (`/healthz`, `/readyz`) y `securityContext` hardened (non-root, sin capabilities, `readOnlyRootFilesystem`) ya vienen soportados nativamente por la imagen — no hubo que ajustar nada para que corriera con esas restricciones.

## Sin Ingress por defecto — investigado, no asumido

Se evaluó compartir el mismo ALB de `hello-world` (via `alb.ingress.kubernetes.io/group.name`) con ruteo por path (`/podinfo`). Problema real: **ALB no reescribe/quita el prefijo del path** al reenviar al backend — `podinfo` solo conoce rutas en la raíz (`/`, `/healthz`, etc.), así que `/podinfo` le llegaría tal cual y devolvería 404.

Se confirmó vía la doc oficial de AWS (release notes de EKS Auto Mode, jul 2026) que el AWS Load Balancer Controller **sí agregó URL rewrite** en v2.13/v2.14 — nuestro controller instalado es v3.5.0, así que la feature debería estar disponible. Pero no se encontró/verificó la sintaxis exacta de la annotation para esta feature específica antes de escribir este repo — en vez de adivinar y arriesgarme a documentar algo que no compila o no funciona, se dejó afuera. `podinfo` queda con `ClusterIP` únicamente (`kubectl port-forward` para probarlo). Ver README, sección "Exposing an app publicly", para las dos opciones reales y lo que falta confirmar antes de usar la opción 2 (rewrite).

## Prerrequisito real para namespaces nuevos

El Fargate Profile que determina si un pod puede siquiera arrancar se define en Terraform, en `aws-eks-cluster/eks.tf` — no acá. Hoy existen profiles para `default`, `kube-system`, `argocd` y `keda`. Si una app nueva necesita su propio namespace (aislamiento, borrar todo con `kubectl delete namespace`), hay que agregar un Fargate Profile ahí ANTES de desplegar acá — si no, el pod queda `Pending` para siempre, sin error obvio de por qué.

## CI: Checkov con framework `kubernetes`, no `terraform`

Mismo Checkov que el resto del ecosistema, pero apuntado a manifiestos K8s planos (`framework: kubernetes` en vez de `terraform`) — Checkov soporta ambos frameworks nativamente. `kubeconform` complementa validando contra el schema real de la API de Kubernetes (typos de campos, valores inválidos) antes de que eso llegue a Argo.

**Gotcha real, no obvio**: el comentario `#checkov:skip=CKV_XXX:motivo` que funciona en todos los `.tf` de este workspace **no hace nada en manifiestos de Kubernetes** — falla en silencio, sin error, el check simplemente sigue apareciendo como FAILED. La sintaxis correcta para K8s es una **annotation** en `metadata`: `checkov.io/skipN: "CKV_XXX=motivo"` (confirmado contra la doc oficial de Checkov después de que el primer intento con comentarios no tuviera efecto). Si un check está asociado al Pod derivado de un `Deployment` (no al Deployment en sí, ej. `CKV2_K8S_6`), puede hacer falta repetir la annotation en `spec.template.metadata.annotations`, no solo en el `metadata` de nivel Deployment.

**Segundo gotcha, encontrado a mano**: el parser de Checkov para estas annotations hace `valor.split("=")` esperando exactamente 2 partes (ID y motivo) — si el texto del motivo contiene **cualquier otro `=`** (ej. explicando código que usa `drop=ALL`, o cualquier snippet con `=`), el split devuelve más de 2 partes y Checkov crashea con `ValueError: too many values to unpack`, no falla ese check puntual — **rompe la corrida entera**. Evitar `=` dentro del texto de motivo de cualquier `checkov.io/skipN`, sin excepción.

**Tercer gotcha, el más caro en tiempo de debug**: Checkov (`checkov -d <dir>`) escanea **cada archivo YAML por separado**, sin pasar primero por `kustomize build` — no entiende que un archivo es un *patch* parcial de Kustomize (ej. `overlays/aks/patch-virtual-node.yaml`, que solo declara `nodeSelector`/`tolerations`, nada más). Lo trata como si fuera un Deployment completo por sí solo, y por supuesto le "faltan" `securityContext`, `resources`, etc. — findings masivos y falsos, que agregar más `checkov.io/skipN` **no arregla**, porque el hallazgo apunta al archivo de patch, no al recurso real ya fusionado (visible en el campo `File:` del output de Checkov — si apunta a un archivo bajo `overlays/*/patch-*.yaml`, es esto, no un problema real del manifiesto). Fix real: `validate.yml` corre `kubectl kustomize` primero, guarda el resultado renderizado en `rendered/`, y Checkov escanea **eso** (`directory: rendered,bootstrap,apps/headlamp/extra`) — nunca la fuente cruda de `apps/*/base` o `apps/*/overlays`. Si se agrega un patch de Kustomize nuevo, no hace falta nada especial para esto — ya queda cubierto por el mismo mecanismo.

## KEDA con trigger cpu no escala nada en AKS Virtual Nodes - confirmado, no un bug de config

Las 3 apps demo (podinfo, game-2048, uptime-kuma) van a Virtual Nodes en AKS (ver `overlays/aks/patch-virtual-node.yaml`, para no competir por el node pool real con Argo CD/KEDA). El `keda-hpa-*` de cada una queda permanentemente en `ScalingActive: False, FailedGetResourceMetric` ahi - `kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods` nunca devuelve una entrada para ningun pod agendado en `virtual-node-aci-linux`, aunque metrics-server si resuelve sin problema pods del node pool real (confirmado con Headlamp, que corre ahi). No es intermitente: se repitio el chequeo varias veces con resultado idéntico.

La documentacion oficial de Microsoft (`learn.microsoft.com/en-us/azure/aks/virtual-nodes`, seccion Limitations) confirma que **initContainers no estan soportados en Virtual Nodes** en absoluto - esto probablemente explica por que metrics-server no puede reportar stats de esos pods (el estado del pod incluye un container terminado que el proveedor ACI no expone de forma que el summary API pueda parsear), aunque el pod arranca y corre igual (`game-2048`'s `init-nginx-dirs` no rompe nada visible - el bug de permisos que resuelve ese initContainer es especifico de EKS Fargate, no reproduce en ACI, asi que aunque el initContainer no se ejecute de verdad ahi no hay sintoma).

**Consecuencia real, no cosmetica**: el trigger `cpu` de KEDA (`type: cpu`, todos los `ScaledObject` de este repo) depende 100% de la metrics API de Kubernetes - sin datos, el HPA subyacente nunca escala, se queda fijo en `minReplicaCount` para siempre, silenciosamente (no hay error visible salvo `kubectl describe hpa`). Esto es especifico de AKS - en EKS Fargate el mismo trigger cpu funciona correctamente (confirmado con carga real, ver README). Si en el futuro se necesita autoescalado real en AKS para una app en Virtual Nodes, la opcion real es un trigger *externo* de KEDA (cron, cola, métrica custom) que no dependa de metrics-server - no cpu/memory.

## Sobre el Argo CD gestionado por AWS (EKS Capability for Argo CD)

Evaluado y descartado por ahora — ver el detalle completo en la conversación que originó este repo. Resumen: fully-managed, corre fuera del cluster en la cuenta de AWS, cero mantenimiento, pero cobra por hora de Capability + por hora de cada `Application` gestionada, y solo despliega a clusters EKS (no portable a AKS si algún día se necesita). Para 1 cluster con pocas apps demo, el self-hosted (lo que se decidió acá) no tiene ese costo y sigue siendo agnóstico de cloud. Reconsiderar si esto escala a múltiples clusters/cuentas.

## No automatizado todavía

Bump del tag de imagen de `podinfo` (o de cualquier app futura) — Dependabot no escanea YAML de Kubernetes plano buscando referencias `image:` de la misma forma que lo hace con Dockerfiles. Si esto llega a valer la pena automatizar, mirar el `image-automation-controller` de Flux o Renovate antes de armar algo a mano.

## Consumidores

Ninguno — proyecto hoja. Argo CD (instalado en cada cluster — `aws-eks-cluster` o `azure-aks-cluster`) es quien lee este repo, no al revés.
