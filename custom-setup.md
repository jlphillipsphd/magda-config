# TL;DR

I am generally following the magda-config guide to installing on an existing K8S cluster: [https://github.com/magda-io/magda-config/blob/master/existing-k8s.md
](https://github.com/magda-io/magda-config/blob/master/existing-k8s.md
)

However, I have the following key additions:

1. Add Google Authentication Plugin support - you will need your Client ID and Client Secret for your Google Cloud Platform OAuth2 Credential
2. Use Cert-Manager for self-signed TLS/SSL cert (for now)
3. Set up Traefik ingress (routes HTTP to HTTPS and provides SSL termination)

## Prerequisites
You will need the following (command-line) tools installed:

* helm (v3+)
* k3d
* kubectl 
* npm (for secrets creator)
* yarn (optional; to generate API keys)


# Make K3D Cluster
I will be using HTTP (80) and HTTPS (443) on `k3d-cluster`. You should check the external IP of the traefik load-balancer once the cluster has started and then add an entry in `/etc/hosts` to point to that IP address.
```
k3d cluster create
```

## Fork then clone the magda-config repo:

You will need to fork the repo: [https://github.com/magda-io/magda-config](https://github.com/magda-io/magda-config)
then use `git clone` to clone your forked copy...

## Install k8s replicator:
```
helm repo add mittwald https://helm.mittwald.de
helm repo update
kubectl create namespace kubernetes-replicator
helm upgrade --namespace kubernetes-replicator --install kubernetes-replicator mittwald/kubernetes-replicator
```

## Install cert-manager
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.6.1 --set installCRDs=true
```

## Create self-signed certificate issuer
```
kubectl apply -f selfsigned-issuer.yaml
```

## Install the secrets creator

Needs administrative privileges.

```
sudo npm install --global @magda/create-secrets
```

# Set up MAGDA

## Edits to chart/Chart.yaml

### Google Authentication Plugin
Add as a dependency following documentation from [https://github.com/magda-io/magda-auth-google](https://github.com/magda-io/magda-auth-google) Put this right after the default ckan plugin.

### Merge these edits
```
  - name: magda-auth-google
    version: 1.2.3
    repository: https://charts.magda.io
```


## Edits to chart/values.yaml

### Google Authentication Plugin

* Note that the secret key will be added to a k8s secret below.
* Be sure to add an "Authorized redirect URIs" on the GCP console: [https://localhost/auth/login/plugin/google/return](https://localhost/auth/login/plugin/google/return)
* Replace the ckan with google in the authPlugins section of the gateway config (see below)

### Merge these edits
```
global:
  externalUrl: https://k3d-cluster
gateway:
  authPlugins:
  - key: "google"
    baseUrl: http://magda-auth-google
  enableHttpsRedirection: true
magda-auth-google:
  googleClientId: "xxxxxx"
```

## Generate secrets
```
create-secrets
```

### Transcript from the command above...
```
magda-create-secrets tool version: 1.2.0-alpha.0
Found previous saved config (January 6th 2022, 7:22:51 pm).
? Do you want to connect to kubernetes cluster to create secrets without going through any questions? NO (Going through all questions)
? Are you creating k8s secrets for google cloud or local testing cluster? Local Testing Kubernetes Cluster
? Which local k8s cluster environment you are going to connect to? docker
? Do you need to access SMTP service for sending data request email? NO
? Do you want to create google-client-secret for oAuth SSO? YES
? Please provide google api access key for oAuth SSO: Your-OAuth2-Secret-ID-Here
? Do you want to create facebook-client-secret for oAuth SSO? NO
? Do you want to create arcgis-client-secret for oAuth SSO? NO
? Do you want to create aaf-client-secret for AAF Rapid Connect SSO? NO
? Do you want to setup HTTP Basic authentication? NO
? Do you want to manually input the password used for databases? Generated password: Ii5hiusepelupiem
? Please enter an access key for your MinIO server: minio_access
? Please enter a secret key for your MinIO server:: minio_secret
? Specify a namespace or leave blank and override by env variable later? YES (Specify a namespace)
? What's the namespace you want to create secrets into (input `default` if you want to use the `default` namespace)? magda
? Do you want to allow environment variables (see --help for full list) to override current settings at runtime? YES (Any environment variable can overide my settings)
? Do you want to connect to kubernetes cluster to create secrets now? YES (Create Secrets in Cluster now)
Failed to get k8s namespace magda or namespace has not been created yet: Error: Command failed: kubectl get namespace magda
? Do you want to create namespace `magda` now? YES
namespace/magda created
Successfully created secret `db-passwords` in namespace `magda`.
Successfully created secret `storage-secrets` in namespace `magda`.
Successfully created secret `oauth-secrets` in namespace `magda`.
Successfully created secret `auth-secrets` in namespace `magda`.
All required secrets have been successfully created!
```

### Configure role binding
Enables the ability to trigger connector jobs from the admin-api? Just in case really...
```
kubectl -n magda apply -f role-binding.yaml
```

# DEPLOY!

Note that the namespace was already created during the **create-secrets** step above...
```
helm repo add magda-io https://charts.magda.io
helm dep up ./chart
helm upgrade --install --namespace magda --timeout 9999s --debug magda ./chart
```

# TLS/SSL Termination

Generally following the ideas in [https://magda.io/docs/how-to-setup-https-to-local-cluster.html](https://magda.io/docs/how-to-setup-https-to-local-cluster.html), but using traefik instead of nginx.

## Enable HTTPS ingress

Create a self-signed cert and ingress route for HTTPS (using traefik):
```
kubectl -n magda apply -f selfsigned-cert.yaml
kubectl -n magda apply -f ingress-traefik-https.yaml
```

## Enable HTTP>HTTPS redirection
Create a namespace-scoped redirect middleware and deploy the ingress route for HTTP>HTTPS redirection.
```
kubectl -n magda apply -f middleware-https-redirect.yaml
kubectl -n magda apply -f ingress-traefik-https-redirect.yaml
```
# Check it out!

Visit: [https://k3d-cluster/](https://k3d-cluster/)

# Create an Admin User

Look up the DB password:

```
kubectl get secrets db-passwords -o yaml -n magda | grep " authorization-db:" | awk '{print $2}' | base64 -d
```

Set up a connection to the database:
```
kubectl port-forward combined-db-0 5432 -n magda
```

Install the admin tool:
```
sudo npm install --global @magda/acs-cmd
```

Make **someone** an admin:
```
# First we get the password for the database
kubectl get secrets db-passwords -o yaml -n magda | grep " authorization-db:" | awk '{print $2}' | base64 -d

# Now we can use it (copy-paste)
POSTGRES_PASSWORD="yourpassword" POSTGRES_USER="client" acs-cmd list users
POSTGRES_PASSWORD="yourpassword" POSTGRES_USER="client" acs-cmd admin set someone-user-id-from-list-above
```

Generate API key:
```
git clone https://github.com/magda-io/magda.git
cd magda
TO-DO

```


