# Troubleshooting Notes

## 1. Overview

This document records the practical issues encountered while building
and operating the Wazuh virtual lab, along with the checks and
resolutions used or recommended during troubleshooting.

The lab consists of an Ubuntu VM running the Wazuh server components and
a Windows endpoint connected through a VirtualBox NAT Network. The
Windows endpoint used for the monitoring exercises is registered as
`WINDOWS-10-HOST` (agent ID `002`), and the Wazuh dashboard is accessed
at `https://10.0.0.7`.

> **Documentation accuracy:** The notes distinguish between issues that
> were encountered and general checks that can be used if the issue
> happens again. Do not describe a suggested check as a confirmed fix
> unless it was tested successfully.

## 2. VirtualBox Networking Issues

### 2.1 Windows and Ubuntu VMs could not communicate

**Problem:** The Wazuh manager runs on Ubuntu and the monitored endpoint
runs on Windows. Both VMs need network connectivity to communicate.

**Checks performed / recommended:**

1.  Confirm that both VMs are attached to the intended VirtualBox NAT
    Network.

2.  Check the Ubuntu IP address:

    ``` bash
    ip address
    ```

3.  Check the Windows IP address:

    ``` powershell
    ipconfig
    ```

4.  Test connectivity from Windows to the Ubuntu manager:

    ``` powershell
    ping 10.0.0.7
    ```

5.  Test the Wazuh agent ports:

    ``` powershell
    Test-NetConnection 10.0.0.7 -Port 1514
    Test-NetConnection 10.0.0.7 -Port 1515
    ```

**Lab configuration:** The lab network uses the `10.0.0.0/24` range,
with the Ubuntu Wazuh server at `10.0.0.7`. The Windows endpoint's
address should be confirmed with `ipconfig`, rather than assumed to
remain unchanged.

**Important:** A failed ping does not always mean the VMs cannot
communicate; ICMP may be blocked. Use the TCP port tests and Wazuh agent
status as additional evidence.

### 2.2 DNS server at `10.0.0.1` did not resolve names

**Problem encountered:** The configured DNS server `10.0.0.1` did not
respond successfully to DNS queries from the Kali Linux VM. The DNS
lookup returned a connection-refused or unsuccessful response.

**Resolution used:** The DNS server was changed to `8.8.8.8`, after
which name resolution worked.

To check the current resolver configuration on Linux:

``` bash
resolvectl status
```

To test DNS resolution:

``` bash
nslookup example.com 8.8.8.8
```

or:

``` bash
dig @8.8.8.8 example.com
```

**Note:** `8.8.8.8` is Google's public DNS resolver. Use it only where
permitted by the network environment. A public resolver may not be
suitable for networks that require internal DNS.

## 3. Wazuh Agent Installation and Enrollment

### 3.1 Agent registration or enrollment problems

**Problem encountered:** An agent enrollment attempt raised a version
compatibility concern between the Wazuh manager and the agent package.

**Recommended checks:**

1.  Check the installed manager version on Ubuntu.
2.  Check the Windows agent version in the installed applications or
    agent logs.
3.  Use an agent package compatible with the manager release.
4.  Verify that the manager's enrollment service is enabled and
    reachable.
5.  Review the manager and agent logs for the exact enrollment error.

On Ubuntu, list registered agents:

``` bash
sudo /var/ossec/bin/agent_control -l
```

On Windows, inspect the agent log:

``` text
C:\Program Files (x86)\ossec-agent\ossec.log
```

On Ubuntu, inspect the manager log:

``` text
/var/ossec/logs/ossec.log
```

Do not delete an existing agent or regenerate its identity as a first
troubleshooting step. First establish whether the problem is a version
mismatch, network reachability, enrollment configuration, or service
state.

### 3.2 Windows agent service was not running or needed verification

Check the service from PowerShell running as Administrator:

``` powershell
Get-Service -Name wazuh
```

If it is stopped, start it:

``` powershell
Start-Service -Name wazuh
```

After changing the agent configuration, restart it:

``` powershell
Restart-Service -Name wazuh
```

Verify again:

``` powershell
Get-Service -Name wazuh
```

The expected service status is `Running`.

### 3.3 Agent is installed but not shown as connected

Check the following in order:

1.  Confirm the Windows service is running.

2.  Verify the manager address in the agent configuration file:

    ``` text
    C:\Program Files (x86)\ossec-agent\ossec.conf
    ```

3.  Confirm that the configured manager address is `10.0.0.7`.

4.  Test the manager communication port:

    ``` powershell
    Test-NetConnection 10.0.0.7 -Port 1514
    ```

