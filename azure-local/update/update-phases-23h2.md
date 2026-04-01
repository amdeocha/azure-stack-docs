---
title: Review update phases of Azure Local, version 23H2
description: Understand the various phases of solution updates applied to Azure Local, version 23H2.
author: ronmiab
ms.author: robess
ms.topic: concept-article
ms.date: 04/01/2026
ms.subservice: hyperconverged
---

# Review update phases of Azure Local

[!INCLUDE [applies-to](../includes/hci-applies-to-23h2.md)]

This article describes the preparation and installation phases of the Azure Local update workflow, including how updates are downloaded, validated, health-checked, and installed. It also explains how update progress is reported at various stages.

For more detailed information on progress reporting, see [Use Azure Update Manager to update Azure Local](azure-update-manager-23h2.md) and [Update Azure Local via PowerShell](update-via-powershell-23h2.md).

## Overview

An Azure Local update follows a two-stage lifecycle:

1. **Preparation**: Download content, validate and extract packages, and run health checks to confirm the cluster is ready.
1. **Installation**: Apply the update across the cluster via an orchestrated action plan.

Each stage produces an `UpdateRun` resource that records step-by-step progress, timing, and any errors encountered. You can query the details of this run on the **Update Progress** page in Azure Update Manager or by using the `Get-SolutionUpdateRun` cmdlet in PowerShell.

:::image type="content" source="media/update-phases-23h2/update-phases-actions.png" alt-text="Diagram of update process with Preparation phase and Installation phase steps." lightbox="media/update-phases-23h2/update-phases-actions.png":::

## Preparation phases

You can trigger preparation independently using `Start-SolutionUpdate -PrepareOnly`, which downloads and validates update content and runs health checks without starting installation. This allows you to pre-stage updates before a maintenance window or verify cluster readiness in advance.

Before preparation, the update might be in an `AdditionalContentRequired` state. This state indicates that the update package (either an SBE update or a combined Solution plus SBE update) requires partner content, and the installed SBE package from the partner doesn't support automatic download of the content.

If the update is in the `AdditionalContentRequired` state, you must import the content before you can begin preparation or installation. For more information, see [Update via PowerShell](update-via-powershell-23h2.md).

The preparation workflow moves through the following phases in order.

### Download

The download phase retrieves the update package from the configured update source.

- **Standard download**: The Update Service downloads the main solution update package (NuGet bundles) directly from the update catalog.
- **Progress tracking**: The `ProgressPercentage` field on the `UpdateStateProperties` property on the Update reports download progress as a value from 0 to 100.

During this phase, the Update object transitions to the `Downloading` state. On failure, the state becomes `DownloadFailed`.

### SBE download connector (if applicable)

SBE updates and some Solution updates require additional content from the hardware vendor's Solution Builder Extension (SBE). When the SBE provides a download connector, the Update Service delegates part of the download to the OEM-supplied logic:

- The Update Service checks whether the installed SBE supports a download connector.
- If supported, an Orchestrator action plan invokes the SBE download action to retrieve OEM-specific packages such as firmware and drivers.
- The hardware vendor typically includes a download connectivity health check that must pass before the download starts.

If download fails while using the SBE download connector, the update state becomes `DownloadFailed`. To see the detailed failure message, examine the preparation details in Azure Update Manager in the portal, or using the `UpdateRun` object from `Get-SolutionUpdateRun`.

### Validate and extract

After all content is downloaded, the Update Service validates the hashes of the downloaded files and extracts the contents.

If validation or extraction fails, the `UpdateRun` records the error and the Update state becomes `PreparationFailed`.

### Health check

Before an update can be installed, a set of pre-update health checks validates that the cluster is in a healthy state and checks for issues that could interfere with successful update installation.

Each health check has an assigned severity:

| Severity | Effect |
| --- | --- |
| **Critical** | Blocks the update. You must remediate this before installation can proceed. |
| **Warning** | Blocks the update by default. You can override this with `Start-SolutionUpdate -IgnoreWarnings`. |
| **Informational** | Advisory only. Doesn't block installation. |

