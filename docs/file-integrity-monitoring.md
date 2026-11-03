# File Integrity Monitoring (FIM)

## 1. Objective

The objective of this practical is to configure and verify File
Integrity Monitoring (FIM) on the Windows 10 endpoint connected to the
Wazuh manager.

The experiment uses a dedicated test directory (`C:\FIM-Lab`) to safely
demonstrate file creation, modification, and deletion. Wazuh events are
then reviewed in the dashboard to verify whether the changes were
detected.


## 2. Lab Environment

  Component             Configuration
  --------------------- -------------------------------------------
  Monitoring platform   Wazuh
  Endpoint              Windows 10 VM (`WINDOWS-10-HOST`)
  FIM test directory    `C:\FIM-Lab`
  Monitoring mode       Real-time monitoring
  Test file             `test-file.txt`
  Events tested         File creation, modification, and deletion

## 3. FIM Configuration

### 3.1 Verify the Wazuh agent

Open PowerShell as Administrator on the Windows 10 VM and check the
agent service:

``` powershell
Get-Service -Name wazuh
```

The service should show `Running` before proceeding.

**Screenshot:**
[01-agent-status.png](../screenshots/fim/01-agent-status.png)

### 3.2 Create the test directory

Create a dedicated directory for the experiment:

``` powershell
New-Item -Path "C:\FIM-Lab" -ItemType Directory -Force
```

**Screenshot:**
[02-fim-test-directory.png](../screenshots/fim/02-fim-test-directory.png)

### 3.3 Configure real-time monitoring

Open the Wazuh agent configuration file:

``` powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Locate the existing `<syscheck>` section. Add the following entry inside
that section. Do not create a duplicate `<syscheck>` block if one
already exists.

``` xml
<directories realtime="yes" check_all="yes">C:\FIM-Lab</directories>
```

Save the configuration file.

-   `realtime="yes"` enables real-time monitoring for the directory.
-   `check_all="yes"` requests checks of the supported file attributes.
-   `C:\FIM-Lab` is the directory monitored in this experiment.

**Screenshot:**
[03-fim-configuration.png](../screenshots/fim/03-fim-configuration.png)

### 3.4 Restart the Wazuh agent

Apply the configuration by restarting the service:

``` powershell
Restart-Service -Name wazuh
Get-Service -Name wazuh
```

Confirm that the service returns to the `Running` state.

**Screenshot:**
[04-agent-restart.png](../screenshots/fim/04-agent-restart.png)

## 4. FIM Testing

Perform each test separately and check the Wazuh dashboard for the
corresponding event. Event arrival and labels can vary depending on the
agent configuration and Wazuh version.

### 4.1 File creation

Create a test file:

``` powershell
Set-Content -Path "C:\FIM-Lab\test-file.txt" -Value "Initial FIM test content"
```

Confirm that the file exists in `C:\FIM-Lab`.

**Screenshot:**
[05-file-creation.png](../screenshots/fim/05-file-creation.png)

### 4.2 File modification

Append a second line to the file:

``` powershell
Add-Content -Path "C:\FIM-Lab\test-file.txt" -Value "This line was added during the FIM modification test."
Get-Content -Path "C:\FIM-Lab\test-file.txt"
```

Confirm that the output contains both the initial and appended text.

**Screenshot:**
[06-file-modification.png](../screenshots/fim/06-file-modification.png)

### 4.3 File deletion

Delete the test file and verify that it no longer exists:

``` powershell
Remove-Item -Path "C:\FIM-Lab\test-file.txt"
Test-Path "C:\FIM-Lab\test-file.txt"
```

The verification command should return `False`.

**Screenshot:**
[07-file-deletion.png](../screenshots/fim/07-file-deletion.png)

## 5. Verification in the Wazuh Dashboard

Open the Wazuh dashboard and navigate to the File Integrity Monitoring
area. Select or filter for the `WINDOWS-10-HOST` endpoint and set a time
range that includes the tests.

Review the events associated with `C:\FIM-Lab\test-file.txt`. Where
available, inspect the event action, file path, timestamp, agent, and
reported file attributes or checksums.

**Screenshot:**
[08-fim-events-overview.png](../screenshots/fim/08-fim-events-overview.png)

Open a relevant event to inspect its details.

**Screenshot:**
[09-fim-event-details.png](../screenshots/fim/09-fim-event-details.png)

## 6. Results

Complete this table only after checking the Wazuh dashboard. Mark an
event as detected only when the corresponding event is visible in Wazuh.

  -----------------------------------------------------------------------
  Test                    Expected result         Observed result
  ----------------------- ----------------------- -----------------------
  File creation           Wazuh reports the new   Pending verification
                          file                    

  File modification       Wazuh reports the file  Pending verification
                          change                  

  File deletion           Wazuh reports the file  Pending verification
                          removal                 
  -----------------------------------------------------------------------

## 7. Troubleshooting

If events do not appear:

1.  Confirm that the Wazuh agent service is running.
2.  Confirm that the monitoring entry is inside the existing
    `<syscheck>` section and that the configuration was saved.
3.  Restart the agent after changing its configuration.
4.  Check the agent log at
    `C:\Program Files (x86)\ossec-agent\ossec.log` for configuration or
    `syscheck` errors.
5.  Confirm that the dashboard is showing the correct endpoint and a
    time range covering the test.
6.  Create a new test file after confirming that monitoring is active. A
    file created before the configuration was applied may not produce
    the expected creation event.

## 8. Conclusion

This practical configures Wazuh FIM to monitor a dedicated directory on
a Windows 10 endpoint. Controlled file creation, modification, and
deletion tests are used to evaluate whether Wazuh reports file changes
in its dashboard.

## 9. References

-   [Wazuh: File integrity
    monitoring](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html)
-   [Wazuh: How it
    works](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-it-works.html)
-   [Wazuh: Configure
    FIM](https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/how-to-configure-fim.html)
-   [Wazuh: Syscheck configuration
    reference](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syscheck.html)