5.  If enrollment is required, check the enrollment port:

    ``` powershell
    Test-NetConnection 10.0.0.7 -Port 1515
    ```

6.  Check the agent and manager logs.

7.  Confirm that the manager lists the expected endpoint name and agent
    ID.

A successful port test verifies TCP reachability only. The endpoint must
also be correctly enrolled and actively communicating to appear
connected.

## 4. Wazuh Dashboard and Monitoring Issues

### 4.1 Dashboard does not open

The lab dashboard is accessed at:

``` text
https://10.0.0.7
```

If it does not load:

1.  Confirm that the Ubuntu VM is running.

2.  Check the dashboard service:

    ``` bash
    sudo systemctl status wazuh-dashboard
    ```

3.  Check the indexer and manager services:

    ``` bash
    sudo systemctl status wazuh-indexer
    sudo systemctl status wazuh-manager
    ```

4.  Confirm that the client machine can reach the Ubuntu VM.

5.  Check whether the browser is showing a certificate warning. In a lab
    environment, a locally generated or self-signed certificate may be
    used. Confirm the server address before proceeding.

### 4.2 FIM dashboard initially showed no recent events

**Problem encountered:** The endpoint summary initially showed **"No
recent events"** in the FIM section.

**Explanation:** This can be normal when no monitored file changes have
occurred in the selected time range, or when the directory has not yet
been configured for monitoring.

**Steps used for the FIM practical:**

1.  Create a dedicated test directory:

    ``` powershell
    New-Item -Path "C:\FIM-Lab" -ItemType Directory -Force
    ```

2.  Add a real-time monitoring entry inside the existing `<syscheck>`
    section in the Windows agent's `ossec.conf`:

    ``` xml
    <directories realtime="yes" check_all="yes">C:\FIM-Lab</directories>
    ```

3.  Save the configuration and restart the agent:

    ``` powershell
    Restart-Service -Name wazuh
    ```

4.  Create, modify, and delete a test file in the monitored directory.

5.  Review the FIM events for the Windows endpoint in the dashboard.

**Observed result:** The Wazuh endpoint summary subsequently displayed
three events for `c:\fim-lab\test-file.txt`:

  Action       Rule description shown
  ------------ -----------------------------
  `added`      File added to the system.
  `modified`   Integrity checksum changed.
  `deleted`    File deleted.

This confirms that the tested file creation, modification, and deletion
were visible in the Wazuh FIM recent-events panel.

Screenshot evidence is stored in:

-   [FIM events overview](../screenshots/fim/08-fim-events-overview.png)
-   [FIM event details](../screenshots/fim/09-fim-event-details.png)

### 4.3 FIM events do not appear after a file change

If a new test event does not appear:

1.  Confirm that the directory exists.
2.  Confirm that the `<directories>` entry is inside the existing
    `<syscheck>` section.
3.  Check that the agent configuration was saved.
4.  Restart the Windows agent.
5.  Inspect `ossec.log` for configuration or `syscheck` errors.
6.  Confirm that the dashboard is showing the correct endpoint and a
    time range that includes the test.
7.  Create a new test file after monitoring is active.

Do not assume that an event was detected until it is visible in the
dashboard or confirmed in the relevant event logs.

### 4.4 Vulnerability and SCA findings need careful interpretation

The Windows endpoint dashboard displayed vulnerability findings and a
CIS Windows 10 Enterprise SCA assessment. These results are separate
from FIM events.

When documenting them:

-   Record the date and time of the scan.
-   Record the endpoint and operating system shown.
-   Do not interpret every vulnerability finding as an actively
    exploited vulnerability.
-   Treat SCA failed checks as configuration-assessment findings that
    require review.
-   Avoid changing security settings solely to improve a score without
    understanding the impact.

The dashboard screenshot used for the FIM evidence has the Vulnerability
Detection panel blurred while keeping the FIM events visible.

## 5. Virtual Machine Storage and Setup Issues

### 5.1 Host system drive was running low on space

**Problem encountered:** The Windows host's C: drive was running low on
storage while VM files were being created on the D: drive.
VirtualBox-related files or default VM locations may still consume space
on C:.

**Recommended checks:**

1.  In VirtualBox, open **File → Tools → Virtual Media Manager** to
    review virtual disk locations.
2.  Check each VM's folder location in its settings or in VirtualBox's
    machine information.
3.  Check the actual size and location of virtual disk files (`.vdi`)
    and snapshots.
4.  Keep sufficient free space on the drive used for VM disks and
    snapshots.

