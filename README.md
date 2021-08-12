# Ambiente CI + CD en Kubernes con Azure: Parte II

A continuación retomaremos nuestro ambiente de MetalCloud desde donde lo dejamos en el workshop. Aunque desplegaremos una aplicacion de ejemplo para nuestro primer API Gateway!

## Pre-requisitos
- Seguir el paso 1 del workshop (brach main de este repo)

## Pre-requisitos en tu maquina

Asumimos que ya tienes instalado:

1. Azure CLI (sino: https://docs.microsoft.com/es-mx/cli/azure/install-azure-cli)
2. Drone CLI (sino: https://readme.drone.io/cli/install/)
3. Kubectl (sino: https://kubernetes.io/es/docs/tasks/tools/install-kubectl/)
4. GIT (sino: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

----------

# Manos a la obra
Pasos para instalar el entorno de CI + CD

# Tema 1: DroneCI

## Paso 1: Mejorar pipelines

> Revisa los pipelines de ./DroneCI/Pipelines/

En ellos se puede ver el uso de "promote" para el despliegue en los distintos ambientes, y restricciones segun la rama. Además de automatizar mediante git el despliegue de ArgoCD.

## Paso 2: Crear plantillas de pipelines

Ejecutar los siguientes comandos:

```console
    cd ./DroneCI/Templates/

    drone template add --namespace "CI-CD-K8s-AZ" --name angular.starlark --data @angular.starlark
    drone template add --namespace "CI-CD-K8s-AZ" --name maven.starlark --data @maven.starlark

    cd ../..
```

> Para obtener tu plantilla en formato YAML ejecuta: 

```console
    drone starlark convert --stdout --source <nombrearchivo>
```

PD: El archivo por defecto que busca es ".drone.star" si no especificas el parametro "--source".

----------------------------------------------------------------

# Tema 2: Kong API Gateway

## Paso 1: Instalar Kong Ingress Controller

Ejecutar los siguientes comandos:

```console
    kubectl create namespace kong

    helm repo add kong https://charts.konghq.com
    helm repo update
    helm install kong kong/kong -n kong --set admin.enabled=true \
        --set env.database=postgres \
        --set env.pg_password=admin1234 \
        --set postgresql.enabled=true \
        --set postgresql.postgresqlUsername=kong \
        --set postgresql.postgresqlPassword=admin1234 \
        --set postgresql.postgresqlDatabase=kong
```

## Paso 2: Instalar konga

Ejecutar los siguientes comandos:

```console
    kubectl apply -f ./Konga/
    kubectl patch service -n kong kong-kong-admin -p '{"metadata":{"annotations": { "konghq.com/protocol" : "https"}}}'
```

> URL API: https://kong-kong-admin.kong.svc:8444/

## Paso 3: Cambiar NGINX por Konga

Para cambiar nuestro NGINX por Konga, primero debemos eliminar nuestro actual Ingress Controller y cambiar nuestro DNS por la IP
de nuestro proxy Kong. Para eliminar NGINX Ingress Controller simplemente debemos ejecutar:

```console
    kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/cloud/deploy.yaml
```

### Reparar Certmanager
kubectl apply -f ./Certmanager/issuer.yaml

### Reparar Kubernetes Dashboard
kubectl delete -f ./Dashboard/ingress.yaml
kubectl apply -f ./Dashboard/ingress.yaml
kubectl patch service -n kubernetes-dashboard kubernetes-dashboard -p '{"metadata":{"annotations": { "konghq.com/protocol" : "https"}}}'

### Reparar DroneCI
kubectl delete -f ./DroneCI/server/droneserver-ingress.yaml
kubectl apply -f ./DroneCI/server/droneserver-ingress.yaml

### Reparar ArgoCD
kubectl delete -f ./ArgoCD/ingress.yaml
kubectl apply -f ./ArgoCD/ingress.yaml
kubectl patch service -n devops argocd-server -p '{"metadata":{"annotations": { "konghq.com/protocol" : "https"}}}'

### Reparar SonarQube
kubectl patch ingress -n devops sonar-sonarqube -p '{"metadata":{"annotations": { "cert-manager.io/cluster-issuer" : "letsencrypt-prod"}}}'
kubectl patch ingress -n devops sonar-sonarqube -p '{"status":{}}'

# Listo!
Ya estas listo para comenzar a configurar tu API Gateway y desplegar proyectos rápidamente gracias a las plantillas!
