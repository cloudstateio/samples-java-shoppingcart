# Cloudstate Sample Shopping Cart Application

## Sample application structure

The sample application consists of 2 services:
* A stateless service `frontend`
* A stateful entity-based service `shopping-cart`

## Building container images

All the latest container images are available publicly at `lightbend-docker-registry.bintray.io/cloudstate-samples`. But feel free to build your own images from sources.

### Frontend service

The `frontend` service is a frontend web application written in TypeScript.
It is backed by a `stateless` service that will serve the compiled JavaScript, html and images. This service makes `grpc-web` calls directly to the other services to get the data that it needs.

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest` container image available at Lightbend Cloudstate samples repository.

Alternatively, you can clone the [cloudstateio/samples-ui-shoppingcart](https://github.com/cloudstateio/samples-ui-shoppingcart) repository and follow the instructions there to build an image and deploy it to your own container image repository.

### Shopping cart service

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart:latest` container image available at Lightbend Cloudstate samples repository.

Alternatively, you can build an image from the sources in the `shopping-cart` directory and push it to your own container image repository.

#### Prerequisites

* Install JDK (Java Development Kit) version 8 or later.
  * We recommend OpenJDK 1.8.0_252 or later. (Check with `java -version`)
  * [SdkMan](https://sdkman.io/) provides convenient way to install and manage multiple Software Development Kits including JDK.
  * Alternatively, Windows, MacOS X, and Linux installer packages are available from [AdoptOpenJDK](https://adoptopenjdk.net/installation.html#installers).

#### Building a container image

Edit `jib` section of `build.gradle` to specify the right registry and tag for the container image

```groovy
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

## Deploying to Lightbend Cloudstate

The following steps use `csctl` to deploy the application to [Ligtbend Cloudstate](https://docs.lbcs.io/).

If you're self-hosting Cloudstate, the instructions for deploying the sample shopping cart application are in the [`deploy` directory](./deploy/README.md)

### Prerequisites

* Get [Your Lightbend Cloudstate Account](https://docs.lbcs.io/gettingstarted/account.html)
* Install [csctl](https://docs.lbcs.io/getting-started/set-up-development-env.html)

### Login to Lightbend Cloudstate

```shell
$ csctl auth login
```

### Create a new project

```shell
$ csctl projects new sample-shopping-cart "Shopping Cart Sample"
```

Wait until you receive an email approving your project!

List projects:

```shell
$ csctl projects list
```

You should see the project listed:

```shell
  NAME                   DESCRIPTION            STATUS   ID
  sample-shopping-cart   Shopping Cart Sample   active   39ad1d96-466a-4d07-b826-b30509bda21b
```

You can change the current project:

```shell
$ csctl config set project 39ad1d96-466a-4d07-b826-b30509bda21b
```

### Deploy the frontend service

A pre-build container image of the frontend service is provided as `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend`.
If you have built your own container image, change the image in the following command to point to the one that you just pushed.

```shell
$ csctl svc deploy frontend lightbend-docker-registry.bintray.io/cloudstate-samples/frontend
```

### Create the store

The shopping cart stateful service relies on a stateful store. Create the store with the following command:

```shell
$ csctl store deploy shopping-store
```

Wait for the store to be created. Check the store status with

```shell
$ csctl stores get
```

Proceed when `STATUS` is `ready`, this can take some time.

### Deploying the shopping cart service

A pre-build container image of the frontend service is provided as `lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart`.
If you have built your own container image, change the image in the following command to point to the one that you just pushed.

```shell
$ csctl svc deploy \
    shopping-cart \
    lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart \
    --with-store shopping-store
```

Wait for the shopping cart service `STATUS` to be `ready`.

```shell
$ csctl svc get
```

### Expose the frontend service

```shell
$ csctl svc expose frontend
```

The output will look like this:

```shell
Service 'frontend' was successfully exposed at: small-fire-5330.us-east1.apps.lbcs.io
```

Make a note of the hostname since it will be used to expose other services on the same host.

### Expose the shopping-cart service

```shell
$ csctl svc expose shopping-cart \
  --hostname small-fire-5330.us-east1.apps.lbcs.io \
  --uri-prefix=/com.example.shoppingcart.ShoppingCart/
```

### Visit the deployed shopping-cart frontend

The sample shopping cart is live. The frontend lives on the hostname previously
generated when deploying the frontend. Append `/pages/index.html` to the
provided hostname to see the shopping-cart frontend.

In the example above, the URL would be:
```
https://small-fire-5330.us-east1.apps.lbcs.io/pages/index.html
```

## Maintenance notes

### License
The license is Apache 2.0, see [LICENSE](LICENSE).

### Maintained by
__This project is NOT supported under the Lightbend subscription.__

This project is maintained mostly by @coreyauger and @cloudstateio.

Feel free to ping the maintainers above for code review or discussions. Pull requests are very welcome–thanks in advance!


### Disclaimer

[DISCLAIMER.txt](DISCLAIMER.txt)
