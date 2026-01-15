+++
title = "Power Saver"
date = "2023-04-22T01:41:46"
slug = "wol-servers"
draft = false
+++

I generally do not need my infrastructure running while I am sleeping. So why not turn it off.

{{< github repo="phantasmfour/powerSaver" showThumbnail=false >}}

-------

I run a two node Proxmox cluster that hosts all my VMs. From previous power outages I have managed to figure out the options to start my important infrastrucure at boot and then start all the others later. 

![](/images/image_powersaver.png)Once you have your VMs setup to come up in the correct order(and at all) you need to work on shutting them all down.

I leverages [Proxmoxer](https://proxmoxer.github.io/docs/2.0/) which is a python module for the Proxmox api. I probably did not use it to its full potential but I did not need to work with the requests module so its good enough.

The first part using the proxmox API is authenticating. With Proxmoxer you are able to give it an IP and an API Token. You can very easily make an API session with their recommended `prox = ProxmoxAPI('ip', user='@', token_name='<token_name>', token_value='<token_value>', verify_ssl=<True|False>, timeout=<timeout_in_seconds>)`

Making an API user in Proxmox is simple via the gui. I first made a user and then I made an API Token for that user. Save off the api token since you only see that once. Your Token name should always be in the gui.

I then had to give permissions to the user I created to actually be able to shutdown the VMs and the nodes. I ended up giving it these permissions

![](/images/image-1_powersaver3.png)These privileges apply on the user. I gave VM.Audit to be able to check the VMs to make sure they are up. I kept getting asked for Sys.Audit in the beginning since I was checking node status. Its not overly permissive and I don't want to try without it so its kept there. Privileges are explained [here](https://pve.proxmox.com/wiki/User_Management#pveum_permission_management).

I wrote a function that pulls all the vm's on the node, then executes the Proxmox API call to shutdown all nodes. I then check all the vms on the node to see if they are still up and wait until they all go down. This is important because I want to shutdown the node next but don't want to kill my VMs before they fully shutdown.

The next function I wrote shutdown the nodes. I had a bit of trouble finding an API command that worked on Proxmox 7 but this one ended up working without any 501 errors `prox(f"nodes/{node}/status").post(command="shutdown")`  


The next function I wrote was the WOL function and a ping function. The ping just checks to see if the host is up or down before I try sending a WOL. The WOL function sends a WOL if the host does not respond to ping and then waits 20 seconds and pings again. If the host is still not up I give it another 10 seconds and then run the loop again. Usually the host is up by the second time around.

I wrote an argparse that checks if you want to startup the nodes or turn them off. I then have the script running via cron on a Pi that I never shutoff and draws way less power.

The hardest part about this whole setup was that my Pi is on a different VLAN than the Proxmox Nodes. I have a fortigate firewall that acts as both a firewall/switch/router(Yes this is not the best). I found a fantastic [post](https://community.fortinet.com/t5/FortiGate/How-to-route-Wake-On-Lan-WOL-magic-packet-through-a-Forti) about how to route WOL through the fortigate.  
It did not all apply to me but it had the part I was missing, static ARP entries. When the nodes went down I was broadcasting traffic to a MAC/IP that the firewall did not have. I entered static ARP entries in the firewall for the two proxmox nodes and after that it worked like a charm. I am pretty loose with my policy between the nodes so WOL was already allowed. Doing captures on the firewall also help a lot to see if the WOL packet is moving correctly or even being sent.

Helpful Fortigate Commands
==========================


`diagnose sniffer packet any 'udp and port 9'`  

`get system arp`  

`config system arp-table`  

`edit 1`  

`set interface INTERFACE_NODE_LIVES_ON`  

`set ip IP.ADDRESS`  

`set mac ff:ff:ff:ff:ff:ff`  

`next`  

`end`


Sources:  
Proxmox API Docs: <https://pve.proxmox.com/pve-docs/api-viewer/index.html#/cluster/backup>  
Shutdown Command Post: <https://forum.proxmox.com/threads/ish-shutdown-all-nodes-via-api.121594/>  
Fortigate WOL Setup: <https://community.fortinet.com/t5/FortiGate/How-to-route-Wake-On-Lan-WOL-magic-packet-through-a-FortiGate-in/ta-p/198103?externalId=FD30104>

Next Steps
----------

I am probably going to do the same thing with my NAS but I will leverage SSH heavily since I don't think there is a public api available. But you can do this with a wide range of devices. It would probably be easier with a smart PDU to turn everything off and back on like switches and firewalls at night but I don't think I am ready to take this that far yet 

