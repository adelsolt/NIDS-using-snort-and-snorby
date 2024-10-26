### Network Intrusion Detection with Snort and Snorby

This project implements Snort for real-time intrusion detection in a virtualized environment. Snorby provides a dashboard for monitoring and managing Snort alerts. The setup uses VMware Workstation with several VMs simulating a network environment for detecting and analyzing security events.
Project Overview

This virtualized project environment sets up a network of VMs to monitor and respond to simulated attacks. Snort is configured on a Debian VM as a Network Intrusion Detection System (NIDS), while Snorby provides a web-based interface for real-time analysis. The environment allows traffic monitoring across multiple network nodes to simulate real-world scenarios and track suspicious activity from a simulated attacker VM.
Architecture and Infrastructure
## Infrastructure Components

    Debian 12 VM: Hosts Snort and Snorby.
        Snort: Monitors network traffic and detects potential intrusions.
        Snorby: Displays Snort alerts on a web interface.
    CentOS 9 VM: Acts as a web/application server to generate normal network traffic.
    CentOS 7 VM: Runs Cloudera to simulate data-intensive operations.
    Kali Linux VM: Simulates an attacker, performing network scans and attacks.

## Network Architecture

    Custom VMware Network (VMnet2): Isolates the environment, simulating a local network.
    IP Configuration: Each VM is assigned a static IP within the subnet (192.168.2.0/24).
    Promiscuous Mode: Enabled on the Debian VM interface to monitor all traffic within VMnet2.

## Project Configuration

### VM Roles and IPs

    Debian VM (Snort + Snorby): 192.168.2.10
    CentOS 9 VM: 192.168.2.11
    CentOS 7 VM: 192.168.2.12
    Kali Linux VM: 192.168.2.13

### Commands for Snort CLI Monitoring and Snorby
#### Snort CLI Monitoring

##### Run Snort to start monitoring traffic:

```
sudo snort -i eth0 -c /etc/snort/snort.conf -A console
```

#### Testing the Setup
##### Simulating Attacks with Kali Linux

    Network Scans (using Nmap)
```
sudo nmap -sS 192.168.2.10

```
##### Verifying Alerts

```
tail -f /var/log/snort/alert

```

