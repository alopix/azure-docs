---
title: Azure Resource Manager deployment modes | Microsoft Docs
description: Describes how to specify whether to use a complete or incremental deployment mode with Azure Resource Manager.
services: azure-resource-manager
documentationcenter: na
author: tfitzmac

ms.service: azure-resource-manager
ms.devlang: na
ms.topic: conceptual
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 01/24/2019
ms.author: tomfitz
---
# Azure Resource Manager deployment modes

When deploying your resources, you specify that the deployment is either an incremental update or a complete update.  The primary difference between these two modes is how Resource Manager handles existing resources in the resource group that aren't in the template. The default mode is incremental.

[!INCLUDE [updated-for-az](../../includes/updated-for-az.md)]

## Incremental and complete deployments

For both modes, Resource Manager tries to create all resources specified in the template. If the resource already exists in the resource group and its settings are unchanged, no operation is taken for that resource. If you change the property values for a resource, the resource is updated with those new values. If you try to update the location or type of an existing resource, the deployment fails with an error. Instead, deploy a new resource with the location or type that you need.

In complete mode, Resource Manager **deletes** resources that exist in the resource group but aren't specified in the template. Resources that are specified in the template, but not deployed because a [condition](resource-manager-templates-resources.md#condition) evaluates to false, aren't deleted.

In incremental mode, Resource Manager **leaves unchanged** resources that exist in the resource group but aren't specified in the template. When redeploying a resource in incremental mode, specify all property values for the resource, not just the ones you're updating. If you don't specify certain properties, Resource Manager interprets the update as overwriting those values.

## Example result

To illustrate the difference between incremental and complete modes, consider the following scenario.

**Resource Group** contains:

* Resource A
* Resource B
* Resource C

**Template** contains:

* Resource A
* Resource B
* Resource D

When deployed in **incremental** mode, the resource group has:

* Resource A
* Resource B
* Resource C
* Resource D

When deployed in **complete** mode, Resource C is deleted. The resource group has:

* Resource A
* Resource B
* Resource D

## Set deployment mode

To set the deployment mode when deploying with PowerShell, use the `Mode` parameter.

```azurepowershell-interactive
New-AzResourceGroupDeployment `
  -Mode Complete `
  -Name ExampleDeployment `
  -ResourceGroupName ExampleResourceGroup `
  -TemplateFile c:\MyTemplates\storage.json
```

To set the deployment mode when deploying with Azure CLI, use the `mode` parameter.

```azurecli-interactive
az group deployment create \
  --name ExampleDeployment \
  --mode Complete \
  --resource-group ExampleGroup \
  --template-file storage.json \
  --parameters storageAccountType=Standard_GRS
```

When using a [linked or nested template](resource-group-linked-templates.md), you must set the `mode` property to `Incremental`. Only root-level templates support the complete deployment mode.

```json
"resources": [
  {
      "apiVersion": "2017-05-10",
      "name": "linkedTemplate",
      "type": "Microsoft.Resources/deployments",
      "properties": {
          "mode": "Incremental",
          <nested-template-or-external-template>
      }
  }
]
```

## Next steps

* To learn about creating Resource Manager templates, see [Authoring Azure Resource Manager templates](resource-group-authoring-templates.md).
* To learn about deploying resources, see [Deploy an application with Azure Resource Manager template](resource-group-template-deploy.md).
* To view the operations for a resource provider, see [Azure REST API](/rest/api/).
