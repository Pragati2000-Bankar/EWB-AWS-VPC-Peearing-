# AWS VPC Peering: EWB Cloud Labs Project 01

Connect two private VPCs with VPC peering and prove private connectivity with EC2, custom route tables, security groups and a bastion host.

- **Student:** Pragati Bankar
- **Region:** Asia Pacific (Mumbai), `ap-south-1`
- **Course:** EWB Courses, AWS Architect Associate

## Architecture

```
Laptop --SSH/22--> Internet gateway (ewb-igw-a)
                          |
 VPC A 10.10.0.0/16       |            VPC B 10.20.0.0/16 (no internet gateway)
 +--------------------------------+   +-----------------------------+
 | ewb-a-public  10.10.1.0/24     |   | ewb-b-private 10.20.1.0/24  |
 |   Bastion 10.10.1.10 (public)  |   |   App-B 10.20.1.10          |
 | ewb-a-private 10.10.2.0/24     |<->|                             |
 |   App-A 10.10.2.10             |  pcx (ewb-a-b-peer)             |
 +--------------------------------+   +-----------------------------+
```

- Administration: laptop to bastion to App-A or App-B using SSH `ProxyJump`.
- Project traffic: App-A and App-B talk over private IPs through the peering connection.

## Resources

| Resource | Name | Value |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public subnet | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet | `ewb-a-private` | `10.10.2.0/24` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private subnet | `ewb-b-private` | `10.20.1.0/24` |
| Internet gateway | `ewb-igw-a` | attached to VPC A |
| Route tables | `ewb-rt-a-public`, `ewb-rt-a-private`, `ewb-rt-b-private` | see below |
| Security groups | `ewb-sg-bastion`, `ewb-sg-app-a`, `ewb-sg-app-b` | see below |
| Peering | `ewb-a-b-peer` | requester VPC A, accepter VPC B |
| Key pair | `ewb-lab` | RSA, `.pem` |
| Instances | `ewb-bastion`, `ewb-app-a`, `ewb-app-b` | Amazon Linux 2023, `t3.micro` |

My actual resource IDs are in the report PDF (section 3).

## Routes

| Route table | Destination | Target |
|---|---|---|
| `ewb-rt-a-public` | `10.10.0.0/16` | local |
| | `0.0.0.0/0` | `ewb-igw-a` |
| | `10.20.0.0/16` | peering connection |
| `ewb-rt-a-private` | `10.10.0.0/16` | local |
| | `10.20.0.0/16` | peering connection |
| `ewb-rt-b-private` | `10.20.0.0/16` | local |
| | `10.10.0.0/16` | peering connection |

## Security group inbound rules

| Security group | Rule | Source |
|---|---|---|
| `ewb-sg-bastion` | SSH TCP 22 | your public IPv4 `/32` |
| `ewb-sg-app-a` | SSH TCP 22 | `10.10.1.10/32` |
| `ewb-sg-app-a` | ICMP Echo Request | `10.20.1.10/32` |
| `ewb-sg-app-b` | SSH TCP 22 | `10.10.1.10/32` |
| `ewb-sg-app-b` | ICMP Echo Request | `10.10.2.10/32` |

Outbound rules stay at the default (all traffic). Never open SSH or ICMP to `0.0.0.0/0`.

## How to build it

1. **VPC A:** create `ewb-vpc-a` (VPC only), two subnets in the same AZ, attach `ewb-igw-a`, create `ewb-rt-a-public` (with `0.0.0.0/0` to the IGW) and `ewb-rt-a-private`, and **explicitly associate** each subnet.
2. **VPC B:** create `ewb-vpc-b`, `ewb-b-private` and `ewb-rt-b-private`, and associate the subnet. No internet gateway.
3. **Security groups:** create the three groups in the correct VPCs with the rules above.
4. **Instances:** create key pair `ewb-lab`, then launch the three instances with the fixed private IPs. Only the bastion gets a public IP.
5. **SSH:** use the config below, then test `ssh ewb-bastion` and `ssh ewb-app-a`.
6. **Baseline:** from App-A run `ping -c 4 10.20.1.10`.
7. **Peering:** create `ewb-a-b-peer` (A requester, B accepter) and **accept** it until Active.
8. **Routes:** add the three peering routes from the table above.
9. **Test and clean up.**

### SSH config (`~/.ssh/config`)

```
Host ewb-bastion
    HostName BASTION_PUBLIC_IP
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

**Windows PowerShell:** there is no `chmod`. Restrict the key with:

```powershell
icacls "$HOME\.ssh\ewb-lab.pem" /inheritance:r
icacls "$HOME\.ssh\ewb-lab.pem" /grant:r "${env:USERNAME}:R"
```

The config file must be named `config` with no `.txt`. Update `HostName` if the bastion public IP changes after a stop/start.

## Tests

| # | Test | Expected | Result |
|---|---|---|---|
| 01 | App-A before peering: `ping -c 4 10.20.1.10` | no replies | 100% loss |
| 02 | App-A after routes: `ping -c 4 10.20.1.10` | 4 replies | 0% loss |
| 03 | App-B after routes: `ping -c 4 10.10.2.10` | 4 replies | 0% loss |
| 04 | `ssh ewb-app-b` from laptop | shell via bastion | shell obtained |
| 05 | App-A and App-B instance details | no public IPv4 | none |
| 06 | Remove ICMP rule on `ewb-sg-app-b`, ping from App-A, then restore | fails, then recovers | run from App-A and capture both screenshots |

## Troubleshooting

| Symptom | Check |
|---|---|
| Bastion SSH times out | your current public IP in `ewb-sg-bastion`, public subnet association, `0.0.0.0/0` route |
| `Permission denied (publickey)` | key path, `ec2-user`, `IdentityFile`; routing is fine if you reach this |
| App-B SSH fails | `10.20.0.0/16` in the public route table, return route in VPC B, SSH source `10.10.1.10/32` |

## Cost note

EC2 instances, EBS volumes and public IPv4 addresses can incur charges. VPCs, subnets, route tables, security groups, the internet gateway and a same-Region peering connection do not. No NAT gateway is used. Confirm current pricing on the AWS pricing pages.
