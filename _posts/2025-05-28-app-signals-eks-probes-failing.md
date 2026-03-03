

After Enabling Cloud Watch Application Signals, EKS readiness probes can fail

When adding OTEL auto instrumentation for do
      annotations:
            instrumentation.opentelemetry.io/inject-dotnet: "true";
            instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-musl-x64" # for Alpine Linux (linux-musl-x64) based images

The readiness probe consistently fails with a connection refused error:
  
    Warning  Unhealthy  6s    kubelet            Readiness probe failed: Get "http://10.1.1.94:5126/readiness": dial tcp 10.1.1.94:5126: connect: connection refused


The issue persists across different EKS deployment modes (Auto Mode, EKS-managed, and Self-Managed CloudWatch Add-on)


k describe pod my-deployment-5c6b88c484-ljdj8
Name:             my-deployment-5c6b88c484-ljdj8
Namespace:        default
Priority:         0
Service Account:  default
Node:             ip-10-1-2-33.ec2.internal/10.1.2.33
Start Time:       Fri, 25 Apr 2025 16:01:49 -0600
Labels:           app=my-pod
                  pod-template-hash=5c6b88c484
Annotations:      instrumentation.opentelemetry.io/inject-dotnet: true
                  instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: linux-musl-x64
Status:           Running
IP:               10.1.2.75
IPs:
  IP:           10.1.2.75
