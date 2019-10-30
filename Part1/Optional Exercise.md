# OSI (Open Systems Interconnection) Model and SSH

Linux has a least two network interfaces:
1 - loopback (lo) interface will have an IP address of 127.0.0.1, which represents the host itself
2 - ethernet 0 (enp0s3) interface is typically the connection to the local network (even in a virtual machine (VM), you’ll still have an eth0 interface that connects to the physical network interface of the host)

### MAC addresses

A *media access control (MAC)* address is the unique identifier assigned to a network interface at layer 2 — the Data Link layer — of the OSI Model. A network interface always has a MAC address—often referred to as the hardware address — even if it does not have an IP address.

### Ping

**Ping** is the most basic network test tool around for testing network reachability. It sends
out an Internet Control Message Protocol (ICMP) packet across the network and notifies you whether there is a response. If a host is up and able to communicate on the network, an ICMP response will be
returned.

### OSI Layers
1. Physical: is the physical layer that includes the physical media used to connect the network (e.g. Gigabit Ethernet, 802.11 wireless)
2. Data link: physical addressing (MAC). For example, Ethernet networking works to encapsulate data and pass that data in the form of *frames*
3. Network: logical (IP) addressing path determination
4. Transport: connection and reliability
5. Session: interhost communication -- logical ports (session establishment between processes)
6. Presentation: data presentation and encryption (formats the data to be presented to the application layer)
7. Application: sers and application processes to access network services, such as electronic massaging like email or network terminals

[![fig1](https://24itworld.files.wordpress.com/2016/08/296299-image0.jpg)](https://24itworld.wordpress.com/2016/08/04/iso-osi-model-layers-of-the-network/)

> Transparent bridges are layer 2 devices that send all frames received on one port out the other bridge ports, based on knowledge of the frame’s destination MAC address. Ethernet switches are multiport network bridges. Multiport network bridges learn of the MAC addresses in the network and intelligently forward frames based on the destination MAC address in the frame.

### SSH, SSL, IPSec and OSI model layer

- Secure Shell (SSH), is an OSI model **application layer** protocol use cryptographic to allow remote login and other network services to operate securely over an unsecured network. 
- Secure Sockets Layer (SSL) runs inside TCP and encrypts the data inside the TCP packets. 
- IPsec replaces IP with an encrypted version of the IP layer.

In practice, SSL and SSH are typically used for different purposes: SSH is most often used for remote log-in, SSL for encrypted web access. On the other hand, IPsec is predominately used in VPNs. Transport Layer Security (TLS) and its predecessor, Secure Sockets Layer (SSL), both of which are frequently referred to as 'SSL', are cryptographic protocols that provide communications security over a computer network. SSL is a layer that fits in between HTTP and TCP. Although SSL can be used to secure any protocol that runs above TCP, the most common application of it is in HTTPS. IPsec is a layer that fits in between TCP and the physical layer. It enhances the IP layer by adding encryption to the data inside it, including the TCP layer if that is what is being sent in the IP packets. IPsec also allows for authentication of both parties communicating and provides methods for secure key exchange. IPsec support is a mandatory part of IPv6.

> SSL/TLS works by binding the identities of entities to cryptographic key pairs via digital documents known as **X.509 certificates**. Each key pair consists of a private key and a public key. The private key is kept secure, and the public key can be widely distributed via a certificate. SSL and SSH are two protocols with very similar functionality -- they both provide the cryptographic elements to build a tunnel for confidential data transport with checked integrity. They differ on the things which are around the tunnel. SSL traditionally uses X.509 certificates for announcing server and client public keys; SSH has its own format.

Reference: [XYZ NETWORK](http://xyznetwork.blogspot.com/2016/04/ssh-ssl-ipsec-osi-model-layer.html)