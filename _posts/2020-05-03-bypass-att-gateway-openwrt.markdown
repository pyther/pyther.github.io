---
layout: single
title:  "Bypass AT&T Fiber Gateway with OpenWrt"
excerpt_separator: <!--more-->
---

AT&T requires the use of their Residential Gateway to access their network.  If
you've attempted to connect your own router directly to the ONT
(Fiber-to-Ethernet adapter), you likely didn't get very far.

Why? Perhaps you want full control of your network. Maybe you are already using
your own router and want to remove an unnecessary extra hop. Whatever your
reason, this can be achieved with OpenWrt and an EAP proxy.

In order to route traffic directly to the ONT
1. WAN Traffic must be tagged with VLAN 0
2. WAN Mac Address must match the MAC Address of the AT&T Gateway
3. IPv6 DUID must be set and match the DUID used by the AT&T Gateway
4. 802.1X Authentication is required


The first three steps are easy, all can be configured with OpenWrt. The last
step, 802.1X Authentication, is where things get a little tricky.

Ideally, we would extract the 802.1X user certificates from the AT&T Router and
let OpenWrt handle the authentication. Extracting certificates from the AT&T
router is not currently possible. There was a root exploit a while back that
allowed some folks to extract certificates, but this has since been patched. 

This can be worked around, by proxying EAP packets between the AT&T Router and
the ONT interfaces, allowing EAP Authentication to take place.


## Configuration
Take a moment and identify the **Mac Address** and **Serial Number** of the AT&T
Gateway/Router. This information can be found on the back of the AT&T Router.

Note: These instructions are specific to OpenWrt but should be adaptable to any Linux based OS.

### Physical Connections
For this setup, you will need a total of 3 NICs. If you lack a third interface,
consider using a USB to Ethernet adapter for the ATT Router connection as
traffic is minimal.

Router Ports:
- eth0 - LAN
- eth1 - WAN, connect to ONT (Fiber to Ethernet Adapter)
- eth2 - ATT, connect to RED broadband port on AT&T Router


### Network Interfaces

#### WAN
Incoming Traffic uses VLAN 0 Priority Tagging. VLAN 0 Priority Tagging is a
method that allow the 802.1P Priority bits to be set for untagged traffic.
Analyzing a few packet captures, the priority bit appears to always be set to
0, Best Effort.

Interestingly, EAP Traffic, is not 802.1Q tagged.

Important Notes
- traffic must be tagged with vlan ZERO (ifname ethX.0)
- macaddr must be set to the mac address on your AT&T Router
- to use DNS servers provided by ATT set `option perdns '1'`

{% highlight sh %}
config interface 'wan'
	option proto 'dhcp'
	option macaddr '88:71:B1:XX:XX:XX' # MAC Address of AT&T Router
	option peerdns '0'
	option ifname 'eth1.0' # vlan 0
	list dns '1.1.1.1' # cloudflare nameserver
	list dns '1.0.0.1' # cloudflare nameserver
{% endhighlight %}

#### ATT
Ensure eth2 is brought up on startup. Needed for EAP Proxy, below.
{% highlight sh %}
config interface 'ATT'
	option proto 'none'
	option ifname 'eth2'
	option auto '1'
	option delegate '0'
	option force_link '1'
{% endhighlight %}

#### LAN
No special configurations need

### EAP Proxy
Since we don't have access to the 802.1X user certificates, we need to proxy
EAP packets between the AT&T Router and the ONT, so the AT&T Router can
authenticate on our behalf.

[pyther/goeap_proxy](goeap_proxy) is one of many EAP Proxy tools that can help
us. The proxy listen on both interfaces for EtherType 0x888E (EAP over LAN)
frames and forwards EAP packets between interfaces.

#### Build
1. Go to [pyther/openwrt-feed](openwrt-feed)
2. Follow instruction in README to build a package for your platform
3. Copy package to your device

#### Setup
1. Install Package: `opkg install /tmp/goeap_proxy_0.200502.3-1_x86_64.ipk`
2. Modify the config file

        root@OpenWrt:~# cat /etc/config/goeap_proxy
        config goeap_proxy 'proxy'
	            option wan 'eth1'
	            option router 'eth2'
3. Start goeap_proxy

        $ /etc/inti.d/goeap_proxy start

4. Check Logs

        $ logread | grep goeap_proxy
        Sat May  2 12:47:30 2020 kern.info goeap_proxy[22848]: eth2: 88:71:b1:a1:b1:c1 > 01:80:c2:00:00:03, EAPOLStart v2, len 0 > eth1
        Sat May  2 12:47:30 2020 kern.info goeap_proxy[22848]: eth1: 00:90:d0:63:ff:01 > 01:80:c2:00:00:03, EAP v1, len 4, Failure (4) id 125 > eth2
        Sat May  2 12:47:30 2020 kern.info goeap_proxy[22848]: eth1: 00:90:d0:63:ff:01 > 01:80:c2:00:00:03, EAP v1, len 15, Request (1) id 126 > eth2
        Sat May  2 12:47:30 2020 kern.info goeap_proxy[22848]: eth1: 00:90:d0:63:ff:01 > 88:71:b1:a1:b1:c1, EAP v1, len 15, Request (1) id 126 > eth2
        Sat May  2 12:47:31 2020 kern.info goeap_proxy[22848]: eth2: 88:71:b1:a1:b1:c1 > 01:80:c2:00:00:03, EAP v2, len 22, Response (2) id 126 > eth1
        ...

Authentication should be complete! Outgoing traffic should now be successful.


### IPv6
To enable IPv6, you first must identify your DHCP unique identifier (DUID).

At the time of this writing, all AT&T Routers appear to share the same DUID
prefix. However, this may change and the best way to identify your DUID is to
sniff DHCPv6 traffic between the AT&T Router and ONT.

Standard DUID: `00:02:00:00:0d:e9:30:30:31:45:34:36:2d:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx`

Replace `xx` with the ASCII values (in hex) of the router's serial number.

Python Code to convert Serial Numbers to Hex
{% highlight python %}
>>> ':'.join('%02x' % ord(c) for c in '12345A123456')
'31:32:33:34:35:41:31:32:33:34:35:36'
{% endhighlight %}

*WARNING: The web interface will not accept the DUID/Clientid as being valid. This must be configured in the config file!*

Interface Configuration
{% highlight sh %}
config interface 'wan6'
	option proto 'dhcpv6'
	option reqaddress 'try'
	option reqprefix 'auto'
	option peerdns '0'
	option clientid '00:02:00:00:0d:e9:30:30:31:45:34:36:2d:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx'
	option ifname 'eth1.0' # tagged vlan0
	list dns '2606:4700:4700::1111' #cloudflare dns
	list dns '2606:4700:4700::1001' #cloudflare dns
{% endhighlight %}

Assign part of the IPv6 assignment to your LAN Intreface
{% highlight sh %}
config interface 'lan'
    ...
    option ip6assign '64'
    option ip6hint '1'
{% endhighlight %}

As AT&T may change your IPv6 Address assignment, you may want to consider
setting a Unique Local Address (ULA) that would be used internally on your
network.
{% highlight sh %}
config globals 'globals'
	option ula_prefix 'fd10:96f2:a1b1::/48'
{% endhighlight %}



[openwrt-feed]: https://github.com/pyther/openwrt-feed
[goeap-proxy]: https://github.com/pyther/goeap_proxy/
