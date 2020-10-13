# Proof of Concept Demo

Andrew's part of the Proof of Concept Demo.

## Table of Contents

* [A-03: API for Platform Certificate Monitoring and Email Alerts](#a-03)
* [A-04: Monitoring and Dashboards](#a-04)
* [LG-06: Log Aggregation and Management](#lg-06) 
* [UH-01: Concurrent Modification - Git Feature Branch Workflow](#uh-01)
* [Demo 10: Platform Certificates](#demo-10)
* [Demo 11: Active Directory (LDAP) Group and User Management](#demo-11)


<a name="a-03"/>

## A-03: API for Platform Certificate Monitoring and Email Alerts

### Prometheus HTTP API

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

### Configuring Email Alerts

Prometheus also includes [Alertmanager]().  This allows you to trigger alerts based on Prometheus queries.  There are large number of queries that come out of the box with OpenShift, including `KubeClientCertficiateExpiration`.  Alertmanager supports certain **receivers**, such as *email* and *PagerDuty*, as well as a generic *REST API* option.

It is common for cluster administrators to setup Email receivers for alerts.

The default Alertmanager config looks like this:

```
"global":
  "resolve_timeout": "5m"
"inhibit_rules":
- "equal":
  - "namespace"
  - "alertname"
  "source_match":
    "severity": "critical"
  "target_match_re":
    "severity": "warning|info"
- "equal":
  - "namespace"
  - "alertname"
  "source_match":
    "severity": "warning"
  "target_match_re":
    "severity": "info"
"receivers":
- "name": "Default"
- "name": "Watchdog"
- "name": "Critical"
"route":
  "group_by":
  - "namespace"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "Default"
  "repeat_interval": "12h"
  "routes":
  - "match":
      "alertname": "Watchdog"
    "receiver": "Watchdog"
  - "match":
      "severity": "critical"
    "receiver": "Critical"
```

We can add email to the list of "receivers":

```
receivers:
- name: email
  email_configs:
  - to: $EMAIL_ACCOUNT
    from: $EMAIL_ACCOUNT
    smarthost: smtp.example.com:587
    auth_username: "$EMAIL_ACCOUNT"
    auth_identity: "$EMAIL_ACCOUNT"
    auth_password: "$EMAIL_AUTH_TOKEN"
```

You can then reference this email receiver to have alerts sent by email.

<a name="a-04"/>

## A-04: Monitoring and Dashboards

Demo: Seating Manifest

<a name="lg-06"/>

## LG-06. Log Aggregation and Management

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

<a name="uh-01"/>

## UH-01: Concurrent Modification - Git Feature Branch Workflow

Red Hat Fuse is based on Apache Camel.  As such, integrations are written in Java code or an XML Domain Specific Language (DSL).  

As such, the best way to ensure there is no concurrent modification of integration code is through proper Git workflows, such as the [Feature Branch Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/feature-branch-workflow) along with policy enforcement with your git repository management system.

With this workflow, each feature (or bug fix) is done in its own branch off of `master`.  There are no commits directly to `master`. This should be enforced through your git repository management system.

Once a developer has completed their task, they will open a *Pull Request* to have their code merged into `master`.  Normally, this would also require another resource to *review* the pull request.

If another developer has since merged their own code into `master` which would cause a conflict, this will be reflected in the pull request.  This conflict will need too be fixed on the branch before it can be merged into master.

<a name="demo-10"/>

## Demo 10: Platform Certificates

From a platform perspective, it is easy to replace the certificates for the OpenShift API and Router with your own certificates.  This will be demonstrated with Let's Encrypt.

To demonstrate a Kubernetes-native way of updating the platform certificates, we will use a Kubernetes `Secret` and `Job` to generate new certificates and apply them.

The `Job` that will be used can be found [in this GitHub repository](https://github.com/pittar/ocp-letsencrypt-job).

The main workflow here is:
1. Determine the *wildacard* and *api* OpenShift URLs.
2. Use the `acme.sh` script to generage new Let's Encrypt certificates.
3. Createt a `tls secret` in the `openshift-ingress` namespace containing the new certificates.
4. Patch the `ingresscontroller` to reference the new certificate secret.

Once this is done, the OpenShift Router and API Server will restart and begin using the new certificates.

<a name="demo-11"/>

## Demo 11: Active Directory (LDAP) Group and User Management

OpenShift can be configured to use Active Directory/LDAP for authentication.  When Active Directory or LDAP is used, admins also have the ability to sync user and group information to the OpenShift cluster.

For this demo, we will use a cloud-based LDAP (JumpCloud) as our LDAP server.  Active Directory works similarly, but requires some minor changes to the configuration.  For the purpose of this demo, LDAP will suffice.

### LDAP / Active Directory Authentication

The configuation resides in an `OAuth` resource.  For example:

```
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: htpasswd-secret
    mappingMethod: claim
    name: htpasswd_provider
    type: HTPasswd
  - name: ldap
    challenge: false
    login: true
    mappingMethod: claim
    type: LDAP
    ldap:
      attributes:
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "uid=ldapuser,ou=Users,o=5ee27e58ad19513fdf256a4b,dc=jumpcloud,dc=com"
      bindPassword:
        name: ldap-secret
      ca:
        name: ca-config-map
      insecure: false
      url: "ldaps://ldap.jumpcloud.com/ou=Users,o=5ee27e58ad19513fdf256a4b,dc=jumpcloud,dc=com?uid"
  tokenConfig:
    accessTokenMaxAgeSeconds: 86400
```

This resource describes two valid authentication methods: `HTPasswd` and `LDAP`

With this configuration in place, a user would be able to login with their LDAP (or Active Directory) credentials.

### LDAP / Active Directory Group Sync

Group Sync is accomplished with another command.  For this, you provide the details of the LDAP as well as the LDAP queries to use to fetch user and group information.  The result is then used to create groups in OpenShift and associate the users from the LDAP/AD.

An example Group Sync custom resource might look like this:

```
kind: LDAPSyncConfig
apiVersion: v1
url: ldaps://ldap.jumpcloud.com
bindDN: uid=ldapadmin,ou=Users,o=5ee27e58ad19513fdf256a4b,dc=jumpcloud,dc=com
bindPassword: <bind password>
rfc2307:
  groupsQuery:
    baseDN: ou=Users,o=5ee27e58ad19513fdf256a4b,dc=jumpcloud,dc=com
    derefAliases: never
    filter: '(|(cn=ocp-*))'
  groupUIDAttribute: dn
  groupNameAttributes:
  - cn
  groupMembershipAttributes:
  - member
  usersQuery:
    baseDN: ou=Users,o=5ee27e58ad19513fdf256a4b,dc=jumpcloud,dc=com
    derefAliases: never
  userUIDAttribute: dn
  userNameAttributes:
  - uid
```

This describes:
* The LDAP to connect to, as well as bind information.
* Groups query and filter
* Users query
* Attributes to use for UID and user names

To sync user and group information to OpenShift, you then run the following command:

```
oc adm groups sync --sync-config=groupsync.yaml --confirm
```

The result is a list of groups that were created.

If you want to see the group membership, you can list the groups:

```
oc get groups
```

### Mapping Permissions to Roles

Now that you have users and groups, you will need to give the appropriate rights to each group!

You can make the `ocp-admins` group `cluster-admin` with the following command:

```
oc adm policy add-cluster-role-to-group cluster-admin ocp-admins
```

For "support" to be able to "view" all projects (read-only):

```
oc adm policy add-cluster-role-to-group cluster-reader ocp-support
```

And to give Developers access to the "Seating" project:

```
oc adm policy add-role-to-group admin ocp-developers -n seating
```