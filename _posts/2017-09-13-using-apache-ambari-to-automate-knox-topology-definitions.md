# Introduction & Motivation

Apache Knox acts as a proxy between Hadoop services and their consumers. Knox topology files define the mapping between requested services and their respective endpoints.
To make Knox aware of new mappings, a Knox administrator has to determine the endpoints for every service to be proxied, and add them to a topology file (XML). When working
with an Ambari-managed cluster, the Knox administrator has the advantage of Ambari's knowledge of the cluster details. However, navigating the Ambari UI to locate the
correct pieces of information to assemble these service endpoint URLs can be challenging, and having a human type them into an XML file introduces the potential for errors.

Fortunately, Ambari's API can be leveraged to automate the determination of these service endpoint URLs and, ultimately, the generation of Knox topology files.

# Knox Topology Files

A topology file has to include a service entry for every Hadoop service to be proxied.

#### Listing 1: Example topology XML file

    <topology>
        <gateway><!-- Provider Configurations --></gateway>
        <!-- Service Endpoint Mappings -->
        <service>
            <role>NAMENODE</role>
            <url>hdfs://c6401.ambari.apache.org:8020</url>
        </service>
        <service>
            <role>JOBTRACKER</role>
            <url>rpc://c6402.ambari.apache.org:8050</url>
        </service>
        <service>
            <role>WEBHDFS</role>
            <url>http://c6401.ambari.apache.org:50070/webhdfs</url>
        </service>
        <service>
            <role>WEBHCAT</role>
            <url>http://c6402.ambari.apache.org:50111/templeton</url>
        </service>
        <service>
            <role>OOZIE</role>
            <url>http://c6402.ambari.apache.org:11000/oozie</url>
        </service>
        <service>
            <role>WEBHBASE</role>
            <url>http://c6401.ambari.apache.org:60080</url>
        </service>
        <service>
            <role>HIVE</role>
            <url>http://c6402.ambari.apache.org:10001/cliservice</url>
        </service>
        <service>
            <role>RESOURCEMANAGER</role>
            <url>http://c6402.ambari.apache.org:8088/ws</url>
        </service>
        <service>
            <role>AMBARI</role>
            <url>http://c6401.ambari.apache.org:8080</url>
        </service>
    </topology>

To populate the *WEBHDFS* role as it is in __Listing 1__, you have to login to the Ambari console, navigate to the *HDFS* configurations, find the *Advanced hdfs-site* configuration, and look for the *dfs.namenode.__http__-address* property value. If you do this, you'll also notice that there is another property named *dfs.namenode.__https__-address*. It would be quite easy for a person to grab the value of the latter property rather than the former. Furthermore, this configuration does not tell you anything about the __/webhdfs__ path required for this service. It may also be important to note the value of the *General* configuration's *dfs.webhdfs.enabled* property.
	
Similarly, for the 
*JOBTRACKER* role, you have to know to navigate to the *YARN* configurations, and check the *Advanced yarn-site* configuration to get the *yarn.resourcemanager.__address__* property value; make sure you don't grab the adjacent *yarn.resourcemanager.__admin.address__* by mistake. And remember that its URL scheme needs to be <em style="background-color: initial;">rpc*.<br>

Hopefully, you can see that correctly populating these service endpoint URLs is not a simple task, especially for individuals who are new to Hadoop, Ambari and/or Knox. HA deployments further increase this difficulty.

# Automated Topology Generation With the Ambari API

## REST API

The <a href="https://github.com/apache/ambari/blob/trunk/ambari-server/docs/api/v1/index.md">Ambari REST API</a> provides the ability to programmatically determine the disparate pieces of information necessary for assembling service endpoint URLs. Software can be written to correctly construct each of these service endpoint URLs from the correct properties, eliminating the potential for human error in identifying the appropriate properties and attempting to construct the corresponding URLs.

There are two especially interesting resources available from the Ambari API:

1. __*/api/v1/clusters/CLUSTER_NAME/services?fields=components/host_components/HostRoles*__

    This resource describes the service component host mappings for a cluster, which is useful for determining the host address for some service components. 
    *(Note: CLUSTER_NAME is a placeholder for an actual cluster name; here and throughout)*
	
    __*Listing 2: Excerpt of service host components API response*__

        “items” : [
          {
            “href” : "http://AMBARI_ADDRESS/api/v1/CLUSTER_NAME/CLUSTER_NAME/services/HIVE",
            “components” : [
              {
                “ServiceComponentInfo” : { },
                “host_components” : [
                  {
                    “HostRoles” : {
                      “cluster_name” : “CLUSTER_NAME”,
                      “component_name” :  “HCAT”,
                      “host_name” : “c6402.ambari.apache.org”
                    }
                  }
                ]
              },
              {
                “ServiceComponentInfo” : { },
                “host_components” : [
                  {
                    “HostRoles” : {
                      “cluster_name” : “CLUSTER_NAME”,
                      “component_name” :  “HIVE_SERVER”,
                      “host_name” : “c6402.ambari.apache.org”
                    }
                  }
                ]
              }
            ]
          },
          {
            “href” : "http://AMBARI_ADDRESS/api/v1/CLUSTER_NAME/CLUSTER_NAME/services/HDFS",
            “ServiceInfo” : {},
            “components” : [
              {
                “ServiceComponentInfo” : { },
                “host_components” : [
                  {
                    “HostRoles” : {
                      “cluster_name” : “CLUSTER_NAME”,
                      “component_name” :  “NAMENODE”,
                      “host_name” : “c6401.ambari.apache.org”
                    }
                  }
                ]
              }
            ]
          }
        ]

    In __Listing 2__, we see that the *HIVE_SERVER* host is *c6402.ambari.apache.org*, and that the *NAMENODE* host is *c6401.ambari.apache.org*. An actual response would be more complete in terms of the number of mappings.

2. __*/api/v1/clusters/ CLUSTER_NAME /configurations/service_config_versions?is_current=true*__

    This resource lists a cluster's active service configuration versions and their respective contents.

    __*Listing 3: Excerpt of service_config_versions API response*__

       "items" : [
	  {
	      "href" : "http://AMBARI_ADDRESS/api/v1/clusters/CLUSTER_NAME/configurations/service_config_versions?service_name=HDFS&service_config_version=2",
	      "cluster_name" : "CLUSTER_NAME",
	      "configurations" : [
	        {
	          "Config" : {
	            "cluster_name" : "CLUSTER_NAME",
	            "stack_id" : "HDP-2.6"
	          },
	          "type" : "ssl-server",
	          "properties" : {
	            "ssl.server.keystore.location" : "/etc/security/serverKeys/keystore.jks",
	            "ssl.server.keystore.password" : "SECRET:ssl-server:1:ssl.server.keystore.password",
	            "ssl.server.keystore.type" : "jks",
	            "ssl.server.truststore.location" : "/etc/security/serverKeys/all.jks",
	            "ssl.server.truststore.password" : "SECRET:ssl-server:1:ssl.server.truststore.password"
	          },
	          "properties_attributes" : { }
	        },
	        {
	          "Config" : {
	            "cluster_name" : "CLUSTER_NAME",
	            "stack_id" : "HDP-2.6"
	          },
	          "type" : "hdfs-site",
	          "tag" : "version1",
	          "version" : 1,
	          "properties" : {
	            "dfs.cluster.administrators" : " hdfs",
	            "dfs.encrypt.data.transfer.cipher.suites" : "AES/CTR/NoPadding",
	            "dfs.hosts.exclude" : "/etc/hadoop/conf/dfs.exclude",
	            "dfs.http.policy" : "HTTP_ONLY",
	            "dfs.https.port" : "50470",
	            "dfs.journalnode.http-address" : "0.0.0.0:8480",
	            "dfs.journalnode.https-address" : "0.0.0.0:8481",
	            "dfs.namenode.http-address" : "c6401.ambari.apache.org:50070",
	            "dfs.namenode.https-address" : "c6401.ambari.apache.org:50470",
	            "dfs.namenode.rpc-address" : "c6401.ambari.apache.org:8020",
	            "dfs.namenode.secondary.http-address" : "c6402.ambari.apache.org:50090",
	            "dfs.webhdfs.enabled" : "true"
	          },
	          "properties_attributes" : {
	            "final" : {
	              "dfs.webhdfs.enabled" : "true",
	              "dfs.namenode.http-address" : "true",
	              "dfs.support.append" : "true",
	              "dfs.namenode.name.dir" : "true",
	              "dfs.datanode.failed.volumes.tolerated" : "true",
	              "dfs.datanode.data.dir" : "true"
	            }
	          }
	        }
	      ]
	    },
	    {
	      "href" : "http://AMBARI_ADDRESS/api/v1/clusters/CLUSTER_NAME/configurations/service_config_versions?service_name=YARN&service_config_version=1",
	      "cluster_name" : "CLUSTER_NAME",
	      "configurations" : [
	        {
	          "Config" : {
	            "cluster_name" : "CLUSTER_NAME",
	            "stack_id" : "HDP-2.6"
	          },
	          "type" : "yarn-site",
	          "properties" : {
	            "yarn.http.policy" : "HTTP_ONLY",
	            "yarn.log.server.url" : "http://c6402.ambari.apache.org:19888/jobhistory/logs",
	            "yarn.log.server.web-service.url" : "http://c6402.ambari.apache.org:8188/ws/v1/applicationhistory",
	            "yarn.nodemanager.address" : "0.0.0.0:45454",
	            "yarn.resourcemanager.address" : "c6402.ambari.apache.org:8050",
	            "yarn.resourcemanager.admin.address" : "c6402.ambari.apache.org:8141",
	            "yarn.resourcemanager.ha.enabled" : "false",
	            "yarn.resourcemanager.hostname" : "c6402.ambari.apache.org",
	            "yarn.resourcemanager.webapp.address" : "c6402.ambari.apache.org:8088",
	            "yarn.resourcemanager.webapp.delegation-token-auth-filter.enabled" : "false",
	            "yarn.resourcemanager.webapp.https.address" : "c6402.ambari.apache.org:8090",
	          },
	          "properties_attributes" : { }
	        },
	      ]
	    }
       ]

    In __Listing 3__, we see the *ssl-server*, *hdfs-site*, and *yarn-site* configurations, which are the active versions for the cluster at the time the resource was accessed. Again, an actual response would be more complete, including all of the active configurations for the cluster.

