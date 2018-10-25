---
title: Tools for Running Knox in Docker
---

Recent development efforts in the [Apache Knox](http://knox.apache.org/) community
have been focused on making it easier to run and manage Knox in container environments.

The intent of this post is to briefly highlight two public projects that provide the
ability to get Knox up and running in a Docker container.

* [knox-docker](https://github.com/pzampino/knox-docker)
* [knox-dev-docker](https://github.com/moresandeep/knox-dev-docker)

[knox-docker](https://github.com/pzampino/knox-docker) provides the ability to
get an instance of a __released__ version of Knox up and running in a container.
Releases prior to [v1.0.0](https://cwiki.apache.org/confluence/display/KNOX/Release+1.0.0)
will be more difficult to use since you have to _bash into_ the resulting container(s)
to effect configuration (including topology) changes. Releases beginning with 
[v1.0.0](https://cwiki.apache.org/confluence/display/KNOX/Release+1.0.0) are a bit
easier to manage due to the associated Admin UI and API enhancements.

[knox-dev-docker](https://github.com/moresandeep/knox-dev-docker) provides the
ability to get an instance of Knox __from a development branch__ up and running
in a container. You can point this to any branch of the [Apache repo](git://git.apache.org/knox.git/)
or any fork of that repo. The Knox sources will be downloaded from the specified source
branch, and built to produce an instance of Knox in a container.

<br>

Both of these projects leverage docker-compose to create a container for the
demo LDAP server and a container for Knox, but you can modify the existing
docker-compose files (or create your own) to add Knox instances or remove the
demo LDAP server (for example).


<br><br><br><br>


