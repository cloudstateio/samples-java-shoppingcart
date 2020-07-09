
# Cloudstate Sample Shopping Cart Application

# Prerequisites

The following assumes that you have completed the steps for setting up your local environment as well as creating an account and project.  If you have not done this you must follow the instructions here:

* [Setting Up your Machine](https://docs.lbcs.dev/gettingstarted/setup.html)
   * Install JDK (Java Development Kit) version 8 or later.
      * We recommend OpenJDK 1.8.0_252 or later. (Check with `java -version`)
      * [SdkMan](https://sdkman.io/) provides convenient way to install and manage multiple Software Development Kits including JDK.
      * Alternatively, Windows, MacOS X, and Linux installer packages are available from [AdoptOpenJDK](https://adoptopenjdk.net/installation.html#installers).
* [Your Lightbend Cloudstate Account](https://docs.lbcs.dev/gettingstarted/account.html)
* [Creating a Project](https://docs.lbcs.dev/gettingstarted/project.html)

## Sample application layout

The sample application consists of 2 services:
* A stateless service `frontend`
* A stateful Entity based service `shopping-cart`

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

The `frontend` service is a frontend web application written in TypeScript. It is backed by a `stateless` service that will serve the compiled JavaScript, html and images. This service makes `grpc-web` calls directly to the other services to get the data that it needs.

#### Getting container image ready

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest` container image available at Lightbend Cloudstate samples repository.

Alternatively, you can clone the [cloudstateio/samples-ui-shoppingcart](https://github.com/cloudstateio/samples-ui-shoppingcart) repository and follow the instructions there to build an image and deploy it to your own container image repository.

#### Deploying the frontend service

##### Lightbend Cloudstate
```
csctl services deploy frontend lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest
```

##### Self-hosted Open-source Cloudstate
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

##### Lightbend Cloudstate
```
csctl stores deploy shopping-store
```

##### Self-hosted Open-source Cloudstate
Deploy the store to your project namespace
```
$ kubectl apply -f shopping-store.yaml -n <project-name>
statefulstore.cloudstate.io/shopping-store created
```

### Shopping Cart Service

```
cd ../shopping-cart
```

#### Building a container image

Edit `jib` section of `build.gradle` to specify the right registry and tag for the container image
```
jib {
  from {
    image = "adoptopenjdk/openjdk8:debian"
  }
  to {
    image = "<my-registry>/shopping-cart"
    tags = ["latest"]
  }
  container {
    mainClass = "io.cloudstate.samples.shoppingcart.Main"
    ports = ["8080"]
  }
}
```
You might also want to set up authentication for your container registry. Please refer to [Jib plugin documentation](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#authentication-methods) for further information.

NOTE: you can get a free public docker registry by signing up at [https://hub.docker.com](https://hub.docker.com/)

Build and push the container image to your container registry
```
./gradlew build jib
```

NOTE: This command builds and pushes the image directly to a container image repository bypassing local Docker (if it is present). However, it is possible to build the image using [Docker](https://www.docker.com/) with `./gradlew build jibDockerBuild`. Please refer to [Jib plugin documentation](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin#build-to-docker-daemon) for further information.

#### Deploying the service

##### Lightbend Cloudstate
```
csctl services deploy shopping-cart vkorenev/shopping-cart:java-0.1 --with-store shopping-store
```

##### Self-hosted Open-source Cloudstate
Deploy the image by changing into the deploy folder and editing `shopping-cart.yaml` to point to the docker image that you just pushed.
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
      name: shopping-store
  containers:
    - image:  lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest # <-- Change this to your repo/image
      name: shopping-cart
```

Deploy the service to your project namespace
```
$ kubectl apply -f shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
```

### Verify they are running
Check that the services are running

##### Lightbend Cloudstate
```
csctl services get
```

##### Self-hosted Open-source Cloudstate
```
$ kubectl get statefulservices -n <project-name>
NAME            AGE    REPLICAS   STATUS
shopping-cart   6m     1          Running
frontend        3m     1          Running
```

To redeploy a new image to the cluster you must delete and then redeploy using the yaml file.  
For example, if we updated the shopping-cart docker image we would do the following.
```
$ kubectl delete statefulservice shopping-cart -n <project-name>
statefulservice.cloudstate.io "shopping-cart" deleted
$ kubectl apply -f shopping-cart.yaml -n <project-name>
statefulservice.cloudstate.io/shopping-cart created
```

## Routes
The last thing that is required is to provide the public routes needed for both the frontend and grpc-web calls.  These exist in the `routes.yaml` file.

##### Lightbend Cloudstate
*TODO this no longer works since kubectl is not supposed to be used anymore*
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

Then expose your frontend
```
csctl services expose frontend
```
In the output, watch for the hostname assigned to the frontend.

Open a web browser and navigate to:

`https://<frontend-hostname>/pages/index.html`

You should now see the shopping cart interface.

##### Self-hosted Open-source Cloudstate
*TODO add instructions for setting up routing*

## Maintenance notes

### License
The license is Apache 2.0, see [LICENSE](LICENSE).

### Maintained by
__This project is NOT supported under the Lightbend subscription.__

This project is maintained mostly by @coreyauger and @cloudstateio.

Feel free to ping the maintainers above for code review or discussions. Pull requests are very welcome–thanks in advance!


### Disclaimer

[DISCLAIMER.txt](DISCLAIMER.txt)
