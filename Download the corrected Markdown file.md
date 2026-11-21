# How to Check Dynatrace OneAgent Status and Version on Multiple Windows EC2 Instances Without RDP

**Author:** Praveen Veerapogu

---

## Introduction

Managing a single Windows server is straightforward. However, when an AWS environment contains multiple Windows EC2 instances, manually signing in to every server simply to verify the Dynatrace OneAgent status quickly becomes repetitive and inefficient.

This guide explains a practical, real-world use case in which **AWS Systems Manager Run Command** is used to check the **Dynatrace OneAgent service status and installed version across multiple Windows EC2 instances simultaneously**, without using Remote Desktop Protocol (RDP) to access each server individually.

This approach is useful for:

- Site Reliability Engineering (SRE) teams
- Cloud Operations teams
- Infrastructure teams
- Observability teams
- AWS administrators
- Engineers managing Dynatrace-monitored Windows EC2 environments

---

## Real-World Scenario

Consider an AWS environment with several Windows EC2 application servers:

```text
Windows EC2-01 -> Dynatrace OneAgent
Windows EC2-02 -> Dynatrace OneAgent
Windows EC2-03 -> Dynatrace OneAgent
Windows EC2-04 -> Dynatrace OneAgent
...
Windows EC2-N  -> Dynatrace OneAgent
```

Suppose the Operations or SRE team notices that monitoring data from one or more servers is missing or is not updating correctly in Dynatrace. The immediate questions are:

- Is Dynatrace OneAgent installed?
- Is the Dynatrace OneAgent Windows service running?
- What is the service startup type?
- Which Dynatrace OneAgent version is installed?
- Are all servers running the expected OneAgent version?

The traditional approach is to connect to each Windows EC2 instance through RDP, open Windows Services or PowerShell, check Dynatrace OneAgent, record the result, disconnect, and repeat the process on the next server.

### Traditional Method

```text
Engineer
   |
   +-- RDP -> EC2-01 -> Check OneAgent
   +-- RDP -> EC2-02 -> Check OneAgent
   +-- RDP -> EC2-03 -> Check OneAgent
   +-- RDP -> EC2-N  -> Check OneAgent
```

This might be manageable for two or three servers, but it is not an efficient operational model for 10, 50, or 100 servers.

---

## A Better Approach: AWS Systems Manager

AWS Systems Manager provides **Run Command**, which allows administrators to remotely execute commands on managed EC2 instances.

```text
AWS Management Console
        |
        v
AWS Systems Manager
        |
        v
Run Command
        |
        v
AWS-RunPowerShellScript
        |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
Windows EC2-01       Windows EC2-02       Windows EC2-03
        |                  |                  |
        v                  v                  v
Dynatrace OneAgent   Dynatrace OneAgent   Dynatrace OneAgent
```

A single PowerShell health check can therefore be executed against multiple Windows EC2 instances from a central location.

---

## What Will We Check?

1. Dynatrace Windows services
2. Dynatrace OneAgent running status
3. Windows service startup type
4. Dynatrace OneAgent installation
5. Installed Dynatrace OneAgent version
6. The same information across multiple EC2 instances

---

## Prerequisites

Before starting, verify that:

- The Windows EC2 instances are running.
- AWS Systems Manager Agent (SSM Agent) is installed and running.
- The EC2 instances appear as managed nodes in AWS Systems Manager.
- An IAM instance profile with the required Systems Manager permissions is attached to each instance. The AWS managed policy commonly used for managed nodes is `AmazonSSMManagedInstanceCore`.
- Dynatrace OneAgent is already installed on the target Windows servers.
- Your AWS identity has permission to execute Systems Manager Run Command.
- The instances can communicate with the required Systems Manager endpoints, either through internet access or VPC endpoints.

> **Important:** If an EC2 instance does not appear as a managed node in AWS Systems Manager, troubleshoot SSM Agent, the IAM instance profile, and network connectivity before troubleshooting Dynatrace.

---

## Part 1: Check Dynatrace OneAgent Service Status

Sign in to the AWS Management Console and navigate to:

```text
AWS Systems Manager
-> Node Management
-> Run Command
-> Run command
```

Under **Command document**, select:

```text
AWS-RunPowerShellScript
```

Next, select the Windows EC2 instances on which Dynatrace OneAgent is installed. Multiple instances can be targeted by the same Run Command.

### PowerShell Script

Enter the following script under **Command parameters**:

```powershell
Write-Output "=========================================="
Write-Output " Dynatrace OneAgent Status Check"
Write-Output " Server: $env:COMPUTERNAME"
Write-Output "=========================================="

$services = Get-Service |
    Where-Object {
        $_.Name -like "*Dynatrace*" -or
        $_.DisplayName -like "*Dynatrace*"
    }

if ($services) {
    Write-Output ""
    Write-Output "Dynatrace services found:"
    Write-Output ""

    $services |
        Select-Object Name, DisplayName, Status, StartType |
        Format-Table -AutoSize

    $stopped = $services |
        Where-Object { $_.Status -ne "Running" }

    if ($stopped) {
        Write-Output ""
        Write-Output "WARNING: One or more Dynatrace services are not running."
    }
    else {
        Write-Output ""
        Write-Output "SUCCESS: All detected Dynatrace services are running."
    }
}
else {
    Write-Output ""
    Write-Output "ERROR: No Dynatrace services were found."
    exit 1
}
```

