# TeamCity deployment in Kubernetes

## Prep
### Helm
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm dependency build
```

### k8s (minikube)
```bash
minikube start
minikube tunnel
```

## Deployment
### emptyDir
By default the helm chart will be deployed by using the emptyDir storage for the teamcity data folder.
```bash
helm install assignment .
```

The superuser credentials are printed to the pod logs.
```bash
-teamcity-...
```

Uninstall the release: `helm uninstall assignment`

### pvc
For using a persistent volume, set `persistence.enabled = true`.
```bash
helm install assignment-pvc . -f pvcEnabled.yaml
```

## configure
### cloud profile
```
# if a LB is configured
export SERVICE_IP=$(kubectl get svc --namespace default assignment-pvc-teamcity --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
xdg-open http://${SERVICE_IP}/admin/editProject.html?projectId=_Root&tab=clouds&action=new&showEditor=true
```
* Profile name: `test`
* Cloud type: `Kubernetes`
* Kubernetes API server URL: `https://kubernetes.default.svc`
* Kubernetes namespace: `agents`
* Authentication strategy: `Default service account`

Test connection should be green now.


Click on `Add image`
* Pod specification: `Run single container`
* Docker image: `jetbrains/teamcity-agent`
* Agent pool: `default`

Finally create the profile

### build
```
xdg-open http://${SERVICE_IP}/admin/createObjectMenu.html?projectId=_Root&showMode=createProjectMenu#createManually
```
* Name: test

Create the build and continue with creating a build configuration.

* Name: test

VCS can be skipped. Next a `Command Line` build step will be added.

* Clustom script: `echo "Hello World"`

Now the build can be triggered manually. It will be executed on the build agent.

## prometheus
Prometheus is installed by using the community provided helm chart.
To connect to the metrics endpoint a new user needs to be created.

```bash
xdg-open http://${SERVICE_IP}/admin/createUser.html
```
* username: `user`
* password: `user`
* Give this user administrative privileges: `true`

Login as the new user and create an access token
```bash
xdg-open http://${SERVICE_IP}/profile.html?item=accessTokens#
```
* Token name: `metrics`

Copy the token to the `serviceMonitor.yaml` file and update the deployment.
```bash
helm upgrade assignment-pvc . -f pvcEnabled.yaml -f serviceMonitor.yaml
```

## grafana
```bash
export GRAFANA_SERVICE_IP=$(kubectl get svc --namespace default assignment-pvc-grafana --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
xdg-open http://${GRAFANA_SERVICE_IP}
```
Login with `admin`/`admin`

Import dashboard
```bash
xdg-open http://${GRAFANA_SERVICE_IP}/dashboard/import
```
* Import via grafana.com: `11669`
