## ---
slug: tq-devlog-1
title: PaaS devlog |#1
authors: [denis]
tags: [paas]
---

Devlog #1: github apps, docker builder, cdk8s

Today I want to share with you my next steps of creating PaaS from scratch.
On this page I will cover how I implement a basic deployment flow.

<!--truncate-->

The service has the first api call. It's a github webhook, when a merge to main/master comes I spin a container from it.

### Github apps

In order to listen users' repository changes it's not necessary to use oauth anymore.
I can create a [github app](https://docs.github.com/en/apps/overview), kinda register an app and let my users to install it.
This app will let me listen the events the users accepted.

To make the github webhook secure they provide sha in the headers so I implemented a [verifier](https://github.com/treenq/treenq/blob/main/pkg/crypto/signature.go).

### App definition

Usually users configure an app using a yaml or click buttons.
We don't consider terraform for now, it's also viable, but we want a quick solution.

I do plan implement a `yaml`, but now I want to focus on definition as code. I believe it's gonna give me more flexibility and give the users faster access to the app resources such as a database credentials.

So I expected a `tq` module in a go app like this:


```go
package tq

import (
	tqsdk "github.com/treenq/treenq/pkg/sdk"
)

func Build() (tqsdk.Space, error) {
	return tqsdk.Space{
		Key:    "treenq-poc",
		Region: "fra1",
		Service: tqsdk.Service{
			DockerfilePath: "Dockerfile",
			SizeSlug:       "basic-xxs",
			Name:           "treenq-poc-service",
			HttpPort:       8000,
			InstanceCount:  1,
		},
	}, nil
}
```

Eventually I want to come up with an idea how a user can map this model to the resources, but now I just want to retrieve the config a user defined such a Dockerfile path and the http port to expose.

So I did a builder package to generate a temporary folder, places this config in there, calls that `Build` func and reads the given config as a json output.

### Build an image

I found an official repo for [buildx](https://github.com/docker/buildx), it's in Go so I can embed it.
But it looks quite a big challenge to solve, it's even scary to imagine the investigation to start using buildx, I would need to:

- run buildx command locally with a buildx debugger
- catch the exact function that builds an image
- understand all the dependencies and the way to build it
- put it into my service and make it work

So now Im fine just call `docker build` from CLI.
I definitely want to get back to buildx, but no clue when.

After tagging the image I need to push it, so I added registry to my docker compose.
It couldn't be simpler:


```yaml
registry:
	image: registry:2.8.3
	ports:
		- "5000:5000"
```

### Deploy

Now I can deploy the image. This part is tricky. I couldn't really understand what Im gonna do here.

So I found [cdk8s](https://cdk8s.io/).
Initially I defined a testing definition.
Seems work, even can return me a yaml output.
Unfortunately, it brings node runtime under the hood, but it pays off now.

So Im taking the `tq` app definition I showed earlier and create a k8s [definition](https://github.com/treenq/treenq/blob/6919be57726109ff3f37ce484db3d2457b4cb01e/src/services/cdk/kube.go#L38) from it.
I do a quite simple deployment, a service and an ingress, and return the generated yaml.

Now I have to apply this yaml to the cluster.

**Cluster?**

Ok, I have to install a cluster.

### Local kube

There are plenty of options to choose, but I start caring about the resources I use.
My app started with postrgres, then registry, now it has a kube. The main issue I might be able to run a cluster in CI for integration tests, so I want to make my kube instance as small as possible.

I found out k3s is the smallest distrubtion. Even though it doesn't follow all the standards, strangers in the internet convinced me I shouldn't ever understand the difference.

So I found an interesting [guide](https://sachua.github.io/post/Lightweight%20Kubernetes%20Using%20Docker%20Compose.html) how to start a cluster locally and it worked well.

In the progress I needed to add some kubelet [arguments](https://github.com/treenq/treenq/blob/6919be57726109ff3f37ce484db3d2457b4cb01e/docker-compose.yaml#L28) and the flight is done.

### Deploy

I wish I could use CLI and call `kubectl apply`, but it's not gonna workout.
The goal to support a huge amount of clusters, so I accept kubeconfig dynamically as an argument.

The examples to make it I found only using regular rest api that kube provides.
But I don't need it, but definitions I have are in yaml.

So I found some dynamic client to create resources unsafely.
Since I expected to not now what resources I create/update it what I need.
Yes, still unsafe, but Im here.

Eventually I combine apimachinery and go client together to split the yaml definition by objects, unmarshal them and try creating or updating an object one by one.

Later on I want to understand better what I need to create or update, but I would need to implement infra diffs for it, it's not gonna happen soon.

### Test

On the testing step I discover my cluster can't pull an image since registry is not a TLS server.
It's solved proving a registry config to my cluster.
So I add another [volume](https://github.com/treenq/treenq/blob/6919be57726109ff3f37ce484db3d2457b4cb01e/docker-compose.yaml#L49) to my cluster to add the registries config


```yaml
mirrors:
  "registry:5000":
    endpoint:
      - "http://registry:5000"

configs:
  "registry:5000":
    tls:
      insecure_skip_verify: true
```