---
layout: post
title: "High Level APIs in Helix"
description: ""
category: posts
tags: ["code","helix"]
---
{% include JB/setup %}

Up until now, [Helix](http://helix.incubator.apache.org) APIs have had (close to) a 1:1 correspondence to how everything is persisted in ZooKeeper. In fact, the Helix data accessor is designed to work exclusively with objects of type [HelixProperty](http://helix.incubator.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/HelixProperty.html), which is often just a thin wrapper on the JSON stored on an actual ZooKeeper ZNode.

To further complicate things, there is a single [HelixManager](http://helix.incubator.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/HelixManager.html) class that manages all interactions to ZooKeeper. However, it contains functions that make no sense depending on how the application is trying to interact with Helix. A controller doesn't need to call `getStateMachineEngine()`, for example.

There are plenty of good reasons to have generic, low-level APIs: code reuse, support for fine-tuning, guarantees of atomicity at the single ZNode level, among others. However, it leads to code like this:

{% highlight java %}
// check the tags of live participants in the system for this resource's tag
public Set<String> getTaggedParticipants(
    String resourceName,
    IdealState idealState) {
  Set<String> taggedParticipants = Sets.newHashSet();
  String resourceTag = idealState.getInstanceGroupTag();
  List<String> liveParticipants = accessor.getChildNames(keyBuilder.liveInstances());
  for (String participantName : liveParticipants) {
    InstanceConfig instanceConfig = accessor.getProperty(keyBuilder.instanceConfig(participantName));
    Set<String> tags = instanceConfig.getTags();
    if (tags.contains(resourceTag)) {
      taggedParticipants.add(participantName);
    }
  }
}
{% endhighlight %}

This is ugly. We had to know that the resource tag is in the `IdealState`, and that we had to get an `InstanceConfig` for every live participant name. Even worse, resource names, participant names, and tags are all plain strings. Every time we pass a `Set<String>` to a new method, it's likely to lose some of its meaning.

Helix 0.7.0 high-level APIs change all this. From configuration to launch to management, 0.7.0 tries to expose APIs that speak in terms of actual Helix concepts (e.g. "Participant", "Controller", "Resource", "Partition", and so on). And now those objects have an explicit hierarchy: a `Cluster` has `Participant`s, `Resource`s, and `Controller`s, and a `Resource` has `Partition`s.

This post is an introduction by example to the new 0.7.0 APIs. The full code is available [here](https://git-wip-us.apache.org/repos/asf?p=incubator-helix.git;a=blob;f=helix-examples/src/main/java/org/apache/helix/examples/LogicalModelExample.java;h=c23341771fc0d0b7f909c7736cbcf5d651c49def;hb=master).

### Configuring a esource
For resourses and other constructs, Helix now makes a clear distinction between configuration properties that Helix defines, and those that the application defines. Specifically for resources, this distinction is in the form of the `RebalancerContext` and the `UserConfig`. Furthermore, all constructs have concrete ID types, in this case a `ResourceId`. No more data structures with raw strings!

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

### Configuring a Participant
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

### Configuring and Adding the Cluster
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

The accessor classes allow CRUD operations on virtually every granularity. Check out the [javadocs](http://helix.incubator.apache.org/javadocs/0.7.0-incubating/reference/org/apache/helix/api/accessor/package-summary.html) to see what you can do with them.

# Starting a Controller and Participant
0.7.0 didn't just add configuration APIs. It also supports actually starting Helix for each role. Here's how to start a controller:

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

### Back to the Example
Let's revisit the example from the beginning of the post, now with the high-level APIs.

{% highlight java %}
// check the tags of live participants in the system for this resource's tag
public Set<ParticipantId> getTaggedParticipants(
    ResourceId resourceId,
    RebalancerContext rebalancerContext) {
  Cluster cluster = accessor.readCluster();
  Set<ParticipantId> taggedParticipants = Sets.newHashSet();
  String resourceTag = rebalancerContext.getParticipantGroupTag();
  Collection<Participant> liveParticipants = cluster.getLiveParticipantMap().values();
  for (Participant participant : liveParticipants) {
    if (participant.hasTag(resourceTag)) {
      taggedParticipants.add(participant.getId());
    }
  }
}
{% endhighlight %}

The code size is similar, but each line makes more sense. Everything falls into the cluster hierarchy, and there's no second lookup to get the participant configuration; the live participant lookup returned concrete objects.

### We're not done!
0.7.0 represents an initial take on refactoring Helix's API. The most important thing with this release is to get it in the hands of system builders to see where it works best and where it falls short. If you try the 0.7.0 API and have suggestions about how it could work better, we'd love to hear them: `user@helix.incubator.apache.org`
