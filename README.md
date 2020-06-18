
# Cloudstate Sample Shopping Cart Application

# Prerequisites

First make sure [your local environment is setup appropriately](../README.md#prerequisites).

## Sample application layout

The sample application consists of 2 services:
* A stateless service `frontend`
* A stateful Entity based service `js-shopping-cart`

Additionally:
* A `cloudstate` directory that contains proto definitions needed.
* A `deploy` directory that contains the deployment yaml files.

### Quick Install

All the latest docker images are available publicly at `lightbend-docker-registry.bintray.io/cloudstate-samples`.

To deploy the shopping-cart application as is, connect to your kubernetes environment and do the following.

```bash
$ cd deploy
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

`https://<project-name>.us-east1.apps.lbcs.dev/pages/index.html`

## Building and deploying the Sample application

### Frontend Service

The frontend service is a front end web application written in typescript. It is backed by a `stateless` service that will serve the compiled javacript, html and images. This service makes `grpc-web` calls directly to the other services to get the data that it needs.

#### Getting container image ready

You can use pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest` container image available at Lightbend Cloudstate samples repository.

Alternatively, you can clone [cloudstateio/samples-ui-shoppingcart](https://github.com/cloudstateio/samples-ui-shoppingcart) repository and follow the instructions there to build an image and deploy it to your own container repository.

#### Deploying the frontend service

Change into the `deploy` folder
```
$ cd deploy
```
If you have built your own container image, edit the `frontend.yaml` to point to the image that you just pushed.
```
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
```
kubectl apply -f frontend.yaml -n <project_name>
statefulservice.cloudstate.io/frontend created
```


### Stateful Store

The shopping cart stateful service relies on a stateful store as defined in `shopping-store.yaml`.

Deploy the store to your project namespace
```
$ kubectl apply -f shopping-store.yaml -n <project-name>
statefulstore.cloudstate.io/shopping-store created
```

### Shopping Cart Service
```
cd ../js-shopping-cart
nvm install
nvm use
npm install
npm run prestart
```
For this service there is no web front end, so we only need to compile the `shoppingcart.proto` into the `user-function.desc`.

Build a docker image with the right registry and tag

NOTE: you can get a free public docker registry by signing up at [https://hub.docker.com](https://hub.docker.com/)
```
docker build . -t <my-registry>/shopping-cart:latest
```

Push the docker image to the registry
```
docker push <my-registry>/shopping-cart:latest
```

Deploy the image by changing into the deploy folder and editing `js-shopping-cart.yaml` to point to the docker image that you just pushed.
```
$ cd ../deploy
$ cat js-shopping-cart.yaml
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
      name: js-shopping-cart
```

Deploy the service to your project namespace
```
$ kubectl apply -f js-shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
```

### Verify they are running
Check that the services are running
```
$ kubectl get statefulservices -n <project-name>
NAME            AGE    REPLICAS   STATUS
shopping-cart   6m     1          Running
frontend        3m     1          Running
```

To redeploy a new image to the cluster you must delete and then redeploy using the yaml file.  
For example if we updated the shopping-cart docker image we would do the following.
```
$ kubectl delete statefulservice shopping-cart -n <project-name>
statefulservice.cloudstate.io "shopping-cart" deleted
$ kubectl apply -f js-shopping-cart.yaml -n <project-name>    
statefulservice.cloudstate.io/shopping-cart created
```

## Routes
The last thing that is required is to provide the public routes needed for both the front end and grpc-web calls.  These exist in the `routes.yaml` file.

```
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
```
kubectl apply -f routes.yaml -n <project-name>
```

Open a web browser and navigate to:

`https://<project-name>.us-east1.apps.lbcs.io/pages/index.html`

You should now see the shopping cart interface.

## Maintenance notes

### License
The license is Apache 2.0, see [LICENSE](LICENSE).

### Maintained by
__This project is NOT supported under the Lightbend subscription.__

This project is maintained mostly by @coreyauger and @cloudstateio.

Feel free to ping the maintainers above for code review or discussions. Pull requests are very welcome–thanks in advance!


### Disclaimer

[DISCLAIMER.txt](DISCLAIMER.txt)
