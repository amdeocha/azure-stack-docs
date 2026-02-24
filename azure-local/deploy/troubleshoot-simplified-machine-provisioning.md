---
title: Troubleshoot Simplified Machine Provisioning for Azure Local (preview)
description: Learn how to troubleshoot simplified machine provisioning for Azure Local (preview).
ms.author: alkohli
author: alkohli
ms.reviewer: alkohli
ms.topic: how-to
ms.date: 02/23/2026
ms.subservice: hyperconverged
---

# Troubleshoot simplified machine provisioning for hyperconverged deployments of Azure Local (preview)

[!INCLUDE [hci-preview](../includes/hci-preview.md)]

This article describes how to troubleshoot simplified machine provisioning. You can use the following methods to troubleshoot:

- Run diagnostic tests.
- Collect a support package.
- Collect logs from your Azure Subscription.

## Run diagnostic tests from the Configurator app

To diagnose and troubleshoot any device issues related to hardware, time server, and network, you can run the diagnostics tests. Follow these steps to run the diagnostic tests from the app:

1. Select the help icon in the top-right corner of the app to open the **Support + troubleshooting** pane.

1. Select **Run diagnostic tests**. The diagnostic tests check the health of the server hardware, time server, and the network connectivity. The tests also check the status of the Azure Arc agent and the extensions.

1. After the tests are completed, the results are displayed. Resolve the issues and retry the operation.

## Collect a support package from the app

A log package is composed of all the relevant logs that can help Microsoft Support troubleshoot any device issues. You can generate a log package via the local web UI. Follow these steps to collect a support package from the app:

1. Select the help icon in the top-right corner of the app to open the **Support + troubleshooting** pane.

1. Select **Create** to begin support package collection. The package collection could take several minutes.

1. After the support package is created, select **Download**. A zipped package is downloaded on your local system. You can unzip the package and view the system log files.

## Collect logs from your Azure subscription

If you can't access the machine using Configurator App, you can get the app logs from the server. Access the logs by connecting with SSH for Azure Linux or PowerShell for Azure Stack HCI.

The logs are stored in the following locations:

| Target operating system | Log files |
|--|--|
| Azure Stack HCI | `C:\Windows\System32\Bootstrap\Logs` |
| Maintenance environment | `/var/log/bootstrap`<br>`/var/log/provisioningextension`<br>`/var/log/trident-full.log`<br>`/var/log/messages`<br>`/var/log/bootstraprestservice` |

