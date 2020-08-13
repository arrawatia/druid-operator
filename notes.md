Nginx ingress controller (not needed)
---
```shell script
helm repo add nginx-stable https://helm.nginx.com/stable
helm upgrade \
  --install nginx-controller nginx-stable/nginx-ingress \
  --namespace nginx-ingress \
  --set controller.service.httpPort.port=20080 \
  --set controller.service.httpsPort.port=20443 \
  --debug
```

Minikube
---
```shell script
minikube start  --cpus 4 --memory 12192 -v 9 --disk-size 60G
minikube addons enable ingress
minikube addons enable ingress-dns
```

Follow the steps here and add minikube as the dns resolver for *.test domain
https://github.com/kubernetes/minikube/tree/master/deploy/addons/ingress-dns

Hint: On mac you will need to kill the mDNSResponder for the dns changes to start working.
```shell script
sudo killall -HUP mDNSResponder
```
Minio
----
```
helm repo add minio https://helm.min.io/
helm upgrade \
    --install minio minio/minio \
    --namespace minio \
    --set resources.requests.memory=2Gi \
    --set persistence.size=500Gi \
    --set accessKey=AKIAIOSFODNN7EXAMPLE \
    --set secretKey="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
    --set ingress.enabled=true \
    --set ingress.hosts={minio.tiny.test} \
    --debug
```

Postgres
---
```shell script
helm repo add bitnami https://charts.bitnami.com/bitnami

helm upgrade \
  --install druid-metadata bitnami/postgresql \
  --namespace tiny-cluster \
  --set postgresqlUsername=druidpguser \
  --set postgresqlPassword=druidpgpassword \
  --set postgresqlDatabase=druiddb \
  --debug
```

Druid
---
```shell script

mkdir -p tmp && cd tmp
RELEASE_VERSION=v0.11.0 curl -LO https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/operator-sdk-${RELEASE_VERSION}-x86_64-apple-darwin
ln -s operator-sdk-v0.11.0-x86_64-apple-darwin operator-sdk
popd
make build
tmp/operator-sdk up local
```
```shell script
kubectl create ns tiny-cluster
kubectl create --namespace tiny-cluster -f deploy/service_account.yaml
# Setup RBAC
kubectl create --namespace tiny-cluster -f deploy/role.yaml
kubectl create --namespace tiny-cluster -f deploy/role_binding.yaml
# Setup the CRD
kubectl create --namespace tiny-cluster -f deploy/crds/druid.apache.org_druids_crd.yaml

# Update the operator manifest to use the druid-operator image name (if you are performing these steps on OSX, see note below)
sed -i 's|REPLACE_IMAGE|druidio/druid-operator:0.0.1|g' deploy/operator.yaml
# On OSX use:
sed -i "" 's|REPLACE_IMAGE|druidio/druid-operator:0.0.1|g' deploy/operator.yaml

# Deploy the druid-operator
make build-docker
kubectl apply --namespace tiny-cluster -f deploy/operator.yaml


# Pull the image manually until I figure out how to add image secrets
eval $(minikube -p minikube docker-env)
docker pull confluent-docker.jfrog.io/confluentinc/cc-druid:v0.46.0

# Check the deployed druid-operator
kubectl describe --namespace tiny-cluster deployment druid-operator
```
```shell script
kubectl apply --namespace tiny-cluster -f examples/tiny-cluster-zk.yaml
```

```shell script
kubectl apply --namespace tiny-cluster -f examples/tiny-cluster.yaml
```


Turnilo
---
```
kubectl run turnilo \
    --namespace tiny-cluster \
    --image=node:12 \
    --port=9090 \
    --command -- bash -c "npm install -g turnilo && turnilo --druid http://druid-tiny-cluster-routers:8088"

kubectl expose pods turnilo --namespace tiny-cluster --port=9090

cat <<EOF | kubectl apply --namespace tiny-cluster -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: turnilo-ingress
spec:
  rules:
  - host: turnilo.tiny.test
    http:
      paths:
      - path: /
        backend:
          serviceName: turnilo
          servicePort: 9090
EOF

```



Grafana
---
```shell script

helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade \
  --install grafana bitnami/grafana \
  --namespace tiny-cluster \
  --set plugins=abhisant-druid-datasource \
  --set admin.password=admin \
  --set ingress.enabled=true \
  --set "ingress.hosts[0].name"=grafana.tiny.test \
  --debug 
```

```shell script
curl 'http://grafana.tiny.test/api/datasources/1' \
  -X 'PUT' \
  -H 'Connection: keep-alive' \
  -H 'Pragma: no-cache' \
  -H 'Cache-Control: no-cache' \
  -H 'accept: application/json, text/plain, */*' \
  -H 'x-grafana-org-id: 1' \
  -H 'User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36' \
  -H 'content-type: application/json' \
  -H 'Origin: http://grafana.tiny.test' \
  -H 'Referer: http://grafana.tiny.test/datasources/edit/1/' \
  -H 'Accept-Language: en-US,en;q=0.9' \
  -H 'Cookie: grafana_session=9d22c01cd0dfc49d7be61d8ab325de5e' \
  --data-binary '{"id":1,"orgId":1,"name":"Druid","type":"abhisant-druid-datasource","typeLogoUrl":"","access":"proxy","url":"druid-tiny-cluster-routers:8088","password":"","user":"","database":"","basicAuth":false,"basicAuthUser":"","basicAuthPassword":"","withCredentials":false,"isDefault":true,"jsonData":{},"secureJsonFields":{},"version":2,"readOnly":false}' \
  --compressed \
  --insecure
```