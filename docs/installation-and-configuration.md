
# Installation and Configuration

## 1. Overview

This document describes the installation and initial configuration of
the Wazuh security monitoring lab. The lab uses an Ubuntu virtual
machine as the Wazuh server and a Windows 10 virtual machine as the
monitored endpoint.

The Windows 10 endpoint is enrolled as `WINDOWS-10-HOST` (agent ID
`002`). The Wazuh dashboard is accessed from the lab network.

> **Documentation note:** The commands below distinguish between the
> server-side setup and Windows agent onboarding. The Wazuh server and
> endpoint have already been brought online in this lab. Where an
> installer version or package filename can vary, verify the installed
> version and use a matching agent package rather than assuming a
> version.

## 2. Lab Environment

  Component                 Configuration
  ------------------------- -------------------------------
  Host operating system     Windows 11
  Virtualization platform   Oracle VirtualBox 7.2.12
  Wazuh server VM           Ubuntu 24.04.4 LTS
  Wazuh server address      `10.0.0.7`
  Monitored endpoint        Windows 10 Home, version 22H2
  Windows build             `19045.2965`
  Windows agent name        `WINDOWS-10-HOST`
  Windows agent ID          `002`
  Virtual network           VirtualBox NAT Network
  Network range             `10.0.0.0/24`
  Dashboard access          `https://10.0.0.7`

The Windows 10 agent communicates with the Wazuh server over the lab
network. Confirm the current IP addresses in the virtual machines before
repeating the setup, as DHCP or network changes can alter an address.

## 3. Virtual Machine Preparation

### 3.1 Prepare the Ubuntu server VM

Create or start the Ubuntu 24.04.4 LTS virtual machine in VirtualBox.
Configure its network adapter to use the lab's NAT Network.

Confirm the server's IP address from Ubuntu:

``` bash
ip address
```

Check network connectivity and confirm that the server is reachable from
the Windows endpoint.

**Screenshot:**
[01-virtualbox-vm-overview.png](../screenshots/lab-architecture/01-virtualbox-vm-overview.png)

**Screenshot:**
[02-ubuntu-vm-configuration.png](../screenshots/lab-architecture/02-ubuntu-vm-configuration.png)

### 3.2 Prepare the Windows 10 endpoint VM

Start the Windows 10 VM and confirm its version and network
configuration.

Run the following in PowerShell:

``` powershell
winver
ipconfig
```

The endpoint used in this lab is Windows 10 Home 22H2, build
`19045.2965`.

**Screenshot:**
[03-windows-vm-configuration.png](../screenshots/lab-architecture/03-windows-vm-configuration.png)

### 3.3 Configure the virtual network

Both VMs must be able to communicate over the VirtualBox NAT Network. In
this lab, the network range is `10.0.0.0/24`.

Confirm the Ubuntu server address is `10.0.0.7` and the Windows endpoint
can reach it. The Windows endpoint used during the lab had the address
`10.0.0.10`; verify this with `ipconfig` before using it in any
configuration.

**Screenshot:**
[04-virtual-network-configuration.png](../screenshots/lab-architecture/04-virtual-network-configuration.png)

## 4. Wazuh Server Installation and Verification

### 4.1 Update Ubuntu

On the Ubuntu VM, update the package index and installed packages:

``` bash
sudo apt update
sudo apt upgrade -y
```

### 4.2 Install the Wazuh server components

For a fresh all-in-one lab deployment, Wazuh provides an installation
assistant. Use the installation instructions for the Wazuh release
selected for the lab, and keep the manager, indexer, dashboard, and
agent package versions compatible.

The general installation-assistant workflow is:

``` bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

> **Important:** `4.x` is a version-family placeholder, not a literal
> release to use. Replace it with the exact supported release path from
> the official Wazuh installation guide. Do not rerun the all-in-one
> installer on an already configured server just to reproduce this
> document; it is intended for a fresh installation and may change an
> existing deployment.

Official guide: [Wazuh
quickstart](https://documentation.wazuh.com/current/quickstart.html).

### 4.3 Verify Wazuh services

On Ubuntu, check the status of the installed components:

``` bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

The services should show as active/running. If a component is not
installed as a system service in the chosen deployment, verify it using
the instructions for that deployment.

Check the manager's registered agents:

``` bash
sudo /var/ossec/bin/agent_control -l
```

The Windows endpoint should appear as `WINDOWS-10-HOST` with agent ID
`002` once enrolled and connected.

**Screenshot:**
[05-wazuh-agent-list.png](../screenshots/installation/01-wazuh-agent-list.png)

## 5. Windows 10 Agent Installation and Enrollment

### 5.1 Obtain a compatible Windows agent installer

Download the Windows Wazuh agent MSI from the official Wazuh packages
repository or the release-specific installation instructions.

Use an agent package compatible with the Wazuh server version. During
this lab, an agent/server version mismatch was encountered during
enrollment, so version compatibility should be checked before
installation or re-enrollment.

Official guide: [Deploy a Wazuh agent on
Windows](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html).

### 5.2 Install and configure the agent

Install the Windows agent using the MSI package. If installing from an
elevated PowerShell prompt, the general command pattern is:

