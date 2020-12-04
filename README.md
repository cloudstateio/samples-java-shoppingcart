# Cloudstate Sample Shopping Cart Application

## Sample application structure

The sample application consists of two services:
* A stateless service `frontend`
* A stateful entity-based service `shopping-cart`

## Building container images

All the latest container images are available publicly at `lightbend-docker-registry.bintray.io/cloudstate-samples`. Feel free to build your own images from sources.

### Frontend service

The `frontend` service is a frontend web application written in TypeScript.
It is backed by a `stateless` service that will serve the compiled JavaScript, html and images. This service makes `grpc-web` calls directly to the other services to get the data that it needs.

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest` container image.

Alternatively, you can clone the [cloudstateio/samples-ui-shoppingcart](https://github.com/cloudstateio/samples-ui-shoppingcart) repository and follow the instructions there to build an image and deploy it to your own container image repository.

### Shopping cart service

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart-java:latest` container image.

Alternatively, you can build an image from the sources in the `shopping-cart` directory and push it to your own container image repository.

#### Prerequisites

* Install JDK (Java Development Kit) version 8 or later.
  * We recommend OpenJDK 1.8.0_252 or later. (Check with `java -version`)
  * [SdkMan](https://sdkman.io/) provides convenient way to install and manage multiple Software Development Kits including JDK.
  * Alternatively, Windows, MacOS X, and Linux installer packages are available from [AdoptOpenJDK](https://adoptopenjdk.net/installation.html#installers).

#### Building a container image
##### Option 1: Using gradle

Edit `jib` section of `build.gradle` to specify the right registry and tag for the container image

```groovy
jib {
  from {
    image = "adoptopenjdk/openjdk8:debian"
  }
  to {
    image = "<my-registry>/shopping-cart-java"
    tags = [version]
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

##### Option 2: Using maven
1. modify maven config file `shopping-cart/pom.xml`
Please modify your docker repo, and generated tag accordingly.
If you use original setting without any change, it generates image `my-docker-repo/shopping-cart-java:my-tag`
```
<image>
  <!-- Please change it to your docker repository info -->
  <name>my-docker-repo/shopping-cart-java:%l</name>
  <build>
    <!-- Base docker image -->
    <from>adoptopenjdk/openjdk8:alpine-jre</from>
    <tags>
      <!-- tag for generated image -->
      <tag>my-tag</tag>
    </tags>
```
2. run the following command to compile proto/java files and generates docker image
```
cd shopping-cart
mvn package
```
3. run command `docker images | head` to make sure the docker image is generated  correctly.
4. (Optional) you can test your image loading in your local docker by command
```
docker run my-docker-repo/shopping-cart-java:my-tag
```
You should see the server loads without error.
5. push your docker image to your docker repo. The commands are
```
docker login
docker tag <src_image_with_tag> <destination_image_with_tag>
docker push
```
NOTE: you can get a free public docker registry by signing up at [https://hub.docker.com](https://hub.docker.com/)

## Deploying to Akka Serverless

The following steps use `akkasls` to deploy the application to [Akka Serverless](https://docs.cloudstate.com/).

If you're writing Cloudstate services to deploy on your own Kubernetes cluster, the instructions for deploying the sample shopping cart application are in the [`deploy` directory](./deploy/README.md)

### Prerequisites

* Get [Your Akka Serverless Account](https://docs.cloudstate.com/getting-started/lightbend-account.html)
* Install [akkasls](https://docs.cloudstate.com/getting-started/set-up-development-env.html)

### Login to Akka Serverless

```shell
$ akkasls auth login
```

### Create a new Akka Serverless project

```shell
$ akkasls projects new sample-shopping-cart "Shopping Cart Sample"
```

Wait until you receive an email approving your project!

List projects:

```shell
$ akkasls projects list
```

You should see the project listed:

```shell
  NAME                   DESCRIPTION            STATUS   ID
  sample-shopping-cart   Shopping Cart Sample   active   39ad1d96-466a-4d07-b826-b30509bda21b
```

You can change the current project:

```shell
$ akkasls config set project sample-shopping-cart
```

### Deploy the frontend service

A pre-built container image of the frontend service is provided as `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend`.
If you have built your own container image, change the image in the following command to point to the one that you just pushed.

```shell
$ akkasls svc deploy frontend lightbend-docker-registry.bintray.io/cloudstate-samples/frontend
```

### Deploying the shopping cart service

A pre-built container image of the shopping cart service is provided as `lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart-java`.
If you have built your own container image, change the image in the following command to point to the one that you just pushed.

```shell
$ akkasls svc deploy shopping-cart \
    lightbend-docker-registry.bintray.io/cloudstate-samples/shopping-cart-java
```

Wait for the shopping cart service `STATUS` to be `ready`.

```shell
$ akkasls svc get
```

### Frontend service

The `frontend` service is a frontend web application written in TypeScript.
It is backed by a _stateless_ service that will serve the compiled JavaScript, HTML and images. This service makes `grpc-web` calls directly to the other services to get the data that it needs.

You can use the pre-built `lightbend-docker-registry.bintray.io/cloudstate-samples/frontend:latest` container image.

Alternatively, you can clone the [cloudstateio/samples-ui-shoppingcart](https://github.com/cloudstateio/samples-ui-shoppingcart) repository and follow the instructions there to build an image and deploy it to your own container image repository.

### Expose the shopping-cart service

```shell
$ akkasls svc expose shopping-cart --enable-cors
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

Feel free to ping the maintainers above for code review or discussions. Pull requests are very welcome.  Thanks in advance!

### Disclaimer

[DISCLAIMER.txt](DISCLAIMER.txt)