If the update was started in **prepare-only** mode, the Update transitions to the `ReadyToInstall` state on health check success. If critical or warning failures are detected (when `-IgnoreWarnings` isn't specified), the status becomes `HealthCheckFailed`.

You can inspect health check results on the update object using:

```powershell
# View health check results
(Get-SolutionUpdate).HealthCheckResult |
Where-Object { ($_.Status -ne "Success") -and ($_.Severity -ne "Informational") } |
Format-List Title, Status, Severity, Description, Remediation
```

For help troubleshooting health check failures, see [Troubleshoot updates](update-troubleshooting-23h2.md).

## Monitor preparation using Get-SolutionUpdateRun

Every call to `Start-SolutionUpdate` (with or without `-PrepareOnly`) creates an `UpdateRun` resource. You can retrieve the specific step details of preparation using:

```powershell
# Get the most recent update run for an update
Get-SolutionUpdate -Id <UpdateResourceId> | Get-SolutionUpdateRun | % Progress | % Steps
```

When a preparation run fails, the `UpdateRun` `State` property is set to `Failed` and the Progress step tree contains error details at the step that encountered the problem.

## Installation phase

After preparation completes, or when `Start-SolutionUpdate` is called without `-PrepareOnly`, the update enters the installation phase.

### Start installation

To start a full update that includes both preparation and installation, use:

```powershell
# Start a full update (preparation + installation)
Get-SolutionUpdate -Id <UpdateResourceId> | Start-SolutionUpdate
```

When the installation begins, the **Update state** transitions to **Installing** and a new `UpdateRun` is created that represents the progress of the installation, replacing the `UpdateRun` that previously represented preparation.

### `UpdateRun` installation progress

During installation, the `UpdateRun`'s `Progress` property contains the full action plan execution tree. This is a hierarchical structure of `Step` objects where each step represents a stage, role, or individual task in the update.

Each step in the progress tree exposes the following properties:

| Property | Type | Description |
| --- | --- | --- |
| Name | string | Name of the step or task. |
| Description | string | Human-readable description. |
| Status | string | `InProgress`, `Success`, or `Error`. |
| StartTimeUtc | DateTime | When the step started executing. |
| EndTimeUtc | DateTime | When the step completed or failed. |
| ErrorMessage | string | Error details if the step failed. |
| ExpectedExecutionTime | TimeSpan | Estimated duration for progress calculation. |
| Steps | Step[] | Child steps forming the execution tree. |

### Monitor installation progress

Due to the complex structure of the `UpdateRun` object, monitor update installation status via the Azure portal.

:::image type="content" source="media/update-phases-23h2/update-run-structure.png" alt-text="Screenshot of the UpdateRun structure." lightbox="media/update-phases-23h2/update-run-structure.png":::

If you need to monitor the update using PowerShell, directly monitor the state of the underlying action plan.

> [!NOTE]
> The `Start-MonitoringActionplanInstanceToComplete` cmdlet should only be used after the system has installed the 2503 update. Before 2503, using this cmdlet to monitor update progress can introduce failures in the orchestration.

```powershell
# Get the action plan instance ID from the update run, then monitor
$run = Get-SolutionUpdate | where State -eq "Installing" | Get-SolutionUpdateRun | where State -eq "InProgress"
$id = ($run.ResourceId -split '/')[-1]
Start-MonitoringActionplanInstanceToComplete -actionPlanInstanceID $id
```

This provides real-time console output that refreshes automatically. Press **Ctrl+C** to exit the monitor without stopping the update.

### Installation state transitions

The Update moves through these states during installation:

| State | Meaning |
| --- | --- |
| Installing | Installation is actively running. |
| Installed | The update finished installing successfully. |
| InstallationFailed | One or more steps failed. |

### Diagnose and resume after installation failure

When installation fails, review the failure details in the Azure portal or via `Get-SolutionUpdateRun`. For troubleshooting guidance, see [Troubleshoot updates](update-troubleshooting-23h2.md).

After you review and mitigate the failure, or determine it's transient, resume the update from the Azure portal or using the `Start-SolutionUpdate` cmdlet:

```powershell
Get-SolutionUpdate | where State -eq "InstallationFailed" | Start-SolutionUpdate
```

After calling this cmdlet, the update state transitions directly from `InstallationFailed` to `Installing`.

## Next steps

Learn more about how to [Troubleshoot updates](./update-troubleshooting-23h2.md).
