# AWS VPC Peering Project

Two AWS VPCs connected with a VPC peering connection. A client reaches a public EC2 instance through an Internet Gateway, and a private EC2 instance in the first VPC reaches a private EC2 instance in the second VPC over private IP addresses.

## Architecture

![VPC Peering Architecture](architecture/vpc-peering-diagram.png)

```
Client
  |
Internet Gateway
  |
+-------------------------- VPC 1 (28.0.0.0/16) ---------------------+      +---- VPC 2 (28.200.0.0/16) ----+
|                                                                     |      |                               |
|  Public subnet 28.0.1.0/24      Private subnet 28.0.2.0/24          |      |  Private subnet 28.200.0.0/16 |
|  [Public EC2] ---------------> [Private EC2] ---- RT ---- Peering ----------- RT ---- [Private EC2]       |
|        |                                                            |      |                               |
|   RT: 0.0.0.0/0 -> IGW                                              |      |                               |
+---------------------------------------------------------------------+      +-------------------------------+
```

| Component | Details |
|---|---|
| VPC 1 | `28.0.0.0/16` |
| Public subnet | `28.0.1.0/24`, has a route to the Internet Gateway, hosts the public EC2 |
| Private subnet | `28.0.2.0/24`, no internet route, hosts a private EC2 |
| Internet Gateway | Attached to VPC 1, used by the public subnet |
| VPC 2 | `28.200.0.0/16` |
| Private subnet (VPC 2) | `28.200.0.0/16`, hosts a private EC2 |
| VPC peering connection | Connects VPC 1 and VPC 2 |

## Route Tables

| Route table | Destination | Target |
|---|---|---|
| Public subnet (VPC 1) | `28.0.0.0/16` | local |
| | `0.0.0.0/0` | Internet Gateway |
| Private subnet (VPC 1) | `28.0.0.0/16` | local |
| | `28.200.0.0/16` | Peering connection |
| VPC 2 subnet | `28.200.0.0/16` | local |
| | `28.0.0.0/16` | Peering connection |

Both peering routes are required. A route on one side only lets packets out but never lets replies return.

## Security Groups

| Instance | Type | Port | Source |
|---|---|---|---|
| Public EC2 | SSH | 22 | Your IP address |
| Private EC2 (VPC 1) | SSH | 22 | `28.0.0.0/16` |
| | All ICMP - IPv4 | N/A | `28.0.0.0/16` |
| Private EC2 (VPC 2) | SSH | 22 | `28.0.0.0/16` |
| | All ICMP - IPv4 | N/A | `28.0.0.0/16` |

## Traffic Flow

1. The client connects to the **public EC2** through the Internet Gateway.
2. From the public EC2, the client reaches the **private EC2** in VPC 1 over the local route.
3. From the private EC2 in VPC 1, traffic to `28.200.0.0/16` goes through the **peering connection** to the private EC2 in VPC 2.
4. Replies return through the peering route in VPC 2's route table.

## Connectivity Tests

From the public EC2:

```bash
ping -c4 <private-ec2-vpc1-ip>
```

From the private EC2 in VPC 1:

```bash
ping -c4 <private-ec2-vpc2-ip>
```

Expected result in each case: `4 packets transmitted, 4 received, 0% packet loss`.

Replace the placeholders with the instances' private IP addresses, and paste your own test output here.

## Troubleshooting Notes

| Problem | Cause | Fix |
|---|---|---|
| EC2 Instance Connect: "Instance is not in a public subnet" | Subnet was using the main route table, with no route to the Internet Gateway | Added `0.0.0.0/0 → IGW` to a custom route table and associated it with the public subnet |
| Ping across the peering connection showed 100% packet loss | Missing return route and ICMP rule in the peer VPC | Added a peering route to the peer route table and an inbound ICMP rule to its security group |

## Screenshots

| Step | File |
|---|---|
| VPC creation | `screenshots/vpc-creation.png` |
| Subnet creation | `screenshots/subnet-creation.png` |
| VPC peering | `screenshots/vpc-peering.png` |
| Route tables | `screenshots/route-tables.png` |
| Security groups | `screenshots/security-groups.png` |
| Connectivity test | `screenshots/connectivity-test.png` |

## Documentation

Step-by-step instructions are in [`documentation/project-steps.md`](documentation/project-steps.md).

## Cleanup

To avoid charges: terminate the EC2 instances, delete the peering connection, detach and delete the Internet Gateway, then delete the subnets, route tables and VPCs.
