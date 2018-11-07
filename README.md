# Gardener Setup Scripts

This is the installation manual for a simple Gardener setup. It is part of the [landscape-setup-template](https://github.com/gardener/landscape-setup-template) project. You can find further information there.

We do recommend this simplified setup for demonstration purposes only. For productive workloads we do recommend that all components (Gardener/Seed/Shoot) run in their own IaaS accounts and that network policies are enabled and properly tested on the seed clusters.


# Prerequisites

Before getting started make sure you have the following at hand:

* You need a cloud account with sufficient quota to set up a Kubernetes cluster with a couple of VMs. **This project currently supports AWS and Openstack.**
* A Linux machine (virtual machine is fine) or a Mac with basic tools such as a git client and the Docker runtime installed.


# Gardener Installation

Follow these steps to install Gardener. Do not proceed to the next step in case of errors.


## TL;DR
If you are already familiar with the installation procedure and just want a short summary of the commands you have to use, here it is:

```
# setup
git clone  --recursive https://github.com/gardener/landscape-setup-template.git landscape
cd landscape/setup
./docker_run.sh
deploy all kubify:
# alternative setup: deploy all etcd:

# -------------------------------------------------------------------

# teardown
undeploy all :kubify
# alternative setup: undeploy all :etcd
./cleanup.sh
```

### Kubectl Aliases

The following aliases can be used within the docker container:
```
k => kubectl
ks => kubectl -n kube-system
kg => kubectl -n garden
kn => kubectl -n
ka => kubectl get --all-namespaces
```

Bash completion works for all of them except for `ka`. 

## Step 1: Clone the Repositories and get Dependencies

Get the `landscape-setup-template` from GitHub and initialize the submodules:

```
git clone  --recursive https://github.com/gardener/landscape-setup-template.git landscape
cd landscape
```

After step 2, this repository will contain all passwords and keys for your landscape. You will be in trouble if you loose them so we recommend that you store this landscape configuration in a private repository. It might be a good idea to change the origin so you do not accidentally publish your secrets to the public template repository.

## Step 2: Configure the Landscape

There is a `landscape_config.yaml` file in the landscape project. This is the only file that you need to modify - all other configuration files will be derived from this and the `landscape_base.yaml` file. The latter one contains the merging instructions as well as technical configurations and it shouldn't be touched unless you know what you are doing. 

#### Building the 'landscape.yaml' File

Both config files - `landscape_config.yaml` and `landscape_base.yaml` - are merged into one `landscape.yaml` file which is then used as configuration for the scripts. Sourcing the `init.sh` file (which happens automatically when entering the docker image) will perform this merge **unless the file already exists.** This means if you change something in one of the original config files after the `landscape.yaml` file has already been created, you need to manually rebuild it in order for the changes to take effect. 

```
./build_landscape_yaml.sh
```

This script will recreate the `landscape.yaml` file. It will also source the `init.sh` file again, as some of the environment variables are extracted from this file.

#### The Base Cluster

Gardener extends the Kubernetes apiserver, so in order to deploy it, you need a Kubernetes cluster first. This setup gives you two options for this:

##### Kubify
You can use [Kubify](https://github.com/gardener/kubify) to create the initial cluster. Kubify uses Terraform to create the cluster and it is integrated into this project - you don't need to create the cluster yourself, just make sure you fill out all relevant parts of the config file.

##### Shoot Cluster
A shoot cluster is a cluster created by a Gardener instance and it can be used as a base cluster for this project. These flags have to be set for the kube-apiserver (if not set, the Gardener will still work, but the dashboard won't):
```
--oidc-issuer-url=https://identity.ingress.<your cluster domain>
--oidc-client-id=kube-kubectl
--oidc-username-claim=email
--oidc-groups-claim=groups
```

For a shoot this can be done by setting `issuerUrl`, `clientID`, `usernameClaim`, and `groupsClaim` in `spec.kubernetes.kubeAPIServer.oidcConfig` in the shoot manifest.

Also make sure that the CIDRs of your base cluster and the from your Gardener spawned shoots don't overlap - if you want to be able to create shoots from the Gardener dashboard later, then don't use the default CIDRs for this base cluster.

Some fields in the `landscape_config.yaml` are marked with `# kubify only`, they can be ignored when using a shoot as the base cluster. The etcd server address defaults to the address for Kubify and needs to be changed for the shoot setup (see the comments in the config file). 

The *kubeconfig* for the base cluster is expected to be named `kubeconfig` and reside in the directory containing this project's directory (next to the `landscape_config.yaml` file).

##### Using an Arbitrary Base Cluster
While this setup has only been tested for clusters created by Kubify or the Gardener (shoot clusters), it is theoretically possible to use the shoot setup method to deploy the Gardener to an arbitrary kubernetes cluster.

## Step 3: Build and Run Docker Container

First, `cd` into the folder containing this project.

Then run the container:

```
./docker_run.sh
```

After this,

* you will be connected to the container via an interactive shell
* the landscape folder will be mounted in that container
* your current working directory will be `setup` folder
* `setup/init.sh` is sourced, meaning
  * the environment variables will be set
  * kubectl will be configured to communicate with your cluster
  * `landscape.yaml` file will have been created if it didn't exist before

The `docker_run.sh` script searches for the image locally and pulls it from an image repository, if it isn't found. 
If pulling the image doesn't work for whatever reason, you can use the `docker_build.sh` script to build the image locally. 

## Step 4-10: Deploying Components

The Gardener deployment is splitted into components. A single component can be easily deployed using 

```
deploy <component name>
```

Please take care that most of the components depend on each other and therefore have to be deployed in the order given below.

The `deploy` command is added to the `PATH` environment variable and can thus be called from anywhere. Bash auto-completion can be used for the component names.

#### Undeploying Components
It is also possible to "undeploy" a component using 

```
undeploy <component name>
```

Components need to be undeployed in the inverse order. Do not undeploy components without undeploying their successors in the component order first. Take care to delete all shoots before undeploying the `gardener` or `seed-config` components (although both undeploy scripts will check for that and trigger a deletion themselves).

#### The 'all' Component
The `all` component is a special component: it serves as a dummy to deploy several components in one go. Usually, manual intervention between deploying components is not necessary and most of them are deployed directly one after the other, so the `all` component makes the "normal" use-case easier. 

For better control which components are deployed, a component range can be given as an argument. The argument should have the form `<start component name>:<end component name>` and then the start component, the end component, and all components in between will be deployed. The order of the arguments is taken from the `$COMPONENT_ORDER` environment variable. 

It is also possible to drop one part of the range or to drop the whole argument. For missing parts the defaults will be used, which are `cert` for the start component and `certmanager` for the end component. 

Upon inspection of the component order, you might notice that the components `kubify` and `etcd` are not deployed by default. This is partly due to historical reasons, as the `all` component evolved from the `deploy_gardener.sh` script, which deployed the components that are also now the default. The other reason is that `etcd` and `kubify` kind of exclude each other - the setup can integrate the Gardener into the Kubify etcd, so an extra etcd is not needed. 
*Calling `deploy all kubify:` won't deploy the `etcd` component although it comes after `kubify` in the component order!* 
Long story short: if you want to create your base cluster using Kubify, you will probably call `deploy all kubify:`, if not, it is probably `deploy all etcd:`.

The `undeploy` command can also be used with the `all` component, but take care that the component order is inverted. 

```
# Examples
# (start and end component are always inclusive)
deploy all                          # deploys 'cert' through 'certmanager' 
deploy all gardener:dashboard       # deploys 'gardener' through 'dashboard' 
deploy all kubify:                  # deploys 'kubify' through 'certmanager', excluding 'etcd' (special case)
deploy all :gardener                # deploys 'cert' through 'gardener'

undeploy all                        # undeploys 'certmanager' through 'cert'
undeploy all :kubify                # undeploys 'certmanager' through 'gardener', excluding 'etcd' (special case)
undeploy all dashboard:etcd         # undeploys 'dashboard' through 'etcd'
```

## Step 4-10: Deploying Components (detailed)

### Step 4: Kubify / etcd

If you want to create a Kubify cluster, deploy the component:

```
deploy kubify
```

The script will wait some time for the cluster to come up and then partly validate that the cluster is ready.

If you get errors during the cluster setup, just try to run the command again.

Once completed the following command should show all deployed pods:

```
root@c41327633d6d:/landscape# kubectl get pods --all-namespaces
NAMESPACE       NAME                                                                  READY     STATUS    RESTARTS   AGE
kube-system     etcd-operator-75dcfcf4f7-xkm4h                                        1/1       Running   0          6m
kube-system     heapster-c8fb4f746-tvts6                                              2/2       Running   0          2m
kube-system     kube-apiserver-hcdnc                                                  1/1       Running   0          6m
[...]
```

---

If you already have a cluster, you don't need Kubify. To deploy an etcd in your cluster, run 

```
deploy etcd
```

This component is not meant to be used in combination with Kubify and might require manual steps to make it work. 

It should also be possible to plug in your own etcd - check the deploy scripts for the `etcd` and `gardener` components for information on where to put the certificates, etc.

### Step 4.5: Workaround (Automated)

There is currently an issue with session affinities in Kubernetes, which can break your cluster. While the problem has been fixed in Kubernetes 1.11 (see [here](https://github.com/kubernetes/kubernetes/commit/f2405cf2942739996af2bb76347c2cb0641153aa)), Kubify currently only supports version 1.9. 

Until that changes, the workaround is to remove the following lines from the `kubernetes` service:

```
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

Kubernetes will add `sessionAffinity: None` on itself.

This workaround is applied automatically at the end of the `kubify` component deploy script, it only needs to be done manually if the script doesn't run through for some reason. 

### Step 5: Generate Certificates

This step will generate a self-signed cluster CA and sign some certificates with it.

```
deploy cert
```

### Step 6: Deploy tiller

Tiller is needed to deploy the Helm charts of Gardener and other components.

```
deploy helm-tiller
```

### Step 7: Deploy Gardener

Now we can deploy Gardener. If the previous steps were executed successfully this should be completed in a couple of seconds.

```
deploy gardener
```

You might see a couple of messages like these:

```
Gardener API server not yet reachable. Waiting...
```

while the script waits for the Gardener to start. Once Gardener is up when the deployment script finished you can verify the correct setup by running the following command:

```
kubectl get shoots
No resources found. 
```

As we do not have a seed cluster yet we cannot create any shoot clusters.
The Gardener itself is installed in the `garden` namespace:

```
kubectl get po -n garden
NAME                                          READY     STATUS    RESTARTS   AGE
gardener-apiserver-56cc665667-nvrjl           1/1       Running   0          6m
gardener-controller-manager-5c9f8db55-hfcts   1/1       Running   0          6m
```

### Step 8: Register Garden Cluster as Seed Cluster

In heterogeneous productive environments one would run Gardener and seed in separate clusters but for simplicity and resource consumption reasons we will register the Gardener cluster that we have just created also as the seed cluster. Make sure that the `seed_config` in the landscape file is correct and matches the region that you are using. Keep in mind that image ids differ between regions as well. Also, valid credentials for the seed provider have to be specified in the `authentication` part of the `landscape_config.yaml` file (the etcd backups of the shoot clusters are stored on the seed). 

```
deploy seed-config
```

#### Configuring Additional Seeds

By default, this step will create a seed for the cloud provider the Gardener has been deployed on and thus creating shoots on this provider will be possible. If you want to create shoots on other cloud providers, you will have to configure additional seeds. There are two options for that:

If the seed-config deploy script is called without any arguments (as shown above), it will create seeds for all providers specified in the `seed_config.seeds` section in the `landscape_config.yaml` file. By default, the only entry in that list is the cloud provider chosen for the Gardener cluster, but you can extend the list.

It is also possible to provide the seed-config deploy script with additional arguments specifying which seeds should be created. Multiple arguments can be given and the script will ignore the list in the `landscape_config.yaml` file when called with arguments. Only the specified seeds will be created, already existing seeds are not affected. If a given seed already exists, it will be updated to the current configuration. 

In both cases, the corresponding variant nodes in `authentication` and `seed_config` have to be filled out in the config file. 

Valid values for seeds are `aws`, `az` (for Azure), `gcp`, and `openstack`. Please note, that while it is possible to create seeds for any cloud provider on any cloud provider, shoot creation may not work across cloud providers for every combination. It should always work if seed (Gardener cluster in this setup) and shoot are on the same provider, though. 

#### Creating a Shoot

That's it! If everything went fine you should now be able to create shoot clusters.
You can start with a sample [manifest](https://github.com/gardener/gardener/blob/master/example/90-shoot-aws.yaml) and create a shoot cluster by standard Kubernetes means:

```
kubectl apply -f shoot-aws.yaml
```

### Step 9: Install Identity and Dashboard

Creating clusters based on a shoot manifest is quite nice but also a little complex. While almost all aspects of a shoot cluster can be configured, it can be quite difficult for beginners, so go on and install the dashboard:

```
deploy identity
[...]
deploy dashboard
[...]
```

### Step 10: Apply Valid Certificates

The following command will install the [cert-manager](https://github.com/jetstack/cert-manager) and request valid letsencrypt certificates for both the identity and dashboard ingresses:

```
deploy certmanager
```

After a few minutes valid certificates should be installed.

### Letsencrypt Quota Limits
Letsencrypt [limits](https://letsencrypt.org/docs/rate-limits/) how many certificates you can get for the same host within a short time. To avoid hitting these limits, you can use the letsencrypt staging server for testing, which has a significantly higher rate limit but produces untrusted certificates.  
The `charts.[certmanager].live` field in the config file allows to switch between live and staging server (remember to rebuild the `landscape.yaml` file after you changed something in the `landscape_config.yaml` file).

#### Accessing the Dashboard

After step 9 you will be able to access the Gardener dashboard. There is a difference in how you access it depending on whether you used the letsencrypt live server or the staging one (and thus have untrusted credentials in the latter case). 

The `print_dashboard_urls.sh` script constructs two URLs from the domain name given in the `landscape.yaml` file and prints them. 

If you have trusted certificates, just use the second one (the one for the dashboard) and everything will be fine. 

If you used the letsencrypt staging server, you will need to visit the first link first. Your browser will show a warning regarding untrusted certificates, you need to ignore that warning. You will then see a nearly blank page with some 404 message. After that, you can open the dashboard link, ignore the certificate warning again and should be able to login. 
If you skip the first link, you will still be able to see the dashboard, but the login button probably won't work. 
While you will be able to login and create projects with the untrusted certificates from the letsencrypt staging server, creating secrets or shoots won't be possible. You'll need trusted certificates for that.

To log into the dashboard, use the options you have specified in the identity chart part of the `landscape_config.yaml`.

# Tearing Down the Landscape

Make sure that you delete all shoot clusters prior to tearing down the cluster. Not deleting `project` resources before deleting the Gardener can also cause troubles, because the namespaces associated with the projects have a finalizer which can't be handled anymore when the Gardener is gone. Both, shoots and projects, can be deleted using the `delete_all.sh` script (give 'shoots' or 'projects' as an argument). To delete a single shoot/project, use [this](https://github.com/gardener/gardener/blob/master/hack/delete) script.

The following command should not return any shoot clusters:

```
kubectl get shoots --all-namespaces
No resources found.
```

If you created your base cluster with the Kubify component, you can destroy it using the undeploy command:
```
undeploy kubify
```

# Cleanup

After destroying the Kubify cluster, there will be some files left that prevent you from simply starting the project up again. 

**ATTENTION: Only do this if you are sure the cluster has been completely destroyed!**
Since this removes the terraform state, an automated deletion of resources won't be possible anymore - you will have to clean up any leftovers manually.

```
./cleanup.sh
```

This will reset your landscape folder to its initial state (including the deletion of `landscape.yaml`).

The script takes an optional "-y" argument to skip the confirmation.