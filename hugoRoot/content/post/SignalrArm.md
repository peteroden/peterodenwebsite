---
title: "SignalR Service with ARM Template"
date: 2018-06-11
description: "Deploy Azure SignalR Service with ARM template with an Azure Function"
image: "/media/images/signalr.png"
categories: ["azure"]
tags: ["azure", "functions", "serverless", "signalr"]
draft: false
---

If you're like me the new Azure SignalR Service announced at Build is really interesting especially in regard to Serverless. So I've been doing some work with Azure SignalR Service and Azure Functions using Anthony Chu's custom extension (https://github.com/anthonychu/AzureAdvocates.WebJobs.Extensions.SignalRService). However, I really like to provide ARM templates along with my code in repos so others can deploy it and check it out with out having to manually deploy a bunch of services, but I couldn't find documentation anywhere for how to deploy SignalR Service via ARM template. So I set out to figure it out on my own and wanted to share what I learned. This creates the SignalR Service, Storage Account and Function App. It also populates the "AzureSignalRConnectionString" App Setting used to pass in the connection string for the SignalR Service connection string used by the extension.

So here's the template. Hope it's helpful!

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "functionName": {
            "type": "string",
            "metadata": {
                "description": "The name of the function app that you wish to create."
            }
        },
        "storageAccountType": {
            "defaultValue": "Standard_LRS",
            "allowedValues": [
                "Standard_LRS",
                "Standard_GRS",
                "Standard_ZRS"
            ],
            "type": "string",
            "metadata": {
                "description": "Storage Account type"
            }
        },
        "SignalRSkuName": {
            "type": "string",
            "allowedValues": [
                "Free_DS2",
                "Basic_DS2"
            ],
            "defaultValue": "Free_DS2",
            "metadata": {
                "description": "SignalR Service Sku"
            }
        },
        "signalRTier": {
            "type": "string",
            "allowedValues": [
                "Free",
                "Basic",
                "Premium"
            ],
            "defaultValue": "Free",
            "metadata": {
                "description": "SignalR Service Tier"
            }
        },
        "signalRCapacity": {
            "type": "int",
            "defaultValue": "1",
            "metadata": {
                "description": "Nuber of instances for the SignalR Service"
            }
        }
    },
    "variables": {
        "functionAppName": "[parameters('functionName')]",
        "appServicePlanName": "[parameters('functionName')]",
        "hostingPlanName": "[parameters('functionName')]",
        "storageAccountName": "[concat(parameters('functionName'), 'storage')]",
        "signalrName": "[concat(parameters('functionName'), 'signalr')]"
    },
    "resources": [{
            "type": "Microsoft.Storage/storageAccounts",
            "sku": {
                "name": "[parameters('storageAccountType')]"
            },
            "kind": "Storage",
            "name": "[variables('storageAccountName')]",
            "apiVersion": "[providers('Microsoft.Storage','storageAccounts').apiVersions[0]]",
            "location": "[resourceGroup().location]"
        },
        {
            "type": "Microsoft.Web/serverfarms",
            "sku": {
                "name": "Y1",
                "tier": "Dynamic",
                "size": "Y1",
                "family": "Y",
                "capacity": 0
            },
            "name": "[variables('hostingPlanName')]",
            "apiVersion": "[providers('Microsoft.Web', 'serverfarms').apiVersions[0]]",
            "location": "[resourceGroup().location]",
            "properties": {
                "name": "[variables('hostingPlanName')]"
            }
        },
        {
            "type": "Microsoft.Web/sites",
            "kind": "functionapp",
            "name": "[variables('functionAppName')]",
            "apiVersion": "[providers('Microsoft.Web', 'sites').apiVersions[0]]",
            "location": "[resourceGroup().location]",
            "properties": {
                "serverFarmId": "[variables('appServicePlanName')]",
                "siteConfig": {
                    "appSettings": [{
                            "name": "FUNCTIONS_EXTENSION_VERSION",
                            "value": "beta"
                        },
                        {
                            "name": "AzureWebJobsDashboard",
                            "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountName'), providers('Microsoft.Storage', 'storageAccounts').apiVersions[0]).keys[0].value)]"
                        },
                        {
                            "name": "AzureWebJobsStorage",
                            "value": "[concat('DefaultEndpointsProtocol=https;AccountName=', variables('storageAccountName'), ';AccountKey=', listKeys(variables('storageAccountName'), providers('Microsoft.Storage', 'storageAccounts').apiVersions[0]).keys[0].value)]"
                        },
                        {
                            "name": "AzureSignalRConnectionString",
                            "value": "[concat('Endpoint=https://', variables('signalrName'), '.service.signalr.net;AccountKey=', listKeys(variables('storageAccountName'), providers('Microsoft.SignalRService', 'SignalR').apiVersions[0]).keys[0].value)]"
                        }
                    ]
                }
            },
            "dependsOn": [
                "[resourceId('Microsoft.Storage/storageAccounts', variables('storageAccountName'))]",
                "[resourceId('Microsoft.Web/serverfarms', variables('hostingPlanName'))]",
                "[resourceId('Microsoft.SignalRService/SignalR', variables('signalrName'))]"
            ]
        },
        {
            "type": "Microsoft.SignalRService/SignalR",
            "sku": {
                "name": "[parameters('SignalRSkuName')]",
                "tier": "[parameters('signalRTier')]",
                "capacity": "[parameters('signalRCapacity')]",
                "size": "DS2"
            },
            "name": "[variables('signalrName')]",
            "apiVersion": "[providers('Microsoft.SignalRService', 'SignalR').apiVersions[0]]",
            "location": "eastus"
        }
    ]
}
```
