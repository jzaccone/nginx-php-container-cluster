## Building and deploying the initial set of containers
Now that the Kubernetes cluster and MySQL, Redis, and Memcached services have been provisioned, it's time to start a set of containers (pods) on the worker nodes. We do this through a set of declarative YAML files, which define not only the containers to start, but the storage and network configuration.

## Review the Docker build files
- The [`scripts/docker/nginx/Dockerfile`](../scripts/docker/nginx/Dockerfile) provides the steps to build a custom NGINX image based on the latest public image. It adds a configuration file that delegates PHP file requests to the load balanced pool of PHP-FPM containers, adds a static HTML page that it will serve itself, and starts up on port 80.
- The [`scripts/docker/php-fpm/Dockerfile`](../scripts/docker/php-fpm/Dockerfile) provides the steps to build a custom PHP-FPM image based on the latest public image. It installs prerequisite packages, configures and builds the MySQL, Redis, and Memcached extensions, copies the custom application code over, runs Composer, sets up read/write access to the storage volume for the PHP-FPM process which does not run as root, and starts up on port 9000.
- The [`scripts/docker/php-cli/Dockerfile`](../scripts/docker/php-cli/Dockerfile) provides the steps to build a custom PHP CLI image based on the latest public image. It installs prerequisite packages, configures and builds the MySQL, Redis, and Memcached extensions, copies the custom application code over, runs Composer, and sets up read/write access to the storage volume for the PHP-FPM process which does not run as root.

## Review the Kubernetes container deployment configuration files
- The [`scripts/kubernetes/persistent-volumes.yaml`](../scripts/kubernetes/persistent-volumes.yaml) files defines a 10 GB storage volume that can be mounted by many containers (`ReadWriteMany`). The containers then reference these volumes in their own configuration files.
- The [`scripts/kubernetes/php-fpm.yaml`](../scripts/kubernetes/php-fpm.yaml) file describes the pod/deployment for the PHP-FPM containers. It specifies how many containers from the given image and tag to start (2, for now), what port to listen on, the environment variables that map to the service credentials, and where to mount the storage volume.
- Similarly The [`scripts/kubernetes/nginx.yaml`](../scripts/kubernetes/nginx.yaml) file describes the pod/deployment for the NGINX containers. It specifies how many containers from the given image and tag to start (1, for now), what port to listen on, the environment variables that map to the service credentials, and where to mount the storage volume.
- Finally, the [`scripts/kubernetes/php-cli.yaml`](../scripts/kubernetes/php-cli.yaml) configures the pool of CLI workers that may be polling a database or queue for messages. It also maps the environment variables and storage volume, but does not expose a service for inbound network access.

## Build container images and push to the private registry
Log into Bluemix and the Container Registry. Make sure your target organization and space is set. If you haven't already installed the Container Registry plugin for the `bx` CLI:

```bash
# Configure the plugin if you haven't yet
bx plugin install container-service -r Bluemix
bx login -a https://api.ng.bluemix.net
bx cs init
```

Next, list the clusters already provisioned on Bluemix, and get the Kubernetes configuration information.
```bash
bx cs clusters #Find your cluster, and input into next command
bx cs cluster-config $CLUSTER_NAME
```

Copy the `export` line from the previous command to configure kubectl to point to your cluster.

```bash
# Configure kubectl
export KUBECONFIG=/Users/$USER_HOME_DIR/.bluemix/plugins/container-service/clusters/$CLUSTER_NAME/kube-config-$DATA_CENTER-$CLUSTER_NAME.yml
```

Finally, test your connection by interacting with your cluster.
```bash
# Confirm cluster is ready
kubectl get nodes

# Run the dashboard which will be available on http://127.0.0.1:8001/ui
kubectl proxy
```

## Optional: Configure your namespace
Right now, this POC is hardcoded to the "jjdojo" namespace. To avoid overwriting other people's images, you may want create and configure your own namespace.

Install the Bluemix container registry CLI plugin:
```bash
bx plugin install container-registry -r Bluemix
bx login
```

Create a namespace:
```bash
bx cr namespace-add <my_namespace>
bx cr namespaces #list namespaces
bx cr login # To enable pushing images
```

Configure scripts with your namespace. You will need to replace "jjdojo" in
- [build-containers.sh](../scripts/build-containers.sh)
- [nginx.yaml](../scripts/kubernetes/nginx.yaml)
- [php-cli.yaml](../scripts/kubernetes/php-cli.yaml)
- [php-fpm.yaml](../scripts/kubernetes/php-fpm.yaml)

Finally, you may have to [create an `imagePull` token](https://console.bluemix.net/docs/containers/cs_cluster.html#bx_registry_other).

## Build the container images
Run this script to build the containers and push them to your registry:
```bash
cd scripts
./build-containers.sh
```

## Deploy the container images to the Kubernetes cluster

```bash
# Create image pull token if needed one time. The kubectl command may not like the wrapped lines, so change it all to one line if needed.
bx cr token-list
bx cr token-get $TOKEN_ID
kubectl --namespace default create secret docker-registry image-pull \
--docker-server="registry.ng.bluemix.net" \
--docker-username="token" \
--docker-password="${TOKEN}" \
--docker-email="${YOUR_EMAIL}"

./deploy-containers.sh
```

The yaml files in this directory reference in same image names (including the name of our registry namespace) as in the `build-containers.sh` script. These is the hand-off point between image build/push, and kubernetes deploy.

## Specify a non-floating LoadBalancer IP
Obtain the available IPs assigned to your cluster (look for "is_public: true")
```bash
kubectl get cm ibm-cloud-provider-vlan-ip-config -n kube-system -o yaml
```

Set `spec.loadBalancerIP` inside [`scripts/kubernetes/nginx.yaml`](../scripts/kubernetes/nginx.yaml)

For example:
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  loadBalancerIP: <ip>
  ...
---
```

## Setup Ingress (replaces LoadBalancer)
So far, we have configured LoadBalancer as the service type for the nginx service. We can use the Ingress type instead to give us more flexibility with specifying routes from a single endpoint and also us to use a hostname instead of floating IPs to access our application. Detailed docs here: https://console.bluemix.net/docs/containers/cs_apps.html#cs_apps_public_ingress.

1) Remove the `type: LoadBalancer` line from [`scripts/kubernetes/nginx.yaml`](../scripts/kubernetes/nginx.yaml)

2) Obtain your "Ingress subdomain".
```bash
bx cs cluster-get <cluster name>
``` 

3) Edit [`scripts/kubernetes/ingress/ingress.yaml`](../scripts/kubernetes/ingress/ingress.yaml) to include your subdomain.

4) Redeploy your nginx service
```bash
kubectl delete service nginx
kubectl apply -f scripts/kubernetes/nginx.yaml
```

5) Deploy the ingress service
```bash
kubectl apply -f scripts/kubernetes/ingress/ingress.yaml
```

6) Once ingress is up (may take a minute), access your application via your domain.

## Tear down the containers
If you want to cleanly install the environment, for example to push a new set of container versions, use the following script:

```bash
./destroy-containers.sh
```
