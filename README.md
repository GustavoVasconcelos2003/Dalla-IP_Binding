# Dalla-IP_Binding

A Bash-based automation tool for provisioning OpenVPN users, associating them with the appropriate client tunnel, generating credentials, configuring access control and, when required, automatically assigning a fixed VPN IP.

## Project Origin

This project originated from a real operational need in a support environment where OpenVPN users had to be created manually.

The original script already automated most of the user provisioning process, including tunnel selection, Linux user creation, password generation, access control and logging.

However, some OpenVPN clients required users to have a **manually assigned/fixed VPN IP**.

In those environments, finding an available IP was a manual process. The administrator had to inspect the existing CCD configuration files and search for `ifconfig-push` entries until an unused address was found.

This project was improved to automate that process.

The updated logic detects whether the selected tunnel uses fixed IPs, identifies the tunnel's subnet, checks which addresses are already assigned and automatically suggests the first available IP.

---

# How It Works

The script can be understood as a sequence of provisioning steps:

```text
User Input
    │
    ▼
Identify Client
    │
    ▼
Find OpenVPN Tunnel
    │
    ▼
Select Tunnel
    │
    ▼
Create Linux User
    │
    ▼
Generate Password
    │
    ▼
Add User to allowusers
    │
    ▼
Check IP Assignment Model
    │
    ├───────────────┐
    │               │
 Dynamic Tunnel   Fixed IP Tunnel
    │               │
    ▼               ▼
   Done       Find Available IP
                    │
                    ▼
              Ask for Confirmation
                    │
                    ▼
              Create CCD file
                    │
                    ▼
              Assign Fixed IP
```

---

## 1. User Input

The process starts by asking for the company/client and username.

The expected naming convention is:

```text
EMPRESA.USER
```

For example:

```text
DALLA.FULANO
```

The company prefix is used to identify the OpenVPN tunnel associated with that client.

---

## 2. Finding the OpenVPN Tunnel

The script searches `/etc/openvpn/ccd` for tunnels matching the company name.

For example, if the user enters:

```text
DALLA.FULANO
```

the script uses the `DALLA` portion to locate the corresponding tunnel configuration.

If multiple tunnels are found, the administrator is asked to select the correct one.

If no tunnel is found, the script stops and asks the administrator to validate the information provided.

This prevents the user from being created against an unknown client environment.

---

## 3. Creating the Linux User

Once the tunnel is selected, the script creates the Linux account used for VPN authentication.

A random password is generated using `pwgen` and the account is created without a regular login shell:

```bash
adduser $nome -M -s /sbin/nologin
```

The generated credentials are displayed to the administrator.

The script also records the provisioning event in:

```text
/var/log/logins
```

The log includes information such as:

* Date and time
* Working directory
* Created username
* Selected client/tunnel

---

## 4. Configuring OpenVPN Access

After creating the Linux user, the username is added to the client's `allowusers` file:

```text
/etc/openvpn/ccd/TUN-CLIENTE/allowusers
```

This file is used to control which users are allowed to authenticate through that OpenVPN tunnel.

At this point, the user is already created and authorized for the selected client.

The next step depends on how the tunnel handles IP addresses.

---

# Fixed IP Detection

## 5. Dynamic vs. Fixed IP Tunnels

Not every OpenVPN tunnel requires manually assigned IP addresses.

The script distinguishes between two scenarios.

### Dynamic IP Tunnel

If the client's CCD directory contains only the expected access-control configuration, the tunnel is treated as dynamic.

In this scenario, OpenVPN manages the client address dynamically.

No individual CCD file is created for the user.

```text
/etc/openvpn/ccd/TUN-CLIENTE/

└── allowusers
```

The provisioning process can therefore finish without manually assigning an IP.

---

### Fixed IP Tunnel

Some clients require every VPN user to have a predefined IP address.

In these environments, the CCD directory contains individual user configuration files in addition to `allowusers`.

For example:

```text
/etc/openvpn/ccd/TUN-CLIENTE/

├── allowusers
├── cliente.joao
├── cliente.maria
└── cliente.pedro
```

Each user configuration contains an `ifconfig-push` directive:

```text
ifconfig-push 10.47.16.148 255.255.255.0
```

The existence of these individual configuration files indicates that the tunnel uses fixed IP assignments.

---

