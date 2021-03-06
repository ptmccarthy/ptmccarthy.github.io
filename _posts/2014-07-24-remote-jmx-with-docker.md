---
layout: post
title: Remote Java Debugging With Docker
---

We've jumped on board the [Docker.io](https://www.docker.io) bandwagon at work, and while it's a great tool for automating and orchestrating dev/test environments, the transition hasn't been without its bumps. One of the recent pain points that I had to help troubleshoot was how to get remote JMX to work with Java applications (specifically, Tomcat6 webapps) running inside of a Docker container. This is something that should have been trivially easy, but documentation and "how to" posts on the topic were scarce.

### The Problem

Tomcat6 relies on remote [JMX](http://en.wikipedia.org/wiki/Java_Management_Extensions) and [JDWP](http://docs.oracle.com/javase/7/docs/technotes/guides/jpda/jdwp-spec.html) being configured both within Tomcat and that its ports be accessible through the network stack. No surprises, typical port-forwarding business.

Following typical configuration advice that can be found all over the web, we can configure JMX and JDWP for Tomcat in `/etc/sysconfig/tomcat6`

```
CATALINA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=62911,server=y,suspend=n"
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote="
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote.port=1898 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
CATALINA_OPTS="${CATALINA_OPTS} -Djava.rmi.server.hostname=10.8.1.106"
```

Note: This configuration is strictly for dev environments so authentication and SSL are disabled. Don't do this in production!!

At this point, if you were going to connect a debugger or JConsole on the same host that Tomcat runs on you'd probably be in good shape.

But we're using Docker to run our Tomcat instance inside of a container. There are [pretty good docs available](https://docs.docker.com/userguide/) for Docker over on their web site, so I'll leave out that bit. I'm going to assume you already have a Tomcat Docker image set up and ready to go.

An important Docker-related note about the Tomcat configuration above is that the `-Djava.rmi.server.hostname` must be set to the *externally accessible* IP address of the Tomcat server. You want to use the address of the Docker host, *not* the Docker-assigned internal IP address.

When we run our Docker Tomcat image to create the container we map the ports we want forwarded from the Docker host to the container. For example, mapping 2022 to 22 for SSH (because we want to leave 22 for the Docker host), 8080 for Tomcat, 1898 for JMX, and 62911 for JDWP looks like:

```
docker run -d -p 2022:22 -p 8080:8080 -p 1898:1898 -p 62911:62911 <image_id>
```

So now you think everything would be peachy. Your ports are all mapped, your application is running, but every time you try to connect to Tomcat via JConsole or your IDE debugger you get `Connection Refused` errors. You can telnet to your various ports to convince yourself they are all open and listening. Tomcat is humming away in its container and you can access your webapp via web browser at port 8080 all day long. So what gives?

### The Cause, and the Solution

As it turns out, sun.management.jmxremote *dynamically assigns a second port to use for [RMI](http://en.wikipedia.org/wiki/Java_remote_method_invocation)*. A well-configured firewall is probably smart enough to handle it, as I assume the connection on this new, dynamically assigned port is initiated as an outbound connection at first. Docker, however, does not seem to be as smart. Docker needs to know all of its port mappings at `docker run` time, and can't map additional ports later. So this is a big problem for us.

After many hours of head-banging, troubleshooting, searching, and basically thinking I was going insane, there is a scantly-mentioned configuration option that was introduced in Java version 7u4 that allows you to override the random RMI port. Not only can you assign a static port, you can also assign it to the same port that JMX listens on for convenience.

The magic secret sauce is:

```
-Dcom.sun.management.jmxremote.rmi.port=1898
```

Which, when added to the rest of the options, looks like

```
CATALINA_OPTS="-Xdebug -Xrunjdwp:transport=dt_socket,address=62911,server=y,suspend=n"
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote="
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote.port=1898 -Dcom.sun.management.jmxremote.rmi.port=1898 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
CATALINA_OPTS="${CATALINA_OPTS} -Djava.rmi.server.hostname=10.8.1.106"
```

At last, JConsole and Intellij can connect to a remote Tomcat server running in a Docker container.


*2016-02-19 Note: Fixed an error in the final configuration snippet pointed out by Mathieu Denis. Thanks!*
