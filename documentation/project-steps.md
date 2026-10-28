# Project Steps: AWS VPC Peering

Build two VPCs, connect them with a peering connection, and test private connectivity between EC2 instances.

## 1. Create the VPCs

1. Go to VPC → Your VPCs → Create VPC.
2. VPC 1: IPv4 CIDR `28.0.0.0/16`.
3. VPC 2: IPv4 CIDR `28.200.0.0/16`.
4. The CIDR blocks must not overlap, or peering is not possible.

## 2. Create the Subnets

| VPC | Name | CIDR |
|---|---|---|
| VPC 1 | Public subnet | `28.0.1.0/24` |
| VPC 1 | Private subnet | `28.0.2.0/24` |
| VPC 2 | Private subnet | `28.200.0.0/16` |

For the public subnet, enable auto-assign public IPv4 address (Actions → Edit subnet settings), or assign a public IP when launching the instance.

## 3. Create and Attach the Internet Gateway

1. Go to VPC → Internet gateways → Create internet gateway.
2. Select it, then Actions → Attach to VPC and choose VPC 1.

## 4. Configure the Route Tables

Public route table (VPC 1)
1. Route tables → Create route table, select VPC 1.
2. Routes → Edit routes → Add route: `0.0.0.0/0` → Internet Gateway.
3. Subnet associations → Edit: select the public subnet.

Private route table (VPC 1)
1. Create a second route table in VPC 1 (or use the main one).
2. Associate it with the private subnet. The peering route is added in step 6.

VPC 2 route table
1. Use the VPC 2 main route table, or create one and associate it with the VPC 2 subnet. The peering route is added in step 6.

## 5. Create the VPC Peering Connection

1. Go to VPC → Peering connections → Create peering connection.
2. Requester: VPC 1. Accepter: VPC 2 (same account, same region).
3. Select the new connection and choose Actions → Accept request.
4. Confirm the status is Active.

## 6. Add the Peering Routes

| Route table | Destination | Target |
|---|---|---|
| VPC 1 private subnet | `28.200.0.0/16` | Peering connection |
| VPC 2 subnet | `28.0.0.0/16` | Peering connection |

## 7. Launch the EC2 Instances

| Instance | VPC | Subnet | Public IP |
|---|---|---|---|
| Public EC2 | VPC 1 | Public subnet | Yes |
| Private EC2 (VPC 1) | VPC 1 | Private subnet | No |
| Private EC2 (VPC 2) | VPC 2 | Private subnet | No |

Use Amazon Linux 2023 and keep the key pair (`.pem`) for each instance.

## 8. Configure Security Groups

| Instance | Type | Port | Source |
|---|---|---|---|
| Public EC2 | SSH | 22 | Your IP address |
| Private EC2 (VPC 1) | SSH | 22 | `28.0.0.0/16` |
| Private EC2 (VPC 1) | All ICMP - IPv4 | N/A | `28.0.0.0/16` |
| Private EC2 (VPC 2) | SSH | 22 | `28.0.0.0/16` |
| Private EC2 (VPC 2) | All ICMP - IPv4 | N/A | `28.0.0.0/16` |

## 9. Connect to the Public EC2

1. Select the public instance and click Connect → EC2 Instance Connect**.
2. Username `ec2-user`, then Connect.

If you see "Instance is not in a public subnet", the subnet has no route to the Internet Gateway. Recheck step 4.

## 10. Test Connectivity

From the public EC2:

```bash
ping -c4 <private-ec2-vpc1-ip>
```

To reach the private instance over SSH, copy its key onto the public instance first:

```bash
chmod 400 private-key.pem
ssh -i private-key.pem ec2-user@<private-ec2-vpc1-ip>
```

From the private EC2 in VPC 1, test the peering link:

```bash
ping -c4 <private-ec2-vpc2-ip>
```

Expected result: `4 packets transmitted, 4 received, 0% packet loss`.

## 11. Cleanup

1. Terminate all EC2 instances.
2. Delete the peering connection.
3. Detach and delete the Internet Gateway.
4. Delete the subnets, custom route tables and both VPCs.
