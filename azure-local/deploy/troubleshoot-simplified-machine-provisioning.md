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

1. Select the help icon in the top-right corner of the app to open **Support + troubleshooting**.

1. Select **Run diagnostic tests**. The diagnostic tests check the health of the server hardware, time server, and the network connectivity. The tests also check the status of the Azure Arc agent and the extensions.

1. After the tests are completed, the results are displayed. Resolve the issues and retry the operation.

## Collect a support package from the app

A log package is composed of all the relevant logs that can help Microsoft Support troubleshoot any device issues. You can generate a log package via the local web UI. Follow these steps to collect a support package from the app:

1. Select the help icon in the top-right corner of the app to open **Support + troubleshooting**.

1. Select **Create** to begin support package collection. The package collection could take several minutes.

1. After the support package is created, select **Download**. A zipped package is downloaded on your local system. You can unzip the package and view the system log files.

## Collect logs from your Azure subscription

If you can't access the machine using the Configurator app, you can get the app logs from the server. Access the logs by connecting with PowerShell.

The logs are stored in the following locations:

| Target operating system | Log files |
|--|--|
| Azure Stack HCI | `C:\Windows\System32\Bootstrap\Logs` |
| Maintenance environment | `/var/log/bootstrap`<br>`/var/log/provisioningextension`<br>`/var/log/trident-full.log`<br>`/var/log/messages`<br>`/var/log/bootstraprestservice` |

## Maintenance environment known issues

**Problem:** The USB Boot enters an infinite USB boot sequence if the BIOS Settings `Boot USB Devices First` is set on the machine.

**Cause:**

When the BIOS setting `Boot USB Devices First` is enabled on the machine, the system enters an infinite USB boot cycle. This setting overrides the configured boot order and always prioritizes booting from any connected USB device. As a result, even after the maintenance environment is successfully installed, subsequent reboots continue to boot from the USB media instead of the internal disk.

This behavior causes a continuous boot loop in which the device repeatedly:

1. Boots from the USB device.
1. Reinstalls the maintenance environment.
1. Reboots.

From the customer’s perspective, the device appears to be in an infinite cycle of installation and reboot, which occurs approximately every 10 minutes.

**Recommendation:**

Disable the `Boot USB Devices First` BIOS option (or the equivalent setting, depending on the hardware model and BIOS version).

TODO1: Engineering: The BIOS screenshot provided in the engineering TSG is for SFF devices; can we please get one for supported Azure Local hardware? dsarkar is working on this.

## Operating system image drop down is empty

**Problem:** In Azure portal, select **Azure Arc** > **Operations** > **Machine provisioning (preview)** > **Get started** > **Provision**. In the **Provision new machines** page, the **Operating system** > **Image** drop down is empty.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png" alt-text="Screenshot 1 showing an empty Image Url drop down." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png":::

**Cause:** The resource provider registration for `Microsoft.AzureStackHCI` is missing.