# Automatic IP Assignment

## 6. Identifying the Tunnel Network

When a fixed-IP tunnel is detected, the script locates the corresponding OpenVPN server configuration:

```text
/etc/openvpn/server/TUN-CLIENTE.conf
```

It reads the `server` directive to identify the tunnel network and subnet mask.

For example:

```text
server 10.47.16.0 255.255.255.0
```

From this information, the script determines the address range that can be used for client assignments.

The current automatic assignment logic supports `/24` networks:

```text
255.255.255.0
```

If another subnet size is detected, the script does not attempt to guess the correct range and instead instructs the administrator to perform the assignment manually.

---

## 7. Finding Addresses Already in Use

The script scans the existing CCD configuration files and extracts addresses assigned through `ifconfig-push`.

For example:

```text
ifconfig-push 10.47.16.148 255.255.255.0
ifconfig-push 10.47.16.120 255.255.255.0
ifconfig-push 10.47.16.121 255.255.255.0
```

The assigned host portions are extracted and organized numerically.

The script then checks the available host range and identifies addresses that are not currently assigned.

---

## 8. Selecting an Available IP

The script evaluates the available addresses in the supported range and builds a list of unused IPs.

For example, if:

```text
10.47.16.120 → used
10.47.16.121 → used
10.47.16.122 → free
10.47.16.123 → free
```

the script suggests:

```text
10.47.16.122
```

as the next available address.

The administrator is shown the available addresses and the suggested IP before anything is written.

---

## 9. Confirming the Assignment

The script does not automatically assign the address without confirmation.

The administrator is asked:

```text
Confirma a fixação deste IP para USER? (s/n):
```

If confirmed, the script creates the user's CCD configuration file:

```text
/etc/openvpn/ccd/TUN-CLIENTE/EMPRESA.USER
```

with:

```text
ifconfig-push 10.47.16.122 255.255.255.0
```

This causes OpenVPN to assign the selected fixed address to that user.

If the administrator declines, no IP configuration is written and the script warns that the address must be assigned manually before the user connects.

---

# Complete Example

A simplified execution flow looks like this:

```text
Administrator
     │
     ▼
Enter: CLIENT.USER
     │
     ▼
Find matching OpenVPN tunnel
     │
     ▼
Select tunnel if necessary
     │
     ▼
Create Linux VPN account
     │
     ▼
Generate random password
     │
     ▼
Add USER to allowusers
     │
     ▼
Does this tunnel use fixed IPs?
     │
     ├── NO ──► Finish
     │
     └── YES
           │
           ▼
      Read TUN-CLIENTE.conf
           │
           ▼
      Identify subnet
           │
           ▼
      Scan existing CCD files
           │
           ▼
      Find assigned IPs
           │
           ▼
      Calculate free IPs
           │
           ▼
      Suggest available IP
           │
           ▼
      Administrator confirms
           │
           ▼
      Create USER CCD file
           │
           ▼
      Write ifconfig-push
           │
           ▼
          Done
```

---

# My Contribution

The original project was developed collaboratively and already automated the main OpenVPN user provisioning workflow.

My contribution was focused on improving the **fixed-IP provisioning process**.

Previously, when a client required manually assigned VPN addresses, the administrator had to inspect the existing CCD configuration files individually to determine which IPs were already in use.

The updated logic automates this process by:

* Detecting when a tunnel uses fixed IP assignments;
* Locating the corresponding OpenVPN server configuration;
* Reading the tunnel network automatically;
* Extracting currently assigned addresses from the CCD files;
* Calculating available addresses;
* Suggesting an available IP;
* Asking for administrator confirmation;
* Creating the user's CCD configuration automatically.

The goal was to eliminate a repetitive manual operation while keeping the administrator in control of the final IP assignment.

---

# Technologies

* Bash
* Linux
* OpenVPN
* OpenSSL
* `pwgen`
* Linux user management
* OpenVPN CCD configuration
* Shell text processing (`grep`, `awk`, `sed`, `cut`, etc.)

# Project Context

This project was developed for use in an environment with multiple OpenVPN client tunnels and different IP assignment requirements.

The same provisioning workflow can therefore handle both:

* **Dynamic tunnels**, where OpenVPN assigns addresses automatically;
* **Fixed-IP tunnels**, where each user requires an individual CCD configuration.
