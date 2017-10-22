---
layout: post
title:  "Container Networking - Veth"
date:   2017-10-14
categories: cni namespace veth linux network
---

I've been playing a lot with Kubernetes lately. After migrating some projects to it, and using it for a while, I've decided to investigate a bit of it's building blocks.

Kubernetes has a lot of exciting topics to study, but at the base of it, there are the containers technologies (e.g., docker, rkt, cri-o), and if you go a level down on those technologies you see that containers don't exists, but that they are a mix of primitives in the Linux kernel used together.

One of this primitives is namespaces, a basic view of it is that they restrict what a process can see, so the process thinks it has the sole system for itself. There are a few namespaces available, but the little experiment I'm going to do in this post is using the network namespace.

Network namespace provides isolation of the resources associated with networking. A new namespace will have its own interfaces, routing table, firewall rules, etc.

The question I was asking myself was "<i>How does 'containers' communicate with each other?</i>" as with everything in computers multiple solutions can be used to address the same problem.

A good mental model to what we want to do connecting two containers: <i>You have two computers and want to wire them with an ethernet cable to be able to communicate back and forth</i>.

**I'm running ubuntu xenial64 with vagrant+virtualbox**

To start, we now need two <s>containers</s>, I mean network namespaces.

Let's first check if we have any existing namespace; we can use the ip tools for it:

{% highlight bash %}
$ ip netns list
(no output)
{% endhighlight %}

Now we create two new namespaces:

{% highlight bash %}
$ sudo ip netns add ns1
$ sudo ip netns add ns2
{% endhighlight %}

If we list again, we now see the two namespaces:

{% highlight bash %}
$ ip netns list
ns2
ns1
{% endhighlight %}

We can verify that those namespaces are isolated, and have their own view of network resources. Running `ip addr` on my ubuntu VM will list all addresses on a device. On my VM I can see three addresses.

{% highlight bash %}
$ ip addr
1: lo # only listing the interface name, not the full output
2: enp0s3 # only listing the interface name, not the full output
3: enp0s8 # only listing the interface name, not the full output
{% endhighlight %}

If you run the same command on the new namespaces, it will only return the address for the loopback interface, because it has an isolated view of the system, not seen what the existing root namespace have configured.

{% highlight bash %}
$ sudo ip netns exec ns1 ip addr
1: lo # only listing the interface name, not the full output
{% endhighlight %}

{% highlight bash %}
$ sudo ip netns exec ns2 ip addr
1: lo # only listing the interface name, not the full output
{% endhighlight %}

Now that we can simulate the two <s>computers</s> - namespaces, we can move to the next step, wiring them for communication, and to do this we need a cable.
Linux has support for a virtual network device **veth**.  We can think that veth is the cable we will use to wire the two computers.
Veth always comes in pairs, and whatever is sent to one side of it will pass to the other side.

To create a veth we also use the aid of the ip tools:

{% highlight bash %}
$ sudo ip link add v1 type veth peer name v2
{% endhighlight %}

With this command we have created a veth, one side of it is named 'v1' and the peer side is named 'v2'.

Listing the interfaces, we can see the new veth just created:

{% highlight bash %}
$ ip link
// ... existing interfaces
4: v2@v1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ba:22:31:bf:fa:04 brd ff:ff:ff:ff:ff:ff
5: v1@v2: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether be:9f:a4:08:c8:9e brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

Continuing on our metaphor, next we need to plug our cable
in both our computers. Each side of the veth goes on one
network namespace, to simulate our wiring.

{% highlight bash %}
$ sudo ip link set v1 netns ns1
{% endhighlight %}

{% highlight bash %}
$ sudo ip link set v2 netns ns2
{% endhighlight %}

Now our my local VM won't display the veth if I list the interfaces. But we can see them listing the interfaces of each namespace.

{% highlight bash %}
$ ip link
# (The veth don't appear locally anymore)
{% endhighlight %}

{% highlight bash %}
$ sudo ip netns exec ns1 ip link
1: lo
10: v1 # one side of the veth
{% endhighlight %}

{% highlight bash %}
$ sudo ip netns exec ns2 ip link
1: lo
10: v2 # the other side of the veth
{% endhighlight %}

The last part to have our two <s>computers</s> - namespaces communicate is to configure the network, each one will have an ip address, and their interfaces must be up.

{% highlight bash %}
$ sudo ip netns exec ns1 ip link set v1 up
$ sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev v1
{% endhighlight %}

{% highlight bash %}
$ sudo ip netns exec ns2 ip link set v2 up
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev v2
{% endhighlight %}

We can now ping one namespace from the other.

{% highlight bash %}
$ sudo ip netns exec ns1 ping 10.0.0.2
{% endhighlight %}

{% highlight bash %}
$ sudo ip netns exec ns2 ping 10.0.0.1
{% endhighlight %}

But a better example is to start a bash process inside each namespace, and create a simple chat using netcat.

Start a bash process inside the namespace ns1:

{% highlight bash %}
$ sudo ip netns exec ns1 bash
{% endhighlight %}

And start a netcat server on the port 8080:

{% highlight bash %}
$ nc -l -p 8080
# write something here (after starting the client)
{% endhighlight %}

In a **different terminal window**, start a bash process in the second namespace:

{% highlight bash %}
$ sudo ip netns exec ns2 bash
{% endhighlight %}

And connect to the other side using netcat:

{% highlight bash %}
$ nc 10.0.0.1 8080
# write something here
{% endhighlight %}

And that's it we did create two network namespace and connected them with a virtual network device (veth).


To clean up, deleted the namespaces:

{% highlight bash %}
$ sudo ip netns del ns1
$ sudo ip netns del ns2
{% endhighlight %}

Quite simple huh? On a next article, I’m going to connect containers using a bridge. :)
