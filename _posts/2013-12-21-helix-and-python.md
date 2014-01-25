---
layout: post
title: "Helix Outside the JVM, Part 1: pyhelix"
description: ""
category: "posts"
tags: ["code","helix"]
---
{% include JB/setup %}

[Helix](http://helix.apache.org) keeps getting [picked up](https://twitter.com/rbranson/status/411305214667259906) as a building block for distributed systems. However, most of these systems are built on languages that are JVM-based. There are currently two ways that Helix can support systems on other platforms. In this post, I'll focus on [pyhelix](https://github.com/kanakb/pyhelix), a Python-based Helix participant and spectator. Part 2 will cover the helix agent module in the main distribution.

### pyhelix

The goal of writing pyhelix was twofold: supporting Helix in a non-JVM language and scoping out what it would take to write a Helix library in a new language. In about a day, we got the major pieces in place and have continued to improve it since.

#### Participant Code

Creating a Python participant isn't all too different from creating one in Java. First, define the state model callbacks:

{% highlight python %}
from pyhelix import statemodel

class OnlineOfflineStateModel(statemodel.StateModel):
    """A class containing callbacks for the OnlineOffline model."""
    def on_become_online_from_offline(self, message):
        print 'OFFLINE --> ONLINE'

    def on_become_offline_from_online(self, message):
        print 'ONLINE --> OFFLINE'

    def on_become_dropped_from_offline(self, message):
        print 'OFFLINE --> DROPPED'

class OnlineOfflineStateModelFactory(statemodel.StateModelFactory):
    """A class that manages per-partition state models."""
    def create_state_model(self, partition_name):
        return OnlineOfflineStateModel()
{% endhighlight %}

Now, start the participant:

{% highlight python %}
from pyhelix import participant

# Configure the participant connection
p = participant.Participant(
    'MyCluster', 'MyHostname', '8001', 'zkhost:2181')

# Register the state model
p.register_state_model_fty('OnlineOffline',
    OnlineOfflineStateModelFactory())

# Optionally register a pre-connect callback
# This is a good place to start up the application
p.register_pre_connect_callback(cb)

# Connect
p.connect()
{% endhighlight %}

#### Spectator Code

When I was thinking of an example to write with this new library, I realized one thing: it's pretty important to know who's in the cluster and serving partitions. So I wrote a basic spectator that registers for changes to the Helix external view, and provides some functions analogous to [RoutingTableProvider](http://helix.apache.org/javadocs/0.6.2-incubating/reference/org/apache/helix/spectator/RoutingTableProvider.html) in the Java library.

Here's a basic spectator:

{% highlight python %}
from pyhelix import spectator

# Configure the spectator connection
conn = spectator.SpectatorConnection('MyCluster', 'zkhost:2181')

# Print out the current participant state mapping
# for a partition of a resource
s = conn.spectate('MyResource')
s.get_state_map(partition_id)
{% endhighlight %}

Normally, the spectator object is long-lived, so when a request comes in, the spectator can look up in its cache which participant to forward to.

### An Example System

To demonstrate pyhelix in action, I wrote what amounts to a [remote python shell](https://github.com/kanakb/pyhelix/wiki/Example:-Remote-Code-Runner). The spectator advertises participants that can accept the program, and forwards the code to the specified participants. Each specified participant runs the code and returns the output, which the spectator shows to the user.

This is quite powerful because the page will always only show nodes that are up, and it can accept expressions like `all` or `random` and use the spectator functions to determine where the programs will be sent.

### So, How'd it Go?

Pretty well. We demonstrated that it's not a massive undertaking to write a client library for Helix in a new language, particularly if the language is high-level. It also gives system builders a new option for working with Helix either through pure Python code, or by using Python to manage external processes. As an added bonus, we don't need much of the rigorous garbage collection resilience that's present in the Java code.

More importantly, it shows exactly what is needed to get Helix working in a new language. It also demonstrates the importance in having language diversity in any systems library. There is a lot of systems code written in C/C++, Go, node.js, and others, and so having something like this library will hopefully drive the community to build similar ports in those languages.

pyhelix is still a work in progress, but it's a lightweight way to get started with Helix. Try it out: [https://github.com/kanakb/pyhelix](https://github.com/kanakb/pyhelix)