If relocating a VM, shut it down completely first and use VirtualBox's
supported move or clone workflow. Do not manually move an active VM or
delete its original virtual disk before verifying that the moved VM
works.

### 5.2 Ubuntu installation showed a black screen or appeared stuck

During Ubuntu VM setup, the installation process showed a black screen
and it was unclear whether the installer was still working.

**Recommended recovery checks:**

1.  Wait briefly to determine whether installation is still in progress.
2.  Check VirtualBox VM activity and resource usage.
3.  Avoid repeatedly powering off the VM while disk installation may be
    in progress.
4.  If it remains unresponsive, shut it down safely where possible and
    review the VM display and graphics-controller settings before
    retrying.
5.  Confirm the Ubuntu ISO and VM resources before starting a fresh
    installation.

Do not delete the virtual disk or restart the installation from scratch
until you have checked whether the existing installation is recoverable.

### 5.3 Copy and paste between host and guest did not work

**Problem encountered:** Copy/paste between the host and a virtual
machine was difficult during setup.

**Checks:**

1.  Confirm the VM is powered off before changing its shared clipboard
    setting.
2.  In VirtualBox VM settings, open **General → Advanced**.
3.  Set **Shared Clipboard** to the required direction, such as
    **Bidirectional**, if appropriate for the lab.
4.  Start the VM and test with non-sensitive text.
5.  For Linux guests, check that the appropriate VirtualBox Guest
    Additions components are installed and running.

Clipboard behavior can depend on the guest OS, Guest Additions version,
and desktop session. Do not paste secrets or credentials into an
untrusted guest.

## 6. Useful Diagnostic Commands

### Windows endpoint (PowerShell as Administrator)

``` powershell
# Check Wazuh agent service
Get-Service -Name wazuh

# Start or restart the agent
Start-Service -Name wazuh
Restart-Service -Name wazuh

# Check Windows IP configuration
ipconfig

# Test manager reachability
Test-NetConnection 10.0.0.7 -Port 1514
Test-NetConnection 10.0.0.7 -Port 1515

# Read the Wazuh agent log
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

### Ubuntu Wazuh server

``` bash
# Check Wazuh services
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# List registered agents
sudo /var/ossec/bin/agent_control -l

# Review manager log
sudo tail -n 50 /var/ossec/logs/ossec.log

# Check listening Wazuh ports
sudo ss -lntp | grep -E ':1514|:1515'
```

### Linux DNS checks

``` bash
# Show resolver configuration
resolvectl status

# Test DNS resolution using the public resolver
nslookup example.com 8.8.8.8
```

## 7. Troubleshooting Summary

  -----------------------------------------------------------------------
  Issue                               Check or resolution
  ----------------------------------- -----------------------------------
  Windows and Ubuntu VM communication Verify both VMs use the intended
                                      NAT Network; check IP addresses and
                                      TCP connectivity

  DNS at `10.0.0.1` did not respond   Changed to `8.8.8.8`; name
                                      resolution then worked

  Agent enrollment/version concern    Check manager and agent versions
                                      and use compatible packages

  Agent service state                 Check with `Get-Service`; start or
                                      restart when required

  Agent not connected                 Verify manager address, ports,
                                      enrollment, and both agent and
                                      manager logs

  FIM initially showed no recent      Configure a monitored directory,
  events                              restart the agent, generate
                                      controlled file changes, and check
                                      the time range

  Host C: drive low on space          Check VM, virtual disk, and
                                      snapshot locations before safely
                                      relocating

  Ubuntu installer black screen       Check VM activity and display
                                      settings before restarting
                                      installation

  Host/guest clipboard not working    Check Shared Clipboard settings and
                                      Guest Additions
  -----------------------------------------------------------------------

## 8. Conclusion

Building the Wazuh lab involved troubleshooting virtual networking, DNS
resolution, agent enrollment and connectivity, VM storage, guest setup,
and monitoring visibility.

The most important monitoring milestone was configuring FIM on the
Windows 10 endpoint and verifying the `added`, `modified`, and `deleted`
events for the controlled test file in the Wazuh dashboard.

These troubleshooting notes document the issues encountered and provide
repeatable diagnostic steps for maintaining the lab. They should be
updated when additional problems are encountered and verified
resolutions are established.

## 9. References

-   [Wazuh
    documentation](https://documentation.wazuh.com/current/index.html)
-   [Wazuh Windows agent
    installation](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
-   [Wazuh agent
    enrollment](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/index.html)
-   [Wazuh File Integrity
    Monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
-   [VirtualBox User Manual](https://www.virtualbox.org/manual/)

