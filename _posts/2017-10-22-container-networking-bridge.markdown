---
layout: post
title:  "Container Networking - Bridge"
date:   2017-10-14
categories: cni namespace bridge linux network
---

In the last [post](https://josedonizetti.github.io/cni/namespace/veth/linux/network/2017/10/14/container-networking-veth.html) we spoke about networking two network namespaces using a Veth pair.  Veth pairs are simple to use, but also limited. If you want to connect multiple network namespaces or give then access to the internet through the host, you are better off using a different solution. On this post, we will explore linux bridges to connect a few network namespaces.

**I'm running ubuntu xenial64 with vagrant+virtualbox**

To start let's install some necessary tooling to help to administrate the bridge.

{% highlight bash %}
sudo apt-get install bridge-utils
{% endhighlight %}

To connect network namespaces using a bridge we also use a veth pair. For each namespace a veth pair will be created, where one side of the veth goes into the namespace and the other peer side on the bridge.

Let's first create our bridge.

{% highlight bash %}
sudo brctl addbr br-test0
{% endhighlight %}

To avoid too much command repetition, let's define a script that will help us create the namespace, veth, and connect everything. Create a script **add-ns-to-br.sh**, and add the following code to it.

{% highlight bash %}
#!/bin/bash

bridge=$1
namespace=$2
addr=$3

vethA=veth-$namespace
vethB=eth0

sudo ip netns add $namespace
sudo ip link add $vethA type veth peer name $vethB

sudo ip link set $vethB netns $namespace
sudo ip netns exec $namespace ip addr add $addr dev $vethB
sudo ip netns exec $namespace ip link set $vethB up

sudo ip link set $vethA up

sudo brctl addif $bridge $vethA
{% endhighlight %}

## Script walkthrough

The script expects three parameters. First the name of the bridge, in our case **br-test0**, then the name of
the network namespace we are creating, and last the ip address to configure into the namespace.

The script first creates a namespace and a veth pair.

{% highlight bash %}
sudo ip netns add $namespace
sudo ip link add $vethA type veth peer name $vethB
{% endhighlight %}

Next, it will configure the veth side used by the namespace,
first, it assigns it to the namespace, to set an ip address for it and put the veth up.

{% highlight bash %}
sudo ip link set $vethB netns $namespace
sudo ip netns exec $namespace ip addr add $addr dev $vethB
sudo ip netns exec $namespace ip link set $vethB up
{% endhighlight %}

Then it sets the other side of the veth up.

{% highlight bash %}
sudo ip link set $vethA up
{% endhighlight %}

And lastly, adds the other veth side to the bridge.

{% highlight bash %}
sudo brctl addif $bridge $vethA
{% endhighlight %}

## Connecting four network namespaces

{% highlight bash %}
./add-ns-to-br.sh br-test0 ns1 10.0.0.1/24
./add-ns-to-br.sh br-test0 ns2 10.0.0.2/24
./add-ns-to-br.sh br-test0 ns3 10.0.0.3/24
./add-ns-to-br.sh br-test0 ns4 10.0.0.4/24
{% endhighlight %}

With our four namespaces created, and configured on the bridge, the last step is to set the bridge interface up.

{% highlight bash %}
sudo ip link set dev br-test0 up
{% endhighlight %}

We can now test our network. For example, from the namespace ns1 we can test we can ping the other namespaces.

{% highlight bash %}
sudo ip netns exec ns1 bash # get a bash into the namespace
ip a # checking ns1 ip address is 10.0.0.1

ping 10.0.0.2 # ping ns2
ping 10.0.0.3 # ping ns3
ping 10.0.0.4 # ping ns4

exit # exits the namespace
{% endhighlight %}

In a next post, we will configure internet access to our bridge, allowing our namespaces to access the internet.

## References

- [https://wiki.linuxfoundation.org/networking/bridge](https://wiki.linuxfoundation.org/networking/bridge)
- [https://goyalankit.com/blog/linux-bridge](https://goyalankit.com/blog/linux-bridge)
- [http://www.opencloudblog.com/?p=66](http://www.opencloudblog.com/?p=66)
- [https://lwn.net/Articles/580893/](https://lwn.net/Articles/580893/)
