# SRE Monitoring for OCP
https://github.com/pingcap/chaos-mesh

## Install Gafana Operator

```shell
oc new-project sre-monitoring
oc apply -f grafana-operator.yaml -n sre-monitoring
#fixes some operator's bug
oc scale deployment grafana-operator --replicas=0
#oc set image deployment/grafana-operator grafana-operator=quay.io/integreatly/grafana-operator:v3.0.0-beta
oc set image deployment/grafana-operator grafana-operator=quay.io/integreatly/grafana-operator:v1.4.0
oc set image deployment/grafana-operator grafana-operator=quay.io/integreatly/grafana-operator:v3.2.0
oc scale deployment grafana-operator --replicas=1
```

## Deploy Grafana - connected to platform Prometheus

```shell
oc create sa platform-prometheus-reader -n sre-monitoring
oc adm policy add-cluster-role-to-user view -z platform-prometheus-reader -n sre-monitoring
export secret_name=$(oc get sa platform-prometheus-reader -o json | jq -r '.secrets[].name | select (.|contains("token"))')
export token=$(oc get secret $secret_name -o jsonpath='{.data.token}' | base64 -d)
helm template grafana-ocp --namespace sre-monitoring --set prometheus_datasource.token=$token | oc apply -f -
# fixes for current bugs of the grafana operator
oc annotate serviceaccount grafana-serviceaccount serviceaccounts.openshift.io/oauth-redirectreference.grafana='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"grafana"}}' -n sre-monitoring

#oc annotate service grafana-service service.alpha.openshift.io/serving-cert-secret-name=grafana-tls -n sre-monitoring
```

## Deploy Service Monitoring

```shell
helm template sre-service-monitor --namespace grafana-operator | oc apply -f -
```

## Monitoring the master-api

```shell
helm template sre-service-monitor --namespace openshift-monitoring --values ./master-api-values.yaml | oc apply -f -
```

## Deploy Grafana - connected to istio Prometheus


```shell
export token=$(oc get secret htpasswd -n istio-system -o jsonpath='{.data.rawPassword}' | base64 -d)
helm template grafana-ocp --namespace sre-monitoring --values ./istio-prometheus-values.yaml --set prometheus_datasource.token=$token | oc apply -f -
# fixes for current bugs of the grafana operator
oc annotate serviceaccount grafana-serviceaccount serviceaccounts.openshift.io/oauth-redirectreference.grafana='{"kind":"OAuthRedirectReference","apiVersion":"v1","reference":{"kind":"Route","name":"sre-service-monitoring"}}' -n sre-monitoring
#oc annotate service grafana-service service.alpha.openshift.io/serving-cert-secret-name=grafana-tls -n sre-monitoring
```

## Deploy bookinfo and generate load

Follow instructions [here] to deploy OCP Service Mesh and bookinfo, the example app.
Follow the steps below to deploy locust, a load generator

```shell
oc new-project locust
export istio_gateway_url=$(oc get route istio-ingressgateway -n istio-system -o jsonpath='{.spec.host}')
oc create configmap locust-tasks --from-file=tasks.py=./locust/locustfile.py -n locust
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
export locust_chart_version=$(helm search repo locust | grep locust | awk '{print $2}')
helm fetch stable/locust --version ${locust_chart_version}
helm template locust-${locust_chart_version}.tgz --namespace locust --set master.config.target-host=http://$istio_gateway_url -f ./locust/values.yaml --name-template locust | oc apply -f -
rm locust-${locust_chart_version}.tgz
oc expose service locust-master-svc --port 8089 --name locust -n locust
```

## Start Swarming

```shell
export locust_url=$(oc get route -n locust locust -o jsonpath='{.spec.host}')
curl -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "locust_count=2&hatch_rate=2" http://${locust_url}/swarm
```

## Deploy Prometheus Operator

```shell
oc new-project sre-monitoring
oc apply -f prometheus-operator.yaml -n sre-monitoring
```

## Deploy Prometheus to watch Istio

```shell
```

## Deploy Prometheus Sre Prometheus

```shell
export token=$(oc get secret htpasswd -n istio-system -o jsonpath='{.data.rawPassword}' | base64 -d)
helm template prometheus-sre --namespace sre-monitoring --set prometheus.istio_endpoint.token=$token | oc apply -f -
```

## New Try for extensible istio monitoring

```shell
export istio_cp_namespace=istio-system
export deploy_namespace=sre-monitoring
oc new-project ${deploy_namespace}
oc patch ServiceMeshMemberRoll/default --type='json' -p='[{"op": "add", "path": "/spec/members/-", "value": "'${deploy_namespace}'" }]' -n ${istio_cp_namespace}
cat prometheus-operator.yaml | envsubst | oc apply -f - -n ${deploy_namespace}
export cert_chain_pem=$(oc get secret -n istio-system istio.default -o json | jq -r '.data["cert-chain.pem"]')
export key_pem=$(oc get secret -n istio-system istio.default -o json | jq -r '.data["key.pem"]')
export root_cert_pem=$(oc get secret -n istio-system istio.default -o json | jq -r '.data["root-cert.pem"]')
oc get ServiceMeshMemberRoll/default -n istio-system -o json | jq -r .spec | j2y > /tmp/members.yaml
helm template prometheus-sre --namespace ${deploy_namespace}  -f /tmp/members.yaml --set istio_control_plane_namespace=${istio_cp_namespace} --set istio_cert.cert_chain=${cert_chain_pem} --set istio_cert.key=${key_pem} --set istio_cert.root_cert=${root_cert_pem} | oc apply -f -
#wait a few minutes
oc patch statefulset/prometheus-sre-prometheus --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--discovery.member-roll-name=default" }]' -n ${deploy_namespace}
oc patch statefulset/prometheus-sre-prometheus --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--discovery.member-roll-namespace='${istio_cp_namespace}'" }]' -n ${deploy_namespace}
oc patch statefulset/prometheus-sre-prometheus --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/volumeMounts/-", "value":  { "name": "istio-certs", "mountPath": "/etc/istio-certs" }  }]' -n ${deploy_namespace}
```

error rate:

sum(rate(istio_requests_total{destination_service_namespace="bookinfo",destination_service="details.bookinfo.svc.cluster.local",response_code!~"5.*"}[5m]))/sum(rate(istio_requests_total{destination_service_namespace="bookinfo",destination_service="details.bookinfo.svc.cluster.local"}[5m]))


## Fault injection

```shell
oc apply -f failure-injection.yaml -n bookinfo
```