# Proof of Concept Demo

Andrew's part of the Proof of Concept Demo.

## Table of Contents

* [1: API for Platform Certificate Monitoring and Email Alerts](#one)
* [2: ](#two) 
* [3: ](#three)
* [4: ](#four)

<a name="one"/>

## 1: API for Platform Certificate Monitoring and Email Alerts

### 1.1: Prometheus HTTP API

[Prometheus](https://docs.openshift.com/container-platform/4.5/operators/operator_sdk/osdk-monitoring-prometheus.html) is used to gather information information from the platform.  These metrics can be queried through the [Prometheus UI](https://docs.openshift.com/container-platform/4.5/monitoring/cluster_monitoring/prometheus-alertmanager-and-grafana.html#monitoring-accessing-prometheus-alerting-ui-grafana-using-the-web-console_accessing-prometheus), or using the [Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/).

To access information about `KubeClientCertificateExperation`, first, forward the prometheus port:
```
oc -n openshift-monitoring port-forward svc/prometheus-operated 9090
```

The Prometheus query for this alert is:
```
apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0 and on(job) histogram_quantile(0.01, sum by(job, le) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))) < 5400
```

In order to use the HTTP API, you have to URL encode the above query, which looks like this:
```
apiserver_client_certificate_expiration_seconds_count%7Bjob%3D%22apiserver%22%7D+%3E+0+and+on%28job%29+histogram_quantile%280.01%2C+sum+by%28job%2C+le%29+%28rate%28apiserver_client_certificate_expiration_seconds_bucket%7Bjob%3D%22apiserver%22%7D%5B5m%5D%29%29%29+%3C+5400
```

You then run the query with the following URL:
```
curl -s http://localhost:9090/api/v1/query?query=<url encoded query>
```

For example:
```
curl -s http://localhost:9090/api/v1/query?query=apiserver_client_certificate_expiration_seconds_count%7Bjob%3D%22apiserver%22%7D+%3E+0+and+on%28job%29+histogram_quantile%280.01%2C+sum+by%28job%2C+le%29+%28rate%28apiserver_client_certificate_expiration_seconds_bucket%7Bjob%3D%22apiserver%22%7D%5B5m%5D%29%29%29+%3C+5400
```

### 1.2: Configuring Email Alerts

Prometheus also includes [Alertmanager]().  This allows you to trigger alerts based on Prometheus queries.  There are large number of queries that come out of the box with OpenShift, including `KubeClientCertficiateExpiration`.  Alertmanager supports certain **receivers**, such as *email* and *PagerDuty*, as well as a generic *REST API* option.

It is common for cluster administrators to setup Email receivers for alerts.

<a name="two"/>

## 2. Log Aggregation

OpenShift includes a cluster logging stack based on the popular *EFK* (Elasticsearch, FluentD, Kibana) stack.  This stack is optional, but fully supported.  It is [installed as a "Day 2" operation](https://docs.openshift.com/container-platform/4.5/logging/cluster-logging-deploying.html).

Cluster Logging is fully integrated with Role Based Access Control.  This means project logs are only accessible by the people who require access.

Log retention can be configured for the three main log types: `application`, `infra`, and `audit`.

An example configuration file might look like this:

```
apiVersion: "logging.openshift.io/v1"
kind: "ClusterLogging"
metadata:
  name: "instance" 
  namespace: "openshift-logging"
spec:
  managementState: "Managed"  
  logStore:
    type: "elasticsearch"  
    retentionPolicy: 
      application:
        maxAge: 1d
      infra:
        maxAge: 7d
      audit:
        maxAge: 7d
    elasticsearch:
      nodeCount: 3 
      storage:
        storageClassName: "<storage-class-name>" 
        size: 200G
      redundancyPolicy: "SingleRedundancy"
  visualization:
    type: "kibana"  
    kibana:
      replicas: 1
  curation:
    type: "curator"
    curator:
      schedule: "30 3 * * *" 
  collection:
    logs:
      type: "fluentd"  
      fluentd: {}
```

This allows the cluster administrator to configure retention periods, as well as the size of the log storage (200G in this example).

The `curator` schedule determines when the curator runs in order to delete expired logs.

<a name="three"/>

## 3: Section 3

<a name="four"/>

## 4: Section 4