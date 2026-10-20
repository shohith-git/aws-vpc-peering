# AWS VPC Peering Project — EWB Cloud Labs

> **Project:** Establish private connectivity between two Amazon VPCs using VPC Peering  
> **Cloud provider:** Amazon Web Services (AWS)  
> **Region:** Asia Pacific (Mumbai) — `ap-south-1`  
> **Operating system:** Amazon Linux 2023  
> **Project type:** Hands-on cloud networking lab

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Architecture](#architecture)
- [Resource Inventory](#resource-inventory)
- [Network and Routing Configuration](#network-and-routing-configuration)
- [Security Configuration](#security-configuration)
- [Implementation Summary](#implementation-summary)
- [Validation and Test Results](#validation-and-test-results)
- [Screenshots and Evidence](#screenshots-and-evidence)
- [Key Learnings](#key-learnings)
- [Cleanup](#cleanup)

## Project Overview

This project demonstrates how to connect two isolated Amazon Virtual Private Clouds (VPCs) using **AWS VPC Peering**. The environment contains a public subnet and a private subnet in VPC A, a private subnet in VPC B, a bastion host, and two private EC2 application instances.

A VPC peering connection provides private network routing between the VPC CIDR ranges. After configuring the peering connection and the required routes, connectivity was tested from App-A to App-B and from App-B to App-A. SSH access to App-B was also verified through the bastion host. A controlled security-group change was performed to demonstrate a connectivity failure and recovery.

## Objectives

- Create two VPCs with non-overlapping CIDR blocks.
- Create and associate public and private subnets with their intended route tables.
- Configure an Internet Gateway for the public subnet in VPC A.
- Launch a bastion host and two private EC2 instances.
- Configure security groups for SSH and ICMP testing.
- Establish and activate a VPC peering connection.
- Add routes in both VPCs so private workloads can communicate.
- Validate baseline failure, bidirectional connectivity, SSH access, and controlled failure/recovery.

## Architecture

```mermaid
flowchart LR
    Client[Administrator workstation]
    IGW[Internet Gateway]
    subgraph A[VPC A — 10.10.0.0/16]
      subgraph APub[Public subnet — 10.10.1.0/24]
        Bastion[Bastion host\n10.10.1.10\nPublic IPv4]
      end
      subgraph APriv[Private subnet — 10.10.2.0/24]
        AppA[App-A\n10.10.2.10]
      end
    end
    Peer{{VPC Peering\npcx-03b56044799603b87\nActive}}
    subgraph B[VPC B — 10.20.0.0/16]
      subgraph BPriv[Private subnet — 10.20.1.0/24]
        AppB[App-B\n10.20.1.10]
      end
    end
    Client -->|SSH| Bastion
    IGW --- Bastion
    Bastion -->|SSH via ProxyJump| AppA
    Bastion -->|SSH via ProxyJump| AppB
    AppA <-->|ICMP / private routing| Peer
    Peer <-->|ICMP / private routing| AppB
```

**Important:** The diagram represents the completed lab configuration. The private application instances do not have public IP addresses. The bastion host is the SSH entry point for the private instances.

## Resource Inventory

| Resource | Name | Configuration |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Public subnet in VPC A | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet in VPC A | `ewb-a-private` | `10.10.2.0/24` |
| Private subnet in VPC B | `ewb-b-private` | `10.20.1.0/24` |
| Internet Gateway | `ewb-igw-a` | Attached to VPC A |
| Public route table | `ewb-rt-a-public` | Associated with `ewb-a-public` |
| Private route table A | `ewb-rt-a-private` | Associated with `ewb-a-private` |
| Private route table B | `ewb-rt-b-private` | Associated with `ewb-b-private` |
| Bastion EC2 | `ewb-bastion` | `10.10.1.10`; public IPv4 enabled |
| App-A EC2 | `ewb-app-a` | `10.10.2.10`; no public IP |
| App-B EC2 | `ewb-app-b` | `10.20.1.10`; no public IP |
| VPC peering connection | `ewb-vpc-a-b-peer` | `pcx-03b56044799603b87`; Active |
| EC2 key pair | `ewb-lab` | Used for SSH authentication |

## Network and Routing Configuration

### Route table entries

| Route table | Destination | Target | Purpose |
|---|---|---|---|
| `ewb-rt-a-public` | `10.10.0.0/16` | `local` | Communication inside VPC A |
| `ewb-rt-a-public` | `0.0.0.0/0` | `ewb-igw-a` | Internet route for the public subnet |
| `ewb-rt-a-public` | `10.20.0.0/16` | VPC peering connection | Routes traffic from the bastion subnet toward VPC B |
| `ewb-rt-a-private` | `10.10.0.0/16` | `local` | Communication inside VPC A |
| `ewb-rt-a-private` | `10.20.0.0/16` | VPC peering connection | Routes App-A traffic toward VPC B |
| `ewb-rt-b-private` | `10.20.0.0/16` | `local` | Communication inside VPC B |
| `ewb-rt-b-private` | `10.10.0.0/16` | VPC peering connection | Return route from VPC B toward VPC A |

The peering connection was created between VPC A (`10.10.0.0/16`) and VPC B (`10.20.0.0/16`) and reached **Active** status. Routes were added to the route tables used by the relevant subnets. The public route table's peering route is needed for the bastion host to reach App-B over the private network.

## Security Configuration

Three security groups were configured for the lab:

- **`ewb-sg-bastion`** — inbound SSH (TCP 22) restricted to the administrator's public IP address with a `/32` mask.
- **`ewb-sg-app-a`** — inbound SSH (TCP 22) from the bastion host's private IP (`10.10.1.10/32`) and ICMP echo requests from App-B (`10.20.1.10/32`).
- **`ewb-sg-app-b`** — inbound SSH (TCP 22) from the bastion host's private IP (`10.10.1.10/32`) and ICMP echo requests from App-A (`10.10.2.10/32`).

The instances use Amazon Linux 2023. The application instances are private and are accessed through the bastion host. The lab uses the default network ACL configuration. Review the actual security-group and network ACL settings in AWS before reusing this design outside the lab.

## Implementation Summary

1. Created VPC A and VPC B with non-overlapping CIDR ranges.
2. Created the public subnet in VPC A and private subnets in VPC A and VPC B.
3. Created and attached the Internet Gateway to VPC A.
4. Created route tables and explicitly associated each subnet with the intended route table.
5. Created security groups and launched the bastion, App-A, and App-B EC2 instances.
6. Configured SSH access from the local workstation to the bastion and private instances through SSH `ProxyJump` entries.
7. Tested App-A to App-B connectivity before peering; the ping failed as expected.
8. Created and activated the VPC peering connection.
9. Added peering routes to the route tables in both VPCs and the public route table used by the bastion subnet.
10. Verified ping connectivity in both directions and SSH access to App-B.
11. Removed App-B's inbound ICMP rule temporarily to produce a controlled failure, restored the rule, and verified recovery.

## Validation and Test Results

| Test | Expected result | Observed result |
|---|---|---|
| App-A to App-B ping before peering | Fail because no peering route exists | **Passed** — 100% packet loss in baseline test |
| VPC peering status | Active | **Passed** — status shown as Active |
| App-A to App-B ping after routing | Succeed | **Passed** — 4 packets received, 0% loss |
| App-B to App-A ping after routing | Succeed | **Passed** — 4 packets received, 0% loss |
| SSH from workstation to bastion | Connect | **Passed** |
| SSH from workstation to App-A via bastion | Connect | **Passed** |
| SSH from workstation to App-B via bastion | Connect | **Passed** |
| Controlled ICMP failure after removing App-B ICMP rule | Ping fails | **Passed** — 100% packet loss while the rule was absent |
| Recovery after restoring App-B ICMP rule | Ping succeeds again | **Passed** |

The results demonstrate that peering alone is not sufficient: the connection must be active, appropriate routes must exist on both sides, and security rules must allow the intended traffic.

## Screenshots and Evidence

The following screenshots are included in the [`screenshots/`](screenshots/) folder. They are embedded below so GitHub can display the evidence directly in this README. Click an image to open it at full size.

### 1. VPCs and subnets

**VPC A details**

![VPC A details](screenshots/01-VPC-A-Details.png)

**VPC B details**

![VPC B details](screenshots/02-VPC-B-Details.png)

**Public subnet in VPC A**

![Public subnet A](screenshots/03-Subnet-A-Public.png)

**Private subnet in VPC A**

![Private subnet A](screenshots/04-Subnet-A-Private.png)

**Private subnet in VPC B**

![Private subnet B](screenshots/05-Subnet-B-Private.png)

### 2. Initial route tables and subnet associations

**Public route table A**

![Public route table A](screenshots/06-Route-Table-A-Public.png)

**Public subnet association**

![Public route table association](screenshots/07-Association-RT-A-Public.png)

**Private route table A**

![Private route table A](screenshots/08-Route-Table-A-Private.png)

**Private subnet A association**

![Private subnet A association](screenshots/09-Association-RT-A-Private.png)

**Private route table B**

![Private route table B](screenshots/10-Route-Table-B-Private.png)

**Private subnet B association**

![Private subnet B association](screenshots/11-Association-RT-B-Private.png)

### 3. Security groups and EC2 instances

**Bastion security group inbound rules**

![Bastion security group inbound rules](screenshots/12-SG-Bastion-Inbound.png)

**App-A security group inbound rules**

![App-A security group inbound rules](screenshots/13-SG-App-A-Inbound.png)

**App-B security group inbound rules**

![App-B security group inbound rules](screenshots/14-SG-App-B-Inbound.png)

**Bastion EC2 details**

![Bastion EC2 details](screenshots/15-EC2-Bastion-Details.png)

**App-A EC2 details**

![App-A EC2 details](screenshots/16-EC2-App-A-Details.png)

**App-B EC2 details**

![App-B EC2 details](screenshots/17-EC2-App-B-Details.png)

### 4. Initial access and baseline test

**SSH connection to the bastion host**

![SSH bastion success](screenshots/18-SSH-Bastion-Success.png)

**SSH connection to App-A through the bastion**

![SSH App-A success](screenshots/19-SSH-App-A-Success.png)

**Baseline ping failure before VPC peering**

![Baseline ping failure](screenshots/20-Baseline-Ping-Failure.png)

### 5. Peering and connectivity validation

**VPC peering connection — Active**

![Active VPC peering connection](screenshots/21-VPC-Peering-Connection-Active.png)

**Successful ping: App-A to App-B**

![App-A to App-B ping success](screenshots/22-App-A-to-App-B-Ping-Success.png)

**Successful ping: App-B to App-A**

![App-B to App-A ping success](screenshots/23-App-B-to-App-A-Ping-Success.png)

### 6. Controlled failure and recovery

**App-B security group during the controlled failure**

![App-B security group during failure](screenshots/24-App-B-SG-Inbound-Rules-During-Failure.png)

**App-A to App-B ping during the controlled failure**

![Controlled failure ping](screenshots/25-Controlled-Failure-App-A-to-App-B-Ping.png)

**App-B ICMP rule restored**

![App-B ICMP rule restored](screenshots/26-App-B-SG-Inbound-ICMP-Rule-Restored.png)

**Ping successful after recovery**

![Recovery ping success](screenshots/27-Recovery-App-A-to-App-B-Ping-Success.png)

### 7. Final route tables and App-B SSH

**Final public route table A**

![Final public route table A](screenshots/28-Route-Table-A-Public-Final-Routes.png)

**Private route table A with peering route**

![Private route table A peering route](screenshots/29-Route-Table-A-Private-Peering-Route.png)

**Private route table B with peering route**

![Private route table B peering route](screenshots/30-Route-Table-B-Private-Peering-Route.png)

**SSH connection to App-B through the bastion**

![App-B SSH success](screenshots/31-App-B-SSH-Success.png)

## Key Learnings

- VPC peering enables private connectivity between two VPCs with non-overlapping CIDR blocks.
- Peering connections do not automatically add routes; route tables must be configured on both sides.
- The route table associated with the bastion's subnet also needs a route to the destination VPC for the bastion to reach private instances across the peering connection.
- Security groups control which traffic is allowed to reach an instance, even when routing is correct.
- A baseline test, a controlled failure, and a recovery test help demonstrate that the configuration behaves as expected.
- SSH `ProxyJump` makes it possible to connect to private EC2 instances without assigning them public IP addresses.

## Cleanup

After completing the VPC Peering lab and capturing the required screenshots, all lab resources were successfully cleaned up to prevent ongoing AWS charges.

## Cleanup Tasks Completed

- [✓] Terminated the three EC2 instances: `ewb-bastion`, `ewb-app-a`, and `ewb-app-b`.
- [✓] Deleted the VPC peering connection.
- [✓] Confirmed that no NAT Gateway was required for the documented design.
- [✓] Detached and deleted the Internet Gateway.
- [✓] Deleted the custom route tables and subnets.
- [✓] Deleted both VPCs after removing their dependent resources.
- [✓] Reviewed Elastic IP addresses, snapshots, and other potentially billable resources.
- [✓] Reviewed the AWS billing console for unwanted remaining resources.

## Cleanup Status

**Status: Completed**

All lab resources were cleaned up after successfully validating:

- VPC peering connectivity between VPC A and VPC B.
- Bidirectional ICMP connectivity between App-A and App-B.
- SSH access to private instances through the bastion host.
- Controlled network failure and successful recovery.

---