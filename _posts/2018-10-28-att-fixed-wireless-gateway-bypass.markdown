---
layout: post
title:  "ATT Fixed Wireless Gateway Bypass"
excerpt_separator: <!--more-->
---

[ATT Fixed Wireless Internet][att-fwi] service consists of an Outdoor Antenna
with a built-in modem and an ATT Residential Gateway. A Power-over-Ethernet
injector is used to power the Antenna/Modem. 

A handful of ATT Fiber customers figured out how to bypass the residential
gateway. Since the same gateways are used for Fixed Wireless Internet, I wonder
if it was possible to do the same.

After some packet captures and testing, I determined the following:

1. Nearly all traffic was tagged for vlan 4001 or vlan 4002
2. The MAC Address of the Linux router needs to the same as the Residential Gateway (or the modem won't respond)
3. DHCP on VLAN 4001 gave a 192.168.11.x address
4. DHCP on VLAN 4002 gave an "Internet" address of 10.x.x.x address (Carrier Grade NAT)

*Note: For ATT Fiber, 802.1x Authentication is required between the residential
gateway and the modem (ONT). 802.1x Authentication is not used for Fixed
Wireless Internet. Be aware of this, if reading other guides.*
<!--more-->

## Traffic Capture

*Optional: I only have a sample size of one device, however I imagine all Fixed
Wireless Modems/Antennas are configured the same. If the mac address is printed
on your residential gateway, you can probably skip this step.*

We need to analyze the traffic between the residential gateway and the
modem/antenna to determine which vlans are being used. We can do this by
creating a bridge and using tcpdump to look at the traffic flowing between the
gateway and modem/antenna.

Make the following connections:
  - Residential Gateway Modem port -> Linux Router (enp1s0)
  - Modem/Antenna -> Linux Router (enp2s0)

Bring up interfaces:
{% highlight sh %}
$ ip route link set up dev enp1s0
$ ip route link set up dev enp2s0
{% endhighlight %}

Create the bridge:
{% highlight sh %}
$ brctl addbr br-att
$ brctl addif br-att enp1s0
$ brctl addif br-att enp2s0
{% endhighlight %}

Analyze the traffic:
{% highlight sh %}
$ tcpdump -vvv -ei br-att
{% endhighlight %}

From the dump, you should be able to identify the **vlans** being used the
**mac address** of the residential gateway. You may find the mac address
printed on the side of the residental gateway. 

Tear down the bridge:
{% highlight sh %}
$ brctl del br-att enp1s0
$ brctl del br-att enp2s0
$ brctl delbr br-att
{% endhighlight %}

Disconnect the residental gateway from the Linux router.

## Configuration

The following configuration uses use the `ip` command. Although this example is
Linux specific, the concepts should work on any platform.

The modem/antenna should be directly connected to your Linux router. The
residential gateway should be disconnected and powered off.

Bring up the WAN interface:
{% highlight sh %}
$ ip route link set up dev enp2s0
{% endhighlight %}

Create a tagged interface:
{% highlight sh %}
$ ip link add link enp2s0 name enp2s0.4002 type vlan id 4002
{% endhighlight %}

Spoof the mac address on the tagged interface:
{% highlight sh %}
$ macchanger --mac=XX:XX:XX:XX:XX:XX enp2s0.4002
{% endhighlight %}

Replace `XX:XX:XX:XX:XX:XX` with the mac address of the residental gateway as identified above.

At this point, the connection should be useable and we can attempt to obtain an IPv4 address.
{% highlight sh %}
$ dhclient -4 -d enp2s0.4002
{% endhighlight %}

If you get a lease for `192.168.11.x`, tear down the tagged interface, and
repeat the steps for any other identified vlans.

If you fail to get a lease, verify the mac address on the tagged interface
(enp2s0.4002). The modem/antenna will not respond to traffic from any other mac
address.

## IPv6 Configuration

In my testing, the antenna/modem won't respond to DHCPv6 requests. The same was
true of the residential gateway. This means requests prefix delegations is out
of the question and we are left using the Router Advertisement (RA) for IPv6
configuration.

If you configure your Linux router to accept router advertisements, you'll
notice that an IPv6 address won't get configured, but a default route will.

Router Advertisement
{% highlight text %}
interface enp2s0.4002
{
	AdvSendAdvert on;
	# Note: {Min,Max}RtrAdvInterval cannot be obtained with radvdump
	AdvManagedFlag off;
	AdvOtherConfigFlag off;
	AdvReachableTime 0;
	AdvRetransTimer 0;
	AdvCurHopLimit 64;
	AdvDefaultLifetime 1800;
	AdvHomeAgentFlag off;
	AdvDefaultPreference medium;
	AdvLinkMTU 1430;
	AdvSourceLLAddress on;

	prefix 2600:380:a826:8893::/64
	{
		AdvValidLifetime 86400;
		AdvPreferredLifetime 14400;
		AdvOnLink off;
		AdvAutonomous off;
		AdvRouterAddr off;
	}; # End of prefix definition


	route 2600:380:a826:8893:35f8:c0d6:8bcb:71e4/128
	{
		AdvRoutePreference medium;
		AdvRouteLifetime 86400;
	}; # End of route definition


	RDNSS 2600:300:20b0:341a::13 2600:300:20b0:341a::14
	{
		AdvRDNSSLifetime 600;
	}; # End of RDNSS definition

};
{% endhighlight %}

As you can see in the prefix section, `AdvAutonomous` is set to `off`. This
means the address can not be used for autonomous address configuration.

When the modem/antenna is powered cycle, a new IPv6 prefix is issued. As I want
my internal networks to have a consistent IPv6 address, I've decided to assign
Unique Link-Local Addresses (ULA) to my internal networks. Using the NETMAP
extension, I can map my ULAs to the IPv6 prefix assigned by ATT.

{% highlight sh %}
$ ip6tables -t nat -A PREROUTING -d 2600:380:7e1b:d87e::/64 -i enp2s0.4002 -j NETMAP --to fd4a:d2f6:a2d6::/48
$ ip6tables -t nat -A POSTROUTING -s fd4a:d2f6:a2d6::/48 -o enp2s0.4002 -j NETMAP --to 2600:380:7e1b:d87e::/64
{% endhighlight %}

Note: The prerouting rule is only included for completeness. It never gets hit,
because ATT NATs all outgoing connections and does not allow new incoming
connections.

I wrote a small bash script that will request and wait for a Router Advertisement
using the ndp binary ([mdlayher/ndp][ndp-go]) and update ip6tables accordingly.
I run the script every 5 minutes via cron. Not the best solution, but works
reasonably well. Ideally it'd be nice to have daemon that listens for RAs and
takes action when there is a change. Here's the script:

{% highlight sh %}
# Send a router solicitation to get network address
# ATT Modem is stupid and sets AdvAutonomous off
# which prevets networkd and friends from assinging an ip address to the interface

INT=enp2s0.4002

if ! networkctl status $INT | grep -q 'State: routable'; then
    echo "Interface is not in a routable state."
    exit 1
fi

addr=$(timeout 10 /opt/bin/ndp -i $INT -a linklocal rs 2>&1 | grep 'prefix information:' | sed 's/[^0-9]*//' | cut -d',' -f1)
rc=$?

if (( rc )); then
    echo "ndp command exit status was ${rc}."
    exit 1
fi

if [[ -z $addr ]]; then
    echo "unable to parse address from ndp command output"
    exit 1
fi

if ! ip6tables-save -t nat | grep "\-A POSTROUTING" | grep -q "$addr"; then
    ip6tables -t nat -F POSTROUTING || exit 1
    ip6tables -t nat -F PREROUTING || exit 1 
    ip6tables -t nat -A PREROUTING -i $INT -j NETMAP -d $addr --to fd4a:d2f6:a2d6::/48 || exit 1
    ip6tables -t nat -A POSTROUTING -o $INT -j NETMAP --to $addr -s fd4a:d2f6:a2d6::/48 || exit 1
    echo "IPv6 NAT rules updated: ${addr}"
fi
{% endhighlight %}

[att-fwi]: https://www.att.com/internet/fixed-wireless.html 
[ndp-go]: https://github.com/mdlayher/ndp
