---
layout: post
title: How to Configure LogMonitor and CloudWatch Logs for ECS Windows Tasks
---

Windows applications often use different logging mechanisms than their Linux counterparts. Instead of relying on `STDOUT`, Windows apps typically utilize Windows-specific logging methods, such as `ETW`, the `Event Log`, or `custom log files` like `IIS logs`. If you’re running a Windows container in an ECS cluster and want to capture Windows and IIS logs in `CloudWatch`, you’ll need to implement `Log Monitor` instrumentation. This setup redirects IIS and system logs to `STDOUT`, allowing the awslogs driver to automatically capture and forward them to `CloudWatch` Logs.


## Setting Up Log Monitor and CloudWatch for IIS Logs

Follow these steps to configure Log Monitor and send IIS logs to CloudWatch:

1. **Identify the Log Providers**. 

    Determine the providers to include in the configuration file using:

    `logman query providers | findstr "<GUID or Provider Name>"` 

    For IIS, you can use:

    `IIS: WWW Server` with GUID `3A2A4E84-4C21-4981-AE10-3FDA0D9B0F83`

2. **Create the LogMonitorConfig.json file**.

    This file specifies which logs to capture. Below is an example configuration capturing system logs, application logs, and IIS logs:

    ```json
    {
      "LogConfig": {
        "sources": [
          {
            "type": "EventLog",
            "startAtOldestRecord": true,
            "eventFormatMultiLine": false,
            "channels": [
              {
                "name": "system",
                "level": "Information"
              },
              {
                "name": "application",
                "level": "Error"
              }
            ]
          },
          {
            "type": "File",
            "directory": "c:\\inetpub\\logs",
            "filter": "*.log",
            "includeSubdirectories": true,
            "includeFileNames": false
          },
          {
            "type": "ETW",
            "eventFormatMultiLine": false,
            "providers": [
              {
                "providerName": "IIS: WWW Server",
                "providerGuid": "3A2A4E84-4C21-4981-AE10-3FDA0D9B0F83",
                "level": "Information"
              },
              {
                "providerName": "Microsoft-Windows-IIS-Logging",
                "providerGuid": "7E8AD27F-B271-4EA2-A783-A47BDE29143B",
                "level": "Information"
              }
            ]
          }
        ]
      }
    }
    ```

3. **Download Log Monitor**
 
    Download [LogMonitor.exe](https://github.com/microsoft/windows-container-tools/releases) from the GitHub repository binaries.

4. **Update Your Dockerfile**

    Integrate Log Monitor into your Docker container. Refer to the usage guide [here]((https://github.com/microsoft/windows-container-tools/tree/main/LogMonitor#usage)).

    Example Dockerfile:

    ```bash
    FROM mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2022

    WORKDIR /LogMonitor

    COPY LogMonitorConfig.json LogMonitor.exe ./

    ENTRYPOINT ["C:\\LogMonitor\\LogMonitor.exe", "C:\\ServiceMonitor.exe", "w3svc"]
    ```

5. **Configure CloudWatch Logs in the Task Definition**

    Enable CloudWatch logs and set up the log driver in your ECS task definition:

    Task definition example:

    ```json
    "logConfiguration": {
        "logDriver": "awslogs",
          "options": {
            "awslogs-group": "/ecs/iis-logs",
            "awslogs-create-group": "true",
            "awslogs-region": "us-east-1",
            "awslogs-stream-prefix": "ecs"
         }
      }
    ```
6. **Rebuild and Deploy**
    - Rebuild the container image and push it to your ECR repository.
    - Update the ECS task definition to use the new image and deploy it to the cluster.

## Viewing Logs in CloudWatch

1. Open the `Amazon ECS console`.
2. Navigate to the cluster containing the task.
3. Select the `Tasks` tab and choose a task to view.
4. Expand the container details by clicking the arrow next to its name.
5. In the `Log Configuration section`, select `View logs in CloudWatch` to access the log stream.


### Example Outputs

- ETW and Log Entries:

  ![cloudwatch](/media/2025/01/iiscloudwatch-01.png)

- Application Exceptions:

  ![applicationerror](/media/2025/01/iiscloudwatch-02.png)
