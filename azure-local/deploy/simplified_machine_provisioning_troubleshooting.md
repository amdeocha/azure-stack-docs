---
title: Troubleshoot Simplified Machine Provisioning for Azure Local (preview)
description: Learn how to troubleshoot simplified machine provisioning for Azure Local (preview).
ms.author: alkohli
ms.reviewer: alkohli
ms.topic: how-to
ms.date: 02/23/2026
ms.service: azure-local
ms.subservice: hyperconverged
---

# Troubleshoot simplified machine provisioning for hyperconverged deployments of Azure Local (preview)

[!INCLUDE [hci-preview](../includes/hci-preview.md)]

## Maintenance environment known issues

Issue: The USB Boot enters an infinite USB boot sequence if the BIOS Settings 'Boot USB Devices First' is set on the NUC.
When the BIOS setting “Boot USB Devices First” is enabled on the NUC, the system enters an infinite USB boot cycle.
This setting overrides the configured boot order and always prioritizes booting from any connected USB device. As a result, even after the ROE is successfully installed, subsequent reboots continue to boot from the USB media instead of the internal disk.
Impact
This behavior causes a continuous boot loop in which the device repeatedly:

Boots from the USB device 
Reinstalls Azure Linux 
Reboots 
From the customer’s perspective, the device appears stuck in an infinite cycle of installation and reboot, occurring approximately every 10 minutes.

Recommendation: Disable the “Boot USB Devices First” BIOS option (or the equivalent setting, depending on the hardware model and BIOS version).

In the diagram below, this setting is correctly configured with the checkbox left unchecked.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-maintenance-environment-known-issues.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-maintenance-environment-known-issues.png":::

## Initial creation failure

Issue: Image Url drop down not showing anything

Possible reason: Resource provider registration for Microsoft.AzureStackHCI is missing

Recommended action: Get it registered (already mentioned in the prereq) and retry.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-1.png":::

Issue: In site, configure new , resource group creation is failing. It means you have an RG policy and currently ZTP is not supported for this scenario.

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-2.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-2.png":::

Issue: A ARM template validation failure

Possible reason: You are trying this for the first time in this tenant

Recommendation: Wait 15 minutes and retry

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-3.png":::

Issue: Internal server error on site default

It means you have policies in your subscriptions on RG creation
1. If the policy is on naming conventions for RG, you can use configure button on sites to give a custom MRG name
1. If however you have policy on rg having tags or rg to be in a specific region, it is currently not supported

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-initial-creation-failure-4.png":::

Issue: Edge machine creation fails with
1. StorageAccountForbidden
There is a policy on storage account creation which we don't support or register provider for Microsoft.Storage is not done
1. DeviceOnboardingConflict
The feature registration and register provider for Microsoft.OnboardingService is missing
1. UpdateArcSettingDataFailed
Register provider for Microsoft.HybridCompute is missing
1. Any other failure, and all above cases please delete edge machine and try again once, post which if it still doesn't work, contact us.

Activity log of edge machine resource group and the managed resource group is a great insight into any of the above failures. Even when contacting us it would be really helpful if you can share the activity logs for both RGs

## Investigate a running system from the cloud

If portal shows ready to connect in edge machine, click on json view

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-1.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-1.png":::v

Go to the arc machine resource id -> Json view then 

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-2.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-investigate-running-system-from-cloud-2.png":::

if client public key is present means TO1 is present but somehow arc is not connecting:

2. If portal shows arc connection is done and waiting for extension installation and you want to see status of extension installation you can either see edge machine json or go to arc and see the extension tab
3. Is portal has mapped that stage also success, you can call the `<EDGE_MACHINE_ARM_ID>/jobs/provisionos` to see latest on OS provisioning and what is going on. 
4. if the stage has crossed download os, keep a look at the target arc resource, mentioned in edge machine again to see same behaviours as 1 and 2.

## Re-attempt a failed OS provisioning

Issue: Today in the first preview we do not have automatic retries from service in case installation of OS fails. We plan to add it in subsequent releases. 
We also plan to add customer supported retry from the portal but up until then one can do a retry themselves as well.

Steps are
Go to the edge machine
change the URL in the portal to add /jobs/provisionos post the edge machine url
See the Json view, the goal is to fire another job with same parameters

:::image type="content" source="media/simplified-machine-provisioning/troubleshooting-reattempt-failed-os-provisioning.png" alt-text="Screenshot showing TODO1" border="false" lightbox="media/simplified-machine-provisioning/troubleshooting-reattempt-failed-os-provisioning.png":::

Do a reput of the URL by only taking the os profile and user details object. Replace the `<PLACEHOLDERS>` with your values.

`PUT /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.AzureStackHCI/edgeMachines/<MACHINE_NAME>/jobs/ProvisionOs?api-version=2025-12-01-preview`

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

One can attempt to delete the edge machine resource at any time and it will take care of deleting relevant objects (like resources under MRG)
[SFF] Edge machine deletion is only blocked whenIt is part of a device pool which is part of an AKS cluster
The device is transitioning and not yet reached a stable state.
In the resource group where ZTP is run, a resource called MOBO broker is present (hidden), it cannot be deleted directly. (If a resource group is deleted then it takes care or it). Otherwise a configuration resource (also hidden) will show up in the resource group. If one deletes that resource MOBO resource is automatically deleted.
One should be careful in deletion the configuration resource as its deletion will bring down devices which have not yet reached "ready to cluster stage". Once it is reached that stage no impact. [SFF] it can even bring down a running cluster
