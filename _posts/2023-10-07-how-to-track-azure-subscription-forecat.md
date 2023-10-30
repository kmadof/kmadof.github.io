---
layout: post
title: How to track Azure subscription forecast?
date: 2023-10-07
excerpt_separator:  <!--more-->
featured_image: /assets/images/posts/2023/laptop-with-financial-chart.jpeg
tags: DevOps Azure FinOps AzureCostManagement AzureCostCLI
featured: true
hidden: true
---

Keeping cost under control is not easy task. Different teams have different needs and it's hard to predict how much resources they will need. In this article I will show you how to track Azure subscription forecast using Azure Cost Management and Billing API.

<!--more-->

### Azure Cost CLI

There is a tool called [Azure Cost CLI](https://github.com/mivano/azure-cost-cli) which allows you to get cost of your Azure subscription. The subscription view provide you with the following information:
- current cost
- forecasted cost
- cost by service name
- cost by location

![Azure CLI view](/assets/images/posts/2023/azure-cost-cli.png)

I created an Azure DevOps template which allows you to run this tool in your pipeline.

```yaml
{% raw %}
parameters:
- name: environment
  type: string
- name: groups
  type: object
  default: []
- name: poolName
  type: string
  default: DevLinux

stages:
- stage: ${{ parameters.environment }}'
  pool:
    name: ${{ parameters.poolName }}
  condition: and(succeeded(), or(eq(variables['build.sourceBranch'], 'refs/heads/master'), ne(variables['Build.Reason'], 'PullRequest'), eq(variables['Build.Reason'], 'Manual')))
  jobs:
  - job: Run_Azure_Cost_CLI
    displayName: ${{ parameters.environment }} Cost
    variables:
    - name: TerraEnv
      value: ${{ lower(parameters.environment) }}
    - ${{ each value in parameters.groups }}:
      - group: ${{ value }}
    workspace:
      clean: all
    steps:
    - checkout: self
      persistCredentials: true
      clean: true

    - script: |
        az login --identity
        az account set --subscription $(TF_VAR_SUBSCRIPTION_ID)
      displayName: 'Login to Azure'

    - bash: |
        installedTools=$(dotnet tool list -g)

        echo "$installedTools"

        # Install the tool only if it isn't installed
        if [[ $installedTools != *"azure-cost-cli"* ]]; then
            dotnet tool install -g azure-cost-cli
        fi

        TOOL_PATH=$HOME/.dotnet/tools
        case ":$PATH:" in
          *:$TOOL_PATH:*) echo Terraform is already in the path;;
          *)
            echo "##[debug]Adding $TOOL_PATH to PATH" 
            echo "##vso[task.prependpath]$TOOL_PATH"
            export PATH=`pwd`:$PATH 
            ;;
        esac 
      displayName: 'Install Azure Cost CLI'

    - script: |
        azure-cost accumulatedCost --subscription $(TF_VAR_SUBSCRIPTION_ID)
      displayName: 'Run Azure Cost CLI'
{% endraw %}
```

And use it for each subscription you want to track:

```yaml
{% raw %}
name: 'Subscriptions Cost $(Date:yyyyMMdd)'
trigger:
  - master

schedules:
- cron: "0 10 * * *"
  displayName: Daily build
  branches:
    include:
    - master
  always: true

stages:
- template: display-cost.yml
  parameters:
    environment: Sub1
    groups:
    - Terraform-Azurerm-Provider-Settings-Sub1

- template: display-cost.yml
  parameters:
    environment: Sub2
    groups:
    - Terraform-Azurerm-Provider-Settings-Sub2
{% endraw %}
```

The tools is very useful but it has one drawback. You get a lot of details and you have to run in for each subscription. It would be nice to have a tool which will allow you to get forecast for all subscriptions in one place.

### Azure Cost Management API

We will use two endpoints of Azure Cost Management API:

- [Query](https://learn.microsoft.com/en-us/rest/api/cost-management/query/usage?tabs=HTTP) - for getting current cost
- [Forecast](https://learn.microsoft.com/en-us/rest/api/cost-management/forecast/usage?tabs=HTTP) - for getting forecast

Both endpoints return data for individual resources and we need to aggregate them.

And to get data for all subscriptions we need to iterate over them. We can do this using Azure CLI and command `az account list`. We will use `homeTenantId` to filter subscriptions for current tenant.


Here is the script which allows you to get current cost and forecast for all subscriptions in your tenant:

```bash
#!/bin/bash

# Based on this
# https://github.com/Azure/azure-cli/issues/17102

# Set the start and end dates in YYYY-MM-DD format
startDate="2023-09-01"
endDate="2023-09-30"

echo "Start date: $startDate"
echo "End date: $endDate"

# Set the query to get the cost data
query='{
    "type": "ActualCost",
    "timeframe": "Custom",
    "timePeriod": {
        "from": "'"$startDate"'",
        "to": "'"$endDate"'"
    },
    "dataset": {
        "granularity": "Monthly",
        "aggregation": {
            "totalCost": {
                "name": "Cost",
                "function": "Sum"
            }
        },
        "grouping": [
            {
                "type": "Dimension",
                "name": "SubscriptionId"
            },
            {
                "type": "Dimension",
                "name": "BillingMonth"
            }
        ]
    }
}'

# Get the current month and year
current_month=$(date +%m)
current_year=$(date +%Y)

# Calculate the first day of the month
first_day_of_month="${current_year}-${current_month}-01"

# Calculate the last day of the month
last_day_of_month=$(gdate -d "${first_day_of_month} +1 month -1 day" +%Y-%m-%d)

echo "First day of the current month: ${first_day_of_month}"
echo "Last day of the current month: ${last_day_of_month}"

queryForecast='{
    "type": "Usage",
    "timeframe": "Custom",
    "timePeriod": {
        "from": "'"$first_day_of_month"'",
        "to": "'"$last_day_of_month"'"
    },
    "dataset": {
        "granularity": "Monthly",
        "aggregation": {
            "totalCost": {
                "name": "Cost",
                "function": "Sum"
            }
        }
    },
    "includeActualCost": true,
    "includeFreshPartialCost": true
}'

currentTenant=$(az account show --query 'homeTenantId' -o tsv)

for id in $(az account list --query "[?homeTenantId=='$currentTenant'].{Id:id}" -o tsv); do
    subscriptionName=$(az account subscription show --subscription-id $id  | jq '.displayName')
    echo "========================================================="
    echo -e "\033[32mSubscription name: $subscriptionName\033[0m"
    subscriptionId=$id

    authToken=$(az account get-access-token --subscription $subscriptionId --resource=https://management.azure.com/ --query accessToken -o tsv)

    # Call the Cost Management API to get the cost data
    response=$(curl -s -X POST -H "Authorization: Bearer $authToken" -H "Content-Type: application/json" -d "$query" "https://management.azure.com/subscriptions/$subscriptionId/providers/Microsoft.CostManagement/query?api-version=2023-03-01")
    
    previousMonth=$(echo $response | jq '.properties.rows[] | first ' | awk '{sum=sum+$0} END{print sum}')
    echo "Previous month: $previousMonth"

    responseForecast=$(curl -s -X POST -H "Authorization: Bearer $authToken" -H "Content-Type: application/json" -d "$queryForecast" "https://management.azure.com/subscriptions/$subscriptionId/providers/Microsoft.CostManagement/forecast?api-version=2023-03-01")
    
    forecast=$(echo $responseForecast | jq '.properties.rows[] | first ' | awk '{sum=sum+$0} END{print sum}')
    echo "Forecast: $forecast"

    sleep 30s
done
```

I hardcoded start and end date to get data for September 2023. You can change it to get data for different period.

As a result you will get something like this:

```
=========================================================
Subscription name: "Sub1"
Previous month: 100.45
Forecast: 53.8079
WARNING: Command group 'account subscription' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
=========================================================
Subscription name: "Sub2"
Previous month: 2334.31
Forecast: 2478.87
WARNING: Command group 'account subscription' is experimental and under development. Reference and support levels: https://aka.ms/CLI_refstatus
=========================================================
```

### Summary

In this article I showed you how to track Azure subscription forecast using Azure Cost Management and Billing API. I hope you will find it useful.

And what do you think about putting this script in Azure DevOps pipeline or maybe Azure Function? Please share your opinions in comments.