If you can access the app, follow the instructions in [Run diagnostic tests](#run-diagnostic-tests-from-the-configurator-app) to troubleshoot the issue and then [Collect a support package](#collect-a-support-package-from-the-app) if needed.

## Investigate a running system from the cloud

If the Azure portal shows that the edge machine has reached the `Ready to connect` stage, select **Json view**:

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-1.png" alt-text="Screenshot 1 showing how to investigate a running system from the cloud." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-1.png":::

Open the URL in `arcMachineResourceId`, then select **Json view**:

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-2.png" alt-text="Screenshot 2 showing how to investigate a running system from the cloud." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-2.png":::

If the client public key is present, it means TO1 is present. If Arc isn't connecting:

1. If the Azure portal shows the Arc connection is done and waiting for extension installation, you can see the status of the extension installation with one of the following methods:

    - Select the edge machine and select **Json view**.

    - Select **Arc** and then the **Extension** tab.

1. If the Azure portal shows that the extension installation succeeded, you can see the status of OS provisioning with the following URL. Replace the `<PLACEHOLDERS>` with your values.

    `/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.AzureStackHCI/edgeMachines/<EDGE_MACHINE_ARM_ID>/jobs/ProvisionOs?api-version=2025-12-01-preview`

1. If the Azure portal shows that provisioning has reached the `Downloading OS` stage, select the edge machine and look at the target Arc resource to see the same behaviors as in the previous steps.

## Maintenance environment known issues

**Problem:** The USB Boot enters an infinite USB boot sequence if the BIOS Settings `Boot USB Devices First` is set on the Next Unit of Computing (NUC).

### Cause

When the BIOS setting `Boot USB Devices First` is enabled on the NUC, the system enters an infinite USB boot cycle.
This setting overrides the configured boot order and always prioritizes booting from any connected USB device. As a result, even after the remote operations environment (ROE) is successfully installed, subsequent reboots continue to boot from the USB media instead of the internal disk.

This behavior causes a continuous boot loop in which the device repeatedly:

1. Boots from the USB device.
1. Reinstalls Azure Linux.
1. Reboots.

From the customer’s perspective, the device appears to be in an infinite cycle of installation and reboot, which occurs approximately every 10 minutes.

### Recommendation

Disable the `Boot USB Devices First` BIOS option (or the equivalent setting, depending on the hardware model and BIOS version).

The following screenshot shows the correct configuration.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-maintenance-environment-known-issues.png" alt-text="Screenshot showing a BIOS configuration with the Boot USB Devices First option disabled." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-maintenance-environment-known-issues.png":::

## Initial creation failure

**Problem:** The **Image Url** drop down is empty.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png" alt-text="Screenshot 1 showing an empty Image Url drop down." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png":::

**Cause:** The resource provider registration for `Microsoft.AzureStackHCI` is missing.

**Recommendation:** Register the resource provider as described in the [prerequisites](simplified-machine-provisioning#azure-prerequisites).

## Resource group creation fails

**Problem:** In site, configure new, resource group creation is failing.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-2.png" alt-text="Screenshot showing a failed resource group creation." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-2.png":::

**Cause:** You have a resource group policy that doesn't support simplified machine provisioning.

**Recommendation:** Create the resource group in a subscription without a resource group policy.

## ARM template validation fails

**Problem:** ARM template validation fails.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png" alt-text="Screenshot showing a failed ARM template validation." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png":::

**Cause:** You're attempting this task for the first time in this tenant.

**Recommendation:** Wait 15 minutes and retry.

## Internal server error on site default

**Problem:** Internal server error on site default

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png" alt-text="Screenshot showing an internal server error on site default." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png":::

**Cause:** You have a resource group policy that doesn't support simplified machine provisioning.

**Recommendation:**

If the policy concerns naming conventions for resource groups, you can select **Configure** on sites to give a custom MRG name.

If the policy concerns resource groups having tags or being created in a specific region, simplified machine provisioning doesn't currently support these scenarios.

## Edge machine creation fails

**Error:** `StorageAccountForbidden`
**Cause:** Your storage account creation policy doesn't support simplified machine provisioning, or you didn't register the `Microsoft.Storage` resource provider.
**Recommendation:** Register the resource provider as described in the [prerequisites](simplified-machine-provisioning#azure-prerequisites).

**Error:** `DeviceOnboardingConflict`
**Cause:** The feature registration is missing, or you didn't register the `Microsoft.OnboardingService` resource provider.
**Recommendation:** Register the resource provider as described in the [prerequisites](simplified-machine-provisioning#azure-prerequisites).

**Error:** `UpdateArcSettingDataFailed`
**Cause:** You didn't register the `Microsoft.HybridCompute` resource provider.
**Recommendation:** Register the resource provider as described in the [prerequisites](simplified-machine-provisioning#azure-prerequisites).

For any of the previously described errors, or for other errors, delete the edge machine and try to create it again. If that doesn't work, contact support. When you contact support, please provide the activity log of the edge machine resource group and the managed resource group if possible. For more information, see [Run diagnostic tests](#run-diagnostic-tests-from-the-configurator-app), [Collect a support package from the app](#collect-a-support-package-from-the-app) and [Collect logs from your Azure subscription](#collect-logs-from-your-azure-subscription).

## Reattempt a failed OS provisioning

**Problem:** Simplified machine provisioning currently doesn't support automatic retries from the service in case the OS installation fails. In a future release it will support automatic retries, as well as allowing the customer to retry provisioning from the Azure portal.

**Recommendation:**

To retry OS provisioning:

1. Open the edge machine and record the URL.

1. Add `/jobs/provisionos` to the URL.

1. Open **Json view** and record the Json.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-reattempt-failed-os-provisioning.png" alt-text="Screenshot showing how to reattempt a failed OS provisioning." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-reattempt-failed-os-provisioning.png":::

1. Make a `PUT` request with the modified URL. Replace the `<PLACEHOLDERS>` with your values.

`PUT /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.AzureStackHCI/edgeMachines/<MACHINE_NAME>/jobs/ProvisionOs?api-version=2025-12-01-preview`

In the content, copy only the `osProfile` and `userDetails` objects.

```json
{
  "properties": {
    "provisioningRequest": {
      "osProfile": {
        "osName": "AzureLinux",
        "osType": "AzureLinux",
        "osVersion": "3.0",
        "osImageLocation": "https://aka.ms/aep/sff/azurelinux/2602a",
        "operationType": "Provision"
      },
      "userDetails": [
        {
          "userName": "clouduser",
          "secretType": "SshPubKey",
          "sshPubKey": ["ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDmMokuQD0b4USnkLA9baVQftdpMKYDunqYiom6qeeNF6Ch2bdw458wKv7qIq4BWFJ5TVBSc2CRhQ2BC0WvokWMvEnOrsi1BLkX0toGBo+P7xTGPxZQrR8iFBkQqa0m1N/wCDMoaghDIdBQmGC6CVuheM5ZF3GDJ9GLzVXwTbhw4/bi9AhGyVyibL+9KjgXzZZOi4OpAxeZyu82ergbxK0Zxqj5dUdV9UeIb8BNEbl6gNP75Y9sysvfbSuomFyIUYb8gD1dCsDBZM54hLolQVF8EzAot1B+pNkDq51Dgos8Tyya94XFd6prCX+wAbRgDHIGN3OgpntQCxR5jseC5ZEBlRUUigl+XOcefAD/6ILc1V4/RV5hAdkK7dHh1IyfQM5sm3hrr/QloZ4a+5RHuuj5U/9sbKv5vLCLi0vcZjm3XrQOpsnvSVLuvpXjZV3LHiuL4W/1ATY+pncTOrwI1q0z5ccEAz1aFRUfjz9heLjBD2v4Ye/O7Lmalspt8lBNTA0= generated-by-azure"]
        }
      ],
    }
}
```

## Clean up resources

You can attempt to delete the edge machine resource at any time and it delete related objects, such as resources under MRG.

Deleting an edge machine is only blocked when:

- The edge machine is part of a device pool that is part of an Azure Kubernetes Service (AKS) cluster.

- The device is transitioning and not yet reached a stable state.

In the resource group where you run simplified machine provisioning, there are two hidden resources: a configuration resource, and a resource called the `MOBO broker`. You can't delete the `MOBO broker` resource directly. If you delete the resource group, the `MOBO broker` resource is deleted with it. Also, if you delete the configuration resource, the `MOBO broker` resource is deleted with it.

Be careful when deleting the configuration resource. Deleting the configuration resource brings down devices that haven't yet reached the `Ready to cluster` stage. Deleting the configuration resource might also bring down running devices.
