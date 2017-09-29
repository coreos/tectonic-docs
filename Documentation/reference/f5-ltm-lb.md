# Using F5 BIG-IP LTM with Tectonic

Installing Tectonic on bare metal or VMWare requires a master DNS record that resolves to any master node(s) in the cluster, and a worker DNS record which resolves to any worker nodes for Ingress. Evaluation clusters may be set up using a DNS entry that points to each node, but production clusters require the use of a load balancer.

When installing on cloud providers such as AWS, Tectonic automatically creates a load balancer (such as an [ELB on AWS][aws-elb]). When installing Tectonic on bare metal or VMWare, you must provide your own load balancer.

This document describes setting up F5 BIG-IP LTM for use as a load balancer with Tectonic.

## Overview

### Prerequisites

* F5 BIG-IP LTM 10.x or higher
    * In the same datacenter as the Tectonic cluster
* Tectonic installer
    * VMware or bare metal environment
* Pre-allocated IP addresses for cluster and pre-created DNS records

## DNS and IP address allocation

Tectonic installation requires static allocation of IP Addresses for bare metal or VMWare. Create these required DNS records before launching Tectonic Installer.

The following table lists an example Tectonic node, Kubernetes API node, and two master and two worker nodes.

| Record | Type | Value |
|------|-------------|:-----:|
|tectonic.example.net | A | 192.168.1.91 |
|api.example.net | A | 192.168.1.92 |
|master-01.example.net | A | 192.168.1.110 |
|master-02.example.net | A | 192.168.1.111 |
|worker-01.example.net | A | 192.168.1.112 |
|worker-02.example.net | A | 192.168.1.113 |

In this example the virtual IP for console traffic will be 192.168.1.91, DNS resolvable at tectonic.example.net and the virtual IP for API traffic will be 192.168.92, DNS resolvable at api.example.net

## Create F5 LTM Pools

First, use the F5 web console to create two LTM pools for directing traffic to Tectonic console and API server.

Create an F5 LTM Pool for the API Server:
1. Log in to F5 BIG-IP LTM web console as an admin user.
2. From the *Main* tab, select *Local Traffic > Pools*, and click *Create*.
3. Enter *Name: tectonic_api_443*.
4. Under *New Members* add the IP address of all MASTER nodes (for example: `192.168.1.110` and `192.168.1.111`) and set *Service Port* to *443 / HTTPS*.
5. Click *Finish* (leaving the rest of the settings at their default).

Create an F5 LTM Pool for Tectonic console:
1. From *Local Traffic > Pools*, click *Create*.
2. Enter *Name: tectonic_console_443*.
3. Under *New Members* add the IP address of all WORKER nodes (for example: `192.168.1.112` and `192.168.1.113`) and set *Service Port* to *443 / HTTPS*.
4. Click *Finish* (leaving the rest of the settings at their default).

## Create F5 Virtual Servers

Next, create two Virtual Servers which use the Pools just created.

Create an F5 LTM Virtual Service for the API Server:
1. From the *Main* tab, select *Local Traffic > Virtual Servers*, and click *Create*.
2. Under *General Properties*, enter *Name: VS-Tectonic-API*.
3. From *Destination* select *Type: Host* and enter an address that matches the DNS entry for the API (for example: `api.example.net / 192.168.1.92`).
4. Select *State: Enabled*.
5. From *Configuration*, select *Type: Performance (Layer 4)*, *Protocol: TCP*, and *SNAT Pool: Auto Map*.
6. Under *Resources*, select *Default Pool: select tectonic_api_443*.
7. Click *Finish* (leaving the rest of the settings at their default).

Create an F5 LTM Virtual Service for Tectonic Console:
1. Click *Create*.
2. Under *General Properties*, enter *Name: VS-Tectonic-Console*.
3. From *Destination* select *Type: Host* and enter an address that matches the DNS entry for Tectonic (for example: `tectonic.example.net / 192.168.1.91`).
4. Select *State: Enabled*.
5. From *Configuration*, select *Type: Performance (Layer 4)*,  *Protocol: TCP*, and *SNAT Pool: Auto Map*.
6. Under *Resources*, select *Default Pool: select tectonic_console_443*.
7. Click *Finish* (leaving the rest of the settings at their default).

## Creating Virtual Service and Pools via CLI

You may also create the virtual service and pools using the command line, rather than the BIG-IP web console. SSH into the F5 LTM as administrator and run the following commands:

```
tmsh create ltm pool tectonic_api_443 members add { 192.168.1.110:https { } 192.168.1.111:https { } }
tmsh create ltm virtual VS-Tectonic-API snat automap pool tectonic_api_443 destination 192.168.1.92:https ip-protocol tcp profiles add { fastL4 }

tmsh create ltm pool tectonic_console_443 members add { 192.168.1.112:https { } 192.168.1.113:https { } }
tmsh create ltm virtual VS-Tectonic-Console snat automap pool tectonic_console_443 destination 192.168.1.91:https ip-protocol tcp profiles add { fastL4 }
tmsh save /sys config
```

Note: The IP addresses used in the CLI example correspond to the IP allocation example above.

## Install Tectonic

Once configured, install Tectonic using the Master DNS and Tectonic DNS defined here to direct traffic through the F5 load balancer.


[aws-elb]: https://aws.amazon.com/elasticloadbalancing/
