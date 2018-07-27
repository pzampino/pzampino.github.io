---
title: Apache Knox HA Deployments
---

# Introduction

Apache Knox can be deployed for High Availability (HA) and scale out by configuring multiple instances with the same configuration and topologies, and placing a load balancer in front of those instances.

Version 1.1.0 includes a number of new features that make it easier to deploy and manage such deployments.

<br>

## ZooKeeper Provider Configuration and Descriptor Management

Provider configurations and descriptors are the recently-added Knox topology building blocks. They can be combined to produce topology deployments.
Knox instances can be configured to monitor a ZooKeeper ensemble to consume and apply changes to provider configurations and descriptors to effect changes to their deployed topologies.
When multiple Knox instances are configured to monitor the same ZooKeeper ensemble in this way, they stay synchronized in terms of their respective deployed topologies.

The following properties must be defined in gateway-site for each Knox instance:

First, a connection to a ZooKeeper ensemble must be defined.<br>
In this example, the _my-zookeeper-client_ part of the property name is an arbitrary identifier:

    <property>
	  <name>gateway.remote.config.registry.my-zookeeper-client</name>
	  <value>type=ZooKeeper;address=zkhost1:2181,zkhost2:2181,zkhost3:2181</value>
    <property>

The remote configuration monitor must then be configured to use *that* ZooKeeper ensemble connection

    <property>
	  <name>gateway.remote.config.monitor.client</name>
	  <value>my-zookeeper-client</value>
    <property>


With this configuration, each Knox instance will monitor the ZooKeeper ensemble for changes to provider configurations and descriptors.
Additions and modifications will result in topology [re]generations and [re]deployments, and deletions will result in topology deletions and undeployments.

If all the Knox instances are being managed by Ambari, then these gateway-site configuration changes can be made once and for all.
For other deployment scenarios, these properties will have to be specified in each Knox instance's gateway-site configuration.

More details about this configuration are available in the [User Guide](http://knox.apache.org/books/knox-1-1-0/user-guide.html#Remote+Configuration+Monitor).

<br>

## Service Discovery & Topology Generation

Service discovery is the facility by which Knox can process a set of declared Hadoop service roles in a descriptor, populating their respective endpoint URLs by interrogating a specified Ambari cluster. With the ZooKeeper monitoring enabled, any changes to provider configurations or descriptors will trigger this service discovery and topology generation to produce an updated topology deployment.

The Ambari cluster details are specified in the descriptors themselves, or defaults can be defined in the gateway-site configuration.

If no discovery address is specified in a descriptor, Knox will check for the following property in gateway-site:

    <property>
	  <name>gateway.discovery.default.address</name>
	  <value>http://ambarihost:8080</value>
    <property>

If no cluster name is specified in a descriptor, Knox will check for the following property in gateway-site:

    <property>
	  <name>gateway.discovery.default.cluster</name>
	  <value>myCluster</value>
    <property>

If no discovery credentials are specified in a descriptor, Knox will refer to the default aliases and attempt to use their values:

* ambari.discovery.user     The default Ambari discovery username
* ambari.discovery.password The default Ambari discovery password

These default aliases are especially useful if a Knox instance will be deploying multiple topologies based on the same cluster.

More details about descriptors and service discovery are available in the [User Guide](http://knox.apache.org/books/knox-1-1-0/user-guide.html#Simplified+Descriptor+Files).
<br>


## ZooKeeper Credential Store Management

Sharing the Knox credential stores among all the Knox instances is critical to support Knox HA deployment and service discovery. Knox employs topology-specific credentials while proxying Hadoop services, and it uses gateway credentials to authenticate with Ambari clusters for service discovery. ZooKeeper makes it easier to manage these credentials by supporting the automatic distribution of those credentials among the participating Knox instances without administrator intervention. The administrator need only make a specific change one time on _one_ of the Knox instances, and all the other Knox instances will effect that change.

For example, defining the following credential aliases on a Knox instance that is configured to monitor a ZooKeeper ensemble will result in these aliases being propagated to other Knox instances which are monitoring the same ZooKeeper ensemble:

    bin/knoxcli.sh create-alias ambari.discovery.user --value maria_dev
	bin/knoxcli.sh create-alias ambari.discovery.password --value maria_dev

<br>


## Cluster Monitoring

Once Knox has performed service discovery against an Ambari-managed cluster, and generated a topology based on that discovery, it has the ability to monitor that cluster's configuration for changes. If so configured, Knox will notice configuration changes that affect deployed topologies, and regenerate and redploy those affected topologies based on the new configuration. This behavior can help Knox continue successfully proxying Hadoop services when a cluster administrator changes service details that affect its endpoint URLs (e.g., host, port, context path), without any intervention by a Knox administrator.

By default, this monitoring is disabled, but simply changing the value of the following property to *true* in the gateway-site configuration enables it:

    <property>
      <name>gateway.cluster.config.monitor.ambari.enabled</name>
      <value>true</value>
      <description>Enable/disable Ambari cluster configuration monitoring.</description>
    </property>

By default, Knox will check for configuration changes every 60 seconds, but this interval can also be configured in gateway-site via the following property:

    <property>
      <name>gateway.cluster.config.monitor.ambari.interval</name>
      <value>60</value>
      <description>The interval (in seconds) for polling Ambari for cluster configuration changes.</description>
    </property>


More details about cluster monitoring are available in the [User Guide](http://knox.apache.org/books/knox-1-1-0/user-guide.html#Cluster+Configuration+Monitoring).
<br>


## Admin UI, API and Knox CLI

While an administrator can add topology-related configuration files directly to a ZooKeeper ensemble being monitored by Knox, there are some Knox facilities that are easier to use to accomplish this. When Knox instances are configured for remote configuration monitoring, provider configuration and descriptor changes effected by the Admin UI or API will
automatically be persisted in the configured ZooKeeper ensemble. As has been discussed, this causes those changes to be applied to every Knox instance which is also monitoring that ZooKeeper ensemble. Thus, for topology-related changes, the Knox administrator does not need to access any of the Knox hosts via ssh/scp or interact with ZooKeeper directly.

Similarly, when the Knox CLI is used to manage credential aliases, those aliases will be persisted in the configured ZooKeeper ensemble, and applied to all the monitoring Knox instances.

<br>


# Summary
Knox support for HA deployments has become significantly easier to manage with the 1.1.0 release. Effecting topology changes for ten Knox instances has been made as easy as doing the same for a single instance. The need for ssh/scp access to Knox hosts has been minimized, and really eliminated for topology-related changes. Finally, the cluster monitoring support allows Knox instances to automatically adapt to service configuration changes, reducing the need for administrators to update the affected topologies.


Complete information is available in the [User Guide](http://knox.apache.org/books/knox-1-1-0/user-guide.html).


<br><br><br><br>


