# Ambiente CI + CD en Kubernes con Azure

A continuación te mostraré los pasos para que puedas montar tu propio sistema de CI + CD en k8s desde cero.

## Pre-requisitos en Azure

Debes crear en Azure lo siguiente:

1. Crear Cluster AKS en Azure (Lo haremos en el Workshop ;) )
2. Crear Zona DNS en Azure
3. Configurar tu dominio para que apunte a Azure 
4. Crear un Registro de contenedor en Azure

## Pre-requisitos en tu maquina

Asumimos que ya tienes instalado:

1. Azure CLI (sino: https://docs.microsoft.com/es-mx/cli/azure/install-azure-cli)
2. Drone CLI (sino: https://readme.drone.io/cli/install/)
3. Kubectl (sino: https://kubernetes.io/es/docs/tasks/tools/install-kubectl/)
4. GIT (sino: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

----------

# Manos a la obra
Pasos para instalar el entorno de CI + CD

# Paso 1: Crear los Namespaces

Ejecutar los siguientes comandos:

`
    kubectl create namespace dev
    kubectl create namespace qa
    kubectl create namespace prod
    kubectl create namespace devops
`

# Paso 2: Instalar Ingress Controller

Ejecutar el siguiente comando:

`
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/cloud/deploy.yaml
`


# Paso 3: Registrar DNS con Ingress

Registrar en DNS los siguientes subdominios apuntando a la IP pública del Ingress Controller:

- k8s.metalcloud.cl
- drone.metalcloud.cl
- argo.metalcloud.cl
- sonar.metalcloud.cl
- dev.metalcloud.cl
- qa.metalcloud.cl
- metalcloud.cl
- www.metalcloud.cl

> Nota: metalcloud.cl es el dominio de prueba, reemplazalo por el tuyo

# Paso 4: Instalar Certmanager

Ejecutar los siguientes comandos:

`
    kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
    kubectl apply -f ./Certmanager/issuer.yaml
`
> Nota: Recuerda cambiar el correo del archivo "issuer.yaml". 

# Paso 5: Instalar Kubernetes Dashboard

Ejecutar los siguientes comandos:

`
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
    kubectl apply -f ./Dashboard/ingress.yaml
    kubectl apply -f ./Dashboard/rbac.yaml
`

> Nota: metalcloud.cl es el dominio de prueba, reemplazalo por el tuyo. Modifica el archivo "./Dashboard/ingress.yaml"

Para obtener un token:

`
    secret=$(kubectl -n kubernetes-dashboard get secret | grep kubernetes-dashboard-token | awk '{print $1}')
    token=$(kubectl -n kubernetes-dashboard get secret $secret --template={{.data.token}} | base64 -d)
    echo "$token" | pbcopy
`

> Nota: Comando "pbcopy" solo para MAC, copia al portapapeles

# Paso 6: Crear Token de GitHub

Crea una aplicación OAuth en GitHub en https://github.com/settings/developers:

![paso1](https://docs.drone.io/screenshots/github_application_create.png)


> Recuerda reemplazar https://drone.domain.com por tu dominio, para el ejemplo: https://drone.metalcloud.cl/


![paso2](https://docs.drone.io/screenshots/github_application_created.png)

> Fuente: https://docs.drone.io/server/provider/github/#:~:text=Create%20a%20GitHub%20OAuth%20application

# Paso 7: Instalar DroneCI

## Establece los secretos

En este punto debes tener un ClientID y ClientSecret de GitHub, por lo tanto, debes modificar el archivo "./DroneCI/server/droneserver-secret.yaml" de la siguiente manera:

    DRONE_GITHUB_CLIENT_ID: <base64 de tu ClientID>
    DRONE_GITHUB_CLIENT_SECRET: <base64 de tu ClientSecret>
    DRONE_USER_CREATE: <base64 de "username:<tu nombre de usuario en GitHub>,admin:true"
    DRONE_SERVER_HOST: <base64 de drone.metalcloud.cl> (reemplazar con tu dominio)

> Nota: metalcloud.cl es el dominio de prueba, reemplazalo por el tuyo. Modifica el archivo "./DroneCI/server/droneserver-ingress.yaml"

## Instala Drone CI
Luego, ejecuta los siguientes comandos:

`
    kubectl apply -f ./DroneCI/postgres/postgres.yaml
    kubectl apply -f ./DroneCI/server/
    kubectl apply -f ./DroneCI/runner/
`




# Paso 8: Instalar ArgoCD

Ejecutar los siguientes comandos:

`
    kubectl apply -n devops -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    kubectl apply -f ./ArgoCD/ingress.yaml
    kubectl apply -f ./ArgoCD/rbac.yaml
`

> Nota: metalcloud.cl es el dominio de prueba, reemplazalo por el tuyo. Modifica el archivo "./ArgoCD/ingress.yaml"

Para obtener la contraseña de admin:

`
    kubectl -n devops get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
`

> Nota: Comando "pbcopy" solo para MAC, copia al portapapeles

# Paso 9: Instalar Sonarqube

Ejecutar los siguientes comandos:

`
    git clone https://github.com/SonarSource/helm-chart-sonarqube.git
    cd helm-chart-sonarqube/charts/sonarqube
    helm dependency update
    helm upgrade --install -f values.yaml -n devops sonar ./ --set "ingress.enabled=true,ingress.hosts[0].name=sonar.metalcloud.cl"

    kubectl patch ingress -n devops sonar-sonarqube -p '{"metadata":{"annotations": { "cert-manager.io/cluster-issuer" : "letsencrypt-prod"}}}'

    kubectl patch ingress -n devops sonar-sonarqube -p '{"spec":{"tls": [ { "hosts": [ "sonar.metalcloud.cl" ], "secretName":  "sonar-secret-tls"  } ] }}'

    cd ../../../
    rm -R helm-chart-sonarqube
`

> Nota: sonar.metalcloud.cl es el dominio de prueba, reemplazalo por el tuyo.

# Paso 10: Generar token de sonarquebe

Ir a https://sonar.metalcloud.cl/ (o tu dominio) e ingresar en:

- My Account –> Security –> Generate Tokens

# Paso 11: Generar token de Telegram

Busca @Botfather en telegram y sigue el bot para registrar tu bot y obtener tu API Key.

> Nota: No olvides que luego debes iniciar el bot con tu cuenta, ya que por ese chat te llegaran los mensajes.

# Paso 12: Crear los secretos

Ejecuta las variables de entorno de tu usuario drone en consola y luego ejecuta los siguientes comandos:

`
drone orgsecret add CI-CD-K8s-AZ telegram_token 1860294509:AAGs2LV6fjU75Bzq6qEqqX5ItzPIrJ0lZlg --allow-pull-request
drone orgsecret add CI-CD-K8s-AZ sonar_host https://sonar.metalcloud.cl --allow-pull-request
drone orgsecret add CI-CD-K8s-AZ sonar_token f9f2f6ab86a193781f8817a164809214417938ce --allow-pull-request

drone orgsecret add CI-CD-K8s-AZ registry_username metalcloud --allow-pull-request
drone orgsecret add CI-CD-K8s-AZ registry_password gFSxydmo79929lswdbUgwPosTMeOvM7+ --allow-pull-request
`

# Paso 13: Listo!

En este punto ya puedes comenzar a jugar con tu ambiente y probar los pipelines y deploys. Siguie el Workshop para ver un ejemplo practico.






----------
# FAQ

**Problemas con Registry**

En caso de tener problemas de permisos para descargar la imagen del Registry, ejecutar los siguientes comandos:

## Solución 1:

Este problema se debe a que nuestro cluster de AKS no tiene las credenciales necesarias para descargar imagenes del registry, por lo tanto, tenemos que asignarle un registry y dejar que azure se encargue de los permisos:

az aks update -n <nombre del cluster> -g <nombre del grupo de recursos> --attach-acr <nombre del registry>

## Solución 2:

Si la Solución 1 no es una opción, siempre podemos hacerlo manualmente. De esta forma creamos un secret en cada namespace que descarguemos imagenes con las credenciales de nuestro registry:

`
    kubectl create secret docker-registry azure-registry \
        --namespace devops \
        --docker-server=<nombre del registry>.azurecr.io \
        --docker-username=<usuario> \
        --docker-password=<contraseña>

    kubectl create secret docker-registry azure-registry \
        --namespace dev \
        --docker-server=<nombre del registry>.azurecr.io \
        --docker-username=<usuario> \
        --docker-password=<contraseña>

    kubectl create secret docker-registry azure-registry \
        --namespace qa \
        --docker-server=<nombre del registry>.azurecr.io \
        --docker-username=<usuario> \
        --docker-password=<contraseña>

    kubectl create secret docker-registry azure-registry \
        --namespace prod \
        --docker-server=<nombre del registry>.azurecr.io \
        --docker-username=<usuario> \
        --docker-password=<contraseña>
`

Finalmente agregamos las siguientes lines en nuestro archivo de manifiesto de kubernetes (yaml):

`
    spec:
      containers:
        ...
    imagePullSecrets:
        - name: azure-registry
`
