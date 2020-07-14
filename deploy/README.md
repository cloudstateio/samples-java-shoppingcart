## Deploying on self-hosted Cloudstate on Kubernetes

### Prerequisites

[Set up Cloudstate](https://github.com/cloudstateio/cloudstate#run-cloudstate) on your Kubernetes cluster.

### Quick install

All the latest docker images are available publicly at `lightbend-docker-registry.bintray.io/cloudstate-samples`.

To deploy the shopping-cart application as is, connect to your kubernetes environment and do the following.

```shell
$ kubectl apply -f . -n <project-name>
# verify stateful store
$ kubectl get -n <project-name> statefulstore
NAME                  AGE
shopping-store   21m
# verify stateful services
$ kubectl -n <project-name>  get statefulservices
NAME            AGE    REPLICAS   STATUS
shopping-cart   7m     1          Running
frontend        4m     1          Running
```

### Deploying the frontend service

If you have built your own container image, edit the `frontend.yaml` to point to the image that you just pushed.

```shell
$ cat frontend.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: frontend
spec:
  containers:
  - image: lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart:latest # <-- Change this to your repo/image
    name: frontend
```

Deploy the service to your project namespace

```shell
kubectl apply -f frontend.yaml -n <project_name>
statefulservice.cloudstate.io/frontend created
```

### Deploying a stateful store

The shopping cart stateful service relies on a stateful store as defined in `shopping-store.yaml`.

Deploy the store to your project namespace
```shell
$ kubectl apply -f shopping-store.yaml -n <project-name>
statefulstore.cloudstate.io/shopping-store created
```

### Deploying the shopping cart service

If you have built your own container image, edit `shopping-cart.yaml` to point to the docker image that you just pushed

```shell
$ cat shopping-cart.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: shopping-cart
spec:
  # Datastore configuration
  storeConfig:
    database: shopping
    statefulStore:
      # Name of a deployed Datastore to use.
      name: shopping-store
  containers:
    - image:  lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest # <-- Change this to your repo/image
      name: shopping-cart
```

Deploy the service to your project namespace

```shell
$ kubectl apply -f shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
```

### Verifying that the services are running

```shell
$ kubectl get statefulservices -n <project-name>
NAME            AGE    REPLICAS   STATUS
shopping-cart   6m     1          Running
frontend        3m     1          Running
```

**TODO** Investigate if/why `kubectl delete` is needed here.

To redeploy a new image to the cluster you must delete and then redeploy using the yaml file. For example, if we updated the shopping-cart docker image we would do the following.

```shell
$ kubectl delete statefulservice shopping-cart -n <project-name>
statefulservice.cloudstate.io "shopping-cart" deleted
$ kubectl apply -f shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
```

### Setting up routes

**TODO** The following instructions do not work for self-hosted Cloudstate. See https://github.com/cloudstateio/samples-java-shoppingcart/issues/10

The last thing that is required is to provide the public routes needed for both the frontend and grpc-web calls.  These exist in the `routes.yaml` file.

```shell
$ cat routes.yaml
apiVersion: cloudstate.io/v1alpha1
kind: Route
metadata:
  name: "shopping-routes"
spec:
  http:
  - name: "shopping-routes"
    match:
    - uri:
        prefix: "/com.example.shoppingcart.ShoppingCart/"
    route:
      service: shopping-cart
  - name: "frontend-routes"
    match:
    - uri:
        prefix: "/"
    route:
      service: frontend
```

Add these routes by performing
```shell
kubectl apply -f routes.yaml -n <project-name>
```

Open a web browser and navigate to:

`https://<project-name>.us-east1.apps.lbcs.io/pages/index.html`

You should now see the shopping cart interface.
