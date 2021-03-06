---
published: true
date: '2017-06-21 10:22 -0600'
title: Enhancing a CiscoLive Demo with Telemetry and Kafka
author: Shelly Cadora
excerpt: Gives the background of a CiscoLive Telemetry and Kafka Demo
tags:
  - iosxr
  - cisco
  - Telemetry
  - MDT
  - YDK
  - Kafka
position: top
---

{% include toc icon="table" title="Enhancing Demos with Telemetry" %}
{% include base_path %}

## Behind the Scenes of Continuous Automation

Every year at Cisco Live, my team helps put together the demos that go into the World of Solutions. Geeks that we are, we get a thrill out of showing off the art of the possible. But the real purpose of a demo is to start a conversation with the folks who stop by our booth.     

This year, a colleague asked me to help integrate model-driven telemetry (MDT) into a Service Provider demo called "Continuous Automation." The goal of the demo is to illustrate how MDT can be used with model-driven APIs to automate a simple provisioning and validation task (it's loosely based on a real customer use case that he's actively working on).  

Pieces of the demo were already in place: a small Python app that utilized the [YDK Python APIs](http://ydk.io) to configure a BGP neighbor and execute a connectivity test from the router when the neighbor came up.  The problem was that the app had no way to know _when_ the neighbor came up.  Enter MDT!

## The Easy Part: Data Model and Router Config
The operational data that we needed was the BGP neighbor session state.  This is easily available in the OpenConfig BGP model:

```
module: openconfig-bgp
   +--rw bgp
      +--rw neighbors
         +--rw neighbor* [neighbor-address]
            +--ro state
               +--ro session-state?   enumeration
```

Translating this into an MDT sensor path config for the router looks like this:

```
telemetry model-driven
 sensor-group BGP
  sensor-path openconfig-bgp:bgp/neighbors/neighbor/state
```

**Note:** For a detailed explanation of MDT router configurations, see my [basic MDT tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/)).
{: .notice--warning}

<div class="notice--">
{{ output | markdownify }}
</div>
Adding a destination-group and a subscription starts the router streaming out the needed data:

```
telemetry model-driven
 destination-group G1
  address-family ipv4 198.18.1.127 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 subscription S1
  sensor-group-id BGP sample-interval 5000
  destination-id G1
```

 But then what?  How do you get data from a TCP stream into a Python app?

## The Other Easy Part: Pipeline and Kafka

My go-to tool for consuming MDT data is [pipeline](https://github.com/cisco/bigmuddy-network-telemetry-pipeline), an open source utility that I've [written about before](http://blogs.cisco.com/sp/introducing-pipeline-a-model-driven-telemetry-collection-service).  If you're not familiar with installing and configuring pipeline, have a read through my [previous tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/).  

For this demo, I used the  ```[testbed]``` input stage in the default pipeline.conf.  With the following lines uncommented, the default pipeline.conf will work "as-is" with the router MDT configuration in the previous section.

{% capture "output" %}

```
[testbed]
stage = xport_input
type = tcp
encap = st
listen = :5432
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

That's good for input, but what about output?  Pipeline can write data to three destinations:
- a file
- time series databases like InfluxDB
- Kafka

Writing to a file would probably work (Python has extensive file handling capabilities) but it seemed clumsy.  Writing to InfluxDB would also have worked (I could use Python REST packages to query the database) but seemed too heavy weight for a simple demo.  That left me with Kafka.  I've been wanting to do a Kafka demo for a while and there are Python packages to work with Kafka, so I figured...why not?  If nothing else, I'll learn something new.

For pipeline to output to Kafka, all you have to do is uncomment the following lines in the ```[mykafka]``` section of the default pipeline.conf. In the example below, I'm running pipeline and Kafka on the same machine, so I used the broker address of "localhost" and the topic called "telemetry."

{% capture "output" %}

```
[mykafka]
stage = xport_output
type = kafka
encoding = json
brokers = localhost:9092
topic = telemetry
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

With those two entries in the pipeline.conf file, I kicked off pipeline as usual:

{% capture "output" %}

```
$ bin/pipeline &
[1] 21975
Startup pipeline
Load config from [pipeline.conf], logging in [pipeline.log]
Wait for ^C to shutdown
$
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

## The Easiest Part
Since I haven't installed Kafka before, I was concerned that it might be the long pole in my demo prep.  But it couldn't have been easier.  I followed the first two steps in the [Apache Kafka Quickstart](https://kafka.apache.org/quickstart) guide.  Boom.  Done.  Didn't even have to alter the default properties files for Kafka and Zookeeper.

## A Quick Python Script
With Kafka, Zookeeper and Pipeline running and the router streaming MDT, all I lacked was a little Python code to subscribe to the topic on Kafka and parse some JSON (by default, pipeline transforms the GPB from the router into a JSON object when it publishes to Kafka). With the [kafka-python client](https://pypi.python.org/pypi/kafka-python), there wasn't much to it.  Here are a few lines of code I used for a quick test (note that the topic is ```telemetry``` and the Pipeline/Kafka stack is running on ```10.30.111.4```):

<div class="highlighter-rouge">
<pre class="highlight">
<code>
from kafka import KafkaConsumer
import json

if __name__ == "__main__":

    session_state = "UNKNOWN"
    consumer = KafkaConsumer('<mark>telemetry</mark>', bootstrap_servers=<mark>["10.30.111.4:9092"]</mark>)

    for msg in consumer:
        telemetry_msg =  msg.value
        telemetry_msg_json = json.loads(telemetry_msg)

        if "Rows" in telemetry_msg_json:
            content_rows = telemetry_msg_json["Rows"]
            for row in content_rows:
            if row["Keys"]["neighbor-address"] == '10.8.0.1':
                    new_session_state = row["Content"]["session-state"]
                    if session_state != new_session_state:
                        print("\nSession state changed from {0:s} to {1:s} at epoch time {2:d}"
                              .format(session_state, new_session_state, row["Timestamp"]))
                        session_state = new_session_state
</code>
</pre>
</div>

## In Sum
There was a little more code to write to tie everything together and tidy it up, but my part of the demo was basically done.  From a telemetry perspective, it was trivial to integrate into the demo by using Kafka.  To recap, the main pieces were:
- Configure the router to stream BGP session state.
- Configure (basically uncomment some lines in the default pipeline.conf) and run pipeline.
- Download and run Kafka and Zookeeper.
- Use the kafka-python package in a Python script to acquire and process the session state from the telemetry topic on Kafka.

Although I didn't get the learning experience that comes from having really complicated things go deeply wrong, this was a fun little exercise.  

If you're in Las Vegas for CiscoLive next week, stop by our booth and talk to us about what you want to do in a model-driven network!
