# Introduction

The [Apache Knox Dynamic Service Endpoint Discovery article](https://community.hortonworks.com/articles/154912/apache-knox-dynamic-service-endpoint-discovery.html)
describes some exciting new functionality available in the 0.14.0 release of Apache Knox. The gateway is now able to dynamically determine the endpoint URLs of
cluster services to proxy from Ambari. The associated benefits are described in that article.

Another benefit of this new functionality, which is not mentioned in that article, is the added ability to dynamically respond to cluster configuration changes that
affect generated Knox topologies by re-generating and re-deploying those topologies. Without this, deployed topologies can be easily disabled when any of the proxied
Hadoop services' configuration changes in the cluster.


# Cluster Monitoring

When Knox deploys a simple topology descriptor, and generates a corresponding topology based on discovered cluster configuration details, it subsequently has the
ability to monitor that cluster configuration for changes. When it discovers a change, it updates all of its topologies that are based on that modified cluster, and
redeploys them. This has the potential to greatly reduce downtime for Knox due to cluster configuration changes.

For example, suppose a descriptor (*docker-sandbox.json*) is deployed, intended to proxy services in the [HDP Docker Sandbox](https://hortonworks.com/downloads/#sandbox).
Following the successful generation and deployment of the *docker-sandbox* topology, Knox can monitor the *Sandbox* cluster managed by Ambari. If an administrator were to
update the *dfs.namenode.http-address* property value in the *hdfs-site* configuration, changing the port number for example, the Knox proxy for the *WEBHDFS* service would
no longer work. However, if the Ambari cluster monitor is enabled, Knox would regenerate and redeploy the *docker-sandbox* topology, such that it would contain the correct
port for the *WEBHDFS* service URL, and Knox clients would continue to work.

By default this monitor is disabled, but it can easily be enabled by setting the __gateway.cluster.config.monitor.ambari.enabled__ property value to __*true*__ in
the __gateway-site__ configuration.

    <property>
        <name>gateway.cluster.config.monitor.ambari.enabled</name>
        <value>true</value>
        <description>Enable/disable Ambari cluster configuration monitoring.</description>
    </property>

Also in the __gateway-site__ configuration, there is a property for controlling the frequency with which Knox will check the clusters for which it has deployed topologies.
For demonstration purposes, you may want to set this as low as 20 or 30 seconds.

    <property>
        <name>gateway.cluster.config.monitor.ambari.interval</name>
        <value>60</value>
        <description>The interval (in seconds) for polling Ambari for cluster configuration changes.</description>
    </property>


# Try It

The [Apache Knox Dynamic Service Endpoint Discovery article](https://community.hortonworks.com/articles/154912/apache-knox-dynamic-service-endpoint-discovery.html)
includes instructions for deploying topologies using simple descriptors, employing service URL discovery. Starting from there, you can enable the Ambari cluster
monitoring, and make a cluster configuration change like the one described in this article. Then, you'll see how Knox responds to the change, and adapts to
continue providing the proxied *WEBHDFS* service to its clients.

1. Set the __*gateway.cluster.config.monitor.ambari.enabled*__ property value to *true* in __*{GATEWAY_HOME}*/conf/gateway-site.xml__
2. Restart the gateway
3. Use Ambari to modify the *hdfs-site dfs.namenode.__http-address__* configuration property value as described in the example.
4. Allow the gateway to notice the configuration change (watch the __*{GATEWAY_HOME}*/logs/gateway.log__ for the messages)
5. Review __*{GATEWAY_HOME}*/conf/topologies/docker-sandbox.xml__, and notice the change to the *WEBHDFS* service URL.

Note: Your sandbox must expose the new port you specified for the __*dfs.namenode.http-address*__ property for Knox to be able to access the new endpoint; otherwise,
even though the topology will be correct, requests will fail due to connection failure.


# Summary

While it doesn't take long to describe, this feature is a significant addition to the value provided by Knox. The ability to dynamically adapt to cluster
service configuration changes reduces the effort required (and the potential for errors) by administrators when making such changes.


__N.B., Statically-defined topologies (i.e., those deployed directly by a regular topology XML file) do NOT benefit from this monitoring support.__


More details are available in the [User Guide](http://knox.apache.org/books/knox-0-14-0/user-guide.html).



