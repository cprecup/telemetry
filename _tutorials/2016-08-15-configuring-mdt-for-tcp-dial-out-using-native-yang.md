---
published: true
date: '2016-08-15 14:05 -0600'
title: Configuring Model-Driven-Telemetry (MDT) for Dial-out Using Native YANG
tags:
  - iosxr
postion: top
---

{% include toc icon="table" title="Configuring MDT dial-out using Native YANG" %}
{% include base_path %}

## Getting the Most out of MDT with Native YANG

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-25-configuring-model-driven-telemetry-mdt-with-yang/), I wrote about how to configure an MDT for gRPC dial-in using the OpenConfig Telemetry YANG model.  In this tutorial, I'll describe how to use the IOS XR Native YANG model to configure MDT with TCP and gRPC dialout.  I will use [ncclient](https://github.com/ncclient/ncclient) as a simple Python NETCONF client, but you can use whatever client you want.

## The Model

The Cisco IOS XR Native YANG model for telemetry is "Cisco-IOS-XR-telemetry-model-driven-cfg."  It can be used to configure any telemetry feature that IOS XR (unlike the OpenConfig telemetry model, which only covers a subset of IOS XR capabilities).

The NETCONF \<get-schema\> operation will give you the contents of the schema but the full YANG output can be really verbose and overwhelming, so I'll pipe the output to the [pyang](https://github.com/mbj4668/pyang) utility for a compact tree view with the following bit of code:

```python
from ncclient import manager
import re
    
xr = manager.connect(host='10.30.111.9', port=830, username='cisco', password='cisco',
                    allow_agent=False,
                    look_for_keys=False,
                    hostkey_verify=False,
                    unknown_host_cb=True)
                    
from subprocess import Popen, PIPE, STDOUT

oc = xr.get_schema('Cisco-IOS-XR-telemetry-model-driven-cfg')
p = Popen(['pyang', '-f', 'tree'], stdout=PIPE, stdin=PIPE, stderr=PIPE) 
print(p.communicate(input=oc.data)[0])

```

And voila:  

{% capture "output" %}

Script Output:

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
   +--rw telemetry-model-driven
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-identifier]
      |     +--rw sensor-paths
      |     |  +--rw sensor-path* [telemetry-sensor-path]
      |     |     +--rw telemetry-sensor-path    string
      |     +--rw enable?                    empty
      |     +--rw sensor-group-identifier    xr:Cisco-ios-xr-string
      +--rw subscriptions
      |  +--rw subscription* [subscription-identifier]
      |     +--rw source-address!
      |     |  +--rw address-family    Af
      |     |  +--rw ip-address?       inet:ipv4-address-no-zone
      |     |  +--rw ipv6-address?     string
      |     +--rw sensor-profiles
      |     |  +--rw sensor-profile* [sensorgroupid]
      |     |     +--rw sample-interval?      uint32
      |     |     +--rw heartbeat-interval?   uint32
      |     |     +--rw supress-redundant?    empty
      |     |     +--rw sensorgroupid         xr:Cisco-ios-xr-string
      |     +--rw destination-profiles
      |     |  +--rw destination-profile* [destination-id]
      |     |     +--rw enable?           empty
      |     |     +--rw destination-id    xr:Cisco-ios-xr-string
      |     +--rw source-qos-marking?        uint32
      |     +--rw subscription-identifier    xr:Cisco-ios-xr-string
      +--rw destination-groups
      |  +--rw destination-group* [destination-id]
      |     +--rw destinations
      |     |  +--rw destination* [address-family]
      |     |     +--rw address-family    Af
      |     |     +--rw ipv4* [ipv4-address destination-port]
      |     |     |  +--rw ipv4-address        inet:ip-address-no-zone
      |     |     |  +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |     |  +--rw protocol!
      |     |     |  |  +--rw protocol        Proto-type
      |     |     |  |  +--rw tls-hostname?   string
      |     |     |  |  +--rw no-tls?         int32
      |     |     |  +--rw encoding?           Encode-type
      |     |     +--rw ipv6* [ipv6-address destination-port]
      |     |        +--rw ipv6-address        xr:Cisco-ios-xr-string
      |     |        +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |        +--rw protocol!
      |     |        |  +--rw protocol        Proto-type
      |     |        |  +--rw tls-hostname?   string
      |     |        |  +--rw no-tls?         int32
      |     |        +--rw encoding?           Encode-type
      |     +--rw destination-id    xr:Cisco-ios-xr-string
      +--rw enable?               empty

