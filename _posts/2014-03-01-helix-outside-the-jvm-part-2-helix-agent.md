---
layout: post
title: "Helix Outside the JVM, Part 2: Helix Agent"
description: ""
category: "posts"
tags: ["code","helix","java"]
---
{% include JB/setup %}

In the [previous installment](http://kanakb.github.io/posts/2013/12/21/helix-and-python), I talked about how a Python-based [Helix](http://helix.apache.org) API enables easy, powerful cluster management for arbitrary distributed systems outside the JVM. However, it's certainly not the only way to do it.

This post will describe how to get an application quickly working with the helix-agent module. For a more detailed description of what helix-agent can provide, see the Helix documentation: [http://helix.apache.org/0.6.2-incubating-docs/tutorial_agent.html](http://helix.apache.org/0.6.2-incubating-docs/tutorial_agent.html)

### Helix Agent

JVM-based apps built using Helix model local state changes as part of state transition callbacks that are invoked when the Helix controller tells the node to do so. Clearly, it's more difficult for a Java library to call into applications that are not built on the JVM. Instead, this module allows you to specify scripts to execute on each state transition.

To make this work, there are two steps: configuration and execution.

#### Configuration

The first thing that must be done is to let Helix know which scripts should be run on a state transition. Helix can automatically monitor these scripts so that their termination indicates the completion of a transition.

One way to specify the scripts to run is through Java code.

{% highlight java %}
// Work at the scope of a resource
HelixConfigScope scope = new HelixConfigScopeBuilder(ConfigScopeProperty.RESOURCE)
    .forCluster(clusterName)
    .forResource(resourceName)
    .build();

// Get an accessor for configurations
ConfigAccessor configAccessor = new ConfigAccessor(_gZkClient);

// Specify the script for OFFLINE --> ONLINE
CommandConfig cmdConfig = new CommandConfig.Builder()
    .setTransition("OFFLINE", "ONLINE")
    .setCommand("simpleHttpClient.py OFFLINE-ONLINE")
    .setCommandWorkingDir(workingDir)
    .setCommandTimeout("5000L")
    .setPidFile(pidFile)
    .build();
configAccessor.set(scope, cmdConfig.toKeyValueMap());

// And so on for other transitions
{% endhighlight %}

The alternative to writing code is simply running some commands to set the configurations.

{% highlight bash %}
# Specify the script for OFFLINE --> ONLINE
./helix-admin.sh --zkSvr localhost:2181 --setConfig CLUSTER clusterName OFFLINE-ONLINE.command="simpleHttpClient.py OFFLINE-ONLINE",OFFLINE-ONLINE.workingDir="/path/to/script", OFFLINE-ONLINE.command.pidfile="/path/to/pidfile"
{% endhighlight %}

#### Execution

The purpose of the transition script is typically to call some running entry point to your actual application. Thus, the transition script is short-lived, and the daemon accepting requests from the transition script is long-lived and tied to the actual application.

The Helix agent should have a lifecycle that matches your long-lived application. Thus, when your application fails or shuts down, that is when the Helix agent should also shut down.

In code, here is how one could model that linked lifecycle:

{% highlight java %}
// Start the app and agent
ExternalCommand serverCmd = ExternalCommand.start(
    workingDir + "/simpleHttpServer.py");
Thread agentThread = new Thread() {
  @Override
  public void run() {
    // Leaving out exception handling
    HelixAgentMain.main(new String[] {
        "--zkSvr", zkAddr,
        "--cluster", clusterName,
        "--instanceName", instanceName,
        "--stateModel", "OnlineOffline"
    });
  }
};

// Wait for the process to terminate
agentThread.start();
serverCmd.waitFor();
agentThread.interrupt();
{% endhighlight %}

Again, this is also doable directly from the command line:

{% highlight bash %}
# Build Helix and start the agent
mvn clean install -DskipTests
chmod +x helix-agent/target/helix-agent-pkg/bin/*
helix-agent/target/helix-agent-pkg/bin/start-helix-agent.sh --zkSvr zkAddr1,zkAddr2 --cluster clusterName --instanceName instanceName --stateModel OnlineOffline

# Here, you can define your own logic to terminate this agent when your process terminates
{% endhighlight %}

What the agent module enables is to create a _meta-participant_ in the cluster so that the single node application code can be written in any language, but will still work with Helix state transitions.