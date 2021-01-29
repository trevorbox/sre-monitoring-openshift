# Deploy managed Grafana for openshift-monitoring

Run this as a cluster admin.

## Setup

```sh
export deploy_namespace=openshift-monitoring-ext
```

Optional - add network policies and quota to mimic production env

```sh
helm upgrade -i --create-namespace admin helm/admin -n ${deploy_namespace}
```

## Deploy operator

```sh
helm upgrade -i --create-namespace grafana-operator helm/operator -n ${deploy_namespace}
```

> Note: you need to manually approve the InstallPlan to install the grafana-operator

## Deploy rbac to allow csv maintenance for developer01 user (changing image)

> Note: this is just an example, you would normally want to map the rolebinding to a group
> Additionally, a ClusterRole could also be used instead of a Role

```sh
helm upgrade -i csv-maintainer helm/rbac -n ${deploy_namespace}
```

## Optional - update dashboards for your OCP version

This will update the json files within helm/grafana/dashboards/openshift-monitoring.

```sh
./export-openshift-monitoring-dashboards.sh
```

## Deploy grafana

```sh
helm upgrade -i --create-namespace grafana helm/grafana -n ${deploy_namespace} --set grafana.datasources.prometheus.openshift_monitoring.password=$(oc extract secret/grafana-datasources -n openshift-monitoring --keys=prometheus.yaml --to=- | grep -zoP '"basicAuthPassword":\s*"\K[^\s,]*(?=\s*",)')
```

## Chargeback testing

This example creates a new metric for aggregating container memory usage by namespaces with matching `example.com/owner-number` labels.

Add the cluster-monitoring label to the namespace to allow openshift-monitoring to consume the prometheus rule.

```sh
oc label namespace ${deploy_namespace} openshift.io/cluster-monitoring='true'
```

Additionally you will need to label namespaces with `example.com/owner-number` to aggregate upon.

For Example:

```sh
oc label namespace ${deploy_namespace} example.com/owner-number='0000'
oc label namespace default example.com/owner-number='0000'
```

Deploy PrometheusRule...

```sh
helm upgrade -i chargeback helm/prometheus -n ${deploy_namespace}
```

If there is load on the namespaces, you should be able to compute averages using the custom metric.

```promql
avg_over_time(label_example_com_owner_number:container_memory_working_set_bytes:sum[24h])
```

After this is complete the `Chargeback` dashboard should also work in Grafana.

> Note: if you change the label be sure to update the chargeback.json dashboard expression as well.
