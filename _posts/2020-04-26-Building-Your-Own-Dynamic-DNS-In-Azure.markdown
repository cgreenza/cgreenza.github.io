---
layout: post
title:  "Build Your Own Dynamic DNS Service In Azure"
date:   2020-04-26 00:00:00 +0200
tags: []
---

Most internet service providers assign dynamic IP addresses to home subscribers. This complicates things if you need to connect to your home router from the internet because the public IP address of your router will occasionally change. The solution is to create a DNS entry and update it whenever the IP address changes. Various dynamic DNS services are available for this.

You can implement your own dynamic DNS service in Azure using the following services:

* **Azure Functions** on the consumption plan. You are only billed when requests are being processed, charges should be minimal and easily within the monthly free grant.
* **Azure DNS Zone** for $0.50 per month plus a few cents for queries
* You'll also need a **domain name**


# Connecting to your home router
Use the domain name (e.g. home.example.com) when connecting to your router. Your device will use DNS to lookup the current IP address of your home router when connecting:
![Connectivity](/assets/dynamic-dns-1.png)


# Router public IP address changes
Your router or other device on your network (e.g. Raspberry Pi) will periodically use DNS to resolve the IP address for your domain name (e.g. home.example.com) and compare it to the router’s current public IP address. If different, it will issue a HTTP request to the Azure Function instructing the DNS record to be updated. This request is secured by a secret Function Key. The Azure Function App is assigned a *system managed identity* in Azure AD that has been granted *DNS Zone Contributor* access on the DNS Zone.
![Changed IP](/assets/dynamic-dns-2.png)


# Setup in Azure
Azure Function code and installation instructions can be found in [GitHub](https://github.com/cgreenza/AzureDynamicDns).

# Setup on your router
I have a [MikroTik RouterOS](https://mikrotik.com/) router and use the following Lua script (scheduled to run every 10 minutes) to detect IP changes and call the Azure Function:

```lua
:local ipv4addr
:set ipv4addr [/ip dhcp-client get [/ip dhcp-client find interface="ether1"] address]
:set ipv4addr [:pick [:tostr $ipv4addr] 0 [:find [:tostr $ipv4addr] "/"]]

:if ([:len $ipv4addr] = 0) do={
   :log error ("DDNS Could not get IP for interface")
   :error ("DDNS Could not get IP for interface")
}

:local resolvedIP [:resolve "home.example.com"];

:if ( $ipv4addr != $resolvedIP) do={
   :log info ("DDNS different $ipv4addr vs $resolvedIP, updating...")
   :local outputfile ("AZ_DDNS" . ".txt")
   /tool fetch mode=http url="https://your_function.azurewebsites.net/api/UpdateDns?code=your_function_key&host=home&ip=$ipv4addr" \
      dst-path=$outputfile

   :log info ( "DDNS update result " . ([/file get ($outputfile) contents]) )
   #/file remove ($outputfile)
} else={
   :log info ("DDNS no change $ipv4addr")
}
```

# Hairpin NAT
Your router needs to be setup for [NAT loopback](https://en.wikipedia.org/wiki/Network_address_translation#NAT_loopback) for the external domain name to work when you're on the home network.
