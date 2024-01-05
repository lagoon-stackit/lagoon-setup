# 1. Certmanager

## 1.1 Add repo
    helm repo add jetstack https://charts.jetstack.io
## 1.2 Install
    kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml

    kubectl apply -f resources/issuer.yml

    helm upgrade --install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.13.3 \
    --set prometheus.enabled=false \
    --set webhook.timeoutSeconds=4 \
    --set ingressShim.defaultIssuerName=letsencrypt-prod \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --set ingressShim.defaultIssuerGroup=cert-manager.io


# 2. Ingress controller
## 2.1. Add repo
    helm repo add nginx-stable https://helm.nginx.com/stable
## 2.2 Install
    helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace -f values/ingress-values.yml
## 2.3 Check ingress ip and add it to the DNS configuration. May take some minutes
    kubectl get svc -n ingress-nginx

# 3. Helloworld app to check ingress and ssl configuration (optional)
    kubectl create namespace helloworld
    kubectl apply -f resources/hello-world.yaml

# 4. Dashboard (optional)
## 4.1. Add repo
    helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
## 4.2. Install
    helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f values/dashboard-values.yml --create-namespace -n kubernetes-dashboard

# 4.3. Harbor
    helm upgrade --install --create-namespace --namespace harbor --wait -f values/harbor-values.yml harbor harbor/harbor

# 5. NFS Server
## 5.1. Add Repo
    helm repo add nfs-ganesha-server-and-external-provisioner https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/
## 5.2. Install
    helm install bulk nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner -f values/nfs-values.yml

# 6. Lagoon core
## 6.1. Add Repo
    helm repo add lagoon https://uselagoon.github.io/lagoon-charts/
## 6.2. Install
    helm upgrade --install --create-namespace --namespace lagoon-core -f values/lagoon-core-values.yml lagoon-core lagoon/lagoon-core
Save password! This is needed to login to ui.kk8s.runs.onstackit.cloud using keycloak.

## 6.3. Delete (If something goes wrong)
    helm delete lagoon-core --namespace lagoon-core
    kubectl delete pvc --all --namespace lagoon-core
    kubectl delete namespace lagoon-core
# 7. Lagoon Remote
## 7.1. Get Rapbt IQ secret and copy the value lagoon-remote-values.yml:lagoon-build-deploy.rabbitMQPassword (without the trailing %)
kubectl -n lagoon-core get secret lagoon-core-broker -o jsonpath="{.data.RABBITMQ_PASSWORD}" | base64 --decode
## 7.2. Copy the Public IP from your cluster to lagoon-remote-values.yml:lagoon-build-deploy.taskSSHHost
kubectl get service lagoon-core-ssh -o custom-columns="NAME:.metadata.name,IP ADDRESS:.status.loadBalancer.ingress[*].ip,HOSTNAME:.status.loadBalancer.ingress[*].hostname" -n lagoon-core
## 7.3. Install
helm upgrade --install --create-namespace --namespace lagoon -f values/lagoon-remote-values.yml lagoon-remote lagoon/lagoon-remote


# 8. Connect remote to core
## 8.1. Login to ui.kk8s.runs.onstackit.cloud and add your ssh key
## 8.2. Add config to lagoon client.
lagoon config add --graphql https://api.kk8s.runs.onstackit.cloud/graphql --ui https://ui.kk8s.runs.onstackit.cloud --hostname  IP_STEP_7.2 --lagoon lagoon-kk8s --port 22
## 8.3. Lagoon login
    lagoon login
## 8.4 copy to token
    cat ~/.lagoon.yml:lagoon-kk8s.toekn
## 8.5. Register lagoon remote to lagoon core:
```
curl -g \
-X POST \
-H "Content-Type: application/json" \
-H "Authorization: Bearer {TOKEN FROM 8.4}" \
-d '{"query":"mutation addKubernetes { addKubernetes(input: { name: \"lagoon-kk8s\",consoleUrl: \"htps://api.kk8s.runs.onstackit.cloud\", token:\"TOKEN_8.4\", routerPattern: \"${environment}-${project}.kk8s.runs.onstackit.cloud\"}){id}}"}' \
https://api.kk8s.runs.onstackit.cloud/graphql
```
## 9. Add first project
```
lagoon add project \
--gitUrl https://gitlab+deploy-token-23:_KyJ7YpJAF1aJYqsQ1qd@git.key-tec.de/cw/lagoon-drupal \
--openshift 1 \
--productionEnvironment main \
--branches "^(main|develop)$" \
--project "lagoon-sample"
```
## 10. Run your first deployment
    lagoon deploy branch -p lagoon-sample -b main