### Understanding the Script

#### Step 1: Identify the Windows Server

```powershell
Write-Output " Server: $env:COMPUTERNAME"
```

This displays the name of the Windows computer on which the command is running. This is particularly useful when the same command is run against multiple EC2 instances.

#### Step 2: Find Dynatrace Windows Services

```powershell
$services = Get-Service |
    Where-Object {
        $_.Name -like "*Dynatrace*" -or
        $_.DisplayName -like "*Dynatrace*"
    }
```

`Get-Service` retrieves the Windows services available on the server. `Where-Object` retains services for which either the internal service name or display name contains `Dynatrace`.

#### Step 3: Display Service Details

```powershell
$services |
    Select-Object Name, DisplayName, Status, StartType |
    Format-Table -AutoSize
```

| Field | Purpose |
|---|---|
| `Name` | Internal Windows service name |
| `DisplayName` | Human-readable service name |
| `Status` | Current service status |
| `StartType` | Windows service startup configuration |

A healthy result may look like this:

```text
Name            DisplayName              Status   StartType
----            -----------              ------   ---------
Dynatrace...    Dynatrace OneAgent...    Running  Automatic
```

#### Step 4: Detect a Non-Running Dynatrace Service

```powershell
$stopped = $services |
    Where-Object { $_.Status -ne "Running" }
```

If a non-running service is detected, the script returns a warning. If all detected Dynatrace services are running, it returns a success message.

#### Step 5: Handle Missing Dynatrace Services

If no matching service is found, the script returns an error and exits with status code `1`.

Possible causes include:

- Dynatrace OneAgent is not installed.
- OneAgent was removed.
- The OneAgent installation is incomplete.
- The expected Windows service is not registered.
- The wrong EC2 instance was targeted.

---

## Review Results Across Multiple EC2 Instances

After running the command, navigate to:

```text
Systems Manager
-> Run Command
-> Command history
-> Select the command
-> Targets and outputs
```

Open each target to review its Dynatrace OneAgent service information.

> **Important:** An SSM command status of `Success` means that Systems Manager successfully delivered and ran the command. It does **not** necessarily mean that Dynatrace OneAgent is installed, running, or healthy. The script output must also be reviewed.

---

## Part 2: Check the Dynatrace OneAgent Version

Create another Run Command using `AWS-RunPowerShellScript`, select the required Windows EC2 instances, and run:

```powershell
$OneAgentCtl = "C:\Program Files\dynatrace\oneagent\agent\tools\oneagentctl.exe"

if (Test-Path $OneAgentCtl) {
    Write-Output ""
    Write-Output "Dynatrace OneAgent Version:"
    & $OneAgentCtl --version
}
else {
    Write-Output "WARNING: oneagentctl.exe was not found at the default location."
}
```

The script defines the default OneAgent control utility path, checks whether it exists with `Test-Path`, and then runs:

```powershell
& $OneAgentCtl --version
```

If `oneagentctl.exe` is not found, investigate whether:

- OneAgent uses a custom installation directory.
- The OneAgent installation is incomplete.
- OneAgent was removed.
- OneAgent is not installed on the target server.

> **Important:** Do not conclude that OneAgent is absent solely because the default `oneagentctl.exe` path is missing. Confirm the installation path and configuration first.

---

## Recommended Combined Health Check

The following reusable script checks both the service status and installed version:

```powershell
Write-Output "=========================================="
Write-Output " Dynatrace OneAgent Health Check"
Write-Output " Server: $env:COMPUTERNAME"
Write-Output " Time: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss K')"
Write-Output "=========================================="

Write-Output ""
Write-Output "[1] SERVICE STATUS"
Write-Output "------------------------------------------"

$services = Get-Service |
    Where-Object {
        $_.Name -like "*Dynatrace*" -or
        $_.DisplayName -like "*Dynatrace*"
    }

$serviceHealthy = $false

if ($services) {
    $services |
        Select-Object Name, DisplayName, Status, StartType |
        Format-Table -AutoSize

    $stopped = $services |
        Where-Object { $_.Status -ne "Running" }

    if ($stopped) {
        Write-Output "RESULT: WARNING - A Dynatrace service issue was detected."
    }
    else {
        $serviceHealthy = $true
        Write-Output "RESULT: HEALTHY - All detected Dynatrace services are running."
    }
}
else {
    Write-Output "RESULT: ERROR - No Dynatrace services were detected."
}

Write-Output ""
Write-Output "[2] ONEAGENT VERSION"
Write-Output "------------------------------------------"

$OneAgentCtl = "C:\Program Files\dynatrace\oneagent\agent\tools\oneagentctl.exe"
$versionDetected = $false

if (Test-Path $OneAgentCtl) {
    & $OneAgentCtl --version
    $versionDetected = $true
}
else {
    Write-Output "WARNING: oneagentctl.exe was not found at the default path."
}

Write-Output ""
Write-Output "=========================================="
Write-Output " Health Check Completed"
Write-Output "=========================================="

if (-not $services -or -not $serviceHealthy -or -not $versionDetected) {
    exit 1
}

exit 0
```

