# Elastisys Compliant Kubernetes Apps

## Build status

![End-to-end tests](https://github.com/elastisys/compliantkubernetes-apps/actions/workflows/end-to-end.yml/badge.svg)

## Overview

This repository is part of the [Compliant Kubernetes][compliantkubernetes] (compliantkubernetes) platform.
The platform consists of the following repositories:

* [compliantkubernetes-kubespray][compliantkubernetes-kubespray] - Code for managing Kubernetes clusters and the infrastructure around them.
* [compliantkubernetes-apps][compliantkubernetes-apps] - Code, configuration and tools for running various services and applications on top of Kubernetes clusters.

The Elastisys Compliant Kubernetes (compliantkubernetes) platform runs two Kubernetes clusters.
One called "service" and one called "workload".

The _service cluster_ provides observability, log aggregation, private container registry with vulnerability scanning and authentication using the following services:

* Prometheus and Grafana
* OpenSearch and OpenSearch Dashboards
* Harbor
* Dex

The _workload cluster_ manages the user applications as well as providing intrusion detection, security policies, log forwarding and monitoring using the following services:

* Falco
* Open Policy Agent
* Fluentd
* Prometheus

[compliantkubernetes]: https://compliantkubernetes.com/
[compliantkubernetes-kubespray]: https://github.com/elastisys/compliantkubernetes-kubespray
[compliantkubernetes-apps]: https://github.com/elastisys/compliantkubernetes-apps

This repository installs all the applications of ck8s on top of already created clusters.
To setup the clusters see [compliantkubernetes-kubespray][compliantkubernetes-kubespray].
A service-cluster (sc) or workload-cluster (wc) can be created separately but all of the applications will not work correctly unless both are running.

All config files will be located under `CK8S_CONFIG_PATH`.
There will be four config files: `common-config.yaml`, `wc-config.yaml`, `sc-config.yaml` and `secrets.yaml`.
See [Quickstart](#Quickstart) for instructions on how to initialize the repo

### Cloud providers

Currently we support four cloud providers: Exoscale, Safespring, Citycloud, Elastx and AWS (beta). In addition to this we support running Compliant Kubernetes on bare metal (beta).

## Setup

The apps are installed using a combination of helm charts and manifests with the help of helmfile and some bash scripts.

### Requirements

* A running cluster based on [compliantkubernetes-kubespray][compliantkubernetes-kubespray]
* [kubectl](https://github.com/kubernetes/kubernetes/releases) (tested with 1.20.7)
* [helm](https://github.com/helm/helm/releases) (tested with 3.8.0)
* [helmfile](https://github.com/roboll/helmfile) (tested with v0.144.0)
* [helm-diff](https://github.com/databus23/helm-diff) (tested with 3.1.2)
* [helm-secrets](https://github.com/futuresimple/helm-secrets) (tested with 3.12.0)
* [jq](https://github.com/stedolan/jq) (tested with jq-1.6)
* [sops](https://github.com/mozilla/sops) (tested with 3.6.1)
* [s3cmd](https://s3tools.org/s3cmd) available directly in ubuntus repositories (tested with 2.0.1)
* [yq4](https://github.com/mikefarah/yq) (tested with 4.26.1)
* [pwgen](https://sourceforge.net/projects/pwgen/) available directly in ubuntus repostitories (tested with 2.08)
* [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html) available directly in ubuntus repostitories (tested with 2.4.41)

Installs requirements using the ansible playbook get-requirements.yaml

```bash
ansible-playbook -e 'ansible_python_interpreter=/usr/bin/python3' --ask-become-pass --connection local --inventory 127.0.0.1, get-requirements.yaml
```

Note that you will need a service and workload cluster.

#### Developer requirements and guidelines

See [DEVELOPMENT.md](DEVELOPMENT.md).

### PGP

Configuration secrets in ck8s are encrypted using [SOPS](https://github.com/mozilla/sops).
We currently only support using PGP when encrypting secrets.
Because of this, before you can start using ck8s, you need to generate your own PGP key:

```bash
gpg --full-generate-key
```

Note that it's generally preferable that you generate and store your primary key and revocation certificate offline.
That way you can make sure you're able to revoke keys in the case of them getting lost, or worse yet, accessed by someone that's not you.

Instead create subkeys for specific devices such as your laptop that you use for encryption and/or signing.

If this is all new to you, here's a [link](https://riseup.net/en/security/message-security/openpgp/best-practices) worth reading!

## Usage

### Quickstart

**You probably want to check the [compliantkubernetes-kubespray][compliantkubernetes-kubespray] repository first, since compliantkubernetes-apps depends on having two clusters already set up.**
In addition to this, you will need to set up the following DNS entries (replace `example.com` with your domain).
- Point these domains to the workload cluster ingress controller:
  - `*.example.com`
- Point these domains to the service cluster ingress controller:
  - `*.ops.example.com`
  - `dex.example.com`
  - `grafana.example.com`
  - `harbor.example.com`
  - `notary.harbor.example.com`
  - `opensearch.example.com`

Assuming you already have everything needed to install the apps, this is what you need to do.

1. Decide on a name for this environment, the cloud provider to use as well as the flavor and set them as environment variables:
   Note that these will be later kept as global values in the common defaults config to prevent them from being inadvertently changed, as they will affect the default options of the configuration when generated or updated.
   To change them remove the common defaults config, set the new environment variables, and then generate a new configuration.

   ```bash
   export CK8S_ENVIRONMENT_NAME=my-ck8s-cluster
   export CK8S_CLOUD_PROVIDER=[exoscale|safespring|citycloud|elastx|aws|baremetal]
   export CK8S_FLAVOR=[dev|prod] # defaults to dev
   ```

1. Then set the path to where the ck8s configuration should be stored and the PGP fingerprint of the key(s) to use for encryption:

   ```bash
   export CK8S_CONFIG_PATH=${HOME}/.ck8s/my-ck8s-cluster
   export CK8S_PGP_FP=<PGP-fingerprint1,PGP-fingerprint2,...>
   ```

1. Initialize your environment and configuration:
   Note that the configuration is split between read-only default configs found in the `defaults/` directory, and the override configs `common-config.yaml`, `sc-config.yaml` and `wc-config.yaml` which are editable and will override any default value.
   The `common-config.yaml` will be applied to both the service and workload cluster, although it will be overriden by the any value set in the `sc-config.yaml` or `wc-config.yaml` respectively.
   When new configs are created this will generate new random passwords for all services.
   When configs are updated this will *not* overwrite existing values in the override configs.
   It will create a backup of the old override configs placed in `backups/`, generate new default configs in `defaults/`, merge common values into `common-config.yaml`, and clear out redundant values set in the override configs that matches the default values.
   See [compliantkubernetes.io](https://compliantkubernetes.io/) if you are uncertain about what order you should do things in.

   ```bash
   ./bin/ck8s init
   ```

1. Edit the configuration files that have been initialized in the configuration path.
   Make sure that the `objectStorage` values are set in `common-config.yaml` or `sc-config.yaml` and `wc-config.yaml`, as well as required credentials in `secrets.yaml` according to your `objectStorage.type`.
   The type may already be set in the default configuration found in the `defaults/` directory depending on your selected cloud provider.
   Set `objectStorage.s3.*` if you are using S3 or `objectStorage.gcs.*` if you are using GCS.

1. Create S3 buckets - optional
   If you have set `objectStorage.type: s3`, then you need to create the buckets specified under `objectStorage.buckets` in your configuration files.
   You can run the script `scripts/S3/entry.sh create` to create the buckets required.
   The script uses `s3cmd` in the background and it uses the `${HOME}/.s3cfg` file for configuration and authentication for your S3 provider.
   There's also a helper script `scripts/S3/generate-s3cfg.sh` that will allow you to generate an appropriate `s3cfg` config file for a few providers.

   ```bash
   # Use your s3cmd config file.
   scripts/S3/entry.sh create

   # Use custom config file for s3cmd.
   scripts/S3/generate-s3cfg.sh aws ${AWS_ACCESS_KEY} ${AWS_ACCESS_SECRET_KEY} s3.eu-north-1.amazonaws.com eu-north-1 > s3cfg-aws
   scripts/S3/entry.sh --s3cfg s3cfg-aws create
   ```

1. Test S3 configuration - optional
   If you enable object storage you also need to make sure that the buckets specified in `objecStorage.buckets` exist.
   You can run the following snippet to ensure that you've configured S3 correctly:

   ```bash
   (
      access_key=$(sops exec-file ${CK8S_CONFIG_PATH}/secrets.yaml 'yq r {} "objectStorage.s3.accessKey"')
      secret_key=$(sops exec-file ${CK8S_CONFIG_PATH}/secrets.yaml 'yq r {} "objectStorage.s3.secretKey"')
      sc_config=$(yq m ${CK8S_CONFIG_PATH}/defaults/common-config.yaml ${CK8S_CONFIG_PATH}/defaults/sc-config.yaml ${CK8S_CONFIG_PATH}/common-config.yaml ${CK8S_CONFIG_PATH}/sc-config.yaml -a overwrite -x)
      region=$(echo ${sc_config} | yq r - 'objectStorage.s3.region')
      host=$(echo ${sc_config} | yq r -  'objectStorage.s3.regionEndpoint')

      for bucket in $(echo ${sc_config} | yq r -  'objectStorage.buckets.*'); do
          s3cmd --access_key=${access_key} --secret_key=${secret_key} \
              --region=${region} --host=${host} \
              ls s3://${bucket} > /dev/null
          [ ${?} = 0 ] && echo "Bucket ${bucket} exists!"
      done
   )
   ```

1. **Note**, for this step each cluster need to be up and running already.
   Deploy the apps:

   ```bash
   ./bin/ck8s apply sc
   ./bin/ck8s apply wc
   ```

1. Test that the cluster is running correctly with:

   ```bash
   ./bin/ck8s test sc
   ./bin/ck8s test wc
   ```

1. You should now have a fully working environment.
   Check the next section for some additional steps to finalize it and set up user access.

### On-boarding and final touches

If you followed the steps in the quickstart above, you should now have deployed the applications and have a fully functioning environment.
However, there are a few steps remaining to make all applications ready for the user.

#### User access

After the cluster setup has completed RBAC resources and namespaces will have been created for the user.
You can configure what namespaces should be created and which users that should get access using the following configuration options in `wc-config.yaml`:

```yaml
user:
  namespaces:
    - demo1
    - demo2
  adminUsers:
    - admin1@example.com
    - admin2@example.com"
```

A **kubeconfig file for the user** (`${CK8S_CONFIG_PATH}/user/kubeconfig.yaml`) can be created by running the script `bin/ck8s kubeconfig user`.
The user kubeconfig will be configured to use the first namespace by default.

**OpenSearch Dashboards** access for the user can be provided either by setting up OIDC or using the internal user database in OpenSearch:
- OIDC:
  - Set `opensearch.sso.enabled=true` in `sc-config.yaml`.
  - Configure extra role mappings under `opensearch.extraRoleMappings` to give the users the necessary roles.
    ```yaml
    extraRoleMappings:
      - mapping_name: kibana_user
        definition:
          users:
            - "configurer"
            - "User Name"
      - mapping_name: kubernetes_log_reader
        definition:
          users:
            - "User Name"
    ```
- Internal user database:
  - Log in to OpenSearch Dashboards using the admin account.
  - Create an account for the user.
  - Give the `kibana_user` and `kubernetes_log_reader` roles to the user.

Users will be able to log in to **Grafana** using dex, but they will have read only access by default.
To give them more privileges, you need to first ask them to log in (so that they show up in the users list) and then change their roles.

**Harbor** works in a multi-tenant way so that each logged in user will be able to create their own projects and manage them as admins (including adding more users as members).
However, users will not be able to see each others (private) projects (unless explicitly invited) and won't have global admin access in Harbor.
This also naturally means that container images uploaded to these private registries cannot automatically be pulled in to the Kubernetes cluster.
The user will first need to add pull secrets that gives some ServiceAccount access to them before they can be used.

For more details and a list of available services see the [user guide](https://compliantkubernetes.io/user-guide/).

### Harbor HA - work in progress

It is possible to run harbor in HA mode.
This section describes the necessary configuration needed to setup harbor in HA mode.
More information about harbor ha can be found [here](https://goharbor.io/docs/2.2.0/install-config/harbor-ha-helm/).

Both Postgres and Redis needs to be external, as harbor does not handle HA deployment of postgres and redis.
It is up to the operator to set these up in a HA mode.

#### Postgres requirements

The following list is requirements on the external postgres

- Password encryption: none or md5
  - [scram-sha-256 is not supported](https://github.com/goharbor/harbor/issues/15731#issuecomment-1100666831).
- Initial empty databases must be created before harbor starts
  - registry
  - notaryserver
  - notarysigner

**Config**

Harbor backup is not designed to work with a external database.
You will have to provide your own backup solution.

In `$CK8S_CONFIG_PATH/sc-config.yaml` set the following configs
```
harbor:
  ...
  backup:
    enabled: false
  database:
    type: external
    external:
      host: "set-me"
      port: "5432"
      username: "set-me"
      # "disable" - No SSL
      # "require" - Always SSL (skip verification)
      # "verify-ca" - Always SSL (verify that the certificate presented by the
      # server was signed by a trusted CA)
      # "verify-full" - Always SSL (verify that the certification presented by the
      # server was signed by a trusted CA and the server host name matches the one
      # in the certificate)
      sslmode: "disable"
```

In `$CK8S_CONFIG_PATH/secrets.yaml` add the postgres user password
```
harbor:
  external:
    databasePassword: set-me
```

#### Redis

**Config**
In `$CK8S_CONFIG_PATH/sc-config.yaml` set the following configs

```
  redis:
    type: external
    external:
      addr: "rfs-redis-harbor.redis-system:26379"
      sentinelMasterSet: "mymaster"
```

### Management of the clusters

The [`bin/ck8s`](bin/ck8s) script provides an entrypoint to the clusters.
It should be used instead of using for example `kubectl`or `helmfile` directly as an operator.
To use the script, set the `CK8S_CONFIG_PATH` to the environment you want to access:

```bash
export CK8S_CONFIG_PATH=${HOME}/.ck8s/my-ck8s-cluster
```

Run the script to see what options are available.

#### Examples

* Bootstrap and deploy apps to the workload cluster:

  ```bash
  ./bin/ck8s apply wc
  ```

* Run tests on the service cluster:

  ```bash
  ./bin/ck8s test sc
  ```

* Port-forward to a Service in the workload cluster:

  ```bash
  ./bin/ck8s ops kubectl wc port-forward svc/<service> --namespace <namespace> <port>
  ```

* Run `helmfile diff` on a helm release:

  ```bash
  ./bin/ck8s ops helmfile sc -l <label=selector> diff
  ```

#### Autocompletion for ck8s in bash

Add this to `~/.bashrc`:

```bash
CK8S_APPS_PATH= # fill this in
source <($CK8S_APPS_PATH/bin/ck8s completion bash)
```

### Removing compliantkubernetes-apps from your cluster
There are two simple scripts that can be used to clean up you clusters.

To clean up the service cluster run:
```bash
./scripts/clean-sc.sh
```
To clean up the workload cluster run:
```bash
./scripts/clean-wc.sh
```

### Operator manual

See <https://compliantkubernetes.io/operator-manual/>.

### Setting up Google as identity provider for dex

1. Go to the [Google console](https://console.cloud.google.com/) and create a project.

2. Go to the [Oauth consent screen](https://console.cloud.google.com/apis/credentials/consent) and name the application with the same name as the project of your google cloud project add the top level domain e.g. `elastisys.se` to Authorized domains.

3. Go to [Credentials](https://console.cloud.google.com/apis/credentials) and press `Create credentials` and select `OAuth client ID`.
   Select `web application` and give it a name and add the URL to dex in the `Authorized Javascript origins` field, e.g. `dex.demo.elastisys.se`.
   Add `<dex url>/callback` to Authorized redirect URIs field, e.g. `dex.demo.elastisys.se/callback`.

4. Configure the following options in `CK8S_CONFIG_PATH/secrets.yaml`

   ```yaml
     dex:
       googleClientID:
       googleClientSecret:
   ```

## Known issues

- When using local volumes, OpenSearch might not start properly unless all worker nodes in the cluster has a local volume attatched to it.
- Users must explicitly be given privileges in Grafana, OpenSearch and Kubernetes instead of automatically getting assigned roles based on group membership when logging in using OIDC.
- The OPA policies are not enforced by default.
  Unfortunately the policies breaks cert-manager so they have been set to "dry-run" by default.
- OpenSearch Dashboards Single Sign On (SSO) via OpenID/Dex requires LetsEncrypt Production.

For more, please the the public GitHub issues: <https://github.com/elastisys/compliantkubernetes-apps/issues>.