```  

{% endcapture %}

<div class="notice--warning">

{{ output | markdownify }}

</div>

You can spend a lot of time understanding the intricacies of YANG and all the details, but all we really need to know for now is that the model has three major sections:  

- The **destination-group** tells the router where to send telemetry data and how. Only needed for dial-out configuration.  

- The **sensor-group** identifies a list of YANG models that the router should stream.  

- The **subscription** ties together the destination-group and the sensor-group.  


Let's see how this works in practice.


## Get-Config

We can use the openconfig-telemetry model to filter for the telemetry config with the ncclient get_config operation. Continuing our python script from above:

```python      
xr_filter = '''<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">'''

c = xr.get_config(source='running', filter=('subtree', xr_filter))

print(c)

```

And here's what we get:  

{% capture "output" %}

Script Output:

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:1ddd326c-e2c8-46b1-8433-11283799b9ce" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv4</address-family>
       <ipv4>
        <ipv4-address>172.30.8.4</ipv4-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>
         <protocol>tcp</protocol>
         <tls-hostname></tls-hostname>
         <no-tls>0</no-tls>
        </protocol>
       </ipv4>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <enable></enable>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <enable></enable>
   <subscriptions>
    <subscription>
     <subscription-identifier>Sub1</subscription-identifier>
     <sensor-profiles>
      <sensor-profile>
       <sensorgroupid>SGroup1</sensorgroupid>
       <sample-interval>30000</sample-interval>
      </sensor-profile>
     </sensor-profiles>
     <destination-profiles>
      <destination-profile>
       <destination-id>DGroup1</destination-id>
       <enable></enable>
      </destination-profile>
     </destination-profiles>
    </subscription>
   </subscriptions>
  </telemetry-model-driven>
 </data>
</rpc-reply>

```  

{% endcapture %}

<div class="notice--warning">

{{ output | markdownify }}

</div>

So what does all that mean to the router?  It breaks down into three parts which you'll recall from the YANG model above:  

- The **destination-group** tells the router where to send telemetry data and how.  The destination group in this configuration ("DGroup1") will send telemetry data to an IPv4 address (172.30.8.4) on port 5432 with a self-describing GPB encoding via TCP.

- The **sensor-group** identifies a list of YANG models that the router should stream.  In this case, the router has a sensor-group called "SGroup1" that will send interface statistics data from the IOS XR Native YANG model for interface stats.

- The **subscription** ties together the destination-group and the sensor-group.  This router has a subscription name "Sub1" that will send the list of models in SGroup1 to DGroup1 at an interval of 30 second (30000 milleseconds).  


If you read the [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/) on configuring MDT with CLI, you might recognize this as the same as the TCP dial-out configuration described there.  If you missed that thrilling installment, the XML above is the YANG equivalent of this CLI:  

{% capture "output" %}

CLI Output:

```
telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 172.30.8.4 port 5432  
   encoding self-describing-gpb  
   protocol tcp
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1 

``` 

{% endcapture %}

<div class="notice--info">

{{ output | markdownify }}

</div>


## Edit-Config

So let's say we want to add a second model (Cisco-IOS-XR-wdsysmon-fd-oper) to SGroup1 to stream cpu utilization data.  We can do that with the following NETCONF operations:

```python
edit_data = '''
<config>
<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
  </telemetry-model-driven>
</config>
'''

xr.edit_config(edit_data, target='candidate', format='xml')
xr.commit()
```

If we do a get-config operation again, this time filtering on just the sensor-groups:  

```python
xr_filter = '''<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg"><sensor-groups>'''

c = xr.get_config(source='running', filter=('subtree', xr_filter))

print(c)

```

... we'll see that SGroup1 has the new sensor-path.  


{% capture "output" %}

Script Output:

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:dbcef1db-83af-43f0-b2fe-153c53fc1f82" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <enable></enable>
     <sensor-paths>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring</telemetry-sensor-path>
      </sensor-path>
      <sensor-path>
       <telemetry-sensor-path>Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
  </telemetry-model-driven>
 </data>
