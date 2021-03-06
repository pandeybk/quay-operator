# quay-operator

[![Build Status](https://github.com/redhat-cop/quay-operator/workflows/CI/CD/badge.svg?branch=master)](https://github.com/redhat-cop/quay-operator/actions?workflow=CI%2FCD)
 [![Docker Repository on Quay](https://quay.io/repository/redhat-cop/quay-operator/status "Docker Repository on Quay")](https://quay.io/repository/redhat-cop/quay-operator)

Operator to manage the lifecycle of [Quay](https://www.openshift.com/products/quay).

## Overview

This repository contains the functionality to provision a Quay ecosystem. Quay is supported by a number of other components, and thus a CustomResourceDefinition called `QuayEcosystem` is used to define the entire architecture. 

The following components are supported to be maintained by the Operator:

* Quay Enterprise
* Redis
* PostgreSQL

## Provisioning a Quay Ecosystem

### Deploy the Operator

The Quay Operator can be deployed on OpenShift or Kubernetes platforms. Quay recommends that it by deployed in a namespace called `quay-enterprise`, however support is available for deploying the operator to a namespace of your choosing. The steps below illustrate the steps necessary to deploy to an OpenShift or Kubernetes environment.

#### OpenShift Deployment

When choosing a namespace other than `quay-enterprise`, the _namespace_ field in the [deploy/cluster_role_binding.yaml](deploy/cluster_role_binding.yaml) must be updated with the new namespace otherwise permission issues will occur when deploying in OpenShift.

The steps described below assume the namespace that will be utilized is called `quay-enterprise`.

```
$ oc new-project quay-enterprise
```

Deploy the cluster resources. Given that a number of elevated permissions are required to resources at a cluster scope the account you are currently logged in must have elevated rights

```
$ oc create -f deploy/crds/redhatcop.redhat.io_quayecosystems_crd.yaml
$ oc create -f deploy/service_account.yaml
$ oc create -f deploy/cluster_role.yaml
$ oc create -f deploy/cluster_role_binding.yaml
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc create -f deploy/operator.yaml
```

**Note: When running on OpenShift 3.x, an alternate Custom Resource Definition (CRD) than the file specified MUST be used. Otherwise an OpenAPI error will be produced**

```
$ oc create -f deploy/crds/redhatcop.redhat.io_quayecosystems_crd-3.x.yaml
```

#### Kubernetes Deployment


The steps described below assume the namespace that will be utilized is called `quay-enterprise`.

```
$ kubectl create namespace quay-enterprise
```

Deploy the cluster resources. Given that a number of elevated permissions are required to resources at a cluster scope the account you are currently logged in must have elevated rights

```
$ kubectl -n quay-enterprise create -f deploy/crds/redhatcop.redhat.io_quayecosystems_crd.yaml
$ kubectl -n quay-enterprise create -f deploy/service_account.yaml
$ kubectl -n quay-enterprise create -f deploy/cluster_role.yaml
$ kubectl -n quay-enterprise create -f deploy/cluster_role_binding.yaml
$ kubectl -n quay-enterprise create -f deploy/role.yaml
$ kubectl -n quay-enterprise create -f deploy/role_binding.yaml
$ kubectl -n quay-enterprise create -f deploy/operator.yaml
```

#### Helm

The Quay Operator can also be deployed as a [Helm](https://helm.sh/) chart.

Create a new OpenShift project or Kubernetes Namespace

```
$ kubectl create namespace quay-enterprise
```

To install the chart run the following command:

Add the helm repository:

```bash
$ helm repo add quay-operator https://redhat-cop.github.io/quay-operator
```

Install the chart

```bash
$ helm install quay-operator quay-operator/quay-operator
```



### Deploy a Quay Ecosystem

Create a pull secret to retrieve Quay images from quay.io. If unsure what to use for the pull secret see [Accessing Red Hat Quay (formerly Quay Enterprise) without a CoreOS login](https://access.redhat.com/solutions/3533201).

```
$ oc create secret generic redhat-pull-secret --from-file=".dockerconfigjson=<location of docker.json file>" --type='kubernetes.io/dockerconfigjson'
```

Create a custom resource to deploy the Quay ecosystem. The following is an example of a `QuayEcosystem` custom resource to support a deployment of the Quay ecosystem

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
```

You can also run the following command to create the `QuayEnterprise` custom resource

```
$ oc create -f deploy/crds/redhatcop_v1alpha1_quayecosystem_cr.yaml
```

This will automatically create and configure Quay Enterprise, PostgreSQL and Redis. The credentials that can be used to access Quay Enterprise is described in the following section.

### Credentials

There are several locations within the Quay ecosystem of tools where sensitive information is stored/used. The Operator supports credentials either provided by the user as _Secrets_, or using defaults.

#### Quay Superuser

During the installation process, a superuser is created in Quay Enterprise with administrative rights. The credentials for this user can be specified or leverage the following default values:

Username: `quay` 
Password: `password`
Email: `quay@redhat.com`

To specify an alternate value, create a secret as shown below:

```
oc create secret generic <secret_name> --from-literal=superuser-username=<username> --from-literal=superuser-password=<password> --from-literal=superuser-email=<email>
```

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    superuserCredentialsSecretName: <secret_name>
    imagePullSecretName: redhat-pull-secret
```

#### Quay Configuration

A dedicated deployment of Quay Enterprise is used to manage the configuration of Quay. Access to the configuration interface is secured and requires authentication in order for access. By default, the following values are used:

Username: `quayconfig`
Password: `quay`

While the username cannot be changed, the password can be defined within a secret. To define a password for the Quay Configuration, execute the following command to create a secret:

```
oc create secret generic <secret_name> --from-literal=config-app-password=<password>
```

Reference the name of the secret in the _QuayEcosystem_ custom resource as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    configSecretName: <secret_name>
    imagePullSecretName: redhat-pull-secret
```

_Note: The superuser password must be at least eight characters._

### Persistent Storage using PostgreSQL

The PostgreSQL relational database is used as the persistent store for Quay Enterprise. PostgreSQL can either be deployed by the operator within the namespace or leverage an existing instance. The determination of whether to provision an instance or not within the current namespace depends on whether the `server` property within the `QuayEcosystem` is defined. 

The following options are a portion of the available options to configure the PostgreSQL database:

| Property | Description | 
| --------- | ---------- |
| `image` | Location of the database image |
| `volumeSize` | Size of the volume in Kubernetes capacity units |

Note: It is important to note that persistent storage for the database will **only** be provisioned if the `volumeSize` property is specified when provisioned by the operator.

Define the values as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    database:
      volumeSize: 10Gi
```


#### Specifying Credentials

The credentials for accessing the server can be specified through a _Secret_ or when being provisioned by the operator, leverage the following default values:

```
Username: `quay`
Password: `quay`
Root Password: `quayAdmin`
Database Name: `quay`
```

To define alternate values, create a secret as shown below:

```
oc create secret generic <secret_name> --from-literal=database-username=<username> --from-literal=database-password=<password> --from-literal=database-root-password=<root-password> --from-literal=database-name=<database-name>
```

Reference the name of the secret in the _QuayEcosystem_ custom resource as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    database:
      credentialsSecretName: <secret_name>
```

#### Using an Existing PostgreSQL Instance

Instead of having the operator deploy an instance of PostgreSQL in the project, an existing instance can be leveraged by specifying the location in the `server` field along with the credentials for access as described in the previous section. The following is an example of how to specify connecting to a remote PostgreSQL instance

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    database:
      credentialsSecretName: <secret_name>
      server: postgresql.databases.example.com
```

### Registry Backends

Quay supports multiple storage backends (configured as an array). The quay operator supports aiding in the facilitation of certain storage backends. The following backends are currently supported:

Please refer to the [Registry Storage Documentation](docs/storage.md) for the options available.

### Repository Mirroring

Quay provides the capability to create container image repositories that exactly match the content of external registries. This functionality can be enabled by setting the `enableRepoMirroring: true` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    enableRepoMirroring: true
```

The following additional options are also available:

* `repoMirrorTLSVerify` - Require HTTPS and verify certificates of Quay registry during mirror
* `repoMirrorServerHostname` - URL for use by the skopeo copy command
* `repoMirrorEnvVars` - Environment variables to be applied to the repository mirror container
* `repoMirrorResources` - Compute resources to be applied to the repository mirror container

### Configuration Files

Files related to the configuration of Quay are located in the `/conf/stack` directory. There are situations for which additional user defined configuration files need to be added to this directory (such as certificates and private keys). The Quay Operator supports the injection of these assets within the `configFiles` property in the `quay` property of the `QuayEcosystem` object where one or more assets can be specified.

Two types of configuration files can be specified by the `type` property:

1. `config` - Configuration files that will be added to the `/conf/stack` directory
2. `extraCaCerts` - Certificates to be trusted container

Configuration files are stored as values within _Secrets_. The first step is to create a secret containing these files. The following command illustrates how a private key can be added: 

```
oc create secret generic quayconfigfile --from-file=<path_to_file>
```

With the secret created, the secret containing the configuration file can be referenced in the `QuayEcosystem` object as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    configFiles:
      - secretName: quayconfigfile
  ```

By default, the `config` type is assumed. If the contents of the secret contains certificates that should be added to the `extra_ca_certs` directory, specify the type as `extraCaCert` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    configFiles:
      - secretName: quayconfigfile
        type: extraCaCert
```

Individual keys within a secret can be referenced to fine tune the resources that are added to the configuration using the `files` property as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    configFiles:
      - secretName: quayconfigfile
        files:
          - key: myprivatekey.pem
            filename: cloudfront.pem
          - key: myExtraCaCert.crt
            type: extraCaCert
```

The example above assumes that two files have been added to a secret called _quayconfigfile_. The file _myprivatekey.pem_ that was added to the secret will be mounted in the quay pod at the path `/conf/stack/cloudfront.pem` since it is a `config` file type and specifies a custom _filename_ that should be projected into the pod. The _myExtraCaCert.crt_ file will be added to the Quay pod within at the path `/conf/stack/extra_certs/myExtraCert.crt`

_Note:_ The `type` property within `files` property overrides the value in the `configFiles` property.

### Skipping Automated Setup

The operator by default is configured to complete the automated setup process for Quay. This can be bypassed by setting the `skipSetup` field to `true` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    skipSetup: true
```

### Methods for Exteral Access

Support is available to access Quay through a number of OpenShift and Kubernetes mechanisms for ingress. When running on OpenShift, a [Route](https://docs.openshift.com/container-platform/4.2/networking/routes/route-configuration.html) is used while a [LoadBalancer Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) and [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) is used. 

All of the properties for defining the configuration for external access can be managed within the `externalAccess` property

The type of external access can be specified by setting the `type` property within `externalAccess` using one of the available options in the table below:

| External Access Type | Description |  Notes |
| --------- | ---------- | ---------- |
| `Route` | [OpenShift Route](https://docs.openshift.com/container-platform/latest/networking/routes/route-configuration.html) | Can only be specified when running in OpenShift |
| `LoadBalancer` | [LoadBalancer Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) | |
| `NodePort` | [NodePort Service](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) | A dns based hostname or IP address **must** be specified using the `hostname` property of the `quay` resource |
| `Ingress` | [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) | Kubernetes native solution for external access | 

An example of how to specify the `type` is shown below

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      type: LoadBalancer
```

#### NodePorts

By default, `NodePort` type Services are allocated a randomly assigned network port between 30000-32767. To support a predictive allocation of resources, the `NodePort` services for Quay and Quay Config can be define using the `nodePort` and `configNodePort` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      type: NodePort
      nodePort: 30100
      configNodePort: 30101
```

#### Ingress

Ingress makes use of a similar concept as an OpenShift route, but requires a separate deployment of an ingress controller that manages external traffic. There are a variety of ingress controllers that can be used and implementation specific properties are typically defined through the use of annotations on the ingress resource.

The following is an example of how to define an _Ingress_ type of External Access using annotations specific for an Nginx controller:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      type: Ingress
      annotations:
        nginx.ingress.kubernetes.io/ssl-passthrough: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
```

Annotations can also be applied for the Config ingress by using the `configAnnotations` property

### Specifying the Quay Route

Quay makes use of an OpenShift route to enable ingress. The hostname for this route is automatically generated as per the configuration of the OpenShift cluster. Alternatively, the hostname for this route can be explicitly specified using the `hostname` property under the _externalAccess_ field as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      hostname: example-quayecosystem-quay-quay-enterprise.apps.openshift.example.com
```


### Specifying a Quay Configuration Route

During the development process, you may want to test the provisioning and setup of Quay Enterprise server. By default, the operator will use the internal service to communicate with the configuration pod. However, when running external to the cluster, you will need to specify the ingress location for which the setup process can use.

Specify the `configHostname` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      configHostname: example-quayecosystem-quay-config-quay-enterprise.apps.openshift.example.com
```

### SSL Certificates

Quay, as a secure registry, makes use of SSL certificates to secure communication between the various components within the ecosystem. Transport to the Quay user interface and container registry is secured via SSL certificates. These certificates are generated at startup with the OpenShift route being configured with a TLS termination type of _Passthrough_.

#### User Provided Certificates

SSL certificates can be provided and used instead of having the operator generate certificates. Certificates can be provided in a secret which is then referenced in the _QuayEcosystem_ custom resource. 

The secret containing custom certificates must define the following keys:

* `tls.cert` -  All of the certificates (root, intermediate, certificate) concatinated into a single file
* `tls.key` - Private key as for the SSL certificate

Create a secret containing the certificate and private key

```
oc create secret tls custom-quay-ssl --key=tls.key=<ssl_private_key> --cert=tls.cert=<ssl_certificate>
```

The secret containing the certificates are referenced using the `secretName` underneath a property called `tls` as defined within the `externalAccess` proprety as shown below

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      tls:
        secretName: custom-quay-ssl
```

### TLS Termination

Quay can be configured to protect connections using SSL certificates. By default, SSL communicated is termminated within Quay. There are several different ways that SSL termination can be configured including omitting the use of certificates altogeter. TLS termination is determined by the `termination` property as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    externalAccess:
      tls:
        termination: passthrough
```

The example above is the default configuration applied to Quay. Alternate options are available as described in the table below:

| TLS Termination Type | Description |  Notes |
| --------- | ---------- | ---------- |
| `passthrough` | SSL communication is terminated at Quay | Default configuration |
| `edge` | SSL commmunication is terminated prior to reaching Quay. Traffic reaching quay is not encrypted (HTTP) |
| `none` | All communication is unencrypted |


### Configuration Deployment After Initial Setup

In order to conserve resources, the configuration deployment of Quay is removed after the initial setup. In certain cases, there may be a need to further configure the quay environment. To specify that the configuration deployment should be retained, the `keepConfigDeployment` property within the _Quay_ object can can be set as `true` as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    keepConfigDeployment: true
```

### Redis Password

By default, the operator managed Redis instance is deployed without a password. A password can be specified by creating a secret containing the password in the key _password_. The following command can be used to create the secret:

```
oc create secret generic <secret_name> --from-literal=password=<password>
```

The secret can then be specified within the _redis_ section using the `` as shown below:


```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  redis:
    credentialsSecretName: <secret_name>
    imagePullSecretName: redhat-pull-secret
```

### Clair

[Clair](https://github.com/coreos/clair) is a vulnerability assessment tool for application container. Support is available to automatically provision and configure both Clair and the integration wtih Quay. A property called `clair` can be specified in the `QuayEcosystem` object along with `enabled: true` within this field in order to deploy Clair. An example is shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
  clair:
    enabled: true
    imagePullSecretName: redhat-pull-secret
```

#### Update Interval

Clair routinely queries CVE databases in order to build its own internal database. By default, this value is set at 500m. You can modify the time interval between checks by setting the `updateInterval` property as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
  clair:
    enabled: true
    imagePullSecretName: redhat-pull-secret
    updateInterval: "60m"
```

The above configuration would have Clair update every 60 minutes.

## Common Attributes

Each of the following components expose a set of similar properties that can be specified in order to customize the runtime execution:

* Quay
* Quay Configuration
* Quay PostgreSQL
* Redis
* Clair
* Clair PostgreSQL

### Image Pull Secret

As referenced in prior sections, an Image Pull Secret can specify the name of the secret containing credentials to an image from a protected registry using the property `imagePullSecret`.

### Image

There may be a desire to make use of an alternate image or source location for each of the components in the Quay ecosystem. The most common use case is to to make use of an image registry that contains all of the needed images instead of being sourced from the public internet. Each component has a property called `image` where the location of the related image can be referenced from.

The following is an example of how a customized image location can be specified:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    image: myregistry.example.com/quay/quay:v3.2.0
```

### Compute Resources

[Compute Resources](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container) such as memory and CPU can be specified in the same form as any other value in a `PodTemplate`. CPU and Memory values for _Requests_ and _Limits_ can be specified under a property called `resources`.

_Note:_ In the case of the QuayConfiguration deployment, `configResources` is the property which should be referenced underneath the `Quay` property.

The following is an example of how compute resources can be specified:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    resources:
      requests:
        memory: 512Mi
```

### Probes

[Readiness and Liveness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/) can be specified in the same form as any other value in a `PodTemplate`. 

The following is how a _readinessProbe_ and _livenessProbe_ can be specified:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    livenessProbe:
      initialDelaySeconds: 120
      httpGet:
        path: /health/instance
        port: 8443
        scheme: HTTPS
    readinessProbe:
      initialDelaySeconds: 10
      httpGet:
        path: /health/instance
        port: 8443
        scheme: HTTPS
```

_Note_: If a value for either property is not specified, an opinionated default value is applied.

### Node Selector

It may be desired that components of the `QuayEcosystem` may need to be deployed to only a subset of available nodes in a Kubernetes cluster. This functionality can be set on each of the resources using the `nodeSelector` property as show below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    nodeSelector:
      node-role.kubernetes.io/infra: true
```

### Deployment Strategy

Each of the core components consist of Kubernetes `Deployments`. This resource supports the method in which new versions are released. This operator supports making use of the `RollingUpdate` and `Recreate` strategies. Either value can be defined by using the `deploymentStrategy` property on the desired resource as shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    deploymentStrategy: RollingUpdate
```

_Note: The absence of a defined value will make use of the `RollingUpdate` strategy_

### Environment Variables

In addition to environment variables that are automatically configured by the operator, users can define their own set of environment variables in order to customize the managed resources. Each core component includes a property called `envVars` where environment variables can be defined. An example is shown below:

```
apiVersion: redhatcop.redhat.io/v1alpha1
kind: QuayEcosystem
metadata:
  name: example-quayecosystem
spec:
  quay:
    imagePullSecretName: redhat-pull-secret
    envVars:
      - name: FOO
        value: bar
```

_Note_: Environment variables for the Quay configuration pod can be managed by specifying the `configEnvVars` property on the `quay` resource

_Caution:_ User defined environment variables are given precedence over those managed by the operator. Undesirable results may occur if conflicting keys are used.

## Troubleshooting

To resolve issues running, configuring and utilzing the operator, the following steps may be utilized:

### Errors during initial setup

The _QuayEcosystem_ custom resource will attempt to provide the progress of the status of the deployment and configuration of Quay. Additional information related to any errors in the setup process can be found by viewing the log messages of the _config_ pod as shown below:

```
oc logs $(oc get pods -l=quay-enterprise-component=config -o name)
```

## Local Development

Execute the following steps to develop the functionality locally. It is recommended that development be done using a cluster with `cluster-admin` permissions. 

Clone the repository, then resolve all dependencies using `go mod`

```
$ export GO111MODULE=on
$ go mod vendor
```

Using the [operator-sdk](https://github.com/operator-framework/operator-sdk), run the operator locally

```
$ operator-sdk up local --namespace=quay-enterprise
```

## Upgrading Quay & Clair

Execute the following steps to upgrade an existing deployment to a new version without upgrading the operator:

```
oc edit quayecosystem/quayecosystem
```

Find and update the following entries:

```
image: quay.io/redhat/clair-jwt:vX.X.X
image: quay.io/redhat/quay:vX.X.X
```

Once saved, the operator will automatically apply the upgrade.

_Note_: If you used a different name than `QuayEcosystem` for the custom resource to deploy your Quay ecosystem, you will have to replace the name to fit the proper value
