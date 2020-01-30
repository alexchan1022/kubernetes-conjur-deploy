# kubernetes-conjur-deploy

This repository contains scripts for automating the deployment of Conjur
followers to a Kubernetes or OpenShift environment. These scripts can also be
used to deploy a full cluster with Master and Standbys for testing and demo
purposes but this is not recommended for a production deployment of Conjur.

**Enterprise Only**. To deploy Conjur OSS, please use the [Conjur OSS helm chart](https://github.com/cyberark/conjur-oss-helm-chart).

---

# Setup

The Conjur deployment scripts pick up configuration details from local
environment variables. The setup instructions below walk you through the
necessary steps for configuring your environment and show you which variables
need to be set before deploying.

All environment variables can be set/defined with:
- `bootstrap.env` file if deploying the follower to Kubernetes or OpenShift
- `dev-bootstrap.env` for all other configurations.

Edit the values per instructions below, source the appropriate file and run
`0_check_dependencies.sh` to verify.

The Conjur appliance image can be loaded with `_load_conjur_tarfile.sh`. The script uses environment variables to locate the tarfile image and the value to use as a tag once it's loaded.
# Usage

## Deploying Conjur Follower

Ensure that:

1- `bootstrap.env` has the `FOLLOWER_SEED` variable set to the seed file created manually [here](#follower-seed) or by a seed service URL from the master by uploading the [policy](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/ConjurDeployFollowers.htm?Highlight=Initialize%20the%20Conjur%20CA%20for%20the%20Kubernetes%20Authenticator#ConfigureDAPforautoenrollmentofFollowers).

2- On the Conjur Master, [Initialize](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/ConjurDeployFollowers.htm?Highlight=Initialize%20the%20Conjur%20CA%20for%20the%20Kubernetes%20Authenticator) the Conjur CA for the Kubernetes Authenticator and add the Kubernetes Authenticator to the DAP authenticators.


Here a `bootstrap.env` example for Kubernetes Platform:

```
# For more details on the required environment
# variables, please see the README

# Make sure you comment out the section for the
# platform you're not using, and fill in the
# appropriate values for each env var

export CONJUR_VERSION=5
export CONJUR_APPLIANCE_IMAGE=conjur-appliance:5.4.3
export CONJUR_ACCOUNT=demo
export CONJUR_APPLIANCE_URL=https://dap-master.mydomain.com

# Only needed if FOLLOWER_SEED is a url to a remote seed service
# Remote Seed Service example: $CONJUR_APPLIANCE_URL/configuration/$CONJUR_ACCOUNT/seed/follower
export FOLLOWER_SEED=$CONJUR_APPLIANCE_URL/configuration/$CONJUR_ACCOUNT/seed/follower

export CONJUR_NAMESPACE_NAME=dap
export CONJUR_SERVICEACCOUNT_NAME=conjur-cluster
export AUTHENTICATOR_ID=k8s-follower
export CONJUR_FOLLOWER_COUNT=2

#######
# OPENSHIFT CONFIG (comment out all lines in this section if not using this platform)
#######
#export PLATFORM=openshift
#export OSHIFT_CLUSTER_ADMIN_USERNAME=[username of cluster admin]
#export OSHIFT_CONJUR_ADMIN_USERNAME=[username of Conjur namespace admin]
#export DOCKER_REGISTRY_PATH=docker-registry-<registry-namespace>.<routing-domain>

#######
# KUBERNETES CONFIG (comment out all lines in this section if not using this platform)
#######
export PLATFORM=kubernetes
export DOCKER_REGISTRY_URL=registry.hub.docker.com
export DOCKER_REGISTRY_PATH=registry.hub.docker.com/<my_path>
## Only if you are using a private repository 
export DOCKER_USERNAME=<my user>
export DOCKER_PASSWORD=<my_password>
export DOCKER_EMAIL=<my_email>
```

If master key encryption is used in the cluster, `CONJUR_DATA_KEY` must be set to the path to a file that contains the
encryption key to use when configuring the follower.

After verifying these settings:


```bash 
### Load environment variables
> source ./bootstrap.env
```

```bash
### Check all dependencies before running 
 > ./0_check_dependencies.sh
 ```

```bash 
### Deploy followers
> ./start 
```

# Configuration

## Conjur Configuration

#### Appliance Image

You need to obtain a Docker image of the Conjur appliance and push it to an
accessible Docker registry. Provide the image and tag like so:

```
export CONJUR_APPLIANCE_IMAGE=<tagged-docker-appliance-image>
```

You will also need to provide an ID for the Conjur authenticator that will later
be used in [Conjur policy](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Operations/Policy/policy-overview.htm?tocpath=Fundamentals%7CPolicy%20Management%7C_____0) to provide your
apps with access to secrets through Conjur:

```
export AUTHENTICATOR_ID=<authenticator-id>
```

This ID should describe the cluster in which Conjur resides. For example, if
you're hosting your dev environment on GKE you might use `gke/dev`.

#### Follower Seed

You will need to provide a follower seed file generated from your Conjur Master.

The seed can be generated:

* Using a Seed Service create by [Conjur policy](https://docs.cyberark.com/Product-Doc/OnlineHelp/AAM-DAP/Latest/en/Content/Integrations/ConjurDeployFollowers.htm?tocpath=Integrations%7CKubernetes%252C%20OpenShift%252C%20and%20GKE%7C_____4#ConfigureDAPforautoenrollmentofFollowers) and export the following variable:

```
$ export FOLLOWER_SEED=$CONJUR_APPLIANCE_URL/configuration/$CONJUR_ACCOUNT/seed/follower
```

* SSH-ing into your Master and running:

*NOTE: If you are running this code to deploy a follower that will run in a separate
cluster from the master, you _must_ force-generate the follower certificate manually
before creating the seed to prevent the resulting certificate from omitting the
future in-cluster subject altname included.*

To generate a follower seed with the appropriate in-cluster subject altname for followers
that are not in the same cluster as master, we first will need to issue a certificate
on master. Skip this step if master is collocated with the follower.
```
$ evoke ca issue --force <follower_external_fqdn> conjur-follower.<conjur_namespace_name>.svc.cluster.local
```

We now need to create the seed archive with the proper information:

```
$ evoke seed follower <follower_external_fqdn> > /tmp/follower-seed.tar
```

If you are on the same node as the master container, you can also export the seed with:
```
$ sudo docker exec <container_id> evoke seed follower <follower_external_fqdn> > /tmp/follower-seed.tar
```
Note: the exported seed file will not be copied to host properly if you use `-t` flag with the
`docker exec` command.

Copy the resulting seed file from the Conjur master to your local filesystem and
set the following environment variable to point to it:

```
export FOLLOWER_SEED=path/to/follower/seed/file
```

The deploy scripts will copy the seed to your follower pods and use it to
configure them as Conjur followers.

*Important note*: Make sure to delete any copies of the seed after use. It
contains sensitive information and can always be regenerated on the Master.

### Platform Configuration

If you are working with OpenShift, you will need to set:

```
export PLATFORM=openshift
export OSHIFT_CLUSTER_ADMIN_USERNAME=<name-of-cluster-admin> # system:admin in minishift
export OSHIFT_CONJUR_ADMIN_USERNAME=<name-of-conjur-namespace-admin> # developer in minishift
```

Otherwise, `$PLATFORM` variable will default to `kubernetes`.

Before deploying Conjur, you must first make sure that you are connected to your
chosen platform with a user that has the `cluster-admin` role. The user must be
able to create namespaces and cluster roles.

#### Conjur Namespace

Provide the name of a namespace in which to deploy Conjur:

```
export CONJUR_NAMESPACE_NAME=<my-namespace>
```

#### The `conjur-authenticator` Cluster Role

Conjur's Kubernetes authenticator requires the following privileges:

- [`"get"`, `"list"`] on `"pods"` for confirming a pod's namespace membership
- [`"create"`, `"get"`] on "pods/exec" for injecting a certificate into a pod

The deploy scripts include a manifest that defines the `conjur-authenticator`
cluster role, which grants these privileges. It will be created by the script 
[4_deploy_conjur_followers.sh](4_deploy_conjur_follower.sh) (note that your user will need to have the 
`cluster-admin` role to do so):

```
# Kubernetes
kubectl apply -f ./kubernetes/conjur-authenticator-role.yaml

# OpenShift
oc apply -f ./openshift/conjur-authenticator-role.yaml
```

### Docker Configuration

[Install Docker](https://www.docker.com/get-docker) from version 17.05 or higher on your local machine if you
do not already have it.


#### Kubernetes

You will need to provide the domain and any additional pathing for the Docker
registry from which your Kubernetes cluster pulls images:

```
export DOCKER_REGISTRY_URL=<registry-domain>
export DOCKER_REGISTRY_PATH=<registry-domain>/<additional-pathing>
```

Note that the deploy scripts will be pushing images to this registry so you will
need to have push access.

If you are using a private registry, you will also need to provide login
credentials that are used by the deployment scripts to create a [secret for
pulling images](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-in-the-cluster-that-holds-your-authorization-token):

```
export DOCKER_USERNAME=<your-username>
export DOCKER_PASSWORD=<your-password>
export DOCKER_EMAIL=<your-email>
```

Please make sure that you are logged in to the registry before deploying.

#### OpenShift

OpenShift users should make sure the [integrated Docker registry](https://docs.okd.io/latest/install_config/registry/deploy_registry_existing_clusters.html)
in your OpenShift environment is available and that you've added it as an
[insecure registry](https://docs.docker.com/registry/insecure/) in your local
Docker engine. You must then specify the path to the OpenShift registry like so:

```
export DOCKER_REGISTRY_PATH=docker-registry-<registry-namespace>.<routing-domain>
```

Please make sure that you are logged in to the registry before deploying.

### Running OpenShift in Minishift

You can use Minishift to run OpenShift locally in a single-node cluster. Minishift provides a convenient way to test out Conjur deployments on a laptop or local machine and also provides an integrated Docker daemon from which to stage and push images into the OpenShift registry. The `./openshift` subdirectory contains two files:
 * `_minishift-boot.env` that defines environment variables to configure Minishift, and
 * `_minishift-start.sh` to startup Minishift.
The script assumes VirtualBox as the hypervisor but others are supported. See https://github.com/minishift/minishift for more information.

Steps to startup Minishift:

1. Ensure VirtualBox is installed
1. `cd openshift`
1. Run `./minishift-start.sh`
1. `source minishift.env` to gain use of the internal Docker daemon
1. `cd ..`
1. Use `dev-bootstrap.env` for your variable configuration
1. Run `./start`

---

# (*Test and Demo Only*) Deploying Conjur Master and Followers 

## Master Cluster configuration

*Please note that running master cluster in OpenShift and Kubernetes environments
is not recommended and should be only done for test and demo setups.*


As mentioned before if you are using these scripts to deploy a full cluster, you will need to set
in `dev-bootstrap.env`:

```
export DEPLOY_MASTER_CLUSTER=true
```

You will also need to set a few environment variable that are only used when
configuring the Conjur master. If you are working with Conjur v4, you will need to set:

```
export CONJUR_VERSION=4
```
along with any other changes you might want.

Otherwise, this variable will default to `5`.

You must also provide an account name and password for the Conjur admin account:

```
export CONJUR_ACCOUNT=<my_account_name>
export CONJUR_ADMIN_PASSWORD=<my_admin_password>
```

Finally, run `./start` to execute the scripts necessary for deploying Conjur.

## Data persistence

The Conjur master and standbys are deployed as a
[Stateful Set](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) on supported target platforms (Kubernetes 1.5+ / OpenShift 3.5+).
Database and configuration data is symlinked and mounted to
[persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
These manifests assume a default [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/)
is set up for the cluster so persistent volume claims will be fulfilled.

Volumes:
- `/opt/conjur/dbdata` - 2GB, database persistence
- `/opt/conjur/data` - 1GB, seed file persistence

## Setup

To configure the Conjur master to persist data, run these commands in the Conjur master container before running `evoke configure master ...`.

```sh-session
# mv /var/lib/postgresql/9.3 /opt/conjur/dbdata/
# ln -sf /opt/conjur/dbdata/9.3 /var/lib/postgresql/9.3

# evoke seed standby > /opt/conjur/data/standby-seed.tar
```

Note that setup is done as part of script [`6_configure_master.sh`](6_configure_master.sh).

## Restore

If the Conjur master pod is rescheduled the persistent volumes will be reattached.
Once the pod is running again, run these commands to restore the master.

```
# rm -rf /var/lib/postgresql/9.3
# ln -sf /opt/conjur/dbdata/9.3 /var/lib/postgresql/9.3

# cp /opt/conjur/data/standby-seed.tar /opt/conjur/data/standby-seed.tar-bkup
# evoke unpack seed /opt/conjur/data/standby-seed.tar
# cp /opt/conjur/data/standby-seed.tar-bkup /opt/conjur/data/standby-seed.tar
# rm /etc/chef/solo.json

# evoke configure master ...  # using the same arguments as the first launch
```

Standbys must also be reconfigured since the Conjur master pod IP changes.

Run [`relaunch_master.sh`](relaunch_master.sh) to try this out in your cluster, after running the deploy.
Our plan is to automate this process with a Kubernetes operator.

## Conjur CLI

The deploy scripts include a manifest for creating a Conjur CLI container within
the Kubernetes environment that can then be used to interact with Conjur. Deploy
the CLI pod and SSH into it:

```
# Kubernetes
kubectl create -f ./kubernetes/conjur-cli.yaml
kubectl exec -it [cli-pod-name] bash

# OpenShift
oc create -f ./openshift/conjur-cli.yaml
oc exec -it <cli-pod-name> bash
```

Once inside the CLI container, use the admin credentials to connect to Conjur:

```
conjur init -h conjur-master
```

Follow our [CLI usage instructions](https://developer.conjur.net/cli#quickstart)
to get started with the Conjur CLI.

## Conjur UI

Visit the Conjur UI URL in your browser and login with the admin credentials to
access the Conjur UI.

---

# Test App Demo

The [kubernetes-conjur-demo repo](https://github.com/conjurdemos/kubernetes-conjur-demo)
deploys test applications that retrieve secrets from Conjur and serves as a
useful reference when setting up your own applications to integrate with Conjur.

# License

This repository is licensed under Apache License 2.0 - see [`LICENSE`](LICENSE) for more details.

