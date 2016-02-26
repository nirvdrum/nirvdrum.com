---
title: "Centralized Selenium Logging with Graylog"
author: Kevin Menard
tags: [opensource, selenium]
layout: post
excerpt: |-
  Selenium Grid is a fantastic way to run a cluster of browsers to speed up your testing. We make very heavy use of it to provide our Web Consistency Testing results. But running many distributed nodes and not knowing which one is running a session at any given time can be problematic from an ops standpoint. Sending all of our logs to a centralized location is one way that we manage all this.
---

[Selenium Grid](http://seleniumhq.org/docs/07_selenium_grid.jsp) is a fantastic way to run a cluster of browsers to speed up your testing.  We make very heavy use of it to provide our Web Consistency Testing results.  But running many distributed nodes and not knowing which one is running a session at any given time can be problematic from an ops standpoint.  Sending all of our logs to a centralized location is one way that we manage all this.

Graylog
-------

We use [Graylog 2](http://www.graylog2.org/) as our integration point (don't let the web page dissuade you &mdash; it's very solid software).  Graylog is built atop [elasticsearch](http://www.elasticsearch.org/), so it has great discovery capabilities and can segment your messages by various facets.  You can then save these configurations as "streams" that you can monitor or optionally have them send you alerts when certain conditions are met.

If you click through a stream, you can see all the messages in that stream.  Here I'm showing a snippet from our "Grid Nodes" stream.  Note how each message notes the host the message was sent from as well as the severity of that message.  We can use that information to drill down into an issue we need to debug or as the basis of an alert for a production problem.

<div class="figure">
  <img src="/images/static/centralized-selenium-logging/message-list.png" />

  Figure 1: Log messages in the "Grid Nodes" stream.
</div>

Configuring Selenium's Logger
-----------------------------

As it turns out, getting all this is fairly cheap.  If you haven't already, you may want to review the article on [Selenium RC Logging for Developers](http://wiki.openqa.org/display/SRC/Selenium+RC+Logging+for+Developers).  The article predates the Selenium 2 merge with WebDriver, but the logging system is the same.

First, you need to create your `logging.properties` file.  Graylog's logging format is called GELF and the GelfHandler serves as the root for the logger configuration:

<div class="figure">
  <pre>
  handlers = org.graylog2.logging.GelfHandler

  .level = ALL

  org.graylog2.logging.GelfHandler.graylogHost = my_graylog_server_hostname
  org.graylog2.logging.GelfHandler.graylogPort = 12201
  org.graylog2.logging.GelfHandler.originHost = ie901
  org.graylog2.logging.GelfHandler.extractStacktrace = true
  org.graylog2.logging.GelfHandler.facility = selenium
  </pre>

  Figure 2: Sample logging.properties config file.
</div>

Most of these values should be straightforward.  The `originHost` value allows you to set the hostname to be used for log messages sent from this host.  If not set, Java will try to figure it out based on your local network configuration.  We've found this often leads to non-descript and long names.  So, we opt to override on each host.  That does imply a host-specific configuration file, but we autogenerate it so it's not a problem in practice.

Next up, start the server with the configuration file.  Note you'll need to [download the gelfj.jar file](https://github.com/t0xa/gelfj/downloads) and its [json-simple.jar dependency](http://code.google.com/p/json-simple/downloads/list) and add them to the classpath when starting Selenium. 

<div class="figure">
  <pre>
  java -cp json-simple.jar:gelfj.jar:selenium-server-standalone.jar \
    -Djava.util.logging.config.file=logging.properties org.openqa.grid.selenium.GridLauncher \
    -role node
  </pre>

  Figure 3: Sample command to start up Selenium with logger configuration.
</div>

And with that, you should now see log messages appear in your Graylog installation.  NB: Since we overload the logger completely, no messages will appear in your console any longer.  Graylog will be the canonical source of all logs from this point forward.  You may want to consider logging to syslog and having that forward to Graylog instead for redundancy.

Bonus: Message Filtering
-----------------------------

If you've ever looked at your Selenium logs you know they can be quite verbose.  E.g., Selenium Grid nodes will log a message for every heartbeat message they send to the grid hub.  While arguably Selenium should make better use of log levels, one way to work around this is to set up [filtering rules in Graylog](https://github.com/Graylog2/graylog2-server/wiki/Message-processing-rewriting).  We do this to drop all heartbeat messages from appearing in Graylog at all, allowing us to focus on log messages that really matter.

<div class="figure">
  <pre class='prettyprint'>
  import org.graylog2.messagehandlers.gelf.GELFMessage

  rule "Drop Selenium Grid heartbeat messages"
    when
      m : GELFMessage( facility == "selenium" && fullMessage matches ".*\\/status\\)?$" )
    then
      m.setFilterOut(true);
      System.out.println("[Drop all Selenium Grid heartbeat messages] : " + m.toString() );
  end
  </pre>

  Figure 4: Sample Graylog rule for dropping verbose log messages.
</div>
