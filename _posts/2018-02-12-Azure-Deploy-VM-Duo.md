---
layout: single
title:  "Azure ARM Template Deployment with Duo Authentication"
date:   2018-02-09 14:17:12 -0500
categories:
  - Azure
tags:
  - Azure
  - Security
---

The previous posts was on securing RDP with multi-factor authentication and can be found here: [Azure: Secure your IaaS VMs with Duo Security](http://erleonard.me/azure/AzureSecurity-Duo/)

Manually installing software once is ok but if anyone knows me, I automate everything I do to ensure that it is always consistent. With that in mind, I was thinking about how can I deploy the Duo Security agent as part of a base build.

For this example, I used the 101-vm-simple-windows QuickStart template found on GitHub and added the DSC extension to the template to install Duo Security. As well, I created a very simple PowerShell DSC script to download and install the agent with the required parameters. 

To make the deployment work, you will need the integration key (IKEY), secret key (SKEY), API Hostname (HOST) from the Duo Security Admin Portal to make the deployment successful.

Here is a sample deployment for PowerShell:

```powershell
$parameters = @{}
$parameters.Add("IKEY", "DIXXXXXXXXXXXXXXXXXXXX")
$parameters.Add("SKEY","xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx")
$parameters.Add("Host","api-xxxxxxxx.duosecurity.com")

New-AzureRmResourceGroupDeployment -Name vmDeployment -ResourceGroupName $ResourceGroup -TemplateParameterObject $parameters -TemplateUri https://raw.githubusercontent.com/erleonard/AzureARMTemplates/master/DuoSecurity/azuredeploy.json
```

**ARM Template**
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "adminUsername": {
      "type": "string",
      "metadata": {
        "description": "Username for the Virtual Machine."
      }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Password for the Virtual Machine."
      }
    },
    "dnsLabelPrefix": {
      "type": "string",
      "metadata": {
        "description": "Unique DNS Name for the Public IP used to access the Virtual Machine."
      }
    },
    "windowsOSVersion": {
      "type": "string",
      "defaultValue": "2016-Datacenter",
      "allowedValues": [
        "2008-R2-SP1",
        "2012-Datacenter",
        "2012-R2-Datacenter",
        "2016-Nano-Server",
        "2016-Datacenter-with-Containers",
        "2016-Datacenter"
      ],
      "metadata": {
        "description": "The Windows version for the VM. This will pick a fully patched image of this given Windows version."
      }
    },
    "_artifactsLocation": {
      "type": "string",
      "metadata": {
        "description": "The location of resources, such as templates and DSC modules, that the template depends on"
      },
      "defaultValue": "https://raw.githubusercontent.com/erleonard/AzureARMTemplates/master/DuoSecurity/DSC/install-duo.zip"
    },
    "IKEY": {
      "type": "string",
      "metadata": {
        "description": "Duo Security IKEY"
      }
    },
      "SKEY": {
      "type": "string",
      "metadata": {
        "description": "Duo Security SKEY"
      }
    },
      "HOST": {
      "type": "string",
      "metadata": {
        "description": "Duo Security host API"
      }
    }
  },
  "variables": {
    "storageAccountName": "[concat(uniquestring(resourceGroup().id), 'sawinvm')]",
    "nicName": "myVMNic",
    "addressPrefix": "10.0.0.0/16",
    "subnetName": "Subnet",
    "subnetPrefix": "10.0.0.0/24",
    "publicIPAddressName": "myPublicIP",
    "vmName": "SimpleWinVM",
    "virtualNetworkName": "MyVNET",
    "subnetRef": "[resourceId('Microsoft.Network/virtualNetworks/subnets', variables('virtualNetworkName'), variables('subnetName'))]",
    "vmExtensionName": "dscExtension"
    
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "name": "[variables('storageAccountName')]",
      "apiVersion": "2016-01-01",
      "location": "[resourceGroup().location]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "Storage",
      "properties": {}
    },
    {
      "apiVersion": "2016-03-30",
      "type": "Microsoft.Network/publicIPAddresses",
      "name": "[variables('publicIPAddressName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "publicIPAllocationMethod": "Dynamic",
        "dnsSettings": {
          "domainNameLabel": "[parameters('dnsLabelPrefix')]"
        }
      }
    },
    {
      "apiVersion": "2016-03-30",
      "type": "Microsoft.Network/virtualNetworks",
      "name": "[variables('virtualNetworkName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "[variables('addressPrefix')]"
          ]
        },
        "subnets": [
          {
            "name": "[variables('subnetName')]",
            "properties": {
              "addressPrefix": "[variables('subnetPrefix')]"
            }
          }
        ]
      }
    },
    {
      "apiVersion": "2016-03-30",
      "type": "Microsoft.Network/networkInterfaces",
      "name": "[variables('nicName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[resourceId('Microsoft.Network/publicIPAddresses/', variables('publicIPAddressName'))]",
        "[resourceId('Microsoft.Network/virtualNetworks/', variables('virtualNetworkName'))]"
      ],
      "properties": {
        "ipConfigurations": [
          {
            "name": "ipconfig1",
            "properties": {
              "privateIPAllocationMethod": "Dynamic",
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses',variables('publicIPAddressName'))]"
              },
              "subnet": {
                "id": "[variables('subnetRef')]"
              }
            }
          }
        ]
      }
    },
    {
      "apiVersion": "2015-06-15",
      "type": "Microsoft.Compute/virtualMachines",
      "name": "[variables('vmName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))]",
        "[resourceId('Microsoft.Network/networkInterfaces/', variables('nicName'))]"
      ],
      "properties": {
        "hardwareProfile": {
          "vmSize": "Standard_A2"
        },
        "osProfile": {
          "computerName": "[variables('vmName')]",
          "adminUsername": "[parameters('adminUsername')]",
          "adminPassword": "[parameters('adminPassword')]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "MicrosoftWindowsServer",
            "offer": "WindowsServer",
            "sku": "[parameters('windowsOSVersion')]",
            "version": "latest"
          },
          "osDisk": {
            "name": "osdisk",
            "vhd": {
              "uri": "[concat(reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob, 'vhds/osdisk.vhd')]"
            },
            "caching": "ReadWrite",
            "createOption": "FromImage"
          },
          "dataDisks": [
            {
              "name": "datadisk1",
              "diskSizeGB": "100",
              "lun": 0,
              "vhd": {
                "uri": "[concat(reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob, 'vhds/datadisk1.vhd')]"
              },
              "createOption": "Empty"
            }
          ]
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces',variables('nicName'))]"
            }
          ]
        },
        "diagnosticsProfile": {
          "bootDiagnostics": {
            "enabled": "true",
            "storageUri": "[reference(resourceId('Microsoft.Storage/storageAccounts/', variables('storageAccountName'))).primaryEndpoints.blob]"
          }
        }
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(variables('vmName'),'/', variables('vmExtensionName'))]",
      "apiVersion": "2015-05-01-preview",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[concat('Microsoft.Compute/virtualMachines/', variables('vmName'))]"
      ],
      "properties": {
        "publisher": "Microsoft.Powershell",
        "type": "DSC",
        "typeHandlerVersion": "2.80",
        "autoUpgradeMinorVersion": true,
        "settings": {
          "ModulesUrl": "[parameters('_artifactsLocation')]",
          "ConfigurationFunction": "install-duo.ps1\\DuoInstall",
          "Properties": {
            "IKEY": "[parameters('IKEY')]",
            "SKEY": "[parameters('SKEY')]",
            "HOST": "[parameters('HOST')]"
          }
        },
        "protectedSettings": null
      }
    }
  ],
  "outputs": {
    "hostname": {
      "type": "string",
      "value": "[reference(variables('publicIPAddressName')).dnsSettings.fqdn]"
    }
  }
}
```

 Adding Duo Security to your base template doesn't take long to complete but by adding multi-factor authentication to your VM it makes it that much more secure.

I hope you enjoyed this article and as always the code is available on Github: [https://github.com/erleonard/AzureARMTemplates/tree/master/DuoSecurity](https://github.com/erleonard/AzureARMTemplates/tree/master/DuoSecurity)