# c-logging
![Version 1.9.0](https://img.shields.io/badge/version-1.9.0-blue)
![Made with love in squad/logging](https://madewithlove.now.sh/af?heart=true&colorB=%2338dcb3&text=squad%2Flogging)
- [c-logging](#c-logging)
  - [Scope](#scope)
  - [Architecture](#architecture)
  - [Repository structure](#repository-structure)
  - [Components](#components)
    - [List of components](#list-of-components)
    - [Resources](#resources)
    - [Useful/official links](#usefulofficial-links)
  - [Compatibility Matrix](#compatibility-matrix)
  - [Stack configuration](#stack-configuration)
    - [Authentication](#authentication)
      - [elastic](#elastic)
      - [logstash](#logstash)
      - [filebeat](#filebeat)
      - [kibana](#kibana)
    - [Services exposed](#services-exposed)
    - [Logging and Dashboards](#logging-and-dashboards)
    - [Logstash configuration](#prometheus-rules)
    - [Monitoring](#monitoring)
    - [Backup](#backup)
  - [Prerequisites](#prerequisites)
  - [Usage procedures](#usage-procedures)
  - [Operational procedures](#operational-procedures)
  - [Known issues](#known-issues)

---

## Scope

- Authentication integrated in OpenShift and Kubernetes for all exposed dashboards.
- Client authentication supported.
- Save logs and make them accessible for the client via Kibana labeling the namespaces with : `c-logging/enabled: "true"`
- We do NOT share env logs with the client (delegated in `d-logging`)
- Elastic logs retention is set to 7 days by default.
- Use standard policy to backup PVs (90 days of retention)
- Capability to add kibana dashboards.
- Multizone support with 3+ Elasticsearch instances (each one in different zones).
- Capability to send logs to an external SIEM following [this guide](./doc/log-forwarding-SIEM.md)
## Architecture

![architecture](./doc/img/c-logging-LLD.png)

The diagram show the relationships between each component of the logging stack. It applies both Kubernetes and OpenShift.

The differences between Kubernetes and Openshift are:

- Ingress access to dashboards:
  - *Kubernetes*: the solution use `d-ingress` component.
  - *OpensShift*: `openshift-default-router` provides the access to external users.
- Authentication to dashboards:
  - *Kubernetes*:
    - *Vanilla/IKS*: `d-auth` is used to authenticate ISCP users and also client users (needs client IdP configured in `d-auth`)
  - *OpensShift*:
    - *ROKS*: the authentication will be provided by IBM Cloud IAM for all kind of users (ISCP and client)
    - *Managed OpenShift (onpremise)*: `d-auth` for OCP will be used to authenticate ISCP users (ISCP-SSO) and the clients will the client IdP configured in oauth (e.g. LDAP)
## Repository structure

<details>
  <summary>Show full directory structure</summary>

```bash
├── k8s
│   ├── default                                --> Default folder for OpenShift deployments without oauth
│   │   ├── crds                               --> Mandatory CRDs to deploy c-logging stack
│   │   ├── c-logging      
|   │   │   ├── backup                         --> K8s sample files with 2 ways to execute the backup (PV or S3).
│   │   │   ├── ilm-slm-policies               --> This Jobs for ilm and slm replace curator 
│   │   │   ├── eck-operator                   --> ECK Operator
│   │   │   ├── elasticsearch                  --> Elastisearch component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── elasticsearch-exporter         --> Elasticsearch-exporter component
│   │   │   ├── filebeat                       --> Filebeat component
│   │   │   ├── kibana                         --> K8s files to deploy kibana in `vanilla` or `iks` clusters
|   │   |   ├── kibana-index-pattern-creation  --> Job for index pattern creation
│   │   │   ├── logstash                       --> Logstash component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── monitoring                     --> All monitoring objects of all component in the stack.
│   │   ├── ns                                 --> Namespace
│   │   ├── ns-config                          --> Docker pull secrets
│   │   └── storage                            --> K8s sample files with PV and PVCs of all component in the stack.
│   ├── oauth                                  --> Folder for OpenShift deployments with oauth
│   │   ├── crds                               --> Mandatory CRDs to deploy c-logging stack
│   │   ├── c-logging      
|   │   │   ├── backup                         --> K8s sample files with 2 ways to execute the backup (PV or S3).
│   │   │   ├── ilm-slm-policies               --> This Jobs for ilm and slm replace curator 
│   │   │   ├── eck-operator                   --> ECK Operator
│   │   │   ├── elasticsearch                  --> Elastisearch component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── elasticsearch-exporter         --> Elasticsearch-exporter component
│   │   │   ├── filebeat                       --> Filebeat component
│   │   │   ├── kibana                         --> K8s files to deploy kibana in `vanilla` or `iks` clusters
|   │   |   ├── kibana-index-pattern-creation  --> Job for index pattern creation
│   │   │   ├── logstash                       --> Logstash component with k8s sample files to deploy in single o multizone clusters.
│   │   │   └── monitoring                     --> All monitoring objects of all component in the stack.   
│   │   ├── ns                                 --> Namespace
│   │   ├── ns-config                          --> Docker pull secrets
│   │   └── storage                            --> K8s sample files with PV and PVCs of all component in the stack.
│      
│   
| 
├── ocp
│   ├── default                                --> Default folder for OpenShift deployments without oauth
│   │   ├── crds                               --> Mandatory CRDs to deploy c-logging stack
│   │   ├── c-logging      
|   │   │   ├── backup                         --> K8s sample files with 2 ways to execute the backup (PV or S3).
│   │   │   ├── ilm-slm-policies               --> This Jobs for ilm and slm replace curator 
│   │   │   ├── eck-operator                   --> ECK Operator
│   │   │   ├── elasticsearch                  --> Elastisearch component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── elasticsearch-exporter         --> Elasticsearch-exporter component
│   │   │   ├── filebeat                       --> Filebeat component
│   │   │   ├── kibana                         --> K8s files to deploy kibana in `ocp` or `roks` clusters
|   │   |   ├── kibana-index-pattern-creation  --> Job for index pattern creation
│   │   │   ├── logstash                       --> Logstash component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── monitoring                     --> All monitoring objects of all component in the stack.
│   │   ├── ns                                 --> Namespace
│   │   ├── ns-config                          --> Docker pull secrets
│   │   └── storage                            --> K8s sample files with PV and PVCs of all component in the stack.
│   ├── oauth                                  --> Folder for OpenShift deployments with oauth
│   │   ├── crds                               --> Mandatory CRDs to deploy c-logging stack
│   │   ├── c-logging      
|   │   │   ├── backup                         --> K8s sample files with 2 ways to execute the backup (PV or S3).
│   │   │   ├── ilm-slm-policies               --> This Jobs for ilm and slm replace curator 
│   │   │   ├── eck-operator                   --> ECK Operator
│   │   │   ├── elasticsearch                  --> Elastisearch component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── elasticsearch-exporter         --> Elasticsearch-exporter component
│   │   │   ├── filebeat                       --> Filebeat component
│   │   │   ├── kibana                         --> K8s files to deploy kibana in `ocp` or `roks` clusters
|   │   |   ├── kibana-index-pattern-creation  --> Job for index pattern creation
│   │   │   ├── logstash                       --> Logstash component with k8s sample files to deploy in single o multizone clusters.
│   │   │   ├── monitoring                     --> All monitoring objects of all component in the stack.
│   │   ├── ns                                 --> Namespace
│   │   ├── ns-config                          --> Docker pull secrets
│   │   └── storage                            --> K8s sample files with PV and PVCs of all component in the stack.
|
├── doc
│   ├── files                                  --> Source documentation files
│   ├── img                                    --> Images to be used in README  
└── misc    
    ├── img                                    --> ArgoCD files, not used on this release, but needed in the future 
    └── scripts                                --> Scripts to generate k8s yaml files from Helm charts used in this repo and Chart values
```

</details>

## Components

### List of components

* [eck-operator](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-operating-eck.html) - (Elastic Cloud on Kubernetes simplifies setup, upgrades, snapshots, scaling, high availability, security, and more for running Elasticsearch and Kibana in Kubernetes.)
* [elasticsearch](https://www.elastic.co/elasticsearch/) - (Elasticsearch is a distributed, RESTful search and analytics engine capable of addressing a growing number of use cases.)
* [elasticsearch-exporter](https://github.com/justwatchcom/elasticsearch_exporter) - (Prometheus exporter for various metrics about ElasticSearch, written in Go.)
* [kibana](https://www.elastic.co/kibana) - (Kibana is a free and open user interface that lets you visualize your Elasticsearch data and navigate the Elastic Stack)
* [logstash](https://www.elastic.co/logstash) - (Logstash is a free and open server-side data processing pipeline that ingests data from a multitude of sources, transforms it, and then sends it )
* [logstash-exporter](https://github.com/bitnami/bitnami-docker-logstash-exporter) - (Prometheus exporter for the metrics available in Logstash).
* [filebeat](https://www.elastic.co/beats/filebeat) - (Filebeat helps you keep the simple things simple by offering a lightweight way to forward and centralize logs and files)
* [busybox](https://busybox.net) - (BusyBox combines tiny versions of many common UNIX utilities into a single small executable)

### Resources

| Component                        | CPU req | CPU Limit | Mem req. | Mem limit |
|:--------------------------------:|:-------:|:---------:|:--------:|:---------:|
| c-logging-elasticsearch-es-node  | 500m    | 1         | 4Gi      | 6Gi       |
| c-logging-elasticsearch-exporter | 50m     | 100m      | 64Mi     | 128Mi     |
| c-logging-filebeat-beat-filebeat | 100m    | 200m      | 100Mi    | 250Mi     |
| c-logging-filebeat-beat-exporter | 50m     | 100m      | 32Mi     | 32Mi      |
| c-logging-kibana-kb              | 500m    | 1         | 1Gi      | 2Gi       |
| c-logging-logstash               | 500m    | 1         | 1Gi      | 2Gi       |
| c-logging-operator               | 100m    | 1         | 150Mi    | 512Mi     |

### Useful/official links

* [elastic-cloud-2.12.0](https://www.elastic.co/guide/en/cloud-on-k8s/2.12/index.html) - Overview of elastic cloud on kubernetes
* [docker-images-8.12.2](https://www.docker.elastic.co/) - Docker images we used to the upgrade for the new images.

### Compatibility Matrix

This unit has been tested in:


    | Component    | Platform |   Tested versions                          |
    |:------------:|:--------:|:------------------------------------------:|
    | c-logging    | IKS      | <=1.30.x  :heavy_check_mark:               |
    | c-logging    | IKS MZ   | <=1.30.x  :heavy_check_mark:               |
    | c-logging    | Vanilla  | <=1.30.x  :heavy_check_mark:               |
    | c-logging    | ROKS     | <=4.17    :heavy_check_mark:               |
    | c-logging    | ROKS     | <=4.17    :heavy_check_mark:               |

## Stack configuration

### Authentication

- For Openshift onpremise, all dashboards has enabled by default `d-auth` authentication through ISCP SSO and other IDPs configured in `d-auth`
- For IKS / Kubernetes vanilla, all dashboards has enabled by default `d-auth` authentication through ISCP SSO and other IDPs configured in `d-auth`. We need an **Oauthserver** (Dex) that it was developed to interact with differents idp "connectors" and serve to apps to connect via proxy to those idps.
- For ROKS, the authentication will be manage by IBM Cloud IAM


About the tool enabled by default `d-auth` is explained here in the new [documentation](https://pages.github.kyndryl.net/iscp-global/doc/operational-guides/secrets-management/secure-apps/?h=) for add the oauth proxy for Kibana.


> xpack.security.enabled must be disable in Elasticsearch, in Kibana is complemented in >8.0



#### Kibana 

| Provider  |           Platform            | Authentication method                      |
|:---------:|:-----------------------------:|--------------------------------------------|
| IBM Cloud | Kubernetes self-managed (IKS) | - *d-auth** (ISCP-SSO, client LDAP...)     |
| IBM Cloud | OpenShift self-managed (ROKS) | - *IAM IBM Cloud**                         |
|  vSphere  |      Kubernetes managed       | - *d-auth** (ISCP-SSO, client LDAP...)     |
|  vSphere  |       OpenShift managed       | - *oauth-proxy* (ISCP-SSO, client LDAP...) |


`*` Default authentication used by c-logging

### Services exposed


- Kibana UI through `d-ingress`/Default OpenShift Router

* **K8s**:
  * Kibana through `d-ingress` with a NGINX Ingress Controller.
  ```
  c-logging-kibana.<targetCluster>.{{ global.mt.instanceSubDomain }}.{{ global.mt.instanceDomain }}
  ```
  where `{{ global.mt.instanceSubDomain }}{{ global.mt.instanceDomain }}` is `pro.eu.d-iscp.net` Example: 
  ```
  c-logging-kibana.oauth.ibc-pro-tst-0001-k01.pro.eu.d-iscp.net
  ```

* **Openshift**:
  * Kibana through routes, components of `default OpenShift Route`. 
  ```
   c-logging-kibana.oauth.apps.< subDomain of default OpenShift Router > 
  ```
  To find the correct subdomain of default Openshift Route, we can check in this way:
  ```
    $ oc get ingresscontroller -n openshift-ingress-operator default -o yaml | grep -i "domain"
    domain: apps.cus01-pro-001-k01.example.io
  ```
  Example:
  ```
  c-logging-kibana.oauth.apps.cus01-pro-001-k01.example.io
  ```

### Logging and dashboards

* Standard solution based on namespaces
* Custom dashboards:

| Dashboard name                    | Description                            |
|:---------------------------------:|:--------------------------------------:|
| c-logging-elasticsearch-dashboard | Dashboard to view elasticsearch health |
| c-logging-logstash-dashboard      | Logstash dashboard                     |
| c-logging-filebeat-dashboard      | Filebeat dashboard                     |

### Backup

* Backups are made by elastic internally. Logging squad configure elastic with two job that configure the delete of index and the backup of those
  
  * [c-logging-ilm](./k8s/default/c-logging/ilm-slm-policies/c-logging-ilm-job.yaml) runs to configure the delete of index that have more than a week (7d)
  
  * [c-logging-slm](./k8s/default/c-logging/ilm-slm-policies/c-logging-slm-job.yaml) runs to configure a nigthly snapshot that save index with a desire retention from the cicd variable {{ snapshotRetention }}

It's important once you install or upgrade the logging stack to ensure that both job are completed succssefully 

Also with S3 variables you can configure to save the backups on S3 bucket

You can check if the nightly-snapshots job are correctly configured on kibana by checking:

![backup-configure](./doc/img/c-logging-backup-configure.png)

## Prerequisites

Follow [this guide](./doc/prerequisites.md) to verify all the prerequisites before the installation of *c-logging* in any supported container platforms.

## Usage procedures


For Kubernetes, follow this [this guide](./doc/usage-k8s.md) to check the procedures to install/update *c-logging*.

For Openshift, follow this [this guide](./doc/usage-oc.md) to check the procedures to install/update *c-logging*.

## Operational procedures

Follow [this guide](./doc/procedures.md) to check the common operational procedures.

Send logs to an external SIEM following [this guide](./doc/log-forwarding-SIEM.md).


## Known issues
