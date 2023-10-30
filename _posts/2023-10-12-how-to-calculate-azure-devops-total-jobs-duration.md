---
layout: post
title: How to calculate Azure DevOps total jobs duration
date: 2023-10-12
excerpt_separator:  <!--more-->
featured_image: /assets/images/posts/2023/laptop-with-charts.jpeg
tags: DevOps AzureDevOps
featured: true
hidden: true
---

I am currently involved in migrating from Azure DevOps to GitHub. The reasons behind this move are definitely something for another article. However, in this piece, I want to focus on a specific task I've been assigned. GitHub Enterprise provides 50,000 minutes of free build time per month, which is substantial. Since we want to estimate price accurately, I need to calculate our current usage. Another challenge arises here: minute-to-minute usage is not always consistent. How is that possible? Well, we employ self-hosted agents, which vary in strength. When comparing [Github-hosted runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources) and Microsoft-hosted agent](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=yaml#hardware), it might seem like they operate on the same machine.

<!--more-->

![Github-hosted runners](/images/2023-10-12-github-hosted-runners.png)

<div class="note-box">
  <p>
    Microsoft-hosted agents that run Windows and Linux images are provisioned on Azure general purpose virtual machines with a 2 core CPU, 7 GB of RAM, and 14 GB of SSD disk space. These virtual machines are co-located in the same geography as your Azure DevOps organization.
  </p>
</div>

You're right if that's how you feel. Both run on [Standard_DS2_v2](https://learn.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series#dsv2-series). Is there something else? They use the same repository to build images for runners and agents called [runner-images](https://github.com/actions/runner-images). We use the same repository to create our images for self-hosted agents, and we also use Standard_DS2_v2. Knowing that, we can assume that minute to minute is equal. But how can we calculate that? After giving it some thought, I have an answer that I would like to share with you.

### Azure DevOps Analytics API

I found the [Azure DevOps Analytics API](https://learn.microsoft.com/en-us/azure/devops/report/analytics/analytics-query-parts?view=azure-devops&tabs=cloud) to be the most suitable for this task. It provides a set of endpoints that allow you to query for data about your organization. The version `4.0.0.` is currently in preview, so it might change in the future. The API is available for Azure DevOps Services, Azure DevOps Server 2022 and Azure DevOps Server 2019.

I utilize the Azure DevOps Analytics API to automate data retrieval and processing for a specific time range. THe script iterates monthly, clearing existing JSON files, constructs API URLs based on given start and end dates, and retrieves pipeline run activity results using a provided Personal Access Token (PAT). The script paginates through API responses, saving each chunk of data in separate JSON files.

```bash
#!/bin/bash

organization='org-name'
project='project-name'

pat="Personal Access Token"

start_date="20230101"
loop_end_date="20231030"

while [ "$start_date" -le "$loop_end_date" ]
do
    rm pipeline-run-activity-results-*.json

    echo "start_date=\"$start_date\""
    end_date=$(gdate -d "$start_date + 1 month - 1 day" +%Y%m%d)
    echo "end_date=\"$end_date\""

    # Counter for file names
    counter=1
    url="https://analytics.dev.azure.com/$organization/$project/_odata/v4.0-preview/PipelineRunActivityResults?%24apply=filter(PipelineRunCompletedDateSK%20ge%20$start_date%20AND%20PipelineRunCompletedDateSK%20le%20$end_date)&%24orderby=%20PipelineRunCompletedDateSK%20asc%20"
    # Loop until there are no more chunks
    while [ "$url" != "null" ]; do
        # Make a curl request and save the response in a file
        output_file="pipeline-run-activity-results-$counter.json"
        
        curl -s -u :$pat "$url" > "$output_file"

        # Extract the next link from the response
        next_link=$(cat "$output_file" | jq -r '.["@odata.nextLink"]')

        # Check if there is a next link, update the URL for the next iteration
        if [ "$next_link" != "null" ]; then
            url="$next_link"
            counter=$((counter + 1))
        else
            url="null"
        fi
    done

    ./total-duration.sh

    start_date=$(gdate -d "$start_date + 1 month" +%Y%m%d)
done
```

The `total-duration.sh` iterates through JSON files matching the pattern pipeline-run-activity-results-*.json, extracting ActivityDurationSeconds from each file, summing them up to calculate the total duration in seconds, and then converting it into minutes. 

```bash
#!/bin/bash

# Initialize total duration in seconds and minutes
total_duration_seconds=0
total_duration_minutes=0

# Loop through all JSON files in the current directory
for file in pipeline-run-activity-results-*.json; do
    # Extract ActivityDurationSeconds values from each file and sum them up
    duration_seconds=$(jq -r '.value[].ActivityDurationSeconds' "$file" | awk '{s+=$1} END {print s}')
    total_duration_seconds=$(awk "BEGIN {print $total_duration_seconds + $duration_seconds}")
done

# Calculate total duration in minutes
total_duration_minutes=$(awk "BEGIN {print $total_duration_seconds / 60}")

# Output the total duration in seconds and minutes
echo "Total Duration: $total_duration_seconds seconds"
echo "Total Duration: $total_duration_minutes minutes"

```
### Summary

In the past six months, our team's total usage of GitHub Actions, as indicated in the provided table, has consistently remained below 50,000 minutes. And tis means that we will not pay for additional minutes as we are in the middle of that limit.

| Start date | End date | Duration [mins]|
| -------- | -------- | -------- |
| 2023 05 01 | 2023 05 01| 24619|
| 2023 06 01 | 2023 06 30| 22912 |
| 2023 07 01 | 2023 07 31| 20601.3|
| 2023 08 01 | 2023 08 31| 22755.2|
| 2023 09 01 | 2023 09 30| 19027.8|
| 2023 10 01 | 2023 10 31| 19073.1|

Please share in comment below how many minutes you use per month. I am curious if you are close to the limit or not.