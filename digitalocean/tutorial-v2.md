# How to Develop on Kubernetes with Okteto

Developing applications for Kubernetes could affect your development productivity when you want to test and debug your applications on Kubernetes. Your traditional inner loop increases as you have to at a minimum: pack your application into a container, push it to a container registry, then tell Kubernetes to pull that image. All of this could easily take five minutes–even more.

In this blog post, you’ll learn how to improve your productivity when developing Kubernetes-native apps. First, you'll create a Kubernetes cluster in DigitalOcean and use it to run a "Hello World" application. You’ll use Okteto to develop and automatically update your application without having to install anything locally–without the need to install compilers, runtimes, Docker or Kubernetes locally 😍. Kubernetes development provides a replicable environment that’s as close to production as possible. Additionally, with Okteto, your inner development loop improves, and that’s crucial for developer productivity.

Enough words, let's get into it!

## Pre-Requisites

Before you begin this tutorial, you'll need the following:

1. [A Kubernetes cluster running in Digital Ocean](https://www.digitalocean.com/docs/kubernetes/quickstart/).
1. `kubectl` [installed and configured](https://www.digitalocean.com/docs/kubernetes/how-to/connect-to-cluster/) to communicate with your cluster.
1. Some familiarity with Kubernetes.

## Step 1: Create the Hello World Application

Let's start with creating a simple "Hello World" in Golang and its manifests you'll use to deploy the app on Kubernetes.

Create a new folder for your application by executing the following command:

```console
$ cd ~
$ mkdir hello_world
$ cd hello_world
```

Open a new file under the name `main.go` with your favorite IDE or text editor. For instance, using *nano* like this:

```console
$ nano main.go
```

`main.go` will be a Golang web server that returns the message *Hello world!*. So, let's use the following code:

```golang
package main

import (
    "fmt"
    "net/http"
)

func main() {
    fmt.Println("Starting hello-world server...")
    http.HandleFunc("/", helloServer)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}

func helloServer(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello world!")
}
```

You need to create a Docker image and push it to the Docker registry so that Kubernetes can pull it, and run the application. To do so, simply use the following Dockerfile in the same folder: 

```yaml
FROM golang:alpine as builder
RUN apk --update --no-cache add bash
WORKDIR /app
ADD . .
RUN go build -o app

FROM alpine as prod
WORKDIR /app
COPY --from=builder /app/app /app/app
EXPOSE 8080
CMD ["./app"]
```

Then, run the following commands to build and push the Docker image. Replace `{{USERNAME}}` with your Docker Hub username:

```console
docker build -t {{USERNAME}}/hello-world:latest .
docker push {{USERNAME}}/hello-world:latest
```

Next, create a new folder for the Kubernetes manifests by executing the following command:

```console
$ mkdir k8s
```

When you use a Kubernetes manifest, you tell Kubernetes how you want your application to run. This time, you'll create a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) object. So, create a new file `deployment.yaml` with your favorite IDE or text editor. For instance, using *nano* like this:

```console
$ nano k8s/deployment.yaml
```

The following content describes a Kubernetes deployment object that runs the `okteto/hello-world:latest` Docker image. In your case, use the name you used when building the Docker image previously (`{{USERNAME}}/hello-world:latest`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: okteto/hello-world:latest
        ports:
        - containerPort: 8080
```

You'll now need a way to access your application. You can expose an application on Kubernetes by creating a [service](https://kubernetes.io/docs/concepts/services-networking/service/) object. Let's continue using manifests. Create a new file `service.yaml` with your favorite IDE or text editor. For instance, using *nano* like this:

```console
$ nano k8s/service.yaml
```

The following content describes a service that exposes the `hello-world` deployment object, which under the hood will use a DigitalOcean load balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  selector:
    app: hello-world
```

Now you have everything ready to deploy the "Hello World" application on Kubernetes.

## Step 2: Deploy the Hello World Application

Besides deploying the "Hello World" application on Kubernetes, you'll validate that it is working correctly. 

So, start by deploying your application on Kubernetes by running the following command:

```console
$ kubectl apply -f k8s
deployment.apps "hello-world" deleted
service "hello-world" deleted
```

After about one minute, you will be able to get the IP of your application from the `EXTERNAL-IP` column when listing the Kubernetes service objects:

```console
$ kubectl get service hello-world
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP      PORT(S)    AGE
hello-world   ClusterIP   10.0.9.181   134.209.139.198  8080/TCP   37s
```

Open your browser and go to the IP of the "Hello World" application (`134.209.139.198` from the above example). Confirm that your application is up and running before continuing with the next step.

Until this moment, what you've done is a traditional way of developing applications with Kubernetes. If you want to change the code in your application, you'll have to build and push the Docker image. Then, pull that image from Kubernetes. All this process could take easily five minutes–or even more. So, let's use Okteto to improve the development inner-loop when working with Kubernetes.

## Step 3: Install Okteto CLI

This step is where you'll be able to improve your development productivity by installing the Okteto CLI. The [Okteto CLI](https://github.com/okteto/okteto) is an open-source project that lets you synchronize application code changes to an application running on Kubernetes. You can continue using your favorite IDE, debuggers, or compilers without having to commit, build, push, or redeploy containers to test your application–as you did in a previous step.

To install the Okteto CLI in macOS or Linux, run the following command:

```console
$ curl https://get.okteto.com -sSfL | sh
```

Or if you are in Windows, download the file https://downloads.okteto.com/cli/okteto.exe and add it to your `$PATH`.

Once the Okteto CLI is installed, you are ready to put the "Hello World" application in development mode.

## Step 4: Put the Hello World Application in Development Mode

Now, Okteto will swap the application running on a Kubernetes cluster with the code you have in your machine. To so, Okteto uses the information provided from an [Okteto manifest](https://okteto.com/docs/reference/manifest/index.html) file. This file declares the Kubernetes deployment object that will swap with your local code. 

Create a new file `okteto.yaml` with your favorite IDE or text editor. For instance, using *nano* like this:

```console
$ nano okteto.yaml
```

Let's use a simple manifest where you define the deployment object name, the Docker base image to use, and a shell–more on this later. Use the following sample content file:

```yaml
name: hello-world
image: okteto/golang:1
command: ["bash"]
```

Prepare to put your application in development mode by running the following command:

```console
$ okteto up
```

```console
 ✓  Development environment activated
 ✓  Files synchronized
    Namespace: default
    Name:      hello-world

Welcome to your development environment. Happy coding!
okteto>
```

The `okteto up` command swaps the "Hello World" application by a development environment, which means:

- The Hello World application container is updated with the docker image `okteto/golang:1`. This image contains the required dev tools to build, test, debug, and run the "Hello World" application.
- A [file synchronization service](https://okteto.com/docs/reference/file-synchronization/index.html) is created to keep your changes up-to-date between your local filesystem and your application pods.
- A remote shell starts in your development environment. Build, test, and run your application as if you were in your local machine.
- Whatever process you run in the remote shell will get the same incoming traffic, the same environment variables, volumes, or secrets than the original "Hello World" application pods–giving you a production-like development environment 💥. 

In the same console, run the application as you would typically do (without building and pushing a Docker image), like this:

```console
okteto> go run main.go
```

```console
Starting hello-world server...
```

The first time you run the application, Go will download your dependencies and compile your application. Wait for this process to finish and test your application by opening your browser and refreshing the page of your application–as you did previously.

## Step 5: Develop directly on Kubernetes

Let's start doing changes to the "Hello World" application to see how these changes are reflected in Kubernetes.

Open the `main.go` file with your favorite IDE or text editor. For instance, using *nano* like this:

```console
$ nano main.go
```

Then, change the response message to be *Hello world from DigitalOcean!*:

```golang
package main

import (
    "fmt"
    "net/http"
)

func main() {
    fmt.Println("Starting hello-world server...")
    http.HandleFunc("/", helloServer)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        panic(err)
    }
}

func helloServer(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello world from DigitalOcean!")
}
```

We'll do things differently now. Instead of building images and redeploying containers to update the "Hello World" application, Okteto will synchronize your changes to your development environment on Kubernetes. 

Cancel the execution of `go run main.go` from the remote shell by pressing `ctrl + c` and rerun the application:

```console
okteto> go run main.go
```

```console
Starting hello-world server...
```

Go back to the browser and reload the page of the "Hello World" application.

Cool! Your code changes were instantly applied to Kubernetes. 

No commit, build, or push required 😎!

## Conclusions

Kubernetes has the potential to be a great development platform, providing replicable, resource-efficient, and production-like development environments. We have shown you how to use [Okteto](https://github.com/okteto/okteto) to iterate your code changes directly on Kubernetes as fast as you can type code. Head over to the [Okteto samples repository](https://github.com/okteto/samples) to see how to use Okteto with different stacks and debuggers.

If you share a Kubernetes cluster with your team, we recommend to give each member access to a sandboxed Kubernetes namespace, configured automatically to be isolated from other developers working on the same cluster. This functionality is provided by the [Okteto](https://marketplace.digitalocean.com/apps/okteto-1) application in the [DigitalOcean Kubernetes marketplace](https://marketplace.digitalocean.com/category/kubernetes). 

[Okteto](https://www.okteto.com) transforms your Kubernetes cluster in a fully-featured development platform with the click of a button 😎.