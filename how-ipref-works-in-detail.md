## How IPREF Works in Detail

IPREF is a method for traversing address spaces - such as private networks behind NAT, NAT6, or filters including cross protocol IPv4/IPv6 - which are otherwise not reachable through native network protocols. The term IPREF is an abbreviation for "IP Addressing with References"

#### Key characteristics of IPREF:

- massively scalable
- cross protocol, cross address space
- strictly layer 3 (no port manipulation, all ports available on all hosts)
- host addresses publishable in DNS
- supports any combination of IPv4/IPv6 local networks and IPv4/IPv6 Internet
- no need for external translators or shared configurations
- does not need any global addresses from any protocol domain, IPv4 or IPv6

#### Perfect for transition to IPv6 Internet

Using IPREF for transitioning to IPv6 Internet simplifies network design, making the entire process much cleaner.

- needs only pure IPv6 Internet (no need for IPv4 translators or services)
- transitions to pure IPv6 local networks (no dual stacks)
- does not need IPv4 global addresses
- eliminates NAT/NAT6 (no  need for port manipulation)
- plays nicely with DNSSEC
- allows to drop IPv4 Internet very early, before a single network is touched
- dramatically speeds up adoption of IPv6 Internet
- substantially lowers cost of transition for enterprises


### Detailed description

- ### Fundamental Concepts

	IPREF is based on an observation that the originating host does not need to know the destination address so long as that address can be referred to in a manner understood by both ends. Similarly, the destination host does not need to know the source address, again, so long as it can be referred to. Thus IPREF uses references to IP addresses instead of real addresses. This approach produces highly scalable, cross protocol, cross address space communication system without a need for any kind of negotiations or shared configurations.

- ### IPREF Address

    ![](./how-ipref-works-in-detail.img1.jpg)
    
    IPREF does not use real addresses. Instead, it uses references which are opaque integers. Such references need contexts to be meaningful, therefore IPREF uses addressing units, called IPREF addresses, that are a combination of a native IP address and a reference. The native address is the context.

    Typically, the context address is an Internet address of an edge router of the source or destination network. It is a convenience, not a requirement, other addresses known to the network may also be used. Originating hosts send packets to the network with this address and have that network calculate the actual address of the destination based on the supplied reference.

    References are allocated by admins of respective networks where hosts reside. There is no negotiations involved since these references are interpreted in the context of respective networks.

