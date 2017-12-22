---
title:  "Monitoring UPS Performance using Slack, SNMP, and Datadog"
---
<img src="/images/2017/180_fam.jpg" style="max-width: 300px; display: block; margin: 0 auto;"/>
Recently, a APC Symmetra LX 100Amp UPS was donated to Computer Science House by the National Technical Institute for the Deaf, a college at  RIT. Taken from the their decommissioned datacenter, we transported the unit to our server room and began planning for the required construction necessary for the installation of a 100 Amp line. Up to this point our infrastructure was powered by three 120V 3-Phase lines.

Once the UPS was installed and properly functioning we began the gradual process of running new PDUs to racks and moving boxes with redundant PSUs to UPS PDUs. Now that the UPS is fully integrated into our data center, we wanted to make sure that we keep a close eye on it.

## Step One: Slack Notifications

At CSH we have a Slack channel dedicated to monitoring notifications. Our system administrators have notification enabled for this channel and when something is posted, we are all alerted. The Network Management Card (NMC) plugged into the UPS has the ability to send notification emails when faults or events occur, so I configured a [Slack Email Integration](https://get.slack.help/hc/en-us/articles/206819278-Send-emails-to-Slack#connect-the-email-app-to-your-workspace) and pointed these alert emails to the forwarding email Slack provides. Now, when an alert is generated it is sent to Slack and is posted in `#monitoring`. 

## Step Two: Actual Data

<img src="/images/2017/board.png" style="max-width: 300px; display: block; margin: 0 auto;"/>

Getting alerts for when things are broken is nice, but being able to view live statistics is infinitely better. In order to get these statistics I configured the NMC to allow SNMP communication and began configuring the Datadog [SNMP integration](https://docs.datadoghq.com/integrations/snmp/). APC provides a [custom MIB](http://www.apc.com/shop/us/en/products/PowerNet-MIB-v4-2-3/P-SFPMIB423) (Management Information Base) for their products. If you aren't familiar with how SNMP works, each monitor is assigned a unique ID and then MIBs map those IDs to human-readable names. For example: `3.6.1.4.1.318.1.1.1.2.2.3` translates to `upsAdvBatteryRunTimeRemaining`. MIBs use a tree based structure so, again using the above example, `upsAdvBatteryRunTimeRemaining` (3) is a object under `upsAdvBattery` (2) which is under the `ups` (2) object, a subset of `hardware` (1), etc. 

Using `pysnmp`, which is available in the EPEL repo in RHEL/CentOS and standard repo in Fedora, I converted the MIB to Python data structures for use in the Datadog integration. 

There is more information about this process in their guide if this is something you are looking to configure. In our setup I placed the exported file in `/etc/dd-agent/mibs/`. You can put them anywhere, but I like to keep everything together. 

Now that Datadog has the MIB information, we can enable and configure the integration. To do this you copy `/etc/dd-agent/conf.d/snmp.yaml.example` to `/etc/dd-agent/conf.d/snmp.yaml`. Depending on the information you want out of the UPS your configuration might vary.

```
init_config:
  mibs_folder: /etc/dd-agent/mibs

instances:
  - community_string: public
    ip_address: 129.21.49.3
    port: 161
    snmp_version: 1
    enforce_mib_constraints: true
    metrics:
      - MIB: PowerNet-MIB
        symbol: upsAdvBatteryCapacity
      - MIB: PowerNet-MIB
        symbol: upsAdvBatteryTemperature
      - MIB: PowerNet-MIB
        symbol: upsAdvBatteryRunTimeRemaining
        forced_type: gauge
      - MIB: PowerNet-MIB
        symbol: upsAdvInputLineVoltage
      - MIB: PowerNet-MIB
        symbol: upsAdvInputFrequency
      - MIB: PowerNet-MIB
        symbol: upsAdvOutputVoltage
      - MIB: PowerNet-MIB
        symbol: upsAdvOutputFrequency
      - MIB: PowerNet-MIB
        symbol: upsAdvOutputLoad
      - MIB: PowerNet-MIB
        symbol: upsAdvOutputCurrent
      - MIB: PowerNet-MIB
        symbol: upsAdvBatteryNumOfBattPacks
      - MIB: PowerNet-MIB
        symbol: upsAdvBatteryNumOfBadBattPacks
```

Due to the nature of how Datadog metrics work (counters, gauges, etc.) input can only be numerical. So metrics like `upsBasicStateOutputState` which translate to strings, aren't of much use here. You will also notice that `upsAdvBatteryRunTimeRemaining` includes the `forced_type: gauge` flag. That is because in the MIB that object is defined as a TimeTick type and isn't automatically converted to an integer. By forcing it to a gauge, the SNMP integration converts the object into time in milliseconds. 

If you notice any of your configured intgegrations missing from the Datadog interface, check `/var/log/datadog/collector.log`. It should provide some hints to why.

If all is configured correctly, you should be able to restart the agent and have metrics show up in the interface. `dd-agent info` should also report that it is recording SNMP information. 

You can now configure alerts and dashboards based on this information in the Datadog web interface. If you haven't had the opportunity to play with Datadog, it is free for students as a part of the [GitHub Student Pack](https://education.github.com/pack) or they have a free trial period as well. 
