---
layout: post
title: Querying metadata in ECS
---

AWS provides two methods to access instance metadata (Instance Metadata Service): IMDSv2 and IMDSv1. While both versions are enabled by default, IMDSv2 offers enhanced security through session-oriented requests. This newer version requires a session token for authentication, unlike IMDSv1's simpler request/response method. For detailed examples and SDK compatibility information, refer to: [How to use the Instance Metadata Service to access instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html). 

## Task metadata available for ECS on Amazon EC2

If your Amazon ECS task is hosted on Amazon EC2, you can access task host metadata using the Instance Metadata Service (IMDS) endpoint: **`http://169.254.169.254/latest/meta-data/`**

You can access to the [instance metadata categories predefined](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-data-categories) or [dynamic data metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#dynamic-data-categories) generated when the instance is launched. 

For example to get the private IPv4 address of the instance "http://169.254.169.254/latest/meta-data/local-ipv4".

Using IMDSv1, no token is required:

```bash
curl http://169.254.169.254/latest/meta-data/local-ipv4
```` 
Output:

```
10.1.2.22
```


IMDSv2 uses token-backed sessions, querying this endpoint having IMDSv2 enabled it will throw HTTP 401 error:

```bash
curl http://169.254.169.254/latest/meta-data/local-ipv4 -v
```

Result:

```bash
* Trying 169.254.169.254:80...
* Connected to 169.254.169.254 (169.254.169.254) port 80 (#0)
> GET /latest/meta-data/local-ipv4 HTTP/1.1
> Host: 169.254.169.254
> User-Agent: curl/7.88.1
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Content-Length: 0
< Date: Wed, 11 Jun 2025 16:36:27 GMT
< Server: EC2ws
< Connection: close
< Content-Type: text/plain
<
* Closing connection 0
* 
```


When using IMDSv2 you need to get an `ec2-metadata-token` first and then use that token in the header to get the metadata information.

```bash
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"` && curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/local-ipv4
```
Result:

```
*   Trying 169.254.169.254:80...
* Connected to 169.254.169.254 (169.254.169.254) port 80 (#0)
> GET /latest/meta-data/local-ipv4 HTTP/1.1
> Host: 169.254.169.254
> User-Agent: curl/7.88.1
> Accept: */*
> X-aws-ec2-metadata-token: AQAEANJUjYU8d-jCP2lrxvPsC3ZgtDtbC41XuK1ddC_ONwyfidlOmw==
>
< HTTP/1.1 200 OK
< X-Aws-Ec2-Metadata-Token-Ttl-Seconds: 21600
< Content-Type: text/plain
< Accept-Ranges: none
< Last-Modified: Wed, 11 Jun 2025 15:38:21 GMT
< Content-Length: 10
< Date: Wed, 11 Jun 2025 15:49:23 GMT
< Server: EC2ws
< Connection: close
<
* Closing connection 0
10.1.2.194
```

> **Important**: This approach just works in your ECS Task is hosted on Amazon EC2 and not for Fargate.

## Using SDK for ECS on Amazon EC2

For example using JavaScript SDK with IMDSv1.


```javascript
const AWS = require("aws-sdk");
const meta = new AWS.MetadataService();

app.get('/', (req, res) => {
    const response = { message: 'Response back from Express REST API' };
    meta.request("/latest/meta-data/local-ipv4", function (err, data) {
        if (err) {
            response.error = "Error fetching local IPv4 address";
            res.status(500).json(response);
        } else {
            response.localIpv4 = data;
            res.json(response);
        }
    });
});
``` 

If IMDSv2 is enabled previous code will fail with the following error:

```
Response back from Express REST API
Error fetching local IPv4 address:  Error: null
    at IncomingMessage.<anonymous> (/app/node_modules/aws-sdk/lib/util.js:930:34)
    at IncomingMessage.emit (node:events:526:35)
    at IncomingMessage.emit (node:domain:488:12)
    at endReadableNT (node:internal/streams/readable:1408:12)
    at process.processTicksAndRejections (node:internal/process/task_queues:82:21) {
  statusCode: 401,
  retryable: false,
  time: 2025-06-11T17:30:50.372Z
}
```


If you are migrating from IMDSv1 to IMDSv2 using EC2 and JavaScript SDK you need to get first the token and then request the metadata endpoint as followed:

```javascript
const AWS = require("aws-sdk");
const meta = new AWS.MetadataService();

app.get('/', (req, res) => {
    console.log('Response back from Express REST API');
    // Fetching the local IPv4 address from AWS Metadata Service
    const response = { message: 'Response back from Express REST API' };
    meta.fetchMetadataToken(function (err, token) {
        if (err) {
            console.error("Error fetching metadata token: ", err);
            response.error = "Error fetching metadata token";
            res.status(500).json(response);
            return;
        } else {
            meta.request("/latest/meta-data/local-ipv4", { headers: { "x-aws-ec2-metadata-token": token } },
                function (err, data) {
                    if (err) {
                        console.error("Error fetching local IPv4 address: ", err);
                        response.error = "Error fetching local IPv4 address";
                        res.status(500).json(response);
                        return;
                    } else {
                        console.log("Local IPv4 address: ", data);
                        response.localIpv4 = data;
                        res.json(response);
                    }
                }
            );
        }
    });
});
```

**Important**: This approach just works in your ECS Task is hosted on Amazon EC2 and not for Fargate. 

If you run this code in ECS Fargate you won't find the endpoint, example:

```
Error fetching local IPv4 address: Error: connect EINVAL 169.254.169.254:80 - Local (0.0.0.0:0)
    at internalConnect (node:net:1090:16)
    at defaultTriggerAsyncIdScope (node:internal/async_hooks:464:18)
    at node:net:1315:9
    at process.processTicksAndRejections (node:internal/process/task_queues:77:11) {
        errno: -22,
        code: 'EINVAL',
        syscall: 'connect',
        address: '169.254.169.254',
        port: 80
    }

```

## Task metadata available for Amazon ECS tasks on Fargate

From ECS Fargate you can't reach the EC2 endpoint **`169.254.169.254`** directly. In order to query the metadata, you need to use the environment variables endpoints injected into each container in a task. There are different task metadata versions, the latest one is v4. Refer to [Amazon ECS task metadata endpoint version 4 for tasks on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v4-fargate.html). These endpoints can varies depending on the Fargate version you are using (v2,v3,v4).

Using Fargate you can use the following endpoint **`169.254.170.2/v4/metadata`**

Each endpoint will provide similar data or extra fields, review the json structure to get specific data. 

- `${ECS_CONTAINER_METADATA_URI_V4}` - Returns metadata for the container.
- `${ECS_CONTAINER_METADATA_URI_V4}/task` - Returns metadata for the task, including a list of the container IDs and names for all of the containers associated with the task. 
- `${ECS_CONTAINER_METADATA_URI_V4}/stats` - Returns Docker stats for the Docker container.


Displaying information using endpoint `ECS_CONTAINER_METADATA_URI`

```bash
echo  $ECS_CONTAINER_METADATA_URI
http://169.254.170.2/v3/c1b6da455882403b887d939f500f0884-2531612879
root@ip-10-1-2-22:/curl http://169.254.170.2/v3/c1b6da455882403b887d939f500f0884-2531612879

```
Output:

```json
{
    "DockerId": "c1b6da455882403b887d939f500f0884-2531612879",
    "Name": "nginx",
    "DockerName": "nginx",
    "Image": "nginx:latest",
    "ImageID": "sha256:6784fb0834aa7dbbe12e3d7471e69c290df3e6ba810dc38b34ae33d3c1c05f7d",
    "Labels": {
        "com.amazonaws.ecs.cluster": "arn:aws:ecs:us-east-1:000000000:cluster/ecs-fargate",
        "com.amazonaws.ecs.container-name": "nginx",
        "com.amazonaws.ecs.task-arn": "arn:aws:ecs:us-east-1:000000000:task/ecs-fargate/c1b6da455882403b887d939f500f0884",
        "com.amazonaws.ecs.task-definition-family": "fargate-nginx",
        "com.amazonaws.ecs.task-definition-version": "5"
    },
    "DesiredStatus": "RUNNING",
    "KnownStatus": "RUNNING",
    "Limits": {
        "CPU": 2
    },
    "CreatedAt": "2025-06-11T15:25:12.692595904Z",
    "StartedAt": "2025-06-11T15:25:12.692595904Z",
    "Type": "NORMAL",
    "Networks": [
        {
            "NetworkMode": "awsvpc",
            "IPv4Addresses": [
                "10.1.2.22"
            ]
        }
    ]
}
```

From the previous output you can get the private ip assigned using this code:

```javascript
const axios = require('axios');

app.get('/fargate', async (req, res) => {
    console.log('Response back from Express REST API for Fargate');
    const response = { message: 'Response back from Express REST API for Fargate' };
    const metadataUrl = process.env.ECS_CONTAINER_METADATA_URI;

    if (!metadataUrl) {
        response.error = "ECS_CONTAINER_METADATA_URI_V4 environment variable not set";
        return res.status(500).json(response);
    }

    try {
        const metaRes = await axios.get(metadataUrl);
        const metaData = metaRes.data;
        response.metadata = metaData;

        // Extract private IP from Networks[0].IPv4Addresses[0]
        let privateIp = null;
        if (
            Array.isArray(metaData.Networks) &&
            metaData.Networks.length > 0 &&
            Array.isArray(metaData.Networks[0].IPv4Addresses) &&
            metaData.Networks[0].IPv4Addresses.length > 0
        ) {
            privateIp = metaData.Networks[0].IPv4Addresses[0];
        }
        response.privateIp = privateIp;

        res.json(response);
    } catch (err) {
        console.error("Error fetching Fargate metadata: ", err);
        response.error = "Error fetching Fargate metadata";
        res.status(500).json(response);
    }
});
```

# References

- [AWS.MetadataService.request does not support IMDSv2 out of the box](https://github.com/aws/aws-sdk-js/issues/3584)
- [Task metadata available for Amazon ECS tasks on EC2](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ec2-metadata.html) 
- [Amazon ECS task metadata endpoint version 4 for tasks on Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-metadata-endpoint-v4-fargate.html)