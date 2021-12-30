# Freedom-IPTV-with-OPNsense
[i]How I configured my OPNsense firewall to enable Canal Digitaal IPTV via Freedom.nl internet (Netherlands)[/i]

First off, I am not a networking professional. Not do I have tons of experience in networking, firewalls or IPTV. I merely saw this as a challenge and since I did not find a lot of Canal Digitaal and Freedom.nl specific information on the internet, I decided to share this info.
If you see any mistakes (particularly ones that may leave my firewall or network vulnerable to attacks!) or potential improvements, please let me know :-)

My network setup looks like this:

[INTERNET] <--> (WAN port; VLANs 4,6)=[OPNsense FW]=(LAN port; VLANs 10,40) <--> (port x; VLANs 10,40)=[Switch]=(port y; VLAN 40) <--> [IP STB] 

Instead of a provider modem, my internet (FttH) connects directly into my OPNsense firewall. VLAN 4 on the physical WAN port is the WANIPTV interface. VLAN 6 is used for regular internet traffic. The OPNsense LAN port has many VLANs for multiple purposes. I'll only mention VLAN 40 here, which is configured as the InternalIPTV interface.
There's a Unify switch, where the IPTV Settop Box is connected to port y. That port is configured in VLAN 40. IGMP Snooping is enabled on the switch for both ports.

