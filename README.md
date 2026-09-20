# Azure Hybrid Hub-and-Spoke Network with pfSense S2S VPN and Azure Firewall

## Project Overview

This project builds and validates a hybrid network between a simulated on-premises branch office and Microsoft Azure. The branch is simulated with VMware, pfSense, and Ubuntu. Azure uses a Hub-and-Spoke architecture with Azure VPN Gateway, Azure Firewall Basic, VNet peering, gateway transit, User Defined Routes (UDRs), NSGs, Network Watcher, and a private Linux VM.

The goal was not simply to make the VPN show **Connected**. The lab validated and troubleshot the complete packet path from an on-premises workstation to a private Azure workload. The Azure environment was built manually first so every component could be understood before rebuilding the same architecture with Terraform.

---

## Final Architecture

```text
Ubuntu workstation 192.168.50.100
            |
Branch LAN 192.168.50.0/24
            |
pfSense
LAN 192.168.50.1
WAN 192.168.71.128
            |
Home Router / NAT
            |
Internet
            |
IKEv2 / IPsec / NAT-T
UDP 500 -> UDP 4500
            |
Azure VPN Gateway
            |
Hub VNet 10.10.0.0/16
            |
Azure Firewall 10.10.3.4
            |
Hub <-> Spoke VNet Peering
            |
Spoke VNet 10.20.0.0/16
            |
Private Linux VM 10.20.0.4
```

## Technologies

- Microsoft Azure
- Azure Virtual Network
- Azure VPN Gateway
- Azure Local Network Gateway
- Azure Site-to-Site VPN
- Azure Firewall Basic and Firewall Policy
- VNet Peering and Gateway Transit
- Azure Route Tables / UDRs
- Network Security Groups
- Azure Network Watcher
- VMware
- pfSense
- Ubuntu Linux
- IKEv2 / IPsec / ESP
- NAT Traversal

---

# 1. IP Addressing

## Branch

| Component | Address |
|---|---|
| Branch LAN | `192.168.50.0/24` |
| pfSense LAN | `192.168.50.1` |
| Ubuntu workstation | `192.168.50.100` |
| pfSense WAN | `192.168.71.128` |
| Internet endpoint | ISP public IP behind upstream NAT |

## Azure

| Component | Address |
|---|---|
| Hub VNet | `10.10.0.0/16` |
| Azure Firewall private IP | `10.10.3.4` |
| Spoke VNet | `10.20.0.0/16` |
| Final private test VM | `10.20.0.4` |

The Hub also contained dedicated Azure infrastructure subnets including `GatewaySubnet`, `AzureFirewallSubnet`, `AzureFirewallManagementSubnet`, and temporarily `AzureBastionSubnet`.

---

# 2. Branch Office

The local VMware environment represented a branch office. pfSense acted as the branch router, firewall, and VPN endpoint. Ubuntu represented an employee workstation.

```text
Ubuntu 192.168.50.100
        |
pfSense LAN 192.168.50.1
        |
pfSense WAN 192.168.71.128
        |
Home Router / NAT
        |
Internet
```

---

# 3. NAT Traversal

pfSense did not directly own the ISP public IP. Its WAN address was private (`192.168.71.128`) and the home router performed NAT.

IKE initially uses UDP 500. When NAT is detected, NAT Traversal encapsulates IPsec in UDP 4500.

```text
pfSense -> Home NAT -> Internet -> Azure VPN Gateway
                 IPsec NAT-T UDP 4500
```

The established pfSense VPN status later showed port 4500, confirming NAT-T operation.

---

# 4. Azure Hub and VPN Gateway

The Hub VNet was created as `10.10.0.0/16`. It hosted centralized connectivity and security services.

Azure VPN Gateway was deployed in the required `GatewaySubnet`. Its public IP became the remote VPN peer configured on pfSense.

A Local Network Gateway was also created. It is not another physical gateway; it is Azure's representation of the remote branch. It described the branch public endpoint and the on-premises address space:

```text
192.168.50.0/24
```

A Site-to-Site connection linked the Azure VPN Gateway and Local Network Gateway using a Pre-Shared Key.

**The real PSK must never be committed to this repository.**

---

# 5. IPsec Phase 1 – IKEv2

Phase 1 establishes the secure control relationship between pfSense and Azure.

Working configuration:

```text
IKE Version:       IKEv2
Authentication:    Pre-Shared Key
Encryption:        AES-128
Integrity:         SHA-256
Diffie-Hellman:    Group 2
Lifetime:          28800 seconds
```

Phase 1 handles peer authentication, cryptographic negotiation, Diffie-Hellman key exchange, and establishment of the IKE Security Association.

## Phase 1 Troubleshooting

The first configuration used DH Group 14. The pfSense log showed:

