Refer to: https://github.com/f5devcentral/modern_app_jumpstart_workshop/blob/main/docs/ingress/setup.md


```
helm --kubeconfig config-siteA.yaml install -f bigipCtrl-siteA.yaml bigipctrl-a f5-stable/f5-bigip-ctlr

helm --kubeconfig config-siteA.yaml uninstall bigipctrl-a
```

## Commands to build k3s:

### to destroy
```
sudo /usr/local/bin/k3s-uninstall.sh
```

### to build

```
advaddress=10.1.70.30
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --egress-selector-mode=disabled \
    --bind-address $advaddress --advertise-address $advaddress \
    --node-ip $advaddress \
    --node-external-ip $advaddress --flannel-iface ens6 \
    --kube-apiserver-arg=feature-gates=LegacyServiceAccountTokenNoAutoGeneration=false" sh -s -
```

### to authenticate from remote

```
kubectl -n kube-system create serviceaccount udf-sa
kubectl create clusterrolebinding udf-sa-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:udf-sa
TOKENNAME=`kubectl -n kube-system get serviceaccount/udf-sa -o jsonpath='{.secrets[0].name}'`
echo $TOKENNAME
TOKEN=`kubectl -n kube-system get secret $TOKENNAME -o jsonpath='{.data.token}' | base64 --decode`
echo $TOKEN

NEWCFG=config-siteB.yaml
## Adjust site name
HOST=`curl -s metadata.udf/deployment | jq '.deployment.components[] | select(.name == "MiniKube site B")| .accessMethods.https[] | select(.label == "K3s API").host' -r`

CA=`openssl s_client -connect $HOST:443 2>&1 </dev/null | sed -ne '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p'|base64 -w 0`

kubectl --kubeconfig=$NEWCFG config set-cluster udf  --server=https://$HOST:443
kubectl --kubeconfig=$NEWCFG config set clusters.udf.certificate-authority-data $CA
TOKENNAME=`kubectl -n kube-system get serviceaccount/udf-sa -o jsonpath='{.secrets[0].name}'`
TOKEN=`kubectl -n kube-system get secret $TOKENNAME -o jsonpath='{.data.token}' | base64 --decode`
kubectl --kubeconfig=$NEWCFG config set-credentials udf-sa --token=$TOKEN

# Set context
kubectl --kubeconfig=$NEWCFG config set-context udf --cluster=udf --namespace=default --user=udf-sa

# Set current context
kubectl --kubeconfig=$NEWCFG config set current-context udf

# Display kubeconfig file
cat $NEWCFG
```