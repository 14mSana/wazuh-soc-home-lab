# Wazuh Windows Agent Onboarding

## 1. Purpose

This document describes how the Windows 10 endpoint was onboarded to the
Wazuh manager and how its registration and connection were verified.

The endpoint appears in the Wazuh environment as:

  Setting                     Value
  --------------------------- -------------------
  Agent name                  `WINDOWS-10-HOST`
  Agent ID                    `002`
  Wazuh manager address       `10.0.0.7`
  Agent communication port    `1514/TCP`
  Common enrollment port      `1515/TCP`
  Endpoint operating system   Windows 10

Agent onboarding connects an endpoint to the Wazuh manager so that the
manager can receive and process security data from that endpoint.

## 2. Prerequisites

Before onboarding the Windows endpoint, confirm the following:

-   The Wazuh manager is installed and running on the Ubuntu VM.
-   The Windows 10 VM and Ubuntu VM can communicate over the configured
    VirtualBox NAT Network.
-   The Windows endpoint can reach the manager address, `10.0.0.7`.
-   The Windows agent package is compatible with the installed Wazuh
    manager version.
-   Administrative privileges are available on Windows.

> **Version compatibility:** Use the Wazuh agent package that matches
> the manager release. Confirm the actual installed versions before
> installation or troubleshooting.

## 3. Obtain and Install the Windows Agent

Download the Windows agent MSI package from the official Wazuh packages
repository or follow the release-specific Windows agent installation
guide:

[Wazuh Windows agent installation
documentation](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)

Install the agent on the Windows 10 VM. The general command pattern for
an MSI installation from an elevated PowerShell window is:

``` powershell
msiexec.exe /i .\wazuh-agent-<VERSION>.msi /q WAZUH_MANAGER="10.0.0.7" WAZUH_AGENT_NAME="WINDOWS-10-HOST"
```

Replace `<VERSION>` with the version of the MSI file actually
downloaded. If the agent is already installed, inspect the existing
installation before running the installer again.

The installation screenshot below records the Windows agent installation
step:

![Windows Wazuh agent
installation](../screenshots/installation/02-wazuh-agent-installation.png)

## 4. Configure the Manager Connection

The Windows agent configuration file is normally located at:

``` text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

Open it with administrative privileges and verify that the `<client>`
section points to the Wazuh manager:

``` xml
<client>
  <server>
    <address>10.0.0.7</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

Preserve the other existing settings and avoid adding a duplicate
`<client>` section.

Agent registration is handled through the Wazuh enrollment mechanism. In
a standard deployment, the enrollment service commonly listens on TCP
port `1515`. The precise enrollment procedure depends on the manager's
enrollment configuration and the agent installation method. Follow the
instructions for the installed Wazuh release; do not reuse another
endpoint's key or invent an enrollment key.

## 5. Start the Agent and Verify Connectivity

Open PowerShell as Administrator on Windows.

Start the agent service:

``` powershell
Start-Service -Name wazuh
```

Check its status:

``` powershell
Get-Service -Name wazuh
```

The service should report `Running`.

Test whether the Windows endpoint can reach the manager's agent
communication and enrollment ports:

``` powershell
Test-NetConnection 10.0.0.7 -Port 1514
Test-NetConnection 10.0.0.7 -Port 1515
```

A successful TCP connection test confirms network reachability to that
port. It does not, by itself, prove that the agent has completed
enrollment or is actively communicating with the manager.

![Windows agent connectivity
test](../screenshots/installation/03-windows-agent-connectivity.png)

## 6. Verify Agent Registration on the Wazuh Manager

On the Ubuntu VM, list the agents registered with the manager:

``` bash
sudo /var/ossec/bin/agent_control -l
```

Locate the Windows endpoint and verify that it appears as
`WINDOWS-10-HOST` with agent ID `002`.

![Wazuh registered agent
list](../screenshots/installation/01-wazuh-agent-list.png)

If the endpoint is not listed, review the enrollment configuration and
manager logs before proceeding.

## 7. Verify the Endpoint in the Wazuh Dashboard

Open the Wazuh dashboard from a browser that can access the Ubuntu VM:

``` text
https://10.0.0.7
```

Navigate to **Endpoints** and select `WINDOWS-10-HOST`. Confirm that the
endpoint is displayed and that its connection status indicates that it
is active/connected.

The dashboard overview provides evidence that the Wazuh interface is
accessible:

![Wazuh dashboard
overview](../screenshots/installation/04-wazuh-dashboard-overview.png)

The endpoint view provides evidence that the Windows agent is visible in
the dashboard:

![Windows endpoint connected in
Wazuh](../screenshots/installation/05-windows-endpoint-connected.png)

## 8. Troubleshooting

### Agent is not running

Check the Windows service:

``` powershell
Get-Service -Name wazuh
```

If appropriate, start it:

``` powershell
Start-Service -Name wazuh
```

### Windows cannot reach the manager

Check the manager address and test the relevant ports:

``` powershell
Test-NetConnection 10.0.0.7 -Port 1514
Test-NetConnection 10.0.0.7 -Port 1515
```

Also verify that both VMs are attached to the intended VirtualBox NAT
Network and that firewall rules permit the required traffic.

### Agent is installed but does not appear as connected

1.  Verify the manager address in the Windows agent's `ossec.conf`.

2.  Confirm that the agent service is running.

3.  Confirm that the agent is registered with the manager.

4.  Check the Windows agent log:

    ``` text
    C:\Program Files (x86)\ossec-agent\ossec.log
    ```

5.  Check the Wazuh manager log on Ubuntu:

    ``` text
    /var/ossec/logs/ossec.log
    ```

6.  Verify that the installed agent and manager versions are compatible.

### Agent registration or version mismatch

Use a Windows agent package compatible with the installed manager
version. If re-enrollment is necessary, follow the official Wazuh agent
enrollment procedure for that release. Do not delete or re-register an
active agent unless you understand the effect on its existing identity
and collected data.

## 9. Outcome

The Windows 10 endpoint is represented in the Wazuh environment as
`WINDOWS-10-HOST` with agent ID `002`. The registered-agent list and
dashboard endpoint view are used to verify onboarding.

After confirming the endpoint is connected, it can be used for the lab's
endpoint monitoring exercises, including vulnerability detection,
Security Configuration Assessment (SCA), and File Integrity Monitoring
(FIM).

## 11. References

-   [Wazuh
    documentation](https://documentation.wazuh.com/current/index.html)
-   [Deploy a Wazuh agent on
    Windows](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
-   [Wazuh agent
    enrollment](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/index.html)
-   [Wazuh agent configuration
    reference](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/client.html)