``` powershell
msiexec.exe /i .\wazuh-agent-<VERSION>.msi /q WAZUH_MANAGER="10.0.0.7" WAZUH_AGENT_NAME="WINDOWS-10-HOST"
```

Replace `<VERSION>` with the exact MSI package version downloaded. If
the agent is already installed, do not run a second installation
blindly; inspect the existing installation and configuration first.

The agent configuration file is normally located at:

``` text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Check that the `<client>` section points to the Wazuh server:

``` xml
<client>
  <server>
    <address>10.0.0.7</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Preserve the rest of the existing configuration file. Do not add a
duplicate `<client>` section if one already exists.

### 5.3 Enroll the agent

Agent enrollment normally uses the Wazuh registration service on TCP
port `1515`, unless the deployment uses a different enrollment method.
The Windows MSI installation may perform enrollment when the appropriate
deployment variables and enrollment settings are supplied.

If manual enrollment is required, use the method documented for the
installed Wazuh version and the server's configured authentication
settings. Do not invent or reuse an agent key. The lab's registered
endpoint is identified in the manager as `WINDOWS-10-HOST`, agent ID
`002`.

### 5.4 Start and verify the Windows agent

Open PowerShell as Administrator:

``` powershell
Start-Service -Name wazuh
Get-Service -Name wazuh
```

The service should show `Running`.

To check basic connectivity to the manager from Windows:

``` powershell
Test-NetConnection 10.0.0.7 -Port 1514
Test-NetConnection 10.0.0.7 -Port 1515
```

Port `1514/TCP` is used for agent communication. Port `1515/TCP` is
commonly used for agent enrollment. A successful TCP test confirms
network reachability to the port; it does not by itself prove that
enrollment or agent communication has completed successfully.

**Screenshot:**
[06-windows-agent-installation.png](../screenshots/installation/02-wazuh-agent-installation.png)

**Screenshot:**
[07-windows-agent-service.png](../screenshots/fim/01-agent-status.png)

**Screenshot:**
[08-windows-agent-connectivity.png](../screenshots/installation/03-windows-agent-connectivity.png)

## 6. Agent Configuration and Log Locations

### 6.1 Windows agent configuration

The primary Windows agent configuration file is:

``` text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

It contains the manager connection settings and the local monitoring
configuration. Back up the file before making changes, preserve existing
sections, and restart the agent after configuration changes.

### 6.2 Windows agent log

The Windows agent log is normally located at:

``` text
C:\Program Files (x86)\ossec-agent\ossec.log
```

Review this log when investigating connection, enrollment, or
configuration errors.

### 6.3 Ubuntu manager log

The Wazuh manager log is located at:

``` text
/var/ossec/logs/ossec.log
```

Review it when investigating agent registration, communication, or
manager-side processing issues.

## 7. Dashboard Access and Endpoint Verification

Open the Wazuh dashboard from a browser that can reach the Ubuntu VM:

``` text
https://10.0.0.7
```

The browser may display a certificate warning in a lab deployment if a
locally generated or self-signed certificate is being used. Confirm that
the address is your own lab server before proceeding.

Navigate to **Endpoints** and select `WINDOWS-10-HOST`. Confirm that the
endpoint is connected and that its agent ID is `002`.

The dashboard is also used to review vulnerability detection, Security
Configuration Assessment (SCA), and File Integrity Monitoring (FIM)
results. Those findings and FIM test steps are documented separately.

**Screenshot:**
[09-wazuh-dashboard-overview.png](../screenshots/installation/04-wazuh-dashboard-overview.png)

**Screenshot:**
[10-windows-endpoint-connected.png](../screenshots/installation/05-windows-endpoint-connected.png)

## 8. Troubleshooting Notes

### Agent does not connect

1.  Confirm that the Windows agent service is running:

    ``` powershell
    Get-Service -Name wazuh
    ```

2.  Confirm the manager address in the Windows agent's `ossec.conf` is
    `10.0.0.7`.

3.  Test connectivity to TCP ports `1514` and `1515` from Windows.

4.  Confirm that the VMs are attached to the intended VirtualBox NAT
    Network.

5.  Check the Windows agent log and Ubuntu manager log.

### Enrollment fails

A version mismatch was encountered during the lab's earlier enrollment
attempt. Check the installed server and agent versions and use
compatible packages. Also verify that the manager's enrollment service
is configured and reachable. Follow the release-specific Wazuh
enrollment instructions.

### Agent appears in the dashboard but is not connected

Check the service state, manager address, network path, and logs on both
systems. On Ubuntu, list registered agents with:

``` bash
sudo /var/ossec/bin/agent_control -l
```

### Useful port-listening check on Ubuntu

``` bash
sudo ss -lntp | grep -E ':1514|:1515'
```

This helps verify whether the expected Wazuh services are listening on
the manager VM.

## 9. References

-   [Wazuh
    documentation](https://documentation.wazuh.com/current/index.html)
-   [Wazuh quickstart
    installation](https://documentation.wazuh.com/current/quickstart.html)
-   [Wazuh Windows agent
    installation](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
-   [Wazuh agent
    enrollment](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/index.html)
-   [Wazuh agent configuration
    reference](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/client.html)
