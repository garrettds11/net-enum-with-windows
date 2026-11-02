For a small cloud environment, the simplest approach is to inventory **identity, network configuration, routes, DNS, active connections, and reachable neighbors** from each Windows server or admin workstation. The native commands below produce most of what you need for a first-pass diagram.

## Core discovery commands

| What you need to map | Command |
| --- | --- |
| Hostname and full system inventory | `hostname` and `systeminfo` |
| IPs, subnets, gateways, DNS, DHCP | `ipconfig /all` |
| Network adapters and MAC addresses | `getmac /v /fo list` |
| Routing paths and default routes | `route print` |
| Neighbor and ARP relationships | `arp -a` |
| Active listeners and connections | `netstat -abno` |
| DNS resolution | `nslookup <name>` |
| Path to another system | `tracert <name-or-ip>` |
| Reachability | `ping <name-or-ip>` |
| Windows network path diagnostics | `pathping <name-or-ip>` |
| Local and domain identity | `whoami /all` |
| Domain and trust information | `nltest /dsgetdc:<domain>` and `nltest /domain_trusts` |
| Shared folders | `net share` |
| Mapped SMB connections | `net use` |
| Running services | `sc query` |
| Firewall profiles and rules | `netsh advfirewall show allprofiles` |
| Proxy configuration | `netsh winhttp show proxy` |

## Best native PowerShell commands

These are easier to filter and export than older command-line utilities.

### 1. Capture the host and operating system

Get-ComputerInfo

Get-CimInstance Win32_ComputerSystem

Get-CimInstance Win32_OperatingSystem

### 2. Capture interfaces, addresses, DNS, and routes

Get-NetAdapter

Get-NetIPConfiguration

Get-NetIPAddress

Get-DnsClientServerAddress

Get-NetRoute

A compact view for diagramming:

Get-NetIPConfiguration |

Select-Object InterfaceAlias, InterfaceDescription,

@{N='IPv4';E={$*.IPv4Address.IPAddress}},*

*@{N='Gateway';E={$*.IPv4DefaultGateway.NextHop}},

@{N='DNS';E={$_.DNSServer.ServerAddresses}}

### 3. Find neighboring systems

Get-NetNeighbor

Get-NetNeighbor -AddressFamily IPv4 |

Where-Object State -ne 'Unreachable'

This is useful for discovering systems visible on the same Layer 2 network. In a cloud environment, however, the cloud network may suppress traditional ARP visibility, so an empty or short list does not imply that no other resources exist.

### 4. Identify active dependencies

Get-NetTCPConnection

Get-NetUDPEndpoint

Focus on established outbound connections and listening services:

Get-NetTCPConnection |

Where-Object State -eq Established |

Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort,

OwningProcess

Get-NetTCPConnection |

Where-Object State -eq Listen |

Select-Object LocalAddress, LocalPort, OwningProcess

Resolve the owning process:

Get-NetTCPConnection |

Where-Object State -eq Established |

ForEach-Object {

[pscustomobject]@{

LocalEndpoint = "$($*.LocalAddress):$($*.LocalPort)"

RemoteEndpoint = "$($*.RemoteAddress):$($*.RemotePort)"

Process = (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).Name

}

}

Run PowerShell as administrator for the most complete process and connection details.

### 5. Capture firewall exposure

Get-NetFirewallProfile

Get-NetFirewallRule -Enabled True

Get-NetFirewallPortFilter

To associate enabled inbound rules with ports:

Get-NetFirewallRule -Enabled True -Direction Inbound |

Get-NetFirewallPortFilter |

Select-Object Protocol, LocalPort, RemotePort

### 6. Test known relationships

Test-Connection server01

Test-NetConnection server01 -Port 443

Test-NetConnection database01 -Port 1433

Resolve-DnsName server01

`Test-NetConnection` is particularly useful when validating the arrows in a diagram:

Test-NetConnection database01 -Port 1433 -InformationLevel Detailed

## Minimal collection script

Run this from each Windows VM and save the outputs to a folder:

$Out = Join-Path $env:USERPROFILE "Desktop\NetworkMap-$env:COMPUTERNAME"

New-Item -ItemType Directory -Path $Out -Force | Out-Null

Get-ComputerInfo |

Out-File "$Out\01-ComputerInfo.txt"

Get-NetAdapter |

Format-List * |

Out-File "$Out\02-Adapters.txt"

Get-NetIPConfiguration |

Format-List * |

Out-File "$Out\03-IPConfiguration.txt"

Get-NetIPAddress |

Export-Csv "$Out\04-IPAddresses.csv" -NoTypeInformation

Get-NetRoute |

Export-Csv "$Out\05-Routes.csv" -NoTypeInformation

Get-DnsClientServerAddress |

Export-Csv "$Out\06-DNS.csv" -NoTypeInformation

Get-NetNeighbor |

Export-Csv "$Out\07-Neighbors.csv" -NoTypeInformation

Get-NetTCPConnection |

Export-Csv "$Out\08-TCPConnections.csv" -NoTypeInformation

Get-NetUDPEndpoint |

Export-Csv "$Out\09-UDPEndpoints.csv" -NoTypeInformation

Get-NetFirewallProfile |

Export-Csv "$Out\10-FirewallProfiles.csv" -NoTypeInformation

Get-NetFirewallRule -Enabled True |

Export-Csv "$Out\11-FirewallRules.csv" -NoTypeInformation

## Turn the output into a diagram

For each asset, capture:

- **Name and role:** web server, jump host, domain controller, database
- **Cloud boundary:** tenant/account, subscription/project, region
- **Network boundary:** virtual network/VPC and subnet
- **Addresses:** private IP, public IP, subnet, gateway
- **Interfaces:** adapter and MAC address
- **Inbound exposure:** listening ports and firewall rules
- **Outbound dependencies:** established remote IPs and ports
- **Infrastructure:** DNS servers, domain controllers, proxies, VPNs
- **Routes:** default gateway and custom routes
- **Trust boundaries:** internet, management, application, and data tiers

A practical layout is:

Cloud/account

Region

Virtual network

Subnet

VM or service

IP address

Role

Listening ports

Dependencies

## Important limitation

Windows commands provide the **host’s view**, not the cloud control plane’s complete view. They typically will not reliably reveal disconnected VMs, managed databases, load balancer configuration, security groups, network ACLs, peering, NAT gateways, private endpoints, or cloud routing objects.

Use the native cloud CLI alongside Windows discovery for completeness:

- Azure: `az network vnet list`, `az network nic list`, `az network nsg list`
- AWS: `aws ec2 describe-vpcs`, `describe-subnets`, `describe-instances`, `describe-security-groups`
- Google Cloud: `gcloud compute networks list`, `subnets list`, `instances list`

The safest workflow is to use Windows commands for **observed connectivity** and the cloud CLI or portal for **configured topology**, then reconcile the two in your diagram.