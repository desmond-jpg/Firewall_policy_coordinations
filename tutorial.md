
# Firewall Policy Coordination Across Rocky Linux(Firewalld) and Ubuntu 24.04(Ufw)

Performing configuration of firewall across different systems like Rocky Linux using the firewalld and Ubuntu 24.04 using the ufw(UNCOMPLICATED FIREWALL) requires using specific commands syntax for each tool. Coordinated policy means applying equivalent rules across both systems, Than a direct synchronization of commands. Traffic shaping and advanced routing require modifying the underlying(iptables/nftables) rules directly, which goes beyong the standart functionaly of firewalld and ufw.

This guide provides a complete step-by-step setup for coordinating firewall policies between Rocky Linux (firewalld) and Ubuntu 24.04 (UFW). It includes synchronized rules, port forwarding (NAT), traffic shaping, IP restrictions, and logging—ensuring both systems enforce consistent security standards.

## Why Firewall policy coordination

Firewall policy cooordination configuration perfom the following:

- **Consistent Security** - Ensures both systems enforce the same security rules, reducing vulnerabilities.
- **Efficient Management** - Simplifies firewall management by applying rules across multiple systems.
- **Compliance** - Helps meet regulatory requirements by maintaining uniform security policies.
- **Reduced Risk** - Minimizes misconfigurations and unauthorized access by standardizing rules.

## Prerequisites

- Two systems: Rocky Linux with firewalld and Ubuntu 24.04 with UFW.
- Root or sudo privileges on both systems.
- Basic familiarity with Linux commands and firewall concepts.

### step 1: Configure firewalld on Rocky Linux(Central Firewall)

Firewalld is the default firewall management tool on Rocky Linux. It uses dynamic zones and rules to control incoming and outgoing network traffic. As the central firewall, Rocky Linux will manage core security policies, handle port access, and control traffic flowing between internal and external networks. This setup ensures a consistent and secure foundation for all connected systems.

#### 1.1 Install and enable firewalld(Rocky Linuxe)

Firewalld is the firewall service that manages network traffic and security rules on Rocky Linux. Before applying any policies, we have to ensure the service is installed, enabled, and running so it can control and enforce your firewall configurations.

```bash
# Install firewalld
sudo dnf install firewalld -y
# Enable and start firewalld
sudo systemctl enable --now firewalld
# Check the status of firewalld
sudo systemctl status firewalld
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2016-28-11.png)

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2016-56-33.png)

### step 2: Coordinated Rules(Rocky Linux)

We need to configure firewalld on Rocky Linux to match the security policies used across your environment. Setting coordinated rules ensures that Rocky Linux applies the same allowed services, ports, and access restrictions as the other systems, creating a consistent and predictable firewall setup.

#### 2.1 Allow common services

To ensure essential services work properly, allow common network services such as SSH, HTTP, and HTTPS through the firewall. This ensures secure remote access and web traffic while maintaining the overall security of the system.

```bash
# Allow SSH, HTTP, and HTTPS
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
# Reload the firewall to apply changes
sudo firewall-cmd --reload
sudo systemctl status firewalld
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2017-01-00.png)

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2017-21-12.png)

#### 2.2 Open custom ports

To enable specific applications or services, open custom ports on the firewall. This allows traffic to pass through the specified ports while maintaining security by only allowing necessary communication.

```bash
# Open custom ports
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=9000/udp
# Reload the firewall to apply changes
sudo firewall-cmd --reload
```

#### 2.3 Restrict access to specific IPs

To tighten security, we can limit access to certain services so that only trusted systems are allowed to connect. In this case, SSH access is restricted to a single Ubuntu machine by removing the general SSH rule and allowing it only from the specified IP address.

```bash
# Restrict SSH access to a specific IP
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.20" port protocol="tcp" port="22" accept'
sudo firewall-cmd --reload
```

### step 3: Port Forwarding(Network Address Translation) on Rocky Linux

Port forwarding allows external traffic on a specific port to be redirected to an internal system. This is useful when services hosted on the Ubuntu machine need to be accessed through the Rocky Linux firewall. By enabling masquerading and creating a forward-port rule, incoming traffic on the chosen port is securely forwarded to the Ubuntu server.

```bash
# Enable masquerading
sudo firewall-cmd --permanent --add-masquerade
```

Add forward-port rule

```bash
# Add forward-port rule
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" forward-port port="8080" protocol="tcp" to-port="80" to-addr="192.168.10.20"'
# Reload the firewall to apply changes
sudo firewall-cmd --reload
```

### step 4: Traffic Shaping on Rocky Linux

Traffic shaping allows you to control how much bandwidth specific services or ports can use. This helps prevent network congestion, ensures fair resource distribution, and protects critical services from being slowed down by heavy traffic. Using the tc command, we can set bandwidth limits and prioritize certain types of network traffic.Limit port 80 traffic to 5Mbps
ensure the interface name matches your system(check trough ip a)

```bash
# Limit port 80 traffic to 5Mbps
sudo tc qdisc add dev ens18 root handle 1: htb default 12
sudo tc class add dev ens18 parent 1: classid 1:12 htb rate 5mbit
sudo tc filter add dev ens18 protocol ip parent 1:0 prio 1 u32 match ip dport 80 0xffff flowid 1:12
```

### step 5: Configure UFW on Ubuntu 24.04

UFW (Uncomplicated Firewall) is the default firewall tool on Ubuntu, designed to make managing network rules simple and effective. Before applying security policies, ensure UFW is installed, enabled, and properly configured to control incoming and outgoing traffic on the system.

#### 5.1 Install and enable

