# Freedom-IPTV-with-OPNsense
### How I configured my OPNsense firewall to enable Canal Digitaal IPTV via Freedom internet (Netherlands)
   

>First off, I am not a networking professional. Nor do I have tons of experience in networking, firewalls or IPTV. I merely saw this as a challenge and since I did not find a lot of Canal Digitaal and Freedom specific information on the internet, I decided to capture my findings here.  

>If you see any mistakes (particularly ones that may leave my firewall or network vulnerable to attacks!) or potential improvements, please let me know :-)

## Network setup
My network looks like this:

[INTERNET] <--> (WAN port; VLANs 4,6)=[OPNsense FW]=(LAN port; VLANs 10,40) <--> (port x; VLANs a,b,c,40)=[Switch]=(port y; VLAN 40) <--> [IP STB] 

Instead of a provider modem, my internet (FttH) connects directly into my OPNsense firewall. VLAN 4 on the physical WAN port is the **WANIPTV** interface. VLAN 6 is used for regular internet traffic. The OPNsense LAN port has many VLANs for multiple purposes. I'll only mention VLAN 40 here, which is configured as the **InternalIPTV** interface.
The IPTV Set-top Box is connected to the switch. That switch port is configured in VLAN 40. IGMP Snooping is enabled on the switch for both ports.

Freedom, the provider, gives some [high-level information for getting IPTV to work with your own equipment](https://www.freedom.nl/helpdesk/internet/algemene-instellingen#televisie) (in Dutch). I found this not particularly helpful, as it lacks enough detail to successfully get this to work. 

## Router to request DHCP address on VLAN 4

Easy enough: interface **WANIPTV** is configured on VLAN 4 and I've enabled DHCP for the interface. I added the following [advanced configuration for the DHCP client](/images/WANIPTV_DHCP_AdvancedConfig.png):

* Send Options: dhcp-class-identifier "IPTV_RG"
* Request Options: subnet-mask, routers, classless-routes
* Require Options: 121

## Static routes configured via VLAN 4 gateway IP

In OPNsense menu, System -> Routes -> Configuration is where I [added the required routes](/images/WANIPTV_StaticRoutes.png). The gateway IP assigned to my WANIPTV interface is 10.10.12.1. This gateway gets created automatically when the interface is configured via DHCP and in my case is called **WANIPTV_DHCP**.

One little adjustment that I made, is that instead of 10.10.0.33/32, I defined a static route to 10.10.0.0/24. I found that this particular route is to enable contacting the DHCP server, but the IP address can be served from different DHCP servers and not always 10.10.0.33. 

Apparently, the required static routes are delivered through DHCP option 121 also. But in my case, adding 121 as a required option did not do the trick.


## NATting traffic on VLAN 4

I've [configured outbound NAT](/images/WANIPTV_OutboundNAT.png) for any traffic from the STB out the WANIPTV interface. Basically just like you do for any traffic from your LAN to the internet.

Furthermore, it is necessary to configure an [inbound NAT rule (or port forward)](/images/WANIPTV_PortForward.png) for RTP traffic (port 554). This is required for the option to restart TV programs to work.
I took some network captures to figure this one out and I decided the source of the RTP stream is 185.24.175.0/24. For now, this seems to be working fine. I may have to update this in the future if I find that I cannot restart programs if the stream comes from a different IP range.

  > The provider suggests to implement RTSP connection tracking, but I could not find any information on this in combination with OPNsense. And since I have only one STB (i.e. the stream always goes to the same target), I opted for the port forward instead.


## Firewal rules

I've got the below [firewall rules configured on the **WANIPTV** interface](/images/WANIPTV_FWrules.png).
|Protocol|Source|Port|Destination|Port|Allow Options|Description|
|---|---|---|---|---|---|---|
|IPv4 * | * | * | WANIPTV address | * |Yes|Any traffic allowed in on WANIPTV address|
|IPv4 * | * | * | IGMP_BROADCAST_NETWORKS | * |Yes|Allow any traffic on WANIPTV to Multicast reserved IP address space|

There's another rule related to the port forward configured earlier, but that rule is auto-generated so I'm not including it in the table.

In order for IGMP traffic to be allowed through the firewall continuously, ["allow options" must be set](/images/FWrules_AllowOptions.png) on the above firewall rules. This is not reflected in the table and has caused me to spend a lot of troubleshooting time before figuring this out.

On the [**InternalIPTV** interface](/images/InternalIPTV_FWrules.png), the below rules are configured.
|Protocol|Source|Port|Destination|Port|Allow Options|Description|
|---|---|---|---|---|---|---|
|IPv4 * | InternalIPTV | * | * | * |Yes| |
|IPv4 * | 185.24.175.0/24 | * | IPTV_STB | * |Yes|Allow RTP streams to STB (enables restarting of TV programs|

In the above firewall rules I use two aliases:
1. IGMP_BROADCAST_NETWORKS is set to 224.0.0.0/4
2. IPTV_STB is set to the static internal IP address of the set-top box on VLAN 40.

## IGMP Proxy
IGMP proxying must be enabled for the multicast streams to make it through to the set-top box.
The os-igmp-proxy can be installed in OPNsense via System -> Firmware -> Plugins. Once installed, the IGMP-service can be configured via [Services -> IGMP Proxy](/images/IGMP_Proxy.png).

One upstream and one downstream interface must be configured. If there are multiple IPTV set-top boxes, multiple downstream interfaces can be configured. There can always be only one upstream interface.
    

|Interface|Type|Network(s)|Description|
|---|---|---|---|
|WANIPTV|Upstream|217.166.225.0/24|CanalDigitaal IPTV|
|InternalIPTV|Downstream|InternalIPTV Network|Internal IPTV|