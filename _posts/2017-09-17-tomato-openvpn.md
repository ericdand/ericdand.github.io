---
layout: post
title: Tomato OpenVPN Configuration Notes
---

Here can be found some rough notes from when I was setting up OpenVPN on my
Tomato router. Mostly written in March 2017, these notes are expanded on
occasionally and can now be found on my website. They may or may not be useful
or even make sense.

# Demystifying the GUI

What does "Redirect Internet traffic" do? My internet traffic is all routed
through the VPN even when it isn't ticked -- is that how it should be?
What about "Redirect through VPN" in the "Routing Policy" tab?

Each option changes a setting in NVRAM. A list of what changes what:

	Advanced>Redirect Internet traffic=vpn_client1_rgw
	Routing Policy>Redirect through VPN=vpn_client1_route

There is also a key, `vpn_clien1_gw` referenced in the file. Inspecting the
HTML of the `vpn-client.asp` page, we find the following tucked into the same
fieldset as "Redirect Internet traffic": 

{% highlight html %}
<span id="client1_gateway" style="display: none"> 
  Gateway:&nbsp;
  <input type="text" name="vpn_client1_gw" value="" maxlength="15" size="17" onchange="verifyFields(this, 1)" id="_vpn_client1_gw">
  <span class="help-block"></span>
</span>
{% endhighlight %}

Deleting the `display:none` to make it appear, it shows up right next to the
checkbox. At first I thought that this field not showing up when "Redirect
Internet Gateway" is ticked was a bug in the Tomato GUI, but it turns out that
it's not: it shows up if you set the interface to "TAP" rather than "TUN".

Furthermore, in `release/src/router/rc/vpn.c:231` we can see this:

{% highlight c %}
sprintf(&buffer[0], "vpn_client%d_rgw", clientNum);
sprintf(&buffer2[0], "vpn_client%d_nopull", clientNum);
if ( nvram_get_int(&buffer[0]) )
{
	sprintf(&buffer[0], "vpn_client%d_gw", clientNum);
	if ( ifType == TAP && nvram_safe_get(&buffer[0])[0] != '\0' )
		fprintf(fp, "route-gateway %s\n", nvram_safe_get(&buffer[0]));
	fprintf(fp, "redirect-gateway def1\n");
} else if ( nvram_get_int(&buffer2[0]) > 0 )
{
	fprintf(fp, "route-nopull\n");
}
{% endhighlight %}

So it looks like if `vpn_client1_rgw` is set, it then checks that the interface
is TAP and that `vpn_client1_gw` is nonempty. It then prints to the file it's
generating, `/etc/openvpn/client1/config.ovpn`. The config options it writes if
`vpn_client1_rgw` is set are:

	route-gateway <gateway>	# if the interface is TAP and the gateway is nonempty
	redirect-gateway def1	# if the interface is not TAP or the gateway is empty

According to the OpenVPN manual, `redirect-gateway def1` redirects all traffic
over the VPN, rather than redirecting only traffic bound for VPN-internal IP
addresses. I assume OpenVPN does this by calling out to iptables, since such
rules exist in my iptables rule set. Curiously, I do not have the "Redirect
Internet traffic" option set, and yet my VPN still redirects all traffic. This
is because my VPN provider's server pushes `redirect-gateway def1` to all
clients. You can ignore this using `--route-nopull`, which rejects all redirect
and DNS related pushes from the server.

# "Create NAT on tunnel" and Firewall

The VPN GUI "firewall" has two settings: Automatic, and Custom. Automatic
configures the router to add the following firewall rules when the VPN is
activated:

	iptables -I INPUT -i tun11 -j ACCEPT
	iptables -I FORWARD -i tun11 -j ACCEPT

It deletes the rules when the VPN is deactivated. If it is set to "Custom",
then it adds no iptables rules.

If "firewall" is set to auto, another option is visible on the UI: "Create NAT
on tunnel". The "Create NAT on tunnel" checkbox (at the bottom of the Basic
tab) creates the following rules for each LAN:

	iptables -t nat -I POSTROUTING -s <LAN IP range with netmask> -o tun11 -j MASQUERADE

This can be found in `vpn.c`, starting at line 394.

Note that masquerading using iptables does not control the actual routing of
traffic; that's done using "ip route," possibly by the OpenVPN client itself.


The Routing Policy tab
----------------------

Shibby added a "Routing Policy" tab, which does what I want without messing
with your own scripts (in theory), but doesn't work as expected on my router. 

When enabled, "Routing Policy" runs a script called `vpnrouting` which marks
all packets which match patterns in the "Routing Policy" tab with the mark
"311". It then creates an `ip route` table, table 311, and a rule that tells
packets marked "311" to use table 311, which in turn tells them to use tun11,
the VPN tunnel interface. Unfortunately, when it goes to add rules to `iptables
-t mangle`, iptables complains and says "No chain/target/match by that name."
It's the MARK target (`-j MARK --set-mark 311`) that gives it problems. 

It seems that on the ASUS RT-AC66U (my router), the kernel module that allows
marking, `xt_MARK`, is absent. I would have to recompile the kernel to get it. 

In the meantime, I added the following rule to get what I want done:

	ip rule add from 192.168.1.64/26 table 311

This takes advantage of the already-built table 311 from Shibby's vpnrouting
script.

TODO: If you deactivate the VPN and table 311 disappears, does internet fall
gracefully back to the regular route? Or does it complain that 311 doesn't
exist, and bail?