</rpc-reply>
```

{% endcapture %}


<div class="notice--warning">

{{ output | markdownify }}

</div>

Now let's add an IPv6 destination to DGroup1 using gRPC dial-out and self-describing GPB encoding. You can do that with the following NETCONF operation:

```python
edit_data = '''
<config>
<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv6</address-family>
       <ipv6>
        <ipv6-address>2001:db8:0:100::b</ipv6-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>
         <protocol>grpc</protocol>
         <tls-hostname></tls-hostname>
         <no-tls>0</no-tls>
        </protocol>
       </ipv6>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
  </telemetry-model-driven>
</config>
'''

xr.edit_config(edit_data, target='candidate', format='xml')
xr.commit()
```

If we do a get-config operation again, this time filtering on just the destination group:  

```python
xr_filter = '''<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg"><destination-groups>'''

c = xr.get_config(source='running', filter=('subtree', xr_filter))

print(c)
```

... we'll see that DGroup1 has the new destination.  


{% capture "output" %}

Script Output:

```
<?xml version="1.0"?>
<rpc-reply message-id="urn:uuid:d3b9beaa-9b69-4f5c-a7a8-5d3dc106ce0f" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
 <data>
  <telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination>
       <address-family>ipv4</address-family>
       <ipv4>
        <ipv4-address>172.30.8.4</ipv4-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>
         <protocol>tcp</protocol>
         <tls-hostname></tls-hostname>
         <no-tls>0</no-tls>
        </protocol>
       </ipv4>
      </destination>
      <destination>
       <address-family>ipv6</address-family>
       <ipv6>
        <ipv6-address>2001:db8:0:100::b</ipv6-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>
         <protocol>grpc</protocol>
         <tls-hostname></tls-hostname>
         <no-tls>0</no-tls>
        </protocol>
       </ipv6>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
  </telemetry-model-driven>
 </data>
</rpc-reply>

```

{% endcapture %}


<div class="notice--warning">

{{ output | markdownify }}

</div>


And if you need some CLI to reassure yourself that it worked, here it is:

{% capture "output" %}

CLI Output:

```
RP/0/RP0/CPU0:SunC#show run telemetry model-driven

telemetry model-driven
 destination-group DGroup1
  address family ipv4 172.30.8.4 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
  address family ipv6 2001:db8:0:100::b port 5432
   encoding self-describing-gpb
   protocol grpc
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription Sub1
  sensor-group-id SGroup1 sample-interval 30000
  destination-id DGroup1

```

{% endcapture %}

<div class="notice--info">

{{ output | markdownify }}

</div>

## Clean-up Time
Since it's always a good idea to be able to remove what you configure, here's the XML instantiation of the YANG model to do that using the "remove" operation.  There are other ways to do this, but this is the most surgical.

```python
edit_data = '''
<config>
<telemetry-model-driven xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-telemetry-model-driven-cfg">
   <sensor-groups>
    <sensor-group>
     <sensor-group-identifier>SGroup1</sensor-group-identifier>
     <enable></enable>
     <sensor-paths>
      <sensor-path nc:operation="remove">
       <telemetry-sensor-path >Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring</telemetry-sensor-path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
  <destination-groups>
    <destination-group>
     <destination-id>DGroup1</destination-id>
     <destinations>
      <destination >
       <address-family>ipv6</address-family>
       <ipv6 nc:operation="delete">
        <ipv6-address>2001:db8:0:100::b</ipv6-address>
        <destination-port>5432</destination-port>
        <encoding>self-describing-gpb</encoding>
        <protocol>
         <protocol>tcp</protocol>
         <tls-hostname></tls-hostname>
         <no-tls>0</no-tls>
        </protocol>
       </ipv6>
      </destination>
     </destinations>
    </destination-group>
   </destination-groups>
  </telemetry-model-driven>
  </config>
'''

xr.edit_config(edit_data, target='candidate', format='xml')
xr.commit()

xr.close_session()
```

## Conclusion

The IOS XR Native telemetry YANG model exposes the full range of functionality in Model-Driven Telemetry.  The examples in this tutorial should get you started with configuring MDT in a programmatic way.  