**Recommendation:** Register the resource provider as described in the [prerequisites](simplified-machine-provisioning.md#azure-prerequisites).

## When you provision new machines, ARM template validation fails

**Problem:** In Azure portal, select **Azure Arc** > **Operations** > **Machine provisioning (preview)** > **Get started** > **Provision**. In the **Provision new machines** page, when you select **Create**, you see the error messages `Arm template validation failed` and `Deployment template validation failed: 'The value for the template parameter 'hciRPServiceprincipalID' at line '1' and column '10174' is not provided. Please see https://aka.ms/arm-create-parameter-file for usage details.'. (Code: InvalidTemplate)`

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png" alt-text="Screenshot showing a failed ARM template validation." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png":::

**Cause:** This issue can happen if you're using Azure Stack HCI for the first time in this subscription.

**Recommendation:** Wait 15 minutes and retry.

## When you provision new machines, you receive an internal server error

**Problem:** In Azure portal, select **Azure Arc** > **Operations** > **Machine provisioning (preview)** > **Get started** > **Provision**. In the **Provision new machines** page, when you select **Create**, you see the error messages `The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.` and `Failed to verify creation of MoboBroker resource. (Code: InternalServerError)`

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png" alt-text="Screenshot showing an internal server error on site default." border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png":::

TODO1: anbacker: I reworded these but please feel free to reword further.

**Causes:** This can have one of the following causes:

1. You have a resource group policy that requires resource groups to be created in a specific region. In this preview release, only the `eastus` region supports simplified machine provisioning.

1. You have a resource group policy that automatically adds tags to newly created resource groups. Simplified machine provisioning doesn't currently support this.

1. You have a resource group policy that enforces naming conventions.

**Recommendations:**

1. Remove the resource group policy that requires resource groups to be created in a specific region.

1. Remove the resource group policy that automatically adds tags to newly created resource groups.

1. When you create a new site, you can select **Configure** to provide a custom resource group name.

## Provisioned machine creation fails with the error message "StorageAccountForbidden"

TODO1: Engineering: Can you please provide more details on the potential issues with storage account creation policies? anbacker is working on this.

**Cause:** Your storage account creation policy doesn't support simplified machine provisioning, or you didn't register the `Microsoft.Storage` resource provider.

**Recommendation:**

1. Register the resource provider as described in the [prerequisites](simplified-machine-provisioning.md#azure-prerequisites).

1. Delete the provisioned machine and create it again.

## Provisioned machine creation fails with the error message "DeviceOnboardingConflict"

**Cause:** You didn't register the `MachineProvision` feature or the `Microsoft.OnboardingService` resource provider.

**Recommendation:**

1. Register the `MachineProvision` feature and required resource providers as described in the [prerequisites](simplified-machine-provisioning.md#azure-prerequisites).

1. Delete the provisioned machine and create it again.

## Provisioned machine creation fails with the error message "UpdateArcSettingDataFailed"

**Cause:** You didn't register the `Microsoft.HybridCompute` resource provider.

**Recommendation:**

1. Register the resource provider as described in the [prerequisites](simplified-machine-provisioning.md#azure-prerequisites).

1. Delete the provisioned machine and create it again.

## Reattempt a failed OS provisioning

**Problem:** Simplified machine provisioning currently doesn't support automatic retries from the service in case the OS installation fails.

**Recommendation:**

To retry OS provisioning:

1. Browse to [Azure Cloud shell](https://shell.azure.com).

1. Azure prompts you to choose Bash or PowerShell if you haven’t selected a default yet. It might also prompt you to create or select a storage account to persist your Cloud Shell files.

TODO1: Engineering: Does the typical customer know to open a text editor like nano, paste this in, edit it, save the file, and run it with `bash` or else set execute permissions on it? Also note we don't actually take command line arguments like that usage string implies, so we might want to instead say "please set these variables"?

1. Create and run the following Bash script. Replace the `<PLACEHOLDERS>` with your values.

    ```bash
    #!/bin/bash
    set -e

    # Script to retry OS provisioning on an edge machine by resubmitting the ProvisionOS job
    # Performs a reput with osProfile and userDetails

    # Input parameters
    SUBSCRIPTION_ID=<subscription-id> 
    RESOURCE_GROUP=<resource-group>
    PROVISIONED_MACHINE_NAME=edge-machine-name>

    # Validate inputs
    if [ -z "$SUBSCRIPTION_ID" ] || [ -z "$RESOURCE_GROUP" ] || [ -z "$PROVISIONED_MACHINE_NAME" ]; then
        echo "Usage: $0 <subscription-id> <resource-group> <edge-machine-name>"
        exit 1
    fi

    echo "Subscription ID: $SUBSCRIPTION_ID"
    echo "Resource Group: $RESOURCE_GROUP"
    echo "Edge Machine Name: $PROVISIONED_MACHINE_NAME"

    # Set the subscription
    echo "Setting subscription..."
    az account set --subscription "$SUBSCRIPTION_ID"

    # Construct the ProvisionOS job endpoint
    PROVISION_OS_URL="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.AzureStackHCI/edgeMachines/$PROVISIONED_MACHINE_NAME/jobs/ProvisionOs"
    EDGE_MACHINE_URL="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.AzureStackHCI/edgeMachines/$PROVISIONED_MACHINE_NAME"
    API_VERSION="2025-12-01-preview"

    echo "Getting edge machine configuration..."
    # GET the edge machine to extract userDetails
    EDGE_MACHINE_CONFIG=$(az rest \
        --method GET \
        --url "https://management.azure.com${EDGE_MACHINE_URL}?api-version=${API_VERSION}" \
        --output json)

    echo "Edge machine configuration retrieved"

    # Extract userDetails and osProfile from edge machine
    USER_DETAILS=$(echo "$EDGE_MACHINE_CONFIG" | jq '.properties.provisioningDetails.userDetails')
    OS_PROFILE=$(echo "$EDGE_MACHINE_CONFIG" | jq '.properties.provisioningDetails.osProfile')
    SITE_RESOURCE_ID=$(echo "$EDGE_MACHINE_CONFIG" | jq -r '.properties.siteDetails.siteResourceId')

    if [ "$USER_DETAILS" == "null" ] || [ "$OS_PROFILE" == "null" ]; then
        echo "Error: Could not extract userDetails or osProfile from the edge machine configuration"
        echo "Edge machine config: $EDGE_MACHINE_CONFIG"
        exit 1
    fi

    # Hardcode target and jobType
    TARGET="HCI"
    JOB_TYPE="ProvisionOs"

    echo "Extracted osProfile and userDetails from edge machine; using hardcoded target: $TARGET and jobType: $JOB_TYPE"

    # Create the PUT body with osProfile, userDetails, target, and jobType
    PUT_BODY=$(jq -n \
        --argjson osProfile "$OS_PROFILE" \
        --argjson userDetails "$USER_DETAILS" \
        --arg target "$TARGET" \
        --arg jobType "$JOB_TYPE" \
        '{
            properties: {
                jobType: $jobType,
                provisioningRequest: {
                    osProfile: $osProfile,
                    userDetails: $userDetails,
                    target: $target
                }
            }
        }')

    echo "Performing reput to retry OS provisioning..."
    # PUT the configuration back to trigger a retry
    az rest \
        --method PUT \
        --url "https://management.azure.com${PROVISION_OS_URL}?api-version=${API_VERSION}" \
        --body "$PUT_BODY" \
        --output json

    echo "ProvisionOS retry submitted successfully"
    ```

1. It takes up to 30 minutes for the retry to complete.

## Clean up resources

You can attempt to delete the provisioned machine resource at any time. Deleting the resource also deletes related objects, such as resources under the managed resource group.

TODO1: Engineering: Can you please review this section? Is this useful to customers? If not, I'll remove it.

In the resource group where you run simplified machine provisioning, there are two hidden resources: a configuration resource, and a resource called the `MOBO broker`. You can't delete the `MOBO broker` resource directly. If you delete the resource group, the `MOBO broker` resource is deleted with it. Also, if you delete the configuration resource, the `MOBO broker` resource is deleted with it.

Be careful when deleting the configuration resource. Deleting the configuration resource brings down devices that haven't yet reached the `Ready to cluster` stage.
