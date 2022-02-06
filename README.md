# Enhanced Kubernetes Network Policies with Cilium

## The Promise of Microservices

There are two main benefits of using microservices architecture.

1. Enable distributed teams to work independently on parts of a distributed system, thus making [Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law) work in our favour.
2. Tame distributed systems complexity by **offloading more and more cross-cutting concerns from application code to the underlying infrastructure.**

In the world where microservices architecture for server side workloads is the dominating paradigm and more and more compute runs on Kubernetes, we have a chance to truly fulfill the microservices technical and organizational promise.

This article will show how to, using Cilium, tackle the authorization concern and move push it to the underlying platform from the application code.

## What is Cilium?

Cilium is an open source software for providing, securing and observing network connectivity between container workloads - cloud native, and fueled by the revolutionary Kernel technology eBPF.
<p style="text-align: center;"><small>Source: https://cilium.io/</small></p>

> If you want to try Cilium yourself, check out their excellent [Interactive Tutorial](https://play.instruqt.com/isovalent/tracks/cilium-getting-started)

### What is eBPF?

> eBPF is a revolutionary technology with origins in the Linux kernel that can run sandboxed programs in an operating system kernel. It is used to safely and efficiently extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules. To learn more about eBPF, visit [Introduction to eBPF](https://ebpf.io/what-is-ebpf/)
<p style="text-align: center;"><small>Source: https://ebpf.io/what-is-ebpf/</small></p>

Here is a [great video](https://www.youtube.com/watch?v=5t7-HM2jlTM&ab_channel=ContainerSolutions) with Liz Rice explaining eBPF in detail.

Below diagram shows how eBPF works on a high level

![ebpf-overview](_media/ebpf-overview.png)
<p style="text-align: center;"><small>Source: https://ebpf.io/what-is-ebpf</p>

## Granular Authorization Control

Let's imagine a scenario where your REST API consists of multiple endpoints exposing a flight booking related resources. This REST API is deployed to a managed K8s cluster, let's say GKE (Google Kubernetes Engine) and is often accessed by other microservices running in the cluster as well as some external services.

From a security point of view you want to follow [Zero Trust Security](https://en.wikipedia.org/wiki/Zero_trust_security_model) and the [Principle of Least Privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) and to achieve this you need to tightly control and verify access to your API and only expose those endpoints that are essential for calling services and not more.

Kubernetes Network Policies can take us half way there.

## Network Policies

Kubernetes [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) define network traffic rules for pods running in a cluster.

Below diagram shows more information about network policies.

> We are going to focus on [Cilium](https://cilium.io/) and show how it can provide enhanced and more powerful policies

![Network Policies](diagrams/rendered/k8s-network-policy.png)

However there is one problem. Our flights booking service exposes multiple REST endpoints and Kubernetes Network policies work only on IP:PORT combination. This means that each service running in the cluster will have access to all endpoints even if it doesn't need it. This clearly violates the Principle of least privilege.

## Network Policies Improved

Cilium addresses this issue by introducing a [CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) _CiliumNetworkPolicy_ which adds the missing functionality and enables us to decoratively create rules governing access to various endpoints of our API.

> As a side note, from architectural point of view, the same could be achieved with an API Management Gateway such as [KONG](https://konghq.com/kong/), but this is a different approach and works only with HTTP services, whereas Cilium being a lower level solution supports Kafka, Databases and more.

Here is a sample CiliumNetworkPolicy YAML file strictly allowing only traffic from pods with selected labels to use GET verb on the /flights resource.

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "readFlightsRule"
spec:
  description: "Allow HTTP GET /flights from env=prod, app=flights_board to app=flights_service"
  endpointSelector:
    matchLabels:
      app: flights_service
  ingress:
  - fromEndpoints:
    - matchLabels:
        env: prod
        app: flights_board
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/flights"
```

Cilium also supports [DNS Policy](https://docs.cilium.io/en/stable/policy/language/#dns-policy-and-ip-discovery) where we can for example allow only incoming external traffic from a load balancer address or pattern matching to a name of service hosted externally.

## What are the benefits

Setting rules on HTTP level (Layer 7) enables offloading of the authorization concerns for APIs to Kubernetes instead of encoding the rules in the application itself.
The benefits of this approach are:

- ability to change the authorization rules independently from application code base development
- possibility to separate application code pipeline from the rules pipeline enabling teams to collaborate
- ability to deploy another instance of the same API in a container image, but with different labels and rules which may depend on namespaces or different conditions
- standardized and centrally controlled security aspect

# Demo Scenario

## Prerequisistes

- minikube > v.1.12.1 or greater
- kubectl
- helm v3

> If you need help installing Cilium, please refer to their [excellent documentation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/).

## Steps

Create minikube cluster

```bash
minikube start --network-plugin=cni --cni=false
```

Install CLI

```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum

sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

rm cilium-linux-amd64.tar.gz{,.sha256sum}
```

Install Cilium on the minikube cluster

```bash
cilium install
```

Deploy sample go api
```bash

```


