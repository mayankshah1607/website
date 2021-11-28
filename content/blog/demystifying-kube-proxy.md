---
title: "Demystifying kube-proxy"
date: 2021-11-20T09:19:29-04:00
description: "This blog post explains how the kube-proxy component of Kubernetes works internally. It goes through various laters of networking abstractions in a top-down approach, ultimately leading to kube-proxy."
keywords: ["kubernetes", "networking"]
draft: false
tags: ["kubernetes", "networking"]
math: false
toc: true
---
Lately, I've been spending some time reading and trying to understand the internals of networking in Kubernetes (purely out of curiosity). One piece of this puzzle that I was particulary interested in learning is how `kube-proxy` works. That is what we'll be looking at in this blog post. I'll try to explain things in a "top-down" fashion; we'll go through various layers of networking abstractions, which will ultimately lead us to the `kube-proxy` itself.

## How to read this blog post?

To get the best out of this blog post, you **need** to have a fair amount of understanding of Kubernetes, container networking, and Linux networking. These topics are vast, and unfortunately, I'm not going to be covering them in-depth here. I'll link some of my favorite resources at the end of this post.

This post is not a "kube-proxy hands-on tutorial". Its mostly just simple explanations, examples and figures/drawings. I've tried my best to keep this post as simple (and concise) as possible and only focus on topics that are directly relevant to the functioning of `kube-proxy`. I can provide the following options to help you navigate your way through this post:

