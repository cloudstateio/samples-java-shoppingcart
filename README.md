
# Cloudstate Sample Shopping Cart Application

The following assumes that you have completed the steps for setting up your local environment as well as creating an account and project.  If you have not done this you must follow the instructions here:

* [Setting Up your Machine](https://docs.lbcs.dev/gettingstarted/setup.html)
   * as well as the [Developer prerequisites](https://docs.lbcs.dev/developing/developing.html#prerequisites)
   * You also need to install the protobuf compiler.
* [Your Lightbend Cloudstate Account](https://docs.lbcs.dev/gettingstarted/account.html)
* [Creating a Project](https://docs.lbcs.dev/gettingstarted/project.html)

## Sample application layout
Grab the sample application from github:
 
[https://github.com/lightbend/cloudstate-samples](https://github.com/lightbend/cloudstate-samples)

The sample application consists of 2 services:
* A stateless service `frontend`
* A stateful Entity based service `js-shopping-cart`

Additionally:
* A `cloudstate` directory that contains proto definitions needed.
* A `deploy` directory that contains the deployment yaml files.

## Quick Install

All the latest docker images are available publicly at `lightbend-docker-registry.bintray.io/`.

To deploy the shopping cart application as is, connect to your kubernetes environment and do the following.

### Storage Setup
* Modify `postgres-store.yaml`
    * Uniquify the store `name`
* Modify `js-shopping-cart.yaml` to match
*   Change `spec|storeConfig|statefulStore|name` to match the name used above

### Deployment
```bash
$ cd deploy
$ kubectl apply -f . -n <project-name>
# To Verify
$ kubectl -n <project-name>  get statefulservices
NAME            REPLICAS   STATUS
frontend        1          Ready
shopping-cart   1          Ready
```

To verify the statefulstore use the following:
```bash
$ kubectl get statefulstore -n corey-demo2
NAME                  AGE
shopping-postgres77   21m
```

To access the front end chat interface open a web browser and navigate to:

`https://<project-name>.us-east1.apps.lbcs.dev/pages/index.html`

If you would like to make changes and build the application, please follow the
instructions in the section below.

## Building and deploying the Sample application

### Frontend Service
The frontend service is a front end web application written in typescript.  It is backed by a `stateless` service that will serve the compiled javacript, html and images.

This service makes `grpc-web` calls directly to the other services to get the data that it needs.  In order to do this we need to compile the proto definitions from the other services as well as generate the grpc-web clients.  This is all done with a shell script `protogen.sh`.  Let's first run the protogen script, then compile the service definition and finally compile our typescript.

```
cd frontend
./protogen.sh
npm install
npm run prestart
npm run-script build
```

This will compile the protobuf and `user-function.desc`.

Build a docker image with the right registry and tag
```
docker build . -t <my-registry>/frontend:latest
```

Push the docker image to the registry
```
docker push <my-registry>/frontend:latest
```

Deploy the image by changing into the deploy folder and editing the `frontend.yaml` to point to your docker image that you just pushed.
```
$ cd ../deploy
$ cat js-frontend.yaml
apiVersion: cloudstate.io/v1alpha1
kind: StatefulService
metadata:
  name: frontend
spec:
  containers:
  - image: coreyauger/frontend:latest    # <-- Change this to your image
    name: frontend
```

Deploy the service to your project namespace
```
kubectl apply -f frontend.yaml -n <project_name>
statefulservice.cloudstate.io/frontend created
````


### Postgres Store

The shopping cart stateful service relies on a stateful store.  The only change required to `postgres-store.yaml` is to give
the store a unique name.  This is set in the `spec:storeConfig:statefulStore:name` field.

### Shopping Cart Service
```
cd ../js-shopping-cart
npm install
npm run prestart
```

This will compile the protobuf and `user-function.desc`
Build a docker image with the right registry and tag
```
docker build . -t <my-registry>/shopping-cart:latest
```

Push the docker image to the registry
```
docker push <my-registry>/shopping-cart:latest
```

Deploy the image by changing into the deploy folder and editing the `shopping-cart.yaml` to point to your docker image that you just pushed.
```
$ cd ../deploy
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
      name: shopping-postgres77     # <-- Change to match the name of the postgres-store
  containers:
  - image: coreyauger/shopping-cart:latest    # <-- Change this to your image
    name: shopping-cart
```

Deploy the service to your project namespace
```
$ kubectl apply -f js-shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
````

### Verify they are running
Check that the services are running
```
$ kubectl get statefulservices -n <project-name>
NAME             REPLICAS   STATUS
shopping-cart    1          Ready
frontent         1          Ready
```

To redeploy a new image to the cluster you must delete and then redeploy using the yaml file.  
For example if we updated the shopping-cart docker image we would do the following.
````
$ kubectl delete statefulservice shopping-cart -n <project-name>
statefulservice.cloudstate.io "shopping-cart" deleted
$ kubectl apply -f js-shopping-cart.yaml -n <project-name>    
statefulservice.cloudstate.io/shopping-cart created
````

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

`https://<project-name>.us-east1.apps.lbcs.dev/pages/index.html`

You should now see the shopping cart interface.

## Maintenance notes

### License
The license is Apache 2.0, see [LICENSE-2.0.txt](LICENSE-2.0.txt).

### Maintained by
__This project is NOT supported under the Lightbend subscription.__

This project is maintained mostly by @coreyauger and @cloudstateio.

Feel free to ping the maintainers above for code review or discussions. Pull requests are very welcome–thanks in advance!


### Disclaimer

[DISCLAIMER.txt](DISCLAIMER.txt)
