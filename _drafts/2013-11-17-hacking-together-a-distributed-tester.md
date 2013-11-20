---
layout: post
title: "Hacking Together a Distributed Tester, Part 1"
description: ""
category: posts
tags: ["distributed", "testing"]
---
{% include JB/setup %}

One of the toughest parts of building a distributed system (or a framework to support distributed systems) is dealing with all the bugs that come up because code is running on more than one machine. [Aphyr](http://aphyr.com/tags/jepsen) has gone into this at length with his *Jepsen* series in which he simulates network partitions to see how partition-tolerant "CP" and "AP" systems really are. There are really two perspectives to keep in mind:

* As a system user, you need to know where the systems you deploy can break down to avoid putting pressure on the system's weakest points.
* As a system implementer, you need to know where the systems you build can break down so that you can keep pushing back the limits.

Ultimately, what you're trying to avoid is ending conversations with:

    "I don't know, it works for me"

Regardless of your role, having a distributed tester makes a lot of sense. This post won't really get into too many specific failure scenarios, but rather will discuss some strategies for building a reusable framework. This framework is not by any means rock-solid, the most efficient approach, or even the right solution for everyone. It is, however, implementable in a short period of time with minimal code and tends to *just work*. The title of this post has "hacking" in it for a reason.

### Overview
Here's the flow that any test framework follows:

    set up test environment
    for each test:
        set up test
        run test
        record success or failure
        clean up test
    clean up test environment

Doing this in a distributed way doesn't change the flow, but it changes what it means to run each of these operations. What we need is a way to execute each of these in a reliable and easy-to-use way.

### Machines

In a best-case scenario, you have some systems that resemble production machines and they live on some network you own. This can be approximated by firing up a few instances on EC2 as well. This is ideal because you'll be running your distributed system on real machines. If you really want to get creative, you can even increase the diversity of the machines to test scenarios like incrementally upgrading parts of your datacenter.

If setting aside machines is out of the question, another strategy is to fire up some virtual machines running your favorite \*nix environment in command line mode. In this case, make sure that you have some way to communicate with these machines and that the machines can communicate with one another. These should be *networked* virtual machines.

Once the machines are set up, there needs to be a way to get these machines to do what you want them to do. Here are some things it would be useful to support to make it easy:

* Ensure that ports are open for ssh so that you can run commands remotely. In this system, we'll treat our local box as the master, and our provisioned systems as minions who do the master's bidding.
* Prepare a configured build environment (for native code), or runtime environment (for non-native code). To be able to test your distributed system, it needs to run on your allocated machines. I'll assume your system is written in a JVM language for the rest of this post, so that would mean having a JRE installed and available.
* Come up with a common directory structure where your deployed code will live.
* Add your local machine's public key to ~/.ssh/authorized_keys so that you can run multiple commands over ssh without having to enter your password each time.

### Language

As long as you like a language that can run arbitrary shell commands, you can pick that language. In fact, that's most languages. It's also useful to be able to define things in a functional way. For example, if you have a task that is an aggregation of multiple operations, it is much easier to express functionally. With this in mind, I'll write examples here in Python, which has solid shell execution support and also treats functions as first-class objects.

### Primitives

Defining the primitives is the toughest thing to do for any framework. Here are some useful ones:

#### SSH and SCP
The first action you need to take before taking a test suite is setting up the remote environment. This usually means copying over your binaries. The easiest way to do that is with a simple SCP.

{% highlight python %}
# Recursively copy to the target machine
subprocess.call('scp -R {0} {1}:{2}'.format(local_dir, host, remote_dir), shell=True)

# Recursively copy from the target machine
subprocess.call('scp -R {0}:{1} {2}'.format(host, remote_dir, local_dir), shell=True)
{% endhighlight %}

Once binaries are built and copied to each remote machine, we need to be able to execute them. We could implement sophisticated libraries to talk over the network or otherwise communicate across machines. However, in the interest of keeping things to implement at a minimum, we can just use SSH.

{% highlight python %}
# Run remotely in the foreground, save stdout
output = subprocess.Popen('ssh -t {0} "{1}"'.format(host, cmd), stdout=subprocess.PIPE, shell=True).communicate()[0]

# Run remotely in the foreground, save return code
result = subprocess.call('ssh -t {0} "{1}"'.format(host, cmd), shell=True)
{% endhighlight %}

What about tasks we want to keep running and kill later? Here's a trick that isn't thread safe:

{% highlight python %}
# Get java pids

# Run remotely in the background

# Get java pids again

# Compare pid sets, the difference should be the new process
{% endhighlight %}

Actually parsing your ps aux for the command you ran is safer, but this works if you're not using your provisioned machines for

#### Command Line Tools
If your app doesn't expose tools that can be run via the command line, then using SSH-based execution is pretty useless. At a minimum, you want to support: starting a component, injecting minor changes to a component, and querying the state of running components.

#### HTTP
Another useful technique for systems that use a RESTful API is to support HTTP calls. In Python, this involves using the urllib and urllib2 libraries.

{% highlight python %}
import json
import urllib2

# Do an HTTP GET and save the JSON output in a Python object
get_result = urllib2.urlopen(url, timeout=TIMEOUT)
obj = json.load(get_result)
{% endhighlight %}
