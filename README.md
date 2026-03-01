# reflector
I use reflector to propagate the TLS secret 'meilinknet' from the 'tls' namespace to all other namespaces. This way, the TLS secret is always available in all namespaces, even when a new application with a new ingress is deployed into the cluster. The 'meilink.net' TLS secret is initially written by a script that runs on enemigo in the root crontab every hour: /usr/local/bin/update-tls-secret-kubernetes.sh. If the TLS secret is missing in a namespace, refelector will copy it from the tls namespace into that namespace. Also, when the TLS secret is updated, it will also get copied to all other namespaces.

Github location for kubernetes-reflector: https://github.com/emberstack/kubernetes-reflector

How to use it: https://www.kubeblogs.com/how-to-use-kubernetes-reflector-simple-secret-management-across-namespaces/

You can pull the chart from its official repo:
```bash
# get repo
helm repo add emberstack https://emberstack.github.io/helm-charts
helm repo update

# search available versions
helm search repo emberstack --versions | grep reflector | head -5
# install
helm upgrade --install reflector emberstack/reflector -n reflector --create-namespace
# or just pull
helm pull emberstack/reflector --version 10.0.14 --untar --untardir charts
```
This repo contains a parent helm chart, and is initialized with: 
```bash
mkdir k8s-reflector
cd k8s-reflector
helm create reflector
mv reflector helm
rm -rf helm/templates/*
```
Edit helm/Chart.yaml:
```Bash
apiVersion: v2
name: reflector
description: A Helm chart for reflector on Kubernetes
type: application
version: 0.1.0 # version of the parent helm chart
appVersion: "10.0.14" # version of reflector being deployed
dependencies:
  - name: reflector
    version: 10.0.14 # version of the helm chart for reflector used
    repository: https://emberstack.github.io/helm-charts
```
Run the dependendency update which will download the kubed helm chart in tgz format and place it in the charts sub-folder as a child helm chart. It will also create the Chart.lock file.
Alternatively, use the chart-update.sh script to update the helm chart automatically.
```Bash
cd helm
helm dependency update
cd charts
tar xvf *tgz
rm -f *tgz
```
Next, update the values.yaml file in the parent helm chart. It is best practice to only have those values in this values.yaml file that overrride the values in values.yaml in the child helm chart.

Test deploy the helm chart - from the helm folder:
```Bash
helm -n reflector install reflector . -f values.yaml --create-namespace
```
To uninstall:
```Bash
helm -n reflector uninstall reflector
kubectl delete ns reflector
```
To have Reflector sync a secret, you need to add some annotations, e.g.:
```Bash
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry-secret
  namespace: default
  annotations:
    reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
    reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
```

