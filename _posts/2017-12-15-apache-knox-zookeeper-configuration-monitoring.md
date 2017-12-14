# Introduction

The [Apache Knox Dynamic Service Endpoint Discovery](https://community.hortonworks.com/content/kbentry/154912/apache-knox-dynamic-service-endpoint-discovery.html)
article introduced the gateway's new ability to dynamically generate topologies using details gleaned from cluster service configurations managed by Ambari. A
key part of this new ability is the addition of simple topology descriptors and provider configurations as input to topology generation.

Knox 0.14.0 also includes the capability of consuming the inputs for these topology deployments from [Apache ZooKeeper](https://zookeeper.apache.org/).

Knox instances can detect changes (additions/removals/modifications) to provider configurations and simple topology descriptors
stored in ZooKeeper, and respond with corresponding topology changes.

<br>

# Remote Configuration Monitor

This behavior is provided by a remote configuration monitor, which effectively watches znodes in the configured ZooKeeper
for content changes.

Specifically, it watches the following znodes:

* __/knox/config/shared-providers__
* __/knox/config/descriptors__

These znodes are treated similarly to their corresponding local directories in __*{GATEWAY_HOME}*/conf/__ in terms of the actions
taken when changes are noticed.

Resource | Change | Result
---------|--------|---------
__/knox/config/shared-providers/*{providerconfig}*__| Add | *{providerconfig}* is downloaded to *{GATEWAY_HOME}*/conf/shared-providers/
__/knox/config/shared-providers/*{providerconfig}*__| Remove | *{providerconfig}* is deleted from *{GATEWAY_HOME}*/conf/shared-providers/
__/knox/config/shared-providers/*{providerconfig}*__| Modify | *{providerconfig}* is downloaded to *{GATEWAY_HOME}*/conf/shared-providers/
__/knox/config/descriptors/*{descriptor}*__| Add | *{descriptor}* is downloaded to *{GATEWAY_HOME}*/conf/descriptors/, and topology generation and deployment is attempted
__/knox/config/descriptors/*{descriptor}*__| Remove | *{descriptor}* is deleted from *{GATEWAY_HOME}*/conf/descriptors/, and corresponding topology is undeployed
__/knox/config/descriptors/*{descriptor}*__| Modify | *{descriptor}* is downloaded to *{GATEWAY_HOME}*/conf/descriptors/, and topology re-generation and re-deployment is attempted

<br>

Multiple Knox instances may monitor the same ZooKeeper, and each will consume these changes to maintain topology consistency among them.
Changes need only be applied once, to the resource maintained in ZooKeeper, to effect change in all the monitoring Knox instances.

<br>

# ZooKeeper

The remote configuration registry monitor, depends on a remote registry client configuration, which defines the connection to the registry to monitor.
This client configuration is what makes the monitor ZooKeeper-specific; ZooKeeper is currently the only supported registry type.

The registry client is configured in __gateway-site.xml__, using the property named __gateway.remote.config.monitor.client__; the value of
this property is a reference to another property (also defined in __gateway-site.xml__).

Multiple registry clients can be configured for use by the gateway components, with properties of the form __gateway.remote.config.registry.*{CLIENT_NAME}*__,
where *{CLIENT_NAME}* is a unique identifier for that particular client. For example, __gateway.remote.config.registry.*sandbox-zookeeper-client*__
would define a registry client named __*sandbox-zookeeper-client*__, which the monitor property could then reference.

    <property>
        <name>gateway.remote.config.registry.sandbox-zookeeper-client</name>
        <value>type=ZooKeeper;address=localhost:2181</value>
        <description>ZooKeeper configuration registry client details.</description>
    </property>

    <property>
        <name>gateway.remote.config.monitor.client</name>
        <value>sandbox-zookeeper-client</value>
        <description>Remote configuration monitor client name.</description>
    </property>

More likely than not, production environments will employ a ZooKeeper ensemble, rather than a single node. In this case, the value of the address component
of the registry client property value will be a comma-delimited list of the ZooKeeper hosts that comprise that ensemble (e.g., host1:2181,host2:2181,host3:2181).

<br>

## Security

Since Knox will be consuming these topology configuration files from ZooKeeper, it's important that the ability for untrusted parties to author content therein be restricted.
It's recommended that ZooKeeper be configured to require client authentication, and that the monitored znodes be created with appropriate ACLs. 

If the monitored znodes do not have ACLs that are acceptable to Knox, the monitor will update them. For example, if the monitor uses a client that is configured for authentication,
but the ACLs on the monitored znodes allow world:anyone access, then the monitor will correct the ACLs, such that __*world:anyone*__ will not have access, and only authenticated clients will.


Property | Value(s) | Description
---------|----------|---------------
__authType__ | *digest*, *kerberos* | The authentication mechanism backing the SASL framework used by ZooKeeper
__principal__| A principal identifier | The principal to be authenticated
__credentialAlias__ | A password alias | For digest authentication only, a gateway alias for the password associated with the principal
__keytab__ | Path to a keytab file | The keytab to use for Kerberos authentication
__useKeyTab__ | *true*, *false* | Whether or not to use the keytab for Kerberos authentication
__useTicketCache__ | *true*, *false* | Whether or not to use the ticket cache for Kerberos authentication 

<br>

### Examples:

__*Digest*__

    <property>
        <name>gateway.remote.config.registry.sandbox-zookeeper-client</name>
        <value>type=ZooKeeper;address=localhost:2181;authType=digest;principal=knox;credentialAlias=sandbox.zk.digest.password</value>
        <description>ZooKeeper configuration registry client details.</description>
    </property>

__*Kerberos*__

    <property>
        <name>gateway.remote.config.registry.sandbox-zookeeper-client</name>
        <value>type=ZooKeeper;address=localhost:2181;authType=kerberos;principal=knox;keytab=/home/user/my.keytab;useKeyTab=true;useTicketCache=false</value>
        <description>ZooKeeper configuration registry client details.</description>
    </property>


<br>

# Managing Content in ZooKeeper

While the ZooKeeper CLI (zkCli.sh) can be used to manage the contents of these monitored znodes, it's not the most convenient tool to use, especially when
it comes to uploading files.

For this reason, the Knox CLI (*{GATEWAY_HOME}*bin/knoxcli.sh) provides the following commands for managing this content in ZooKeeper:

Command | Description
--------|-------------
__list-registry-clients__ | Lists the configured registry clients available.
__upload-provider-config *filePath* --registry-client *name* [--entry-name entryName]__ | Uploads a provider configuration as a child of the /knox/config/shared-providers znode.
__upload-descriptor *filePath* --registry-client *name* [--entry-name *entryName*]__ | Uploads a descriptor as a child of the /knox/config/descriptors znode.
__delete-provider-config *providerConfig* --registry-client *name*__ | Deletes a provider configuration from the /knox/config/shared-providers znode.
__delete-descriptor *descriptor* --registry-client *name*__ | Deletes a descriptor from the /knox/config/shared-providers znode.

<br>
More details about the available commands are available in [the Knox CLI section of the User Guide](http://knox.apache.org/books/knox-0-14-0/user-guide.html#Knox+CLI).

<br>

# Benefits

Some of the benefits of maintaining this configuration in ZooKeeper include:

* Convenience in propagating changes to multiple Knox instance (e.g., Knox HA)
    * Add or remove a descriptor once in ZooKeeper, and all monitoring Knox instances will update their topology deployments accordingly.
    * Update a single provider configuration once in ZooKeeper, and have it applied to every topology whose descriptor referenced it, across multiple Knox instances.

<br>

# Try It

 0. [Docker Sandbox](https://hortonworks.com/products/sandbox) [https://hortonworks.com/downloads/#sandbox](https://hortonworks.com/downloads/#sandbox)
     
 1. Ensure that ZooKeeper is running and accessible
     
 2. Add the necessary properties to __gateway-site.xml__

    <img src="/assets/img/zk-demo-gatewayconfig.png"/>


 3. Create a providers configuration and a simple topology descriptor

    __sandbox-providers.xml__

        <gateway>
            <provider>
                <role>authentication</role>
                <name>ShiroProvider</name>
                <enabled>true</enabled>
                <param name="sessionTimeout" value="30"/>
                <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
                <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
                <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>
                <param name="main.ldapRealm.userDnTemplate" value="uid={0},ou=people,dc=hadoop,dc=apache,dc=org"/>
                <param name="main.ldapRealm.contextFactory.url" value="ldap://localhost:33389"/>
                <param name="main.ldapRealm.contextFactory.authenticationMechanism" value="simple"/>
                <param name="urls./**" value="authcBasic"/>
            </provider>
        </gateway>

    __docker-sandbox.json__

        {
          "discovery-address":"http://localhost:8080",
          "discovery-user":"maria_dev",
          "provider-config-ref":"sandbox-providers",
          "cluster":"Sandbox",
          "services":[
            {"name":"NAMENODE"},
            {"name":"JOBTRACKER"},
            {"name":"WEBHDFS"},
            {"name":"WEBHCAT"},
            {"name":"OOZIE"},
            {"name":"WEBHBASE"},
            {"name":"RESOURCEMANAGER"}
          ]
        }

 4. Provision the necessary aliases in the gateway instance(s)

        {GATEWAY_HOME}/bin/knoxcli.sh create-alias ambari.discovery.password --value maria_dev


 5. Start the demo ldap server and the gateway

        {GATEWAY_HOME}/bin/ldap.sh start
	    {GATEWAY_HOME}/bin/gateway.sh start

    __*{GATEWAY_HOME}*/logs/gateway.log:__
	<br><br><img src="/assets/img/log-monitor-started.png"/>

 6. *{GATEWAY_HOME}*/bin/knoxcli.sh upload-provider-config sandbox-providers.xml --registry-client sandbox-zookeeper-client

	    ls -l {GATEWAY_HOME}/conf/shared-providers/


 7. *{GATEWAY_HOME}*/bin/knoxcli.sh upload-descriptor docker-sandbox.json --registry-client sandbox-zookeeper-client

	    ls -l {GATEWAY_HOME}/conf/descriptors/
    	ls -l {GATEWAY_HOME}/conf/topologies/

    __Note the service URLs in *{GATEWAY_HOME}*/conf/topologies/docker-sandbox.xml__
    <br><br><img src="/assets/img/topo-generated-service-urls.png"/>

 8. Add the hostmap provider to the providers configuration

        <gateway>
            <provider>
                <role>authentication</role>
                <name>ShiroProvider</name>
                <enabled>true</enabled>
                <param name="sessionTimeout" value="30"/>
                <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
                <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
                <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>
                <param name="main.ldapRealm.userDnTemplate" value="uid={0},ou=people,dc=hadoop,dc=apache,dc=org"/>
                <param name="main.ldapRealm.contextFactory.url" value="ldap://localhost:33389"/>
                <param name="main.ldapRealm.contextFactory.authenticationMechanism" value="simple"/>
                <param name="urls./**" value="authcBasic"/>
            </provider>
            <provider>
                <role>hostmap</role>
                <name>static</name>
                <enabled>true</enabled>
                <param name="localhost" value="sandbox,sandbox.hortonworks.com"/>
            </provider>
        </gateway>

	And upload the change to ZooKeeper:

        {GATEWAY_HOME}/bin/knoxcli.sh upload-provider-config sandbox-providers.xml --registry-client sandbox-zookeeper-client

    __Note the *hostmap* provider in *{GATEWAY_HOME}*/conf/topologies/docker-sandbox.xml__
    <br><br><img src="/assets/img/topo-updated-providers.png"/>

 9. __*{GATEWAY_HOME}*/bin/knoxcli.sh delete-descriptor docker-sandbox.json --registry-client sandbox-zookeeper-client__

        ls -l {GATEWAY_HOME}/conf/descriptors/
    	ls -l {GATEWAY_HOME}/conf/topologies/

10. __*{GATEWAY_HOME}*/bin/knoxcli.sh delete-provider-config sandbox-providers.xml --registry-client sandbox-zookeeper-client__

        ls -l {GATEWAY_HOME}/conf/shared-providers/

<br>

# A Note About Aliases

Since a simple descriptor may trigger service URL discovery, it's important that the necessary Ambari alias(es) be provisioned in every Knox instance that may consume the descriptor.
For example, if a simple descriptor *my-cluster.json* declares a *discovery-pwd-alias* named *my-ambari-alias*, that alias must be provisioned for every Knox instance that will consume
*my-cluster.json*. Otherwise, service URL discovery will not work because Knox won't be able to be authenticated by Ambari to get the service configuration details.


<br><br><br><br>

