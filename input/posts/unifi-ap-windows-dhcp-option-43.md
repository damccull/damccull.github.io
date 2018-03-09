Title: UniFi Access Points and Windows DHCP Server
Published: 3/9/2018 03:08
Tags:
    - Ubiquiti
    - UniFi
    - Networking
Lead: Learn to use a vendor-specific DHCP Option 43 to inform UniFi Access Points about a UniFi Controller on another network.
---
UniFi Access Points (APs) are fantastic, but can be difficult to adopt from a UniFi Controller if they never show up. Many different DHCP servers can be configured to tell the APs where the Controller is. You can learn to configure several DHCP servers [here][other-dhcp-stuff] but, to my knowledge, noone has yet written a tutorial on how to do this with Windows DHCP Server. This article aims to teach you just how to do that.



[other-dhcp-stuff]: https://help.ubnt.com/hc/en-us/articles/204909754-UniFi-Device-Adoption-Methods-for-Remote-UniFi-Controllers#DHCP "Ubiquity's Adoption tutorial"