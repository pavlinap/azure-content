<properties 
	pageTitle="Provision a web app that uses a SQL Database" 
	description="Use an Azure Resource Manager template to deploy a web app that includes a SQL Database." 
	services="app-service" 
	documentationCenter="" 
	authors="tfitzmac" 
	manager="wpickett" 
	editor="jimbe"/>

<tags 
	ms.service="app-service" 
	ms.workload="na" 
	ms.tgt_pltfrm="na" 
	ms.devlang="na" 
	ms.topic="article" 
	ms.date="02/26/2016" 
	ms.author="tomfitz"/>

# Provision a web app with a SQL Database

In this topic, you will learn how to create an Azure Resource Manager template that deploys a web app and SQL Database. You will learn how to define which resources are deployed and 
how to define parameters that are specified when the deployment is executed. You can use this template for your own deployments, or customize it to meet your requirements.

For more information about creating templates, see [Authoring Azure Resource Manager Templates](../resource-group-authoring-templates.md).

For more information about deploying apps, see [Deploy a complex application predictably in Azure](app-service-deploy-complex-application-predictably.md).

For the complete template, see [Web App With SQL Database template](https://github.com/Azure/azure-quickstart-templates/blob/master/201-web-app-sql-database/azuredeploy.json).

[AZURE.INCLUDE [app-service-web-to-api-and-mobile](../../includes/app-service-web-to-api-and-mobile.md)] 

## What you will deploy

In this template, you will deploy:

- a web app
- SQL Database server
- SQL Database
- AutoScale settings
- Alert rules
- App Insights

To run the deployment automatically, click the following button:

[![Deploy to Azure](./media/app-service-web-arm-with-sql-database-provision/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2Fazure-quickstart-templates%2Fmaster%2F201-web-app-sql-database%2Fazuredeploy.json)

## Parameters to specify

[AZURE.INCLUDE [app-service-web-deploy-web-parameters](../../includes/app-service-web-deploy-web-parameters.md)]

### serverName

The name of the new database server to create.

    "serverName": {
      "type": "string"
    }

### serverLocation

The location of the database server. For best performance, this location should be the same as the location of the web app.

    "serverLocation": {
      "type": "string"
    }

### administratorLogin

The account name to use for the database server administrator.

    "administratorLogin": {
      "type": "string"
    }

### administratorLoginPassword

The password to use for the database server administrator.

    "administratorLoginPassword": {
      "type": "securestring"
    }

### databaseName

The name of the new database to create.

    "databaseName": {
      "type": "string"
    }

### collation

The database collation to use for governing the proper use of characters.

    "collation": {
      "type": "string",
      "defaultValue": "SQL_Latin1_General_CP1_CI_AS"
    }

### edition

The type of database to create.

    "edition": {
      "type": "string",
      "defaultValue": "Standard",
      "metadata": {
        "description": "The type of database to create. The available options are: Web, Business, Basic, Standard, and Premium."
      }
    }

### maxSizeBytes

The maximum size, in bytes, for the database.

    "maxSizeBytes": {
      "type": "string",
      "defaultValue": "1073741824"
    }

### requestedServiceObjectiveName

The name corresponding to the performance level for edition. 

    "requestedServiceObjectiveName": {
      "type": "string",
      "defaultValue": "S0",
      "metadata": {
        "description": "The name corresponding to the performance level for edition. The available options are: Shared, Basic, S0, S1, S2, S3, P1, P2, and P3."
      }
    }


## Resources to deploy

### SQL Server and Database

Creates a new SQL Server and database. The name of the server is specified in the **serverName** parameter and the location specified in the **serverLocation** parameter. When creating the new server,
you must provide a login name and password for the database server administrator. 

    {
      "name": "[parameters('serverName')]",
      "type": "Microsoft.Sql/servers",
      "location": "[parameters('serverLocation')]",
      "apiVersion": "2014-04-01-preview",
      "properties": {
        "administratorLogin": "[parameters('administratorLogin')]",
        "administratorLoginPassword": "[parameters('administratorLoginPassword')]",
        "version": "12.0"
      },
      "resources": [
        {
          "name": "[parameters('databaseName')]",
          "type": "databases",
          "location": "[parameters('serverLocation')]",
          "apiVersion": "2014-04-01-preview",
          "dependsOn": [
            "[concat('Microsoft.Sql/servers/', parameters('serverName'))]"
          ],
          "properties": {
            "edition": "[parameters('edition')]",
            "collation": "[parameters('collation')]",
            "maxSizeBytes": "[parameters('maxSizeBytes')]",
            "requestedServiceObjectiveName": "[parameters('requestedServiceObjectiveName')]"
          }
        },
        {
          "apiVersion": "2014-04-01-preview",
          "dependsOn": [
            "[concat('Microsoft.Sql/servers/', parameters('serverName'))]"
          ],
          "location": "[parameters('serverLocation')]",
          "name": "AllowAllWindowsAzureIps",
          "properties": {
            "endIpAddress": "0.0.0.0",
            "startIpAddress": "0.0.0.0"
          },
          "type": "firewallrules"
        }
      ]
    },


[AZURE.INCLUDE [app-service-web-deploy-web-host](../../includes/app-service-web-deploy-web-host.md)]


### Web app

    {
      "apiVersion": "2015-08-01",
      "name": "[parameters('siteName')]",
      "type": "Microsoft.Web/sites",
      "location": "[parameters('siteLocation')]",
      "dependsOn": [
        "[concat('Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
      ],
      "tags": {
        "[concat('hidden-related:', resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]": "empty"
      },
      "properties": {
        "name": "[parameters('siteName')]",
        "serverFarmId": "[parameters('hostingPlanName')]"
      },
      "resources": [
        {
          "apiVersion": "2015-08-01",
          "type": "config",
          "name": "connectionstrings",
          "dependsOn": [
            "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
          ],
          "properties": {
            "DefaultConnection": {
              "value": "[concat('Data Source=tcp:', reference(concat('Microsoft.Sql/servers/', parameters('serverName'))).fullyQualifiedDomainName, ',1433;Initial Catalog=', parameters('databaseName'), ';User Id=', parameters('administratorLogin'), '@', parameters('serverName'), ';Password=', parameters('administratorLoginPassword'), ';')]",
              "type": "SQLAzure"
            }
          }
        }
      ]
    },

### AutoScale

    {
      "apiVersion": "2014-04-01",
      "name": "[concat(parameters('hostingPlanName'), '-', resourceGroup().name)]",
      "type": "Microsoft.Insights/autoscalesettings",
      "location": "East US",
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]": "Resource"
      },
      "dependsOn": [
        "[concat('Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
      ],
      "properties": {
        "profiles": [
          {
            "name": "Default",
            "capacity": {
              "minimum": 1,
              "maximum": 2,
              "default": 1
            },
            "rules": [
              {
                "metricTrigger": {
                  "metricName": "CpuPercentage",
                  "metricResourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]",
                  "timeGrain": "PT1M",
                  "statistic": "Average",
                  "timeWindow": "PT10M",
                  "timeAggregation": "Average",
                  "operator": "GreaterThan",
                  "threshold": 80
                },
                "scaleAction": {
                  "direction": "Increase",
                  "type": "ChangeCount",
                  "value": 1,
                  "cooldown": "PT10M"
                }
              },
              {
                "metricTrigger": {
                  "metricName": "CpuPercentage",
                  "metricResourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]",
                  "timeGrain": "PT1M",
                  "statistic": "Average",
                  "timeWindow": "PT1H",
                  "timeAggregation": "Average",
                  "operator": "LessThan",
                  "threshold": 60
                },
                "scaleAction": {
                  "direction": "Decrease",
                  "type": "ChangeCount",
                  "value": 1,
                  "cooldown": "PT1H"
                }
              }
            ]
          }
        ],
        "enabled": false,
        "name": "[concat(parameters('hostingPlanName'), '-', resourceGroup().name)]",
        "targetResourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
      }
    },

### Alert rules for status codes 403 and 500's, High CPU, and HTTP Queue Length 

    //Alert-Rules --> 5xx
    {
      "apiVersion": "2014-04-01",
      "name": "[concat('ServerErrors ', parameters('siteName'))]",
      "type": "Microsoft.Insights/alertrules",
      "location": "East US",
      "dependsOn": [
        "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
      ],
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]": "Resource"
      },
      "properties": {
        "name": "[concat('ServerErrors ', parameters('siteName'))]",
        "description": "[concat(parameters('siteName'), ' has some server errors, status code 5xx.')]",
        "isEnabled": false,
        "condition": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.ThresholdRuleCondition",
          "dataSource": {
            "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleMetricDataSource",
            "resourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]",
            "metricName": "Http5xx"
          },
          "operator": "GreaterThan",
          "threshold": 0,
          "windowSize": "PT5M"
        },
        "action": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleEmailAction",
          "sendToServiceOwners": true,
          "customEmails": [ ]
        }
      }
    },
    //Alert-Rules --> 403
    {
      "apiVersion": "2014-04-01",
      "name": "[concat('ForbiddenRequests ', parameters('siteName'))]",
      "type": "Microsoft.Insights/alertrules",
      "location": "East US",
      "dependsOn": [
        "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
      ],
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]": "Resource"
      },
      "properties": {
        "name": "[concat('ForbiddenRequests ', parameters('siteName'))]",
        "description": "[concat(parameters('siteName'), ' has some requests that are forbidden, status code 403.')]",
        "isEnabled": false,
        "condition": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.ThresholdRuleCondition",
          "dataSource": {
            "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleMetricDataSource",
            "resourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]",
            "metricName": "Http403"
          },
          "operator": "GreaterThan",
          "threshold": 0,
          "windowSize": "PT5M"
        },
        "action": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleEmailAction",
          "sendToServiceOwners": true,
          "customEmails": [ ]
        }
      }
    },
    //Alert-Rules --> High CPU
    {
      "apiVersion": "2014-04-01",
      "name": "[concat('CPUHigh ', parameters('hostingPlanName'))]",
      "type": "Microsoft.Insights/alertrules",
      "location": "East US",
      "dependsOn": [
        "[concat('Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
      ],
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]": "Resource"
      },
      "properties": {
        "name": "[concat('CPUHigh ', parameters('hostingPlanName'))]",
        "description": "[concat('The average CPU is high across all the instances of ', parameters('hostingPlanName'))]",
        "isEnabled": false,
        "condition": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.ThresholdRuleCondition",
          "dataSource": {
            "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleMetricDataSource",
            "resourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]",
            "metricName": "CpuPercentage"
          },
          "operator": "GreaterThan",
          "threshold": 90,
          "windowSize": "PT15M"
        },
        "action": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleEmailAction",
          "sendToServiceOwners": true,
          "customEmails": [ ]
        }
      }
    },
    //Alert-Rules --> HTTP Queue Length
    {
      "apiVersion": "2014-04-01",
      "name": "[concat('LongHttpQueue ', parameters('hostingPlanName'))]",
      "type": "Microsoft.Insights/alertrules",
      "location": "East US",
      "dependsOn": [
        "[concat('Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
      ],
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]": "Resource"
      },
      "properties": {
        "name": "[concat('LongHttpQueue ', parameters('hostingPlanName'))]",
        "description": "[concat('The HTTP queue for the instances of ', parameters('hostingPlanName'), ' has a large number of pending requests.')]",
        "isEnabled": false,
        "condition": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.ThresholdRuleCondition",
          "dataSource": {
            "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleMetricDataSource",
            "resourceUri": "[concat(resourceGroup().id, '/providers/Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]",
            "metricName": "HttpQueueLength"
          },
          "operator": "GreaterThan",
          "threshold": 100,
          "windowSize": "PT5M"
        },
        "action": {
          "odata.type": "Microsoft.Azure.Management.Insights.Models.RuleEmailAction",
          "sendToServiceOwners": true,
          "customEmails": [ ]
        }
      }
    },

### App Insights

    {
      "apiVersion": "2014-04-01",
      "name": "[parameters('siteName')]",
      "type": "Microsoft.Insights/components",
      "location": "Central US",
      "dependsOn": [
        "[concat('Microsoft.Web/sites/', parameters('siteName'))]"
      ],
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]": "Resource"
      },
      "properties": {
        "ApplicationId": "[parameters('siteName')]"
      }
    }

## Commands to run deployment

[AZURE.INCLUDE [app-service-deploy-commands](../../includes/app-service-deploy-commands.md)]

### PowerShell

    New-AzureRmResourceGroupDeployment -TemplateUri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/201-web-app-sql-database/azuredeploy.json

### Azure CLI

    azure group deployment create --template-uri https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/201-web-app-sql-database/azuredeploy.json


 
