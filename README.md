kernel_pactch_for_udp
=====================

kernel patch for binding a single UDP packet to specific interface via IPV6_PKTINFO, or IP_PKTINFO

when we send a UDP packet with option IPV6_PKTINFO, or IP_PKTINFO, and specify out bound interface from user space, 
it will be used as key to lookup route table, instead of sk->sk_bound_dev_if, in udp_sendmsg, but in hook NF_INET_LOCAL_OUT,
kernel will check if route entry match skb, if not, kernel will reroute this skb, in this case, it will because skb and route
 entry doesn't match due to extra out bound interface key when lookup route table, so it will reroute at ip_route_me_harder,
this time it pick up sk->sk_bound_dev_if as key to lookup route table, and this cause out bound interface in IPV6_PKTINFO, or IP_PKTINFO
 totally ignored, specially when we don't bind this socket to some interface, 
 its routing result will depend on route table, 
 
 there are two result: 
 
 1) it find a route entry, this packet will route to that destionation
 
 2) it doesn't, this packet will drop silently.
 
for example:
there are two WAN interface, 

interface WAN1: IP address 11.11.11.11, mac address 11:11:11:11:11:11

interface WAN2: IP address 22.22.22.22, mac address 22:22:22:22:22:22

we send a DNS query with binding interface WAN1 in IPV6_PKTINFO, or IP_PKTINFO
when it looup route table first time, its destination interface is WAN1, and its source ip address is 11.11.11.11,

when reroute, we assume that default route interface is WAN2(if we send a UDP packet with option IPV6_PKTINFO, or IP_PKTINFO, it is most likely that out bound interface is not default route interface, or we don't need this option), it find default route entry, so it will route out via interface WAN2,

but it won't change its source IP address because it already did in first time lookup, the packet will send out with IP address 11.11.11.11, and mac address 22:22:22:22:22:22, and when server reply, interface WAN1 will get that reply.

if it doesn't find a route entry in second time, this DNS query will be dropped silently.

and there is another solution except this patch, which is add a policy route rule for interface WAN1 with source address lookup, but I think in this scenario 
it is not a good option.

any questions, drop me a email: weishigoname@hotmail.com
