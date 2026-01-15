+++
title = "Host Detector"
date = "2023-04-18T23:47:25"
slug = "host-detector"
draft = false
+++

There are probably already way better ways of doing host detection on your network but I was always thinking of creating one on my own.

{{< github repo="phantasmfour/hostDetector" showThumbnail=false >}}

I run my own local DNS server which I liked to keep filed with the VMs and devices on my network. I don't like however running into situations where I don't know what a host is on my network so I wanted something to remind me to put in DNS records for new hosts.

What Does Your Script Do?
-------------------------

* Scans a hardcoded list of networks
* Resolves DNS on all of them
* Get the MAC Address
* Check if it is in a whitelist of no alert MACs
* Check if we already found these exact hosts
* Send the results to Discord


How does it do all this?
------------------------

**Scan a hardcoded list of networks**  
I am using the python-nmap module. I originally was going to just write this script in bash but found the python-nmap module easier to work with. I run an `nmap -sn` on each subnet. You are able to run this without needing root as long as you allow http on your network. You can run it with ICMP but you do need root.

**Resolve DNS on all of them**  
dns.resolver python library is really cool. It is the new python 3.9 way of doing DNS resolutions. Originally I was crafting reverse DNS queries myself by reversing the octets and adding a PTR to the end. However they gave me deprecation warnings about this and I found you can do it in one line  
`resolver.resolve(dns.reversename.from_address(host), 'PTR')`

**Get the MAC Address**  
getmac is a python library that again does not need you to run as root. I only query MAC addresses when the host does not have a dns record to speed this up. I am pretty sure this is just using ARP since there is really no other way and I remember seeing it could not figure out the MAC of my host considering I don't have an ARP cache for myself

**Check if it is in a whitelist of no alert MACs**  
I have some hosts on IOT networks which I don't care about in DNS. Or maybe temporary hosts that only need access for a few day. No reason to have these in DNS in my opinon but I do hate alerts about unecessary things. So I created a txt file that I just read in and check if the MAC was already whitelisted.

**Check if we already found these exact hosts**  
In order to not abuse Discord webhook I want to make sure I am not alerting for things I already know about. If 10.0.0.1 does not have a DNS and I already alerted on it today I probably don't care. So I again have another local txt file that I check if anything is in there. If there is nothing I clobber it with what I have currently unfound. If there is something in there check if it is an exact match and if its not clobber it ad send to Discord. I rotate(just by time and cron) this file everyday at 12AM so I will get new alerts at least once a day for outstanding items. Keeps the noise down.

**Send the results to Discord**  
Discord we hooks are very easy to integrate with Python. Its like three lines

`from discord import Webhook, RequestsWebhookAdapter`  

`webhook=Webhook.from_url("https://discord.com/api/webhooks",adapter=RequestsWebhookAdapter())`  

`webhook.send(f"New Hosts on the network with no DNS Record: {unfoundList}")`


Things I could do better
------------------------

Fill rotation and probably use a better file to store recently sent items along with a whitelist of MACs.  
I think it is good enough though and did not want to spend too much time on it.