We need to install and enable ufw on the Ubuntu machine.

```bash
# Install ufw
sudo apt install ufw -y 
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2017-43-52.png)

First we need to allow ssh before enabling ufw to avoid ssh be blocked

```bash
sudo ufw allow ssh
# Or explicitly with the port
sudo ufw allow 22/tcp
sudo ufw status
sudo ufw enable
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2018-19-10.png)

### step 6: Coordinates Rules - Ubuntu

To maintain consistent security across your systems, Ubuntu must follow the same firewall policies as Rocky Linux. By applying coordinated rules in UFW, you ensure both servers allow, block, and control traffic in a uniform way, creating a predictable and secure network environment.

#### 6.1 Allow common service

We need to allow common services such as SSH, HTTP, and HTTPS, to ensure the Ubuntu system can communicate securely and function properly while still being protected by the firewall.

```bash
# Allow common services
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
```

#### 6.2 Open custom ports

Some applications require ports that are not open by default. Allowing custom ports in UFW ensures these services can communicate while the system remains protected by controlled firewall rules

```bash
# Open custom ports
sudo ufw allow 8080/tcp
sudo ufw allow 9000/udp
sudo ufw status
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2018-27-47.png)

#### 6.3 Restrict access to specific IPs

To improve security, we can limit SSH access so that only a trusted machine is allowed to connect. The rule below allows SSH only from the specified Rocky Linux IP address and denies all other connections to port 22.

```bash
# Restrict access to specific IPs # your actual ip address
sudo ufw allow from 192.168.10.10 to any port 22 proto tcp
sudo ufw deny 22/tcp
```

### step 7: Port forwarding(Network Address Translation) on Ubuntu

Port forwarding allows traffic arriving at a specific port on the Ubuntu system to be redirected to another internal host or service. By configuring NAT - Network Address Translation rules and enabling IP forwarding, you can securely forward external requests to internal servers, ensuring services are accessible while maintaining firewall control.

#### 7.1 Edit UFW Network Address Translation Rules

To configure port forwarding on Ubuntu, you need to modify UFW’s NAT rules. Editing the `/etc/ufw/before`, rules file allows you to define how incoming traffic on specific ports should be redirected to internal hosts, enabling controlled and secure access to services.

```bash
# Edit UFW Network Address Translation Rules
sudo nano /etc/ufw/before.rules
```

Add the following rules to the file

```ruby

*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# Forward port 8080 → 192.168.10.10:80
-A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.10.10:80
-A POSTROUTING -j MASQUERADE

COMMIT
```

#### 7.2 Enable IPv4 forwarding

Enabling IPv4 forwarding allows the Ubuntu system to act as a router, forwarding traffic between networks. This is essential for NAT and port forwarding to function correctly, ensuring that redirected traffic can reach its intended destination.

Edit

```bash
# Enable IPv4 forwarding
sudo nano /etc/ufw/sysctl.conf
```

uncomments:

```bash
net/ipv4/ip_forward=1
```

#### 7.3 Allowing forwarding in ufw

To enable forwarding in UFW, we need to modify the default forward policy. By setting the `DEFAULT_FORWARD_POLICY` to "ACCEPT", you allow the system to forward traffic between networks, which is necessary for NAT and port forwarding to work properly.

Edit

```bash
sudo nano /etc/default/ufw
```

set

```ini
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Reload ufw

```bash
# Reload ufw to apply the changes   
sudo ufw reload
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2020-28-20.png)

### step 8: Traffic Shaping on Ubuntu

Traffic shaping is a technique used to control the amount of bandwidth used by specific types of traffic. This is particularly useful for managing network resources and ensuring that critical services receive the necessary bandwidth while limiting less important traffic.

limit port 80 traffic to 5Mbps,
ensure the interface name matches your system(check through ip a)

```bash
# Traffic shaping on Ubuntu
sudo tc qdisc add dev ens33 root handle 1: htb default 12
sudo tc class add dev ens33 parent 1: classid 1:12 htb rate 5mbit
sudo tc filter add dev ens33 protocol ip parent 1:0 prio 1 u32 match ip dport 80 0xffff flowid 1:12
```

### step 10 Logging

By enabling firewall logging its allows you to monitor denied or allowed connections and track potential security threats. With firewalld, logging provides visibility into network activity, helping with troubleshooting, auditing, and security analysis.

On Rocky Linux

```bash
# Enable logging for firewalld
sudo firewall-cmd --set-log-denied=all
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2020-53-22.png)
On Ubuntu

By enabling UFW logging its allows you to track allowed and denied network connections. This helps monitor traffic patterns, detect unauthorized access attempts, and troubleshoot firewall-related issues.

```bash
# Enable logging for UFW
sudo ufw logging medium
```

### step 10: Verification

#### Rocky Linux

We need to verify that the firewall rules are correctly applied and that the traffic is being forwarded as expected.

```bash
# Verify firewall rules
sudo firewall-cmd --list-all
```

```bash
sudo firewall-cmd  --state
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2020-56-26.png)

#### Ubuntu

We need to verify that the firewall rules are correctly applied and that the traffic is being forwarded as expected.

```bash
# Verify firewall rules
sudo ufw status verbose
```

sample output

![screenshot](images.png/Screenshot%20from%202025-12-14%2020-58-17.png)

Conclusion

We have now successfully configure the firewall policy coordination , By configuring coordinated firewall rules on Rocky Linux and Ubuntu 24.04, we have  establish a unified and secure network environment across diffferent systems. Both firewalld and UFW ultimately control nftables, so aligning their policies ensures consistent access control, synchronized port forwarding, unified logging, and predictable traffic behavior.
