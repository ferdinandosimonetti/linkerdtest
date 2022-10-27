# install, multicluster, failover

## env preparation
```
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ export KUBECONFIG="./east.yml:./west.yml"
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl config get-contexts
CURRENT   NAME   CLUSTER   AUTHINFO                                     NAMESPACE
*         east   east      clusterAdmin_DefaultResourceGroup-PAR_east   
          west   west      clusterAdmin_DefaultResourceGroup-PAR_west   
```
## install cert-manager on both clusters
https://cert-manager.io/docs/installation/helm/
```
for c in east west
do
    helm install --kube-context $c \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --set installCRDs=true
done
```
## create local CA (if already present, skip)
https://smallstep.com/docs/step-cli/installation
```
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca --no-password --insecure --not-after=87600h
```
## create linkerd namespace and secret for the CA
```
for c in east west
do
    kubectl --context $c create namespace linkerd
    kubectl --context $c create secret tls \
    linkerd-trust-anchor \
    --cert=ca.crt \
    --key=ca.key \
    --namespace=linkerd
done
```
## create Issuer and Certificate from the secret
```
for c in east west
do
    kubectl --context $c apply -f linkerdIssuer.yml
    kubectl --context $c apply -f linkerdCertificate.yml
done
```
## install Linkerd CRDs and Control Plane
```
for c in east west
do
    helm --kube-context $c install linkerd-crds --namespace linkerd linkerd/linkerd-crds
    sleep 2
    helm --kube-context $c install linkerd-control-plane --namespace linkerd \
    --set-file identityTrustAnchorsPEM=ca.crt \
    --set identity.issuer.scheme=kubernetes.io/tls \
    linkerd/linkerd-control-plane
done
```
## install Multicluster add-on
```
for c in east west
do
    helm --kube-context $c install linkerd-multicluster --namespace linkerd-multicluster \
    --create-namespace \
    linkerd/linkerd-multicluster
done
```
## install Failover add-on
https://github.com/linkerd/linkerd-failover
```
for c in east west
do
    helm --kube-context $c install linkerd-smi --namespace linkerd-smi \
    --create-namespace \
    linkerd-smi/linkerd-smi
done
sleep 5
for c in east west
do
    helm --kube-context $c install linkerd-failover --namespace linkerd-failover \
    --create-namespace \
    linkerd/linkerd-failover
done
```
## linking two clusters
https://linkerd.io/2.12/tasks/multicluster/

Give resources running on **west** cluster knowledge about **east**'s gateway

```
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ linkerd --context=east multicluster link --cluster-name east | kubectl --context=west apply -f -
secret/cluster-credentials-east created
link.multicluster.linkerd.io/east created
clusterrole.rbac.authorization.k8s.io/linkerd-service-mirror-access-local-resources-east created
clusterrolebinding.rbac.authorization.k8s.io/linkerd-service-mirror-access-local-resources-east created
role.rbac.authorization.k8s.io/linkerd-service-mirror-read-remote-creds-east created
rolebinding.rbac.authorization.k8s.io/linkerd-service-mirror-read-remote-creds-east created
serviceaccount/linkerd-service-mirror-east created
deployment.apps/linkerd-service-mirror-east created
service/probe-gateway-east created
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ linkerd --context=west multicluster check
linkerd-multicluster
--------------------
√ Link CRD exists
√ Link resources are valid
        * east
√ remote cluster access credentials are valid
        * east
√ clusters share trust anchors
        * east
√ service mirror controller has required permissions
        * east
√ service mirror controllers are running
        * east
√ all gateway mirrors are healthy
        * east
√ multicluster extension proxies are healthy
√ multicluster extension proxies are up-to-date
√ multicluster extension proxies and cli versions match

Status check results are √
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ linkerd --context=west multicluster gateways
CLUSTER  ALIVE    NUM_SVC      LATENCY  
east     True           0          2ms  
```
## creating two test services
```
for ctx in west east; do
  echo "Adding test services on cluster: ${ctx} ........."
  kubectl --context=${ctx} apply \
    -n test -k "github.com/linkerd/website/multicluster/${ctx}/"
  kubectl --context=${ctx} -n test \
    rollout status deploy/podinfo || break
  echo "-------------"
done
````
## "exporting" the east one (using **mirror** feature)
```
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl --context=east label svc -n test podinfo mirror.linkerd.io/exported=true
service/podinfo labeled
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl --context=west get svc -n test
NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
frontend       ClusterIP   10.0.125.0   <none>        8080/TCP            7m25s
podinfo        ClusterIP   10.0.255.6   <none>        9898/TCP,9999/TCP   7m24s
podinfo-east   ClusterIP   10.0.99.32   <none>        9898/TCP,9999/TCP   16s
```
## checking the endpoints
```
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl --context=west -n test get endpoints podinfo-east \
>   -o 'custom-columns=ENDPOINT_IP:.subsets[*].addresses[*].ip'
nkerd-multicluster get svc linkerd-gateway \
  -o "custom-columns=GATEWAY_IP:.status.loadBalancer.ingress[*].ip"ENDPOINT_IP
20.74.29.222
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl --context=east -n linkerd-multicluster get svc linkerd-gateway \
>   -o "custom-columns=GATEWAY_IP:.status.loadBalancer.ingress[*].ip"
GATEWAY_IP
20.74.29.222
```
Endpoints of *podinfo-east* on west cluster are pointing to east's public gateway
## Trying to query mirrored "east" service from an *already meshed* pod gives the correct answer ("east" content)
```
ferdi@DESKTOP-NL6I2OD:~/linkerdtest$ kubectl --context=west -n test exec -c nginx -it \
>   $(kubectl --context=west -n test get po -l app=frontend \
>     --no-headers -o custom-columns=:.metadata.name) \
>   -- /bin/sh -c "apk add curl && curl http://podinfo-east:9898"
fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.16/community/x86_64/APKINDEX.tar.gz
OK: 26 MiB in 42 packages
{
  "hostname": "podinfo-66b6f88cfb-dxpgp",
  "version": "4.0.2",
  "revision": "b4138fdb4dce7b34b6fc46069f70bb295aa8963c",
  "color": "#007bff",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from east",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.14.3",
  "num_goroutine": "9",
  "num_cpu": "2"
}
```

## installing VIZ extension

```
for c in east west
do
    helm --kube-context $c install linkerd-viz -n linkerd-viz --create-namespace linkerd/linkerd-viz
done
```

## trying traffic split
https://linkerd.io/2.12/tasks/linkerd-smi/
```
kubectl create namespace trafficsplit-sample
linkerd inject https://raw.githubusercontent.com/linkerd/linkerd2/main/test/integration/viz/trafficsplit/testdata/application.yaml | kubectl -n trafficsplit-sample apply -f -
kubectl apply -f linkerdTrafficSplit.yml 

```