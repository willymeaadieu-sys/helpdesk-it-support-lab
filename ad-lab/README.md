


Active Directory Lab — Setup

Part of the [Help Desk & IT Support Lab](../README.md) portfolio project.

This section covers building a working Active Directory environment from scratch — domain controller promotion, organizational unit design, user provisioning, and security group configuration. This is the foundation used in the [Scenarios](../scenarios/README.md) section, where the environment is used to simulate real Tier 1 support tickets.

## Environment Overview

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| OS | Windows Server 2022 Standard (Desktop Experience) |
| Domain Controller | DC01 |
| Forest / Domain Name | helpdesk.lab |
| NetBIOS Name | HELPDESK |
| Static IP | 192.168.101.10 |
| DNS | Self-hosted (127.0.0.1) |
| Network Mode | Internal Network (isolated from host LAN) |

## 1. VM Provisioning

Created a new VirtualBox VM (`DC01-WinServer2022`) with 8GB RAM, 4 vCPUs, and a 60GB dynamically allocated disk. Network adapter set to **Internal Network** to keep the domain controller isolated from the host's home network.

**Troubleshooting note:** initial attempt used VirtualBox's unattended guest OS installation feature, which failed with a `Windows cannot find the Microsoft Software License Terms` error — a known compatibility issue between unattended install answer files and Windows Server evaluation ISOs. Resolved by recreating the VM and performing a standard manual installation instead.

(<img width="1016" height="802" alt="VM setup error and fix" src="https://github.com/user-attachments/assets/191c2dfd-8e74-43ed-844e-9264260c4602" />)

## 2. Windows Server Installation

Installed Windows Server 2022 Standard Evaluation (Desktop Experience) via manual setup — language/edition selection, license acceptance, custom install to the primary virtual disk.

(<img width="1232" height="959" alt="Server Manager" src="https://github.com/user-attachments/assets/ff4729b6-b975-4781-b198-9c5ac6e323d1" />
)

## 3. Initial Server Configuration

- Renamed the server from its auto-generated name (`WIN-FLEOO4V775N`) to `DC01`
- Configured a static IPv4 address (192.168.101.10 / 255.255.255.0) with DNS pointed at itself (127.0.0.1), since this server would become its own DNS provider once promoted

(<img width="1244" height="952" alt="Computer rename dialog" src="https://github.com/user-attachments/assets/b2c5b416-19b4-4020-8e9d-f6faa6ac6fa7" />
)

## 4. Active Directory Domain Services Installation

Installed the AD DS role via Server Manager's Add Roles and Features wizard, then promoted the server to a domain controller for a new forest:

- **Forest / root domain:** helpdesk.lab
- **Domain/Forest functional level:** Windows Server 2016
- **DNS Server role:** installed alongside AD DS (self-hosted DNS)

(<img width="1256" height="952" alt="AD DS and DNS roles installed" src="https://github.com/user-attachments/assets/e34d63fc-eee0-41d8-8b54-ea2f416fdf46" />
)

## 5. Post-Promotion Verification

Confirmed the domain controller was fully operational using both `ipconfig /all` and `Get-ADDomain`:

- Static IP persisted correctly after reboot, DHCP disabled
- DNS suffix and hostname correctly reflect `DC01.helpdesk.lab`
- `Get-ADDomain` confirms DC01 holds all FSMO roles (PDC Emulator, RID Master, Infrastructure Master) — expected for a single-DC forest

(<img width="1438" height="724" alt="ipconfig confirmation" src="https://github.com/user-attachments/assets/3372c8ae-b951-45e6-99d2-3e4483e05057" />)
(<img width="1378" height="868" alt="Get-ADDomain output" src="https://github.com/user-attachments/assets/2bd8ede0-95af-4c0f-8fa9-088e07b93177" />)

## 6. Organizational Unit Structure

Built an OU structure modeled on a small company with three departments, keeping computer objects and service accounts separated from user accounts:

```
helpdesk.lab
├── Employees
│   ├── IT
│   ├── Sales
│   └── HR
├── ComputersHD
└── Service Accounts
```

(<img width="1128" height="786" alt="OU structure tree" src="https://github.com/user-attachments/assets/7516bac3-5eb3-4de8-87f6-3b461eac9b65" />
)

## 7. User Provisioning

Created five test user accounts distributed across departments to simulate a real employee directory:

| Name | Username | Department (OU) |
|---|---|---|
| John Martinez | jmartinez | IT |
| Sarah Chen | schen | IT |
| Mike Johnson | mjohnson | Sales |
| Lisa Rodriguez | lrodriguez | Sales |
| Amanda Foster | afoster | HR |

(<img width="1114" height="680" alt="OU structure IT" src="https://github.com/user-attachments/assets/7fec4185-3b7d-4da0-a6e2-fe9943735cf8" />)
<img width="1122" height="672" alt="OU structre HR" src="https://github.com/user-attachments/assets/47f5c5d9-6ed7-4671-8494-57c71f49042c" />
<img width="1126" height="668" alt="OU structure Sales" src="https://github.com/user-attachments/assets/5850bc91-12ec-44f9-a635-ca900502e245" />

## 8. Security Groups

Created department-aligned security groups (Global scope, Security type) and assigned matching users to each, to support permission-based access control scenarios later:

- **IT-Staff** — John Martinez, Sarah Chen
- **Sales-Staff** — Mike Johnson, Lisa Rodriguez
- **HR-Staff** — Amanda Foster

(<img width="1124" height="786" alt="OU structure Security groups" src="https://github.com/user-attachments/assets/87caf94a-5163-492f-b30b-9293d935b863" />
)
(<img width="1124" height="778" alt="OU Structure security groups IT" src="https://github.com/user-attachments/assets/727c771e-ae23-4841-83ca-db143e4dd8fc" />
)