Controlled By:  ReplicaSet/my-deployment-5c6b88c484
Init Containers:
  opentelemetry-auto-instrumentation-dotnet:
    Container ID:  containerd://bf46f416895cfbc5cf4254aa568382542d4ea71fdbc90a605ce85f1c00f4c0bc
    Image:         602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/observability/adot-autoinstrumentation-dotnet:v1.6.0
    Image ID:      602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/observability/adot-autoinstrumentation-dotnet@sha256:f40f92d709418206c9106d7dd36b02161da043b704bf27197a131b97dc94547b
    Port:          <none>
    Host Port:     <none>
    Command:
      cp
      -a
      /autoinstrumentation/.
      /otel-auto-instrumentation-dotnet
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Fri, 25 Apr 2025 16:01:49 -0600
      Finished:     Fri, 25 Apr 2025 16:01:50 -0600
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        50m
      memory:     128Mi
    Environment:  <none>
    Mounts:
      /otel-auto-instrumentation-dotnet from opentelemetry-auto-instrumentation-dotnet (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rckgq (ro)
Containers:
  dotnet-alpine:
    Container ID:   containerd://28962ff97e889f164817cc3257acaecc02e0d0f2a414d089d15a32d65c6f436e
    Image:          egachi/my-test:latest
    Image ID:       docker.io/egachi/my-test@sha256:ef2d9a70bf7db4212d62c7f0a3587da7f21e4ff6038fe331eb1db36496c30349
    Port:           5126/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 25 Apr 2025 16:01:50 -0600
    Ready:          False
    Restart Count:  0
    Readiness:      http-get http://:5126/readiness delay=5s timeout=5s period=15s #success=1 #failure=3
    Environment:
      OTEL_AWS_APPLICATION_SIGNALS_ENABLED:            true
      OTEL_AWS_APPLICATION_SIGNALS_RUNTIME_ENABLED:    true
      OTEL_TRACES_SAMPLER_ARG:                         endpoint=http://cloudwatch-agent.amazon-cloudwatch:2000
      OTEL_TRACES_SAMPLER:                             xray
      OTEL_EXPORTER_OTLP_PROTOCOL:                     http/protobuf
      OTEL_EXPORTER_OTLP_ENDPOINT:                     http://cloudwatch-agent.amazon-cloudwatch:4316
      OTEL_EXPORTER_OTLP_TRACES_ENDPOINT:              http://cloudwatch-agent.amazon-cloudwatch:4316/v1/traces
      OTEL_AWS_APPLICATION_SIGNALS_EXPORTER_ENDPOINT:  http://cloudwatch-agent.amazon-cloudwatch:4316/v1/metrics
      OTEL_METRICS_EXPORTER:                           none
      OTEL_DOTNET_DISTRO:                              aws_distro
      OTEL_DOTNET_CONFIGURATOR:                        aws_configurator
      OTEL_LOGS_EXPORTER:                              none
      OTEL_DOTNET_AUTO_PLUGINS:                        AWS.Distro.OpenTelemetry.AutoInstrumentation.Plugin, AWS.Distro.OpenTelemetry.AutoInstrumentation
      CORECLR_ENABLE_PROFILING:                        1
      CORECLR_PROFILER:                                {918728DD-259F-4A6A-AC2B-B85E1B658318}
      CORECLR_PROFILER_PATH:                           /otel-auto-instrumentation-dotnet/linux-musl-x64/OpenTelemetry.AutoInstrumentation.Native.so
      DOTNET_STARTUP_HOOKS:                            /otel-auto-instrumentation-dotnet/net/OpenTelemetry.AutoInstrumentation.StartupHook.dll
      DOTNET_ADDITIONAL_DEPS:                          /otel-auto-instrumentation-dotnet/AdditionalDeps
      OTEL_DOTNET_AUTO_HOME:                           /otel-auto-instrumentation-dotnet
      DOTNET_SHARED_STORE:                             /otel-auto-instrumentation-dotnet/store
      OTEL_SERVICE_NAME:                               my-deployment
      OTEL_RESOURCE_ATTRIBUTES_POD_NAME:               my-deployment-5c6b88c484-ljdj8 (v1:metadata.name)
      OTEL_RESOURCE_ATTRIBUTES_NODE_NAME:               (v1:spec.nodeName)
      OTEL_PROPAGATORS:                                tracecontext,baggage,b3,xray
      OTEL_RESOURCE_ATTRIBUTES:                        com.amazonaws.cloudwatch.entity.internal.service.name.source=K8sWorkload,k8s.container.name=dotnet-alpine,k8s.deployment.name=my-deployment,k8s.namespace.name=default,k8s.node.name=$(OTEL_RESOURCE_ATTRIBUTES_NODE_NAME),k8s.pod.name=$(OTEL_RESOURCE_ATTRIBUTES_POD_NAME),k8s.replicaset.name=my-deployment-5c6b88c484,service.version=latest
    Mounts:
      /otel-auto-instrumentation-dotnet from opentelemetry-auto-instrumentation-dotnet (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rckgq (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       False
  ContainersReady             False
  PodScheduled                True
Volumes:
  kube-api-access-rckgq:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
  opentelemetry-auto-instrumentation-dotnet:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:   200Mi
QoS Class:       Burstable
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  74s                default-scheduler  Successfully assigned default/my-deployment-5c6b88c484-ljdj8 to ip-10-1-2-33.ec2.internal
  Normal   Pulled     75s                kubelet            Container image "602401143452.dkr.ecr.us-east-1.amazonaws.com/eks/observability/adot-autoinstrumentation-dotnet:v1.6.0" already present on machine
  Normal   Created    75s                kubelet            Created container opentelemetry-auto-instrumentation-dotnet
  Normal   Started    75s                kubelet            Started container opentelemetry-auto-instrumentation-dotnet
  Normal   Pulling    74s                kubelet            Pulling image "egachi/my-test:latest"
  Normal   Pulled     74s                kubelet            Successfully pulled image "egachi/my-test:latest" in 156ms (156ms including waiting). Image size: 52120699 bytes.
  Normal   Created    74s                kubelet            Created container dotnet-alpine
  Normal   Started    74s                kubelet            Started container dotnet-alpine
  Warning  Unhealthy  10s (x5 over 60s)  kubelet            Readiness probe failed: Get "http://10.1.2.75:5126/readiness": dial tcp 10.1.2.75:5126: connect: connection refused