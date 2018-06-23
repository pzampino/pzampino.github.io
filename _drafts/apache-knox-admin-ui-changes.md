---
title: New and Improved Apache Knox Admin UI
---

# Introduction

Apache Knox 1.0.0 added support for topology generation from simplified descriptors and shareable provider configurations. Rather than XML, these topology source
configurations can be defined in either JSON or YAML. While these can be hand-coded and manually copied to the Knox host(s), the Knox Admin UI provides a more
user-friendly facility for authoring them, which also does __not__ require ssh/scp access to the Knox host(s). Furthermore, using the Admin UI simplifies the management
of topologies in Knox HA deployments by eliminating the need to copy files to multiple Knox hosts.

The Admin UI has provided the ability to view, add, modify and delete topologies since it was created. Starting with Apache Knox 1.1.0, the ability to do the same
for provider configurations and descriptors has been added.

The Admin UI can be accessed at the following URL (where __*hostname*__ is the name of a Knox host) : __https://*hostname*:8443/gateway/manager/admin-ui__

This post will focus on working with the provider configuration and descriptor resources. For these types, the <img src="/assets/img/adminui-changes/plus-icon.png" style="height:20px;vertical-align:bottom"> icon next to the resource list header presents the respective facility for creating a new resource of that type.<br>
Modification options, including deletion, are available from the detail view for an individual resource.

<br>

# Provider Configurations

By default, there is a provider configuration named __*default-providers*__.

<img src="/assets/img/adminui-changes/adminui_default-providers_view.png"/>

## Editing
For each provider in a given provider configuration, the attributes can be modified:
* The provider can be enabled/disabled
* Parameters can be added (<img src="/assets/img/adminui-changes/plus-icon.png" style="height:20px;vertical-align:bottom">) or removed (<img src="/assets/img/adminui-changes/x-icon.png" style="height:12px;vertical-align:middle">)
* Parameter values can be modified (by clicking on the value)
  <img src="/assets/img/adminui-changes/adminui_edit.png"/>

<br>
To persist changes, the <img src="/assets/img/adminui-changes/save-icon.png" style="height:32px;vertical-align:bottom"> button must be clicked. To revert *unsaved* changes, click the  <img src="/assets/img/adminui-changes/undo-icon.png" style="height:32px;vertical-align:bottom"> button or simply choose another resource.
<br>

## Wizards
The Admin UI gives administrators the ability to define provider configurations based on their functional needs. So, rather than having to know the provider names
and their respective parameter names, the provider configuration wizard can be used.

A provider configuration is a named set of providers. The wizard allows an administrator to specify the name, and add providers to it.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig.png"/>

To add a provider, first a category must be chosen. For this example, let's add an authentication provider.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_categories.png"/>

After choosing a category, the type within that category must be selected. For this example, let's choose LDAP authentication.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_authentication.png"/>

Finally, for the selected type, the type-specific parameter values can be specified. Here, the demo LDAP server details can be configured.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_authentication_ldap.png"/>

After adding a provider, others can be added similarly.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_add-more.png"/>

## Composite Provider Types
The wizard for some provider types, such as the HA provider, behave a little differently than the other provider types.

For example, when you choose the HA provider category, you subsequently choose a service role (e.g., WEBHDFS), and specify the parameter values for that service
role's entry in the HA provider.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_ha.png"/>

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_ha_webhdfs.png"/>

If multiple services are configured in this way, the result is still a single HA provider, which contains all of the service role configurations.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_ha_summary.png"/>


## Persisting the New Provider Configuration

After adding all the desired providers to the new configuration, choosing <img src="/assets/img/adminui-changes/ok-button.png" style="height:24px;vertical-align:bottom"> persists it.

Following our example, the resulting provider configuration has two providers: authentication and ha.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_listing.png"/>

Expanding these providers, we find the details that were specified in the various wizard steps.

<img src="/assets/img/adminui-changes/adminui_new-providerconfig_details.png"/>

Now, this provider configuration can be referenced by one or more descriptors to be incorporated into the respective generated topologies.

<br>

# Descriptors

Out of the box, there are no descriptors in the __conf/descriptors/__ directory; thus, there are no descriptors listed in the Admin UI.
Fortunately, the Admin UI makes it easy to add one.

A descriptor is essentially a named set of service roles to be proxied with a provider configuration reference.
The __new descriptor__ dialog provides the ability to specify the name, which will also be the name of the resulting topology. It also
allows one or more supported service roles to be selected for inclusion.

<img src="/assets/img/adminui-changes/adminui_new-descriptor.png"/>

The provider configuration reference can entered manually, or the provider configuration selector can be used, to specify the name of an
existing provider configuration.

<img src="/assets/img/adminui-changes/adminui_new-descriptor_providers.png"/>

Optionally, discovery details can also be specified to direct Knox to discover the endpoints for the declared service roles from the Ambari-managed
target cluster.

<img src="/assets/img/adminui-changes/adminui_new-descriptor_discovery.png"/>

Choosing <img src="/assets/img/adminui-changes/ok-button.png" style="height:24px;vertical-align:bottom"> results in the persistence of the descriptor, and subsequently, the generation and deployment of the associated topology.

<img src="/assets/img/adminui-changes/adminui_descriptor_view_discovery.png"/>


<br>


# Knox HA Considerations

If the Knox instance which is hosting the Admin UI is configured for remote configuration monitoring, then provider configuration and descriptor changes will
be persisted in the configured ZooKeeper ensemble. Then, every Knox instance which is also configured to monitor configuration in this same ZooKeeper will apply
those changes, and [re]generate/[re]deploy the affected topologies. In this way, Knox HA deployments can be managed by making changes once, and from any of the
Knox instances.

More details are available in the [User Guide](http://knox.apache.org/books/knox-1-0-0/user-guide.html#Remote+Configuration+Monitor).

<br>


# Summary

These improvements to the Admin UI make it even easier to take advantage of the service discovery and topology generation features of Knox. Now, administrators
don't even have to bother writing JSON or YAML files directly, and they don't need to access the Knox host(s) via ssh/scp to effect topology changes. Finally, by
configuring multiple Knox instances in an HA manner, changes to provider configurations and descriptors are automatically propagated to all participating Knox
instances, simplifying management of Knox HA deployments.


More details are available in the [User Guide](http://knox.apache.org/books/knox-1-1-0/user-guide.html).


<br><br><br><br>