The explicit exit codes make the command result more operationally useful: `0` indicates that all checks passed, while `1` indicates that at least one check requires attention.

---

## Troubleshooting Scenarios

### Scenario 1: SSM Succeeds and OneAgent Is Running

```text
SSM command: Success
OneAgent:   Running
```

Systems Manager connectivity is working, the command executed successfully, and the detected Dynatrace services are running.

### Scenario 2: SSM Succeeds but OneAgent Is Stopped

```text
SSM command: Success
OneAgent:   Stopped
```

Systems Manager connectivity and PowerShell execution are working, but the problem exists at the Dynatrace or Windows service layer. On production systems, investigate why the service stopped before restarting it.

### Scenario 3: No Dynatrace Service Is Found

Investigate whether OneAgent was installed or removed, whether installation failed, whether the correct instance was targeted, and whether the expected Windows service is registered.

### Scenario 4: `oneagentctl.exe` Is Not Found

Check for a custom installation directory, an incomplete installation, removal of OneAgent, or unsuccessful onboarding of the server to Dynatrace.

---

## Targeting Larger AWS Environments

For a small number of Windows servers, selecting individual EC2 instances in Run Command is acceptable. For larger environments, use consistent EC2 tags, for example:

```text
Environment = Production
Monitoring  = Dynatrace
OS          = Windows
Application = EasyTravel
```

Tag-based targeting is more scalable and less error-prone than maintaining a static list of EC2 instance IDs.

---

## Security Considerations

AWS Systems Manager Run Command provides powerful administrative capabilities. Protect it appropriately:

- Apply least-privilege AWS IAM permissions.
- Use the appropriate EC2 IAM instance profile.
- Restrict who can execute Systems Manager Run Command.
- Use AWS CloudTrail and Systems Manager logging for auditing.
- Review access to administrative PowerShell execution against production instances.
- Where required, send command output to an encrypted Amazon S3 bucket or Amazon CloudWatch Logs.
- Avoid placing passwords, tokens, or other secrets directly in command text.

The goal is not to replace RDP with unrestricted remote command execution. The goal is to use a controlled, auditable, and repeatable mechanism for operational checks that do not require interactive server access.

---

## Key Benefits

- No need to establish an RDP session with every Windows EC2 instance.
- Check multiple EC2 instances simultaneously.
- Centralized execution through AWS Systems Manager.
- Quickly identify stopped Dynatrace services.
- Verify Windows service startup configuration.
- Check installed Dynatrace OneAgent versions.
- Reduce repetitive operational work.
- Support incident response and maintenance validation.
- Scale targeting through EC2 tags.

---

## Quick Reference

### AWS Console Path

```text
AWS Systems Manager
-> Node Management
-> Run Command
-> Run command
```

### Command Document

```text
AWS-RunPowerShellScript
```

### Default OneAgent Control Utility Path

```text
C:\Program Files\dynatrace\oneagent\agent\tools\oneagentctl.exe
```

### OneAgent Version Command

```powershell
& "C:\Program Files\dynatrace\oneagent\agent\tools\oneagentctl.exe" --version
```

---

## Final Takeaway

When multiple Windows EC2 instances are monitored using Dynatrace OneAgent, signing in to every server simply to check the agent status or version is not an efficient operational model.

AWS Systems Manager Run Command provides a practical alternative. It enables engineers to run a consistent PowerShell validation against a targeted Windows EC2 fleet and centrally determine:

- Whether Dynatrace services are detected
- Whether the detected services are running
- The configured service startup type
- Whether `oneagentctl.exe` is available
- Which OneAgent version is installed

> **Operational principle:** If the same administrative health check must be performed repeatedly across multiple servers, consider whether AWS Systems Manager can perform it centrally, consistently, securely, and audibly.

---

## Conclusion

AWS Systems Manager and Dynatrace can work together to simplify day-to-day monitoring operations across Windows EC2 environments. Using Systems Manager Run Command, administrators can remotely validate Dynatrace OneAgent service status, Windows service startup configuration, installation, and version across multiple instances without establishing individual RDP sessions.

For small environments, this saves time. For larger environments, it provides the foundation for a more standardized, repeatable, and scalable operational process.

---

## Author

**Praveen Veerapogu**

---

## Suggested Tags

`Dynatrace` `AWS` `AWS Systems Manager` `Amazon EC2` `OneAgent` `Windows Server` `SRE` `Observability` `Cloud Operations` `PowerShell` `AWS SSM`

## Suggested Search Description

> Learn how to check Dynatrace OneAgent service status and version across multiple Windows EC2 instances without RDP by using AWS Systems Manager Run Command and PowerShell.