- _" I'd like to start from the top and work my way to the bottom"_ - start right away from the [next](#overview-of-the-kubernetes-networking-model) section which talks about the Kubernetes networking model.

- _" I'm already familiar with Kubernetes networking"_ - skip to the [iptables](#iptables) section to understand how `kube-proxy` leverages some Linux networking features internally.

- _" I just want to know how `kube-proxy` really works"_ - skip to the [kube-proxy](#kube-proxy) section.

- _" I read this post, and now I'm interested in learning about Kubernetes networking in greater depth"_ - awesome! Jump to the [extra reading](#extra-reading) section. This list is not exhaustive but should definitely help you.

> _Most figures in this post have been simplified to a great extent to hide the underlying complexities of Linux and container networking. This has been done intentionally for ease of understanding._

## Overview of the Kubernetes networking model

Let us first try to understand how two Pods can communicate in a Kubernetes cluster. 

For any two Pods to talk to each other, they need to be assigned _unique_ IP addresses. These IP addresses have to be unique across the cluster. Pods should be able to reach each other using these IP addresses, without relying on _network address translation_, regardless of the host they're running on. 

"How does Kubernetes do that?", I hear you ask. Well, it _kind of_ doesn't. There are a hundred different ways of achieving this, each with a different use-case in mind. And Kubernetes can't possibly implement each of these networking solutions natively. Instead, Kubernetes dictates some networking requirements (as we saw earlier), and it is the responsobility of _CNI_ (plugins) to ensure that these requirements are met.

Some CNI plugins do a lot more than just ensuring Pods have IP addresses and that they can talk to each other. I'm not going to explain CNI in depth because it is another vast topic in itself, and I won't do justice by trying to explain it in a single paragraph. But the crucial thing to remember here is that CNI ensures that Pods have [L3 connectivity](https://osi-model.com/network-layer/) without relying on NAT.

![sdf](https://i.imgur.com/PxVO432.png "Figure 1. Pod connectivity in Kubernetes (note that the implementation details of container-level networking has not been shown here)")

### Services

At this point, we've understood how two Pods talk to each other using their IP addresses. However, because of the ephemeral nature of Pods, it is almost never a good idea to directly use Pod IP addresses. Pod IP addresses are not persisted across restarts and can change without warning, in response to events that cause Pod restarts (such as application failures, rollouts, rollbacks, scale-up/down, etc.).

Moreover, when you have multiple Pod replicas being managed by some parent object such as a _Deployment_, keeping track of all the IP addresses on the client-side could introduce significant overhead.

Services were hence introduced to address this set of problems.

Kubernetes Service objects allow you to assign a single virtual IP address to a set of Pods. It works by keeping track of the state and IP addresses for a group of Pods and proxying / load-balancing traffic to them (later we'll see that the Service itself does not proxy traffic and how the `kube-proxy` is actually responsible for this).

With this, the client can now use the Service IP address instead of relying on the individual IP addresses of the underlying Pods.

![](https://i.imgur.com/0OpHnYU.png "Figure 2. Services in Kubernetes")

> _Pods can also use internally available DNS names instead of Service IP addresses. This DNS system is powered by [CoreDNS](https://coredns.io/)._
### Endpoints

We now know that Services proxy and load-balance incoming traffic to the desired Pod(s). But how do Services know which Pods to track, and which Pods are ready to accept traffic? The answer is _Endpoints_.

Endpoints have a 1:1 relationship with Services, and are created for keeping track of IP addresses of running Pods their corresponding Services are proxying for. They're always kept in sync with the state and IP addresses of their target Pods. You can think of Endpoints as a lookup table for Services to fetch the target IP addresses of Pods.

![](https://i.imgur.com/Wf6tVO4.png "Figure 3. Extending figure 2 to demonstrate Endpoints in Kubernetes")

In the example above (Figure 3), _Pod B3_ is in an unready state and that information is reflected onto its parent Endpoint object. That's how _Service B_ knows **not to** send any traffic to _Pod B3_.

> _It might be interesting to note that `Endpoint`s do not scale well with the size of the cluster and the number of Services running on it. `EndpointSlice`s were introduced to tackle issues around scalability with `Endpoint`s. Read more about it [here](https://kubernetes.io/docs/concepts/Services-networking/Endpoint-slices/)._

In a moment you'll see how Endpoints and Services are relevant to `kube-proxy`. But before that, let's quickly brush through the basics of some Linux networking features that `kube-proxy` heavily relies on.

## iptables

iptables is an extremely useful Linux firewall utility that allows you to control the packets that enter the kernel. For example, using iptables, you can tell your Linux kernel to drop all packets coming from IP address `12.32.81.101`. Or, you can tell it to change the source IP address of all packets to `127.0.0.1` and make it seem like they're coming from localhost. It does this by interfacing with the [`netfilter`](https://www.netfilter.org/) module of the Linux kernel, which lets you intercept and mutate packets.

`iptables` is essential to understanding how `kube-proxy` works. They're not exactly the most interesting thing to work with, and because of that, I'm going to keep this section short. The important thing to understand here is that iptables allow you to program how the Linux kernel should treat packets. Later we'll see how `kube-proxy` makes use of this.

> _You'll find several blog posts and tutorials on the internet that do a fantastic job at explaining iptables (I'll link them at the end of this post)._


## IPVS

While iptables are undoubtedly super useful, they can sometimes become inefficient in the presence of a large number of routing rules. iptables evaluate rules sequentially. They're essentially a large block of _if-else_ conditions. This could become hard to scale as the number of rules grows. Moreover, they offer very limited functionality in terms of L4 load-balancing.

IPVS (which stands for _IP Virtual Server_) was introduced to tackle such challenges. It offers multiple load-balancing modes (such as round-robin, least connection, shortest expected delay, etc.), allowing traffic to be distributed more effectively in contrast to iptables. IPVS also has support for session affinity which can be useful for scenarios such as maximizing cache hits. Once again, I’m not going to go too much into detail here, but please feel free to explore IPVS in greater depth.  In the next section, we'll combine everything we've learned so far and understand how `kube-proxy` works.

## kube-proxy

`kube-proxy` runs on each node of a Kubernetes cluster. It watches `Service` and `Endpoint` (and `EndpointSlice`) objects and accordingly updates the routing rules on its host nodes to allow communicating over Services. `kube-proxy` has 4 modes of execution - `iptables`, `ipvs`, `userspace` and `kernelspace`. The default mode (as of writing this blog post) is `iptables`, and somewhat tricker to understand compared to the rest. Lets understand this mode using a simple example.

Assume that you have a Kubernetes cluster with 2 nodes, each running `kube-proxy` in the `iptables` mode. Also assume that this cluster has 2 Pods, _Pod A_ (the client) and _Pod B_ (the server), each with its own unique IP address (thanks CNI plugin!), running on either nodes. The figure below (Figure 4) illustrates our imaginary environment.

> _Before proceeding, I'd like to remind once again that the figures below do not show the implementation details of container-level networking, i.e, how a packet reaches the `eth0` interface from a Pod._

![](https://i.imgur.com/vQIgFl6.png "Figure 4. Representation of our imaginary Kubernetes environment")

So far there's no involvement from `kube-proxy` as _Pod A_ can directly talk to _Pod B_ using the assigned IP address. Let us now expose _Pod B_ over a `Service` (we'll call it _Service B_) as shown in _Figure 5_.

![](https://i.imgur.com/XZjxAi4.png "Figure 5. Exposing Pod B using a Service")

This is where things get interesting. _Pod A_ can now reach _Pod B_ at its Service IP address (i.e, 10.0.1.3 in this example), regardless of the state and IP address of the underlying Pod. To understand how `kube-proxy` handles this for us, we'll analyse the lifecycle of a packet in two parts: _Pod to Service_ (client to server) and _Service to Pod_ (server to client).

### Pod to Service

When we created _Service B_, the first thing that happened was the creation of a corresponding _Endpoint_ object that stores a list of Pod IP addresses to forward traffic to. 

Once the Endpoint was updated with the correct IP address of the Pod, all _kube-proxies_ updated the _iptables_ on their host nodes with a new rule. I won't dive into the technical details of this new iptables rule, but here's how it translates in English - _"All outgoing packets to 10.0.1.3 (IP address of Service B), should instead go to 10.0.1.2 (IP addresss of Pod B, as obtained from the Endpoint object)"._

![](https://i.imgur.com/aCFA30P.png "Figure 6. The packet originating from Pod A has its destination IP addresss re-written with the help of iptables")

> _The process of changing the destination IP address of a packet is commonly referred to as DNAT (stands for Destination Network Address Translation)._

Interesting, right? Well, the story doesn't end here. Continue reading to understand how `kube-proxy` handles the response back from _Pod B_.

### Service to Pod

Now that _Pod B_ successfully receives the request from _Pod A_ (via _Service B_), its going to have to send a response back. This may sound pretty straight forward - _Pod B_ already knows the source IP address from the packet, so just use that to send a response back. But, there's a small catch.

_Pod A_ would expect a response only from the IP address it intended to send the packet to, i.e, _Service B_. In this case however, _Pod A_ receives a response from _Pod B's_ IP address. As a result, _Pod A_ is going to drop this packet.

To solve this problem, _kube-proxies_ write yet another iptables rule on their host node.  I won't get into the technical details of this iptables rule, but in English, it would translate to - _"If you (iptables / Linux kernel) re-wrote the destination IP address of outgoing packets (to that of the Service), **please** remember to also re-write the source IP address (to that of the _Service_) for incoming response packets. Thanks!"_

> _iptables use a Linux feature called _conntrack_ to remember the routing choices it previously made. That's how iptables remember to re-write the source IP address of incoming packets._

![](https://i.imgur.com/qcHzFNw.png "Figure 7. The packet arriving from Pod B has its source IP address modified, so that Pod A doesn't freak out")

> _The process of changing the source IP address of a packet is commonly referred to as SNAT (stands for Source Network Address Translation)._

And that's pretty much how `kube-proxy` works in the iptables mode! 

## Some final thoughts

While it may look like magic from the outside, the mechanisms used by Kubernetes to handle compex networking tasks are rather interesting to learn. Networking is a field that is just as complex as it is deep, and covering every concept in a single blog post would be nearly impossible. I plan on writing about other areas of networking in Kubernetes, in the near future. These topics will include CNI, DNS, Ingress and hopefully a lot more.

Thank you so much for taking the time to read this post. If you found it helpful, please do consider sharing it with your friends and colleagues. Feedback and (constrictive) critisism is also always more than welcome.

## Extra reading

- [A Guide to the Kubernetes Networking Model](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/) by _Kevin Sookcheff_
- [Networking and Kubernetes: A Layered Approach](https://www.oreilly.com/library/view/networking-and-kubernetes/9781492081647/) by _James Strong_ and _Vallery Lancey_
- [A container networking overview](https://jvns.ca/blog/2016/12/22/container-networking/) by _Jula Evans_
- [Network containers](https://docs.docker.com/engine/tutorials/networkingcontainers/)
- [An In-Depth Guide to iptables, the Linux Firewall](https://www.booleanworld.com/depth-guide-iptables-linux-firewall/) by _Supriyo Biswas_
- [iptables man page](https://linux.die.net/man/8/iptables)
- [Introduction to Linux interfaces for virtual networking](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking) by _Hangbin Liu_
