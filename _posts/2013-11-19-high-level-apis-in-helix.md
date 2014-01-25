---
layout: post
title: "High Level APIs in Helix"
description: ""
category: posts
tags: ["code","helix"]
---
{% include JB/setup %}

Up until now, [Helix](http://helix.apache.org) APIs have had (close to) a 1:1 correspondence to how everything is persisted in ZooKeeper. In fact, the Helix data accessor is designed to work exclusively with objects of type [HelixProperty](http://helix.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/HelixProperty.html), which is often just a thin wrapper on the JSON stored on an actual ZooKeeper ZNode.

To further complicate things, there is a single [HelixManager](http://helix.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/HelixManager.html) class that manages all interactions to ZooKeeper. However, it contains functions that make no sense depending on how the application is trying to interact with Helix. A controller doesn't need to call `getStateMachineEngine()`, for example.

There are plenty of good reasons to have generic, low-level APIs: code reuse, support for fine-tuning, guarantees of atomicity at the single ZNode level, among others. However, it leads to a detachment from the concepts the framework is trying to present. And because Helix stores things as JSON, there are a lot of strings everywhere, like:

{% highlight java %}
Map<String, Map<String, String>> mapFields =
    property.getRecord().getMapFields();
{% endhighlight %}

At this level, each of those strings has lost its meaning.

Helix 0.7.0 high-level APIs change all this. From configuration to launch to management, 0.7.0 tries to expose APIs that speak in terms of actual Helix concepts (e.g. "Participant", "Controller", "Resource", "Partition", and so on). And now those objects have an explicit hierarchy: a `Cluster` has roles (e.g. `Participant`s, `Controller`s, and `Spectator`s) and `Resource`s, and a `Resource` has `Partition`s.

This post is an introduction by example to the new 0.7.0 APIs. The full code is available [here](https://git-wip-us.apache.org/repos/asf?p=helix.git;a=blob;f=helix-examples/src/main/java/org/apache/helix/examples/LogicalModelExample.java;h=c23341771fc0d0b7f909c7736cbcf5d651c49def;hb=master).

### The Hierarchy
Below is a summary of the logical hierarchy of classes added to 0.7.0. An arrow here indicates a \"has-a\" relation. This hierarchy is replicated in the configuration classes (i.e. ClusterConfig, ResourceConfig, ParticipantConfig, etc), as well as the snapshot classes (i.e. Cluster, Resource, Participant, etc).

<a href="{{ site.url }}/assets/070hierarchy.png"><img src="{{ site.url }}/assets/070hierarchy.png" alt="Helix API Hierarchy" width="100%" height="100%"/></a>

For those familiar with the existing APIs, this isn't a major departure. Many of the leaf nodes are the same low-level classes, like `CurrentState` and `ExternalView`. However, each has been integrated into this hierarchy. When you read a cluster, you get its participants and resources, and so on.

There is one major difference that those aware of Helix concepts may have noticed: the omission of `IdealState` in the hierarchy. Over time, this class has become a dumping ground for arbitrary resource properties. Worse, for non-data resources, many of the concepts of IdealState are difficult to relate to. Thus, Helix now has an interface called `RebalancerContext` which captures the real goal of an IdealState of configuring how to assign resources to participants while allowing users to introduce application-specific constructs. See [this tutorial](http://helix.apache.org/site-releases/0.7.0-incubating-site/tutorial_user_def_rebalancer.html) for more.

Furthermore, in the past, the Helix admin API encouraged applications to simply set simple fields, list fields, and map fields on `HelixProperty` objects as a way to have accessible user-specific configurations. This was clunky and confusing. Instead, Helix now has a `UserConfig` class for all of these items, and it is available at multiple levels of the hierarchy (cluster, participant, resource, and partition; see the boxed UserConfigs in the diagram). Now there is a clear separation and protection against name collisions for fields that Helix users set and fields that Helix itself sets.

Notice that some of the hierarchy has bubbles indicating that the type has a concrete ID class to identify it, all constructs have concrete ID types, in this case a `ResourceId`. No more data structures with raw strings!

### Interfaces
There are a few different types of new APIs in 0.7.0. For instance, there are cluster snapshots, configurations, IDs, and accessors. Snapshots are intended to be a high-level equivalent to `ClusterDataCache`, configurations are intended to be a high-level equivalent to `HelixProperty` objects, IDs are replacements for arbitrary strings in Helix code, and accessors support CRUD with those classes in a more hierarchical way than `HelixAdmin`. Most of these interfaces are available in the [org.apache.helix.api](http://helix.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/api/package-summary.html) package.

Helix 0.7.0 also introduced a new way of managing Helix connections. There is now a reusable `HelixConnection` class that can get all of the logical accessors for the cluster, as well as objects for each Helix role (e.g. Controllers and Participants) that can be started and stopped. The HelixConnection API is [here](http://helix.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/HelixConnection.html); it is meant to be a cleaner version of the too-large `HelixManager`.

Now, we can see these interfaces in action.

### Create a State Model Definition
This process is relatively unchanged. The tutorial for creating a new one can be found [here](http://helix.apache.org/site-releases/0.7.0-incubating-site/tutorial_state.html). Our resource uses a lock-unlock model, which is basically where one replica per partition is in _LOCKED_ state and all others are in _RELEASED_ state (i.e. each partition is an exclusive lock).

### Configure a Resource
For resourses, Helix now makes a clear distinction between configuration properties that Helix defines, and those that the application defines. This distinction is in the form of the `RebalancerContext` and the `UserConfig`.

{% highlight java %}
// identify the resource with a concrete type
ResourceId resourceId = ResourceId.from("exampleResource");

// specify the rebalancer configuration
// this resource will be rebalanced in FULL_AUTO mode
FullAutoRebalancerContext rebalancerContext =
    new FullAutoRebalancerContext.Builder(resourceId)
    .replicaCount(1) // 1 replica per partition
    .addPartitions(2) // 2 partitions
    .stateModelDefId(stateModelDef.getStateModelDefId())
    .build();

// create (optional) resource user config properties
UserConfig userConfig = new UserConfig(Scope.resource(resourceId));
userConfig.setBooleanField("sampleBoolean", true);

// create the configuration
ResourceConfig resource =
    new ResourceConfig.Builder(resourceId)
    .rebalancerContext(rebalancerContext)
    .userConfig(userConfig)
    .build();
{% endhighlight %}

### Configure a Participant
It's the same idea for configuring a participant. Add Helix-specific properties, attach a UserConfig, and it's ready to go.

{% highlight java %}
// identify the participant with a concrete type
ParticipantId participantId = ParticipantId.from("localhost_1234");

// create (optional) participant user config properties
UserConfig userConfig = new UserConfig(Scope.participant(participantId));
// ... fill in userConfig ...

// create the configuration
ParticipantConfig participant =
    new ParticipantConfig.Builder(participantId)
    .hostName("localhost").port(1234)
    .userConfig(userConfig)
    .build();
{% endhighlight %}

### Create and Configure the Cluster
Now that we have a resource and a participant configured, let's add them to a cluster.

{% highlight java %}
// identify the cluster with a concrete type
ClusterId clusterId = ClusterId.from("exampleCluster");

// create (optional) cluster user config properties
UserConfig userConfig = new UserConfig(Scope.cluster(clusterId));
userConfig.setIntField("sampleInt", 1);

// fully specify the cluster with a ClusterConfig
ClusterConfig.Builder clusterBuilder =
    new ClusterConfig.Builder(clusterId)
    .addResource(resource)
    // .addResource(resource2)
    .addParticipant(participant)
    // .addParticipant(participant2)
    .addStateModelDefinition(lockUnlock)
    // .addStateModelDefinition(masterSlave)
    .autoJoin(true)
    .userConfig(userConfig);

{% endhighlight %}

We can even throw in some constraints at different granularities.

{% highlight java %}
// add a state constraint with cluster scope
// only one replica per partition allowed in LOCKED state
clusterBuilder.addStateUpperBoundConstraint(
    Scope.cluster(clusterId),
    lockUnlock.getStateModelDefId(),
    State.from("LOCKED"),
    1);

// add a transition constraint (this time with a resource scope)
// RELEASED --> LOCKED transitions will happen one at a time now
clusterBuilder.addTransitionConstraint(
    Scope.resource(resource.getId()),
    lockUnlock.getStateModelDefId(),
    Transition.from(State.from("RELEASED"),
    State.from("LOCKED")),
    1);

ClusterConfig cluster = clusterBuilder.build();
{% endhighlight %}

Now, we can persist everything.

{% highlight java %}
// set up a connection to work with ZooKeeper-persisted data
HelixConnection connection = new ZkHelixConnection(args[0]);
connection.connect();

// add the cluster in one step
ClusterAccessor accessor = connection.createClusterAccessor(clusterId);
accessor.createCluster(cluster);
{% endhighlight %}

Obviously specifying all resources and participants ahead of time isn't right for everyone, so adding and changing things later is supported, too:

{% highlight java %}
// add a new resource and participant
accessor.addResourceToCluster(resource2);
accessor.addParticipantToCluster(participant2);

// change the resource rebalancer context
ResourceAccessor resourceAccessor = connection.createResourceAccessor(clusterId);
resourceAccessor.setRebalancerContext(resourceId, newContext);
{% endhighlight %}

The accessor classes allow CRUD operations on virtually every granularity. Check out the [javadocs](http://helix.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/api/accessor/package-summary.html) to see what you can do with them.

# Start a Controller and Participant
Helix 0.7.0 didn't just add configuration APIs. It also supports actually starting Helix for each role. Here's how to start a controller:

{% highlight java %}
ControllerId controllerId = ControllerId.from("exampleController");
HelixController helixController = connection.createController(clusterId, controllerId);
helixController.startAsync();
{% endhighlight %}

Starting a participant looks almost the same:

{% highlight java %}
HelixParticipant helixParticipant = connection.createParticipant(
    clusterId, participantId);
helixParticipant.getStateMachineEngine().registerStateModelFactory(
    lockUnlock.getStateModelDefId(),
    new LockUnlockFactory());
helixParticipant.startAsync();
{% endhighlight %}

In both cases, we're speaking in concrete classes for each role: `HelixController` and `HelixParticipant`. Before, the role was an enum flag that was passed into the monolithic `HelixManager` class. Now, each role-specific class only contains methods that make sense for that role.

### We're not done!
Helix 0.7.0 represents an initial take on refactoring Helix's API. The most important thing with this release is to get it in the hands of system builders to see where it works best and where it falls short. If you try the 0.7.0 API and have suggestions about how it could work better, we'd love to hear them: `user@helix.apache.org`

### Update
The master branch of Helix renamed `RebalancerContext` to `RebalancerConfig` to more accurately represent it as something that is typically set beforehand. In addition, there is now something called a `ControllerContext` that the rebalancer can set and will be persisted across controller pipeline runs.

### Update 2
Helix is now a top level project, so changing the links from [helix.incubator.apache.org](helix.incubator.apache.org) to [helix.apache.org](helix.incubator.apache.org).