# Topology Generation

Furthermore, a tool that subsequently generates Knox topology files based on the cluster configuration information would deal with the errors associated with copying endpoint URLs into the XML manually.

I've created a rudimentary implementation of such a tool to demonstrate it's usefulness. It's a small set of Python scripts, which can be found [here](https://github.com/pzampino/knox-topology-gen/).

This tool accepts simple YAML descriptors as input toward generating proper Knox topology files.

#### Listing 4: demo.yml

    ---
    # Discovery info source
    discovery-registry : http://c6401.ambari.apache.org:8080
    
    # Provider config reference, the contents of which will be
    # included in the resulting topology descriptor.
    # The contents of this reference has a <gateway/> root, and
    # contains <provider/> configurations.
    provider-config-ref : ambari-cluster-policy.xml
    
    # The cluster for which the service details should be discovered
    cluster: mycluster
    
    # The services to declare in the resulting topology descriptor,
    # whose URLs will be discovered (unless a value is specified)
    services:
        - name: NAMENODE
        - name: JOBTRACKER
        - name: WEBHDFS
        - name: WEBHCAT
        - name: OOZIE
        - name: WEBHBASE
        - name: HIVE
        - name: RESOURCEMANAGER
        - name: AMBARI
          url: http://c6401.ambari.apache.org:8080
        - name: AMBARIUI
          url: http://c6401.ambari.apache.org:8080

The *__discovery-registry__* is simply the Ambari API address for the instance managing the cluster for which the endpoint URLs are desired.

The *__provider-config-ref__* entry points to *ambari-cluster-policy.xml*; This referenced file provides the *<gateway/>* element content that is found in a topology file, and the tool copies this referenced content directly into its XML output. A nice side-effect of using provider configuration references is that these configurations can be more easily shared among Knox topologies.

It is necessary to specify a *__cluster__* because a Knox topology is a mapping for a specific cluster.

You'll also notice that some of the *__services__* entries have an associated *__url__*, while others do not. This is simply a way to tell the tool that it doesn't need to discover these URLs. The endpoint URLs for those services for which there is no *__url__* specified will be populated from the details discovered from Ambari.

# Running the tool

A very simple script is provided for testing the TopologyBuilder python object. , and it's output looks like the following:

    knox-topology-gen pzampino$ python test_build_topology.py
    
    NAMENODE        : hdfs://c6401.ambari.apache.org:8020<br>
    JOBTRACKER      : rpc://c6402.ambari.apache.org:8050<br>
    WEBHDFS         : http://c6401.ambari.apache.org:50070/webhdfs<br>
    WEBHCAT         : http://c6402.ambari.apache.org:50111/templeton<br>
    OOZIE           : http://c6402.ambari.apache.org:11000/oozie<br>
    WEBHBASE        : http://c6401.ambari.apache.org:60080<br>
    HIVE            : http://c6402.ambari.apache.org:10001/cliservice<br>
    RESOURCEMANAGER : http://c6402.ambari.apache.org:8088/ws<br>
    AMBARI          : http://c6401.ambari.apache.org:8080<br>
    AMBARIUI        : http://c6401.ambari.apache.org:8080<br>
    
    Generated demo.xml
    

Note that <i style="background-color: initial;">__demo.xml__</i> is a complete Knox topology file, which can be copied to the <i style="background-color: initial;">$GATEWAY_HOME/conf/topologies</i> directory for deployment.

Check out the [source](https://github.com/pzampino/knox-topology-gen/) for this tool for more details. Hopefully, this information is a help to those who need to define Knox topologies.

