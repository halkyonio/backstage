# Backstage

Table of Contents
=================

* [Prerequisites](#prerequisites)
* [Instructions](#instructions)
   * [New project](#new-project)
   * [Build and upload the container image](#build-and-upload-the-container-image)
* [Plugins](#plugins)
   * [k8s](#k8s)
* [Cleanup](#cleanup)
* [Rancher desktop](#rancher-desktop)

## Prerequisites

- Node (>= 16), npm and npx installed
- docker and kind available

## Instructions

Here are the steps I followed to install backstage on a k8s cluster according to the backstage documentation
and feedback that I got from the backstage engineers. 

You can use an existing project created using `@backstage/create-app` - version [0.4.28](https://www.npmjs.com/package/@backstage/create-app/v/0.4.28) 
and available [here](https://github.com/halkyonio/my-backstage.git) and implementing the following plugins:
- k8s: https://backstage.io/docs/features/kubernetes/overview
- CCF: https://github.com/cloud-carbon-footprint/ccf-backstage-plugin 

or create a new project as described hereafter

### New project 

Open a terminal and execute this command within the folder where you want to launch backstage
```bash
npx @backstage/create-app
npx: installed 70 in 12.614s
? Enter a name for the app [required] my-backstage
...
🥇  Successfully created my-backstage

 All set! Now you might want to:
  Run the app: cd my-backstage && yarn dev
  Set up the software catalog: https://backstage.io/docs/features/software-catalog/configuration
  Add authentication: https://backstage.io/docs/auth/
```
When the `yarn build` is over, move to the folder of the created project, add the package of the sqlite3 DB.
```bash
cd my-backstage 
yarn add --cwd packages/backend better-sqlite3
```
**Note**: Set this property if a more recent version of node is installed `yarn config set ignore-engines true`

Test it locally by launching this command and next access the UI at this address: `http://localhost:3000/`
```bash
yarn dev
```

### Build and upload the container image

Next, build the image and upload it within your local registry (or kind cluster)
```bash
yarn build
yarn build-image -t backstage:dev
kind load docker-image backstage:dev
```
**Note**: To support to build the `TechDocs` using the backend pod, some Dockerfile changes are needed to install the mkdocs package. See this [commit](https://github.com/halkyonio/my-backstage/commit/2d93a33901128ef78b3ef31906c26c59e6e0bc59)

We can now create the YAML values file needed by the Helm chart to expose the ingress route, get the extra config from a configMap and 
use the image built
```bash
DOMAIN_NAME="<VM_IP>.sslip.io"
cat <<EOF > $(pwd)/my-values.yml
ingress:
  enabled: true
  host: backstage.$DOMAIN_NAME
  className: nginx
backstage:
  image:
    registry: "docker.io/library"
    repository: "backstage"
    tag: "dev"
  extraAppConfig:
    - filename: app-config.extra.yaml
      configMapRef: my-app-config         
EOF
```
and deploy it
```bash
helm upgrade --install \
  my-backstage \
  backstage \
  --repo https://vinzscam.github.io/backstage-chart \
  -f $(pwd)/my-values.yml \
  --create-namespace \
  -n backstage
```

We can now create our `app-config.extra.yaml` backstage config file:
```bash
DOMAIN_NAME="<VM_IP>.sslip.io"
cat <<EOF > $(pwd)/app-config.extra.yaml
app:
  baseUrl: http://backstage.$DOMAIN_NAME
  title: Backstage
backend:
  baseUrl: http://backstage.$DOMAIN_NAME
  cors:
    origin: http://backstage.$DOMAIN_NAME
    methods: [GET, POST, PUT, DELETE]
    credentials: true      
  csp:
    connect-src: ['self','http:','https:']
  database:
    client: better-sqlite3
    connection: ':memory:'
  cache:
    store: memory
techdocs:
  builder: 'local' # Alternatives - 'external'
  generator:
    runIn: 'local' # Alternatives - 'local'
  publisher:
    type: 'local' # Alternatives - 'googleGcs' or 'awsS3'. Read documentation for using alternatives.
catalog:
  locations:
  - type: url
    target: https://github.com/mclarke47/dice-roller/blob/master/catalog-info.yaml    
EOF
```

**Note**: If you use the [my-backstage](https://github.com/halkyonio/my-backstage.git) github project, then configure too as described hereafter the `kubernetes` plugin ! 

Create the `configMap` containing our extra parameters and rollout the backstage app to reload its config
```bash
kubectl create configmap my-app-config -n backstage \
  --from-file=app-config.extra.yaml=$(pwd)/app-config.extra.yaml
  
kubectl rollout restart deployment/my-backstage -n backstage
```
Grab the URL of backstage and access it from your browser
```bash
BACKSTAGE_URL=$(kubectl get ingress/my-backstage -n backstage -o json | jq -r '.spec.rules[0].host')
echo "http://${BACKSTAGE_URL}"
```

## Plugins

### k8s

To use the backstage kubernetes plugin, it is needed to install 2 packages:
```bash
# From your Backstage root directory
yarn add --cwd packages/app @backstage/plugin-kubernetes
yarn add --cwd packages/backend @backstage/plugin-kubernetes-backend
```

and to set up the `Index` and `Backend` packages as described here: https://backstage.io/docs/features/kubernetes/installation

As we need some additional k8s resources deployed (backstage serviceaccount having the RBAC cluster admin role, the dice-roller example, ...) we will then deploy them:
```bash
kubectl apply -f manifests/dice-roller.yml
kubectl apply -f manifests/backstage-rbac.yml
```
Next, the existing ConfigMap must be extended to include the kubernetes config

```bash
BACKSTAGE_SA_TOKEN=$(kubectl -n backstage get secret $(kubectl -n backstage get sa backstage -o=json | jq -r '.secrets[0].name') -o=json | jq -r '.data["token"]' | base64 --decode)
cat <<EOF >> $(pwd)/app-config.extra.yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
  - type: 'config'
    clusters:
      - url: https://kubernetes.default.svc
        name: kind
        authProvider: 'serviceAccount'
        skipTLSVerify: true
        skipMetricsLookup: true
        serviceAccountToken: ${BACKSTAGE_SA_TOKEN}
EOF

kubectl create configmap my-app-config -n backstage \
  --from-file=app-config.extra.yaml=$(pwd)/app-config.extra.yaml \
  -o yaml \
  --dry-run=client | kubectl apply -n backstage -f -

kubectl rollout restart deployment/my-backstage -n backstage
```
Open the url of the `dice-roller` service (e.g. https://backstage.VM_IP.sslip.io/catalog/default/component/dice-roller/kubernetes) and click on the `kubernetes` tab, and you will see

![](k8s-plugin.png)

## Cleanup

To uninstall the chart
```bash
helm uninstall my-backstage -n backstage
```
**Note**: If the resources of the chart must be changed locally, then pull/untar the project:
```bash
helm pull https://github.com/vinzscam/backstage-chart/releases/download/backstage-0.2.0/backstage-0.2.0.tgz --untar --untardir ./
```
## Rancher desktop

If you plan to use `rancher-desktop`, then I recommend to disable `traefix` and to install the `ingress nginx` controller
```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.hostPort.enabled=true
```

The command to be used to build the backstage image is `nerdctl build --namespace k8s.io -f packages/backend/Dockerfile -t backstage:dev .` 
instead of `yarn build-image -t backstage:dev`
