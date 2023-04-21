===========
Tayga NAT64
===========

------------
Introduction
------------
IPv6-only networks are less complex to plan, configure, maintain and troubleshoot than dual-stack networks. But many services on the Internet
are still IPv4-only. NAT64 preserves access to these services by performing IPv6-to-IPv4 translation. The NAT64 implementation currently
available for OPNsense is the Tayga plugin. Tayga behaves like an external device connected to OPNsense via a point-to-point interface.
It is therefore a Hop in the Path. And requires additional internally used IP addresses.

.. Note::
   This how-to focuses on providing IPv6-only LANs with access to IPv4-only services. However, this is not the only use case for NAT64.
   See :doc:`/manual/how-tos/tayga_generic.rst` for a more generic how-to.

.. Note::
   You may also use any other DNS64 capable DNS server (see DNS64 section). If you use the well-known prefix any Public DNS64 will work, too.

-------------
Prerequisites
-------------
OPNsense must be configured with working dual-stack Internet access on WAN. And you may want to have a IPv6 enabled VPN or LAN.

.. Note::
   You don't need to run an IPv6-only network to use NAT64, you can have a dual-stacked network with NAT64 to do a slow transition
   or to centrally handle NAT64 if want to simplify IPv6-only deployments further down your network topology.
   Generally speaking a network that offers dual-stack and NAT64 has the highest compatibility (IPv4-only devices, devices without an
   integrated CLAT and hardcoded IPv4, dual-stacked devices, IPv6-only devices). This configuration is often used for (private) clouds,
   as it allows to choose within each project to be IPv4-only, IPv6-only, or dual-stack.

----------
Installing
----------
Go to :menuselection:`System --> Firmware --> Plugins` and install the `os-tayga` plugin.

-------------
Configuration
-------------
Then go to :menuselection:`Services --> Tayga`.

:Enable:
   Tick this configuration to enable Tayga.

:IPv4 Address:
   Will show up in traceroutes from the IPv4 side to the IPv6 side. Can be left to its default value unless you changed the `IPv4 Pool`.
   Should be located in the `IPv4 Pool` subnet.

:IPv4 NAT64 Interface Address:
   Can be left to its default value unless this conflicts with your network. Must not be located in the `IPv4 Pool` subnet and must not be
   used by another interface or VIP.

:IPv6 Address:
   Will show up in traceroutes from the IPv6 side to the IPv4 side. Can be left empty if the `IPv6 Prefix` is a GUA or the `IPv4 Address` is
   a non-RFC1918 address. Tayga will then auto-generate its IPv6 address by mapping the `IPv4 Address` into the `IPv6 Prefix`.
   For example, if the `IPv6 Prefix` 2001:db8:64:64::/96 and `IPv4 Address` 192.168.255.1 are being used, Tayga's IPv6 address will be
   2001:db8:64:64::192.168.255.1 (2001:db8:64:64::c0a8:ff01).

   .. Warning::
      Tayga can't auto-generate its `IPv6 Address` if the default well-known `IPv6 Prefix` 64:ff9b::/96 and a private (RFC1918) `IPv4 Address`
      are being used. In this case, you have to manually specify an unused address from your site's GUA or ULA prefix.

:IPv6 NAT64 Interface Address:
   Must not be located in the `IPv6 Prefix` subnet and must not be used by another interface or VIP.

   .. Warning::
      The default value must not be used since 2001:db8::/32 is a documentation-only prefix.

:IPv6 Prefix:
   The IPv6 prefix which Tayga uses to translate IPv4 addresses. Use the default well-known prefix 64:ff9b::/96.

   .. Note::
      When using the well-known prefix 64:ff9b::/96, Tayga will prohibit IPv6 hosts from contacting IPv4 hosts that have private (RFC1918)
      addresses. This is not relevant when using NAT64 for accessing IPv4 services on the Internet. However, if access to local services with
      private IPv4 addresses is required, a GUA /96 prefix must be used.
      While technically possible, using a ULA prefix for NAT64 is not recommended. This can cause issues with certain hosts, especially those
      which support 464XLAT.
      However if your main goal is to use the well-known prefix but with support for RFC1918 addresses, use fc00:64:ff9b::/96 internally
      **and** create an additional NPTv6 rule, so that clients will use the well-known prefix where as Tayga will think they don't.
      This is a simple workaround for the Tayga limitation that prevents private IPv4 addresses with the well-known prefix.

:IPv4 Pool:
   The virtual IPv4 addresses which Tayga maps to LAN IPv6 addresses. Can be left to its default value unless this overlaps with existing
   subnets in your network. Must be sufficiently large to fit all devices in your IPv6-only LAN(s).

:Custom IPv6 Routing:
   This is an advanced setting for selective routing scenarios. It will prevent installing the route which routes the IPv6 Prefix to Tayga.
   This requires assigning and locking the nat64 interface, enabling dynamic gateway policy, configuring a dynamic IPv6 gateway and adding
   custom routes.

--------------
Firewall rules
--------------
Tayga uses a tunnel interface for packet exchange with the system. Rules are required to prevent the firewall from blocking these packets.
Additionally, an outbound NAT rule may be required if you don't have routed IPv4 address space (most cases).

Go to :menuselection:`Firewall --> Rules --> Tayga`, add a new rule, set the `TCP/IP Version` to `IPv4+IPv6`, leave all other settings to
their default values and save.

.. Note::
   If you just enabled Tayga and can't find :menuselection:`Firewall --> Rules --> Tayga`, go to :menuselection:`Interfaces --> Assignments`,
   click `Save` and reload the page.

Go to :menuselection:`Firewall --> Settings --> Normalization`, add a new rule, set the `Interface` to `Tayga`, leave all other settings to
their default values and save.

.. Note::
   This rule is required for proper handling of fragmented packets.

Go to :menuselection:`Firewall --> NAT --> Outbound`, add a new rule, set `Source address` to `Single host or network`, enter your Tayga
`IPv4 Pool`, leave all other settings to their default values and save.

.. Note::
   This rule is required to hide the Tayga IPv4 Pool behind the OPNsenses own IP. You kan skip this rule if it is properly routed.
   If you have neither return packages won't be received and your connections fail.

Don't forget to apply the firewall changes to make NAT64 fully operational.

-----
DNS64
-----
In most scenarios, you also want to configure DNS64 in addition to NAT64. If you use OPNsense's :doc:`/manual/unbound` DNS resolver,
DNS64 can be enabled by going to :menuselection:`Services --> Unbound DNS --> General` and ticking `Enable DNS64 Support`.
If you don't use the default 64:ff9b::/96 prefix, you also have to enter your /96 prefix there.

.. Note::
   If you don't use the well-known prefix you should take care that it is still usable by clients via NATv6 and routes regardless.
   It is good practice to have the well-known prefix available in an IPv6 network for best compatibility.
   Generally speaking it is never necessary nor adviced to use another prefix for NAT64 for generic internet access.
   Custom prefixes are mainly used for usecases where you have your own IPv4 address space and want to use it for inbound connections
   without dualstacking your internal network. So it is a very uncommon configuration to use it for DNS64.

-------
Testing
-------
You can use a service like https://internet.nl/connection/ to verify that devices in your IPv6-only LAN have IPv6 and IP4 Internet access.