```text
NO_PROPOSAL_CHOSEN
```

The log proved that pfSense sent the request, Azure received it and replied, but Azure rejected the proposed cryptographic combination.

DH was changed from Group 14 to Group 2. Phase 1 then established successfully.

This demonstrated an important troubleshooting principle: the Internet path was working; the failure was cryptographic negotiation, not basic connectivity.

---

# 6. Diffie-Hellman

Diffie-Hellman allows the VPN peers to derive shared key material without transmitting the final shared secret across the Internet.

In this lab, the negotiated Phase 1 used DH Group 2. The important lesson was not to guess at VPN failures: use IKE logs to identify proposal mismatches.

---

# 7. IPsec Phase 2 – Child SA

Phase 1 establishes the VPN relationship. Phase 2 protects the actual data networks.

Initial Phase 2:

```text
Local:  192.168.50.0/24
Remote: 10.10.0.0/16
Protocol: ESP
Encryption: AES-128
Integrity: SHA-256
```

This protected branch-to-Hub traffic.

Later, the actual test workload was placed in the Spoke `10.20.0.0/16`. A second Phase 2 was therefore required:

```text
Local:  192.168.50.0/24
Remote: 10.20.0.0/16
```

Final protected networks:

```text
192.168.50.0/24 <-> 10.10.0.0/16
192.168.50.0/24 <-> 10.20.0.0/16
```

Without the second selector, traffic to `10.20.0.4` did not even enter the pfSense IPsec interface.

---

# 8. pfSense Firewall Rules

The VPN tunnel alone does not automatically permit traffic.

## LAN Rule

Branch-initiated Azure traffic was permitted with a LAN rule using:

```text
Action: Pass
Protocol: IPv4 Any
Source: LAN subnets
Destination: Azure network
```

A useful lesson was the difference between `LAN address` and `LAN subnets`. `LAN address` means pfSense itself (`192.168.50.1`); `LAN subnets` represents the branch network (`192.168.50.0/24`).

## IPsec Rules

Return traffic arriving from Azure was allowed with rules for:

```text
10.10.0.0/16 -> LAN subnets
10.20.0.0/16 -> LAN subnets
```

---

# 9. Hub-and-Spoke Peering

The Hub (`10.10.0.0/16`) and Spoke (`10.20.0.0/16`) were connected with VNet peering.

The two sides required different gateway settings because they perform different jobs.

## Hub Side

The Hub owns the VPN Gateway.

```text
Allow virtual network access: Enabled
Allow forwarded traffic: Enabled
Allow gateway/route server to forward traffic: Enabled
```

## Spoke Side

The Spoke uses the gateway located in the Hub.

```text
Allow virtual network access: Enabled
Allow forwarded traffic: Enabled
Use remote gateway / route server: Enabled
```

Memory aid:

```text
HUB SHARES THE GATEWAY.
SPOKE USES THE GATEWAY.
```

---

# 10. Azure Firewall Basic

Azure Firewall Basic was deployed in the Hub.

Dedicated subnets were required:

```text
AzureFirewallSubnet
AzureFirewallManagementSubnet
```

The management subnet provides a separate management path for the Basic firewall.

The firewall private IP was:

```text
10.10.3.4
```

This private IP—not the public IP—was used as the next hop for internal Azure routing.

The public IP is relevant to Internet-facing SNAT/DNAT scenarios. Internal Hub/Spoke routing uses the firewall private address.

---

# 11. Public IP Quota Troubleshooting

The lab subscription allowed only three Public IP addresses.

Existing resources included the VPN Gateway public IP and Azure Bastion public IP. Azure Firewall Basic required two public IP resources for the deployment.

That would have exceeded quota.

Azure Bastion was not required for the final test, so Bastion and its public IP were deleted. This left capacity for:

```text
VPN Gateway PIP        = 1
Firewall data PIP      = 1
Firewall management PIP = 1
Total                  = 3
```

This was a real cloud quota troubleshooting scenario rather than a networking configuration error.

---

# 12. Azure Firewall Policy

A Firewall Policy was attached to Azure Firewall.

Rule collection:

```text
Branch-Spoke-Rules
```

Branch to Spoke:

```text
Name: Allow-Branch-Spoke
Source: 192.168.50.0/24
Destination: 10.20.0.0/16
Protocol: Any
Ports: *
Action: Allow
```

Spoke to Branch:

```text
Name: Allow-Spoke-Branch
Source: 10.20.0.0/16
Destination: 192.168.50.0/24
Protocol: Any
Ports: *
Action: Allow
```

During configuration, Azure temporarily rejected a second update because the policy was still in `Updating` state. Waiting for the asynchronous update to complete resolved it.

---