- ### Packet Exchange

	![](./how-ipref-works-in-detail.img2.jpg)

	Conceptually, packet exchange is based exclusively on IPREF addresses. However, local networks run native protocols, IPv4 or IPv6, which do not understand IPREF addresses. The gateways encode IPREF addresses into native local addresses which can be used by local hosts. Encoded addresses come from a dedicated private address range, such as 10.128.0.0/10 or fdee:eeee::/64. The gateways replace these encoded addresses with corresponding IPREF addresses before sending packets out. On the way in, they replace incoming IPREF addresses with encoded local addresses before injecting packets to the local network.

	Let's say an IPv4 host A1 wants to send a packet to an IPv6 host B6:
	
	1. #### Host A1 finds out the address of host B6
	
		Host  A1 issues a DNS query for host B6. The query goes through  the local resolver.
		
		Local resolver receives a response:
		
			B6 has address: 2001:db8:bb:2::8 + 2466
		
		Since local hosts don't understand IPREF addresses, the resolver asks gateway GWA to allocate an encoded address for it. The gateway responds:
		
			map: 2001:db8:bb:2::8 + 2466  =  10.128.48.62
			
		The resolver returns the encoded result of the DNS query to host A1:
		
			B6 has address:  10.128.48.62
			
	1. #### Host A1 sends a packet out to B6
	
		Host A1 places its own address as source and the address of host B6 returned by the local resolver as destination.
		
			packet out: | 172.17.1.75 | 10.128.48.62 | payload |
			
	1. #### Packet arrives at gateway GWA
	
			packet in:  | 172.17.1.75 | 10.128.48.62 | payload |
			
		The gateway notices that it does not have a mapping for the source address, so it creates one on the fly:
		
			map: 172.17.1.75 = 2001:db8:aa:1::9 + 1579
			
		It replaces source address with the just created corresponding IPREF address. It replaces encoded destination address with the original IPREF address returned by the DNS query. It also repackages IPv4 packet into IPv6 since the Internet is IPv6.
		
			packet out: | 2001:db8:aa:1::9 + 1579 | 2001:db8:bb:2::8 + 2466 | payload |


	1. #### Packet arrives at gateway GWB
	
			packet in:  | 2001:db8:aa:1::9 + 1579 | 2001:db8:bb:2::8 + 2466 | payload |
			
		The gateway notices that it does not have an encoding for the source IPREF address, so it creates one on the fly:
		
			map: 2001:db8:aa:1::9 + 1579 = fdee:eeee::1377:5951
			
		It replaces source IPREF address with the just created local encoded address. It replaces destination IPREF address with the local address of host B6
		
			packet out: | fdee:eeee::1377:5951 | 2001:db8:bb:12::28 | payload |
			
			
	1. #### Packet arrives at host B6
	
			packet in:  | fdee:eeee::1377:5951 | 2001:db8:bb:12::28 | payload |
			
		Host B6 recognizes destination address as its own and consumes the packet. OS passes the packet to an application implied by layer 4.
		
	1. #### Host B6 sends a reply back to A1
	
		Host B6 gets a reply payload from the application, swaps source and destination addresses, and sends the packet back to A1
		
			packet out: | 2001:db8:bb:12::28 | fdee:eeee::1377:5951 | payload |
			
	1. #### Packet arrives at gateway GWB
	
			packet in:  | 2001:db8:bb:12::28 | fdee:eeee::1377:5951 | payload |
			
		The gateway replaces local source address with corresponding IPREF address previously allocated and advertised in DNS. It replaces destination local encoded address with the corresponding IPREF address from mapping created earlier.
		
			packet out: | 2001:db8:bb:2::8 + 2466 | 2001:db8:aa:1::9 + 1579 | payload |
			
	1. #### Packet arrives at gateway GWA
	
			packet in:  | 2001:db8:bb:2::8 + 2466 | 2001:db8:aa:1::9 + 1579 | payload |
			
		The gateway replaces source IPREF address with the corresponding local encoded address from mapping created earlier. It replaces destination IPREF address with the local address of host A1. It also realizes that the local network is IPv4, so it repackages the packet into IPv4.
		
			packet out: | 10.128.48.62 | 172.17.1.75 | payload |
			
	1. #### Packet arrives at host A1
	
			packet in:  | 10.128.48.62 | 172.17.1.75 | payload |
			
		Host A1 recognizes destination address as its own and consumes the packet. OS passes the packet to the application that sent the original packet.
		

- ### IPREF with DNS

	![](./how-ipref-works-in-detail.img3.jpg)

	IPREF addresses are publishable via DNS. These addresses will most likely require a new RR type. Until a new record type is allocated, TXT records may be used. In the diagram above, an AA record is used for illustration purpose.
	
	Local networks may publish IPREF addresses of their servers using standard authoritative DNS servers. All such published IPREF addresses must be communicated to the local IPREF gateway so that it can properly map destination addresses of incoming packets to local hosts.
	
	For resolution of DNS names, a modified recursive resolver, that recognizes IPREF addresses, is required. The resolver must be able to encode returned IPREF addresses into local private addresses in cooperation with local IPREF gateway. This is needed because local hosts do not understand IPREF addresses. Instead, local hosts use encoded equivalents which are then replaced with the actual IPREF addresses by the gateway.
	
	IPREF address records may be protected by DNSSEC. Packets leaving local networks contain the exact addresses returned from DNS. Internally, in the local networks, encoded versions of these addresses are used which complicates things a little but the gateways replace these encoded addresses with actual IPREF addresses listed in DNS before sending packets out of the local network.