# 13. Spoke-to-Branch UDR

A route table was created for the Spoke workload subnet.

```text
Destination: 192.168.50.0/24
Next hop type: Virtual appliance
Next hop IP: 10.10.3.4
```

Meaning:

```text
Spoke VM
  |
  | destination 192.168.50.x
  v
Azure Firewall 10.10.3.4
  |
  v
VPN Gateway
  |
  v
IPsec
  |
  v
pfSense
  |
  v
Branch
```

---

# 14. Wrong UDR Association

The route itself was correct, but it was initially associated with the wrong Spoke subnet.

The test VM `10.20.0.4` was in the `10.20.0.0/24` subnet, while the route table was attached elsewhere.

Network Watcher initially reported:

```text
Next Hop Type: VirtualNetworkGateway
```

After associating the route table with the subnet containing `10.20.0.4`, Network Watcher reported:

```text
Next Hop Type: VirtualAppliance
Next Hop IP: 10.10.3.4
```

Lesson: creating a UDR is not enough. Its route table must be associated with the correct subnet.

---

# 15. Symmetric Routing

Azure Firewall is stateful, so the design needed consistent paths in both directions.

Desired flow:

```text
Branch -> VPN Gateway -> Azure Firewall -> Spoke
Spoke  -> Azure Firewall -> VPN Gateway -> Branch
```

A gateway-side route was used to steer Spoke-bound traffic toward the firewall:

```text
Destination: 10.20.0.0/16
Next hop type: Virtual appliance
Next hop IP: 10.10.3.4
```

This prevented one direction from bypassing the firewall while the return path crossed it.

---

# 16. Private Spoke VM

A Linux VM was deployed in the Spoke:

```text
Private IP: 10.20.0.4
Public IP: None
```

No Public IP was used because the test was specifically intended to validate private hybrid networking.

---

# 17. NSG Test Rule

The workload NSG initially did not allow the ICMP test from the branch.

A temporary inbound rule was added:

```text
Source: 192.168.50.0/24
Protocol: ICMPv4
Destination: Any
Action: Allow
Priority: 100
```

In production, firewall and NSG rules should be restricted to only the protocols and ports actually required.

---

# 18. Azure Network Watcher

Network Watcher Next Hop was used with:

```text
Source: 10.20.0.4
Destination: 192.168.50.100
```

Before fixing the UDR association:

```text
Next Hop: VirtualNetworkGateway
```

After fixing it:

```text
Next Hop: VirtualAppliance
IP: 10.10.3.4
```

This proved the Spoke VM's branch-bound traffic was being routed through Azure Firewall.

---

# 19. pfSense Packet Capture

When branch-to-Spoke ping still failed, pfSense packet capture was used.

```text
Interface: IPsec (enc0)
Host filter: 10.20.0.4
Protocol: ICMP
```

The capture initially showed no packets.

That immediately indicated the traffic was not entering IPsec, so continuing to modify Azure Firewall or the VM NSG would have been the wrong troubleshooting direction.

Inspection of Phase 2 revealed that only `10.10.0.0/16` was configured. Adding the second selector for `10.20.0.0/16` resolved the missing VPN path.

---

# 20. Final Connectivity Test

Source:

```text
Ubuntu branch workstation
192.168.50.100
```

Destination:

```text
Private Azure Spoke VM
10.20.0.4
```

Command:

```bash
ping 10.20.0.4
```

The ping succeeded.

This validated the complete hybrid path:

```text
192.168.50.100
    |
pfSense
    |
IKEv2 / IPsec / NAT-T
    |
Azure VPN Gateway
    |
Hub
    |
Azure Firewall 10.10.3.4
    |
VNet Peering
    |
Spoke
    |
10.20.0.4
```

---

# 21. Troubleshooting Summary

## IKE Failure

```text
Error: NO_PROPOSAL_CHOSEN
Cause: IKE proposal mismatch
Fix: DH Group 14 -> DH Group 2
```

## pfSense Behind NAT

```text
pfSense WAN: 192.168.71.128
Fix/behavior: NAT-T over UDP 4500
```

## Public IP Quota

```text
Quota: 3
Problem: VPN Gateway + Bastion + two Firewall PIPs exceeded quota
Fix: Remove unused Bastion/PIP
```

## Gateway Transit

```text
Problem: Spoke needed Hub VPN Gateway
Fix:
Hub -> allow gateway transit
Spoke -> use remote gateway
```

## Wrong UDR Subnet

```text
Problem: Correct route attached to wrong subnet
Fix: Associate route table with subnet containing 10.20.0.4
```

## Asymmetric Routing

```text
Problem: One direction could bypass stateful firewall
Fix: Route both directions through Azure Firewall
```

## NSG

```text
Problem: ICMP test blocked
Fix: Allow ICMPv4 from 192.168.50.0/24
```

## Missing Phase 2

```text
Problem:
192.168.50.0/24 <-> 10.10.0.0/16 existed
but Spoke 10.20.0.0/16 was missing

Fix:
192.168.50.0/24 <-> 10.20.0.0/16
```

---

# 22. Troubleshooting Methodology

The lab reinforced a hop-by-hop troubleshooting workflow:

```text
1. Can the VPN peers reach each other?
2. Does IKE Phase 1 establish?
3. Does IPsec Phase 2 establish?
4. Does Phase 2 include the destination network?
5. Does pfSense permit the traffic?
6. Does Azure know the branch route?
7. Is gateway transit configured?
8. Is the UDR associated with the correct subnet?
9. Does Azure Firewall allow the flow?
10. Is routing symmetric?
11. Does the workload NSG allow the flow?
12. Validate with Network Watcher and packet capture.
```

The key principle was to identify the broken hop before changing configuration.

---

# 23. Skills Practiced

- Azure Hub-and-Spoke networking
- Azure VPN Gateway
- Local Network Gateway
- Site-to-Site VPN
- IKEv2
- IPsec Phase 1 and Phase 2
- IKE SA / Child SA concepts
- AES and SHA-256
- Diffie-Hellman
- NAT Traversal
- UDP 500 and UDP 4500
- ESP
- pfSense routing and firewall policy
- VNet peering
- Gateway transit
- Azure Firewall
- Azure Firewall Policy
- Stateful inspection
- User Defined Routes
- Virtual appliance next hops
- Symmetric routing
- NSGs
- Azure Network Watcher
- Next Hop diagnostics
- Packet capture
- Azure quota troubleshooting
- Private hybrid connectivity

---

# 24. Security Notes

- Do not commit the real VPN PSK.
- Do not commit passwords or private keys.
- Blur secrets in screenshots before publishing.
- Broad `Any` firewall rules were used for controlled lab validation; production rules should use least privilege.
- Restrict NSGs to required sources, destinations, protocols, and ports.
- Avoid unnecessary Public IPs.
- Keep workloads private when possible.
- Production cryptographic settings should follow current organizational and vendor requirements rather than blindly copying a lab configuration.

---

# 25. Cost Cleanup

Azure Firewall and Azure VPN Gateway can create meaningful charges.

After successful testing and screenshot collection, the lab Azure Resource Group was deleted to remove the Azure resources.

The local VMware environment was retained:

```text
pfSense
Ubuntu branch workstation
```

These local machines will be reused for the Terraform version.

---

# 26. Terraform Rebuild – Next Phase

The next phase will recreate the Azure side of this exact manually validated architecture with Terraform.

Planned Terraform resources:

- Resource Group
- Hub VNet
- Gateway subnet
- Firewall subnet(s)
- Spoke VNet
- Spoke subnet
- Hub/Spoke peerings
- Gateway transit settings
- Public IP resources
- Azure VPN Gateway
- Local Network Gateway
- VPN connection
- Azure Firewall Basic
- Azure Firewall Policy
- Firewall network rules
- Route tables
- UDRs
- Route table associations
- NSG and rules
- Private test VM
- Useful outputs

The pfSense branch will remain the simulated on-premises environment.

The Terraform objective is to reproduce infrastructure that has already been built, understood, troubleshot, and validated manually.


```

---

# 29. Final Result

Successful private connectivity was validated from:

```text
On-Premises Ubuntu
192.168.50.100
```

through:

```text
pfSense
-> IKEv2/IPsec/NAT-T
-> Azure VPN Gateway
-> Azure Hub
-> Azure Firewall 10.10.3.4
-> VNet Peering
-> Azure Spoke
```

to:

```text
Private Linux VM
10.20.0.4
```

Final validation:

```bash
ping 10.20.0.4
```

**Result: Successful end-to-end hybrid connectivity.**

---

## Project Status

- Manual architecture build: **Completed**
- Site-to-Site VPN: **Completed**
- Hub-and-Spoke networking: **Completed**
- Azure Firewall routing: **Completed**
- Troubleshooting: **Completed**
- End-to-end validation: **Completed**
- Azure cost cleanup: **Initiated after validation**
- Local VMware branch: **Retained**
- Next phase: **Terraform implementation**

---

## Closing Notes

The most valuable part of this project was not simply achieving a successful ping.

The lab required troubleshooting at multiple layers: IKE negotiation, Diffie-Hellman mismatch, NAT Traversal, IPsec Phase 2 selectors, pfSense firewall policy, gateway transit, UDR association, Azure Firewall routing, symmetric traffic flow, NSG filtering, Azure Network Watcher, and packet capture.

