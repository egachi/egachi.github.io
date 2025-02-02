In scenarios that you want to run a Daemonset, but want to run daemonset pods in specific nodes or avoid running pods without deleting the daemonset for a particular timeframe. You can use labes with nodeSelector. 

The nature of a DaemonSet was designed to run one pod per node in your Kubernetes cluster, if you have 3 nodes, you will see 3 daemonset pods running.

To effectively reduce the number of Daemonset pods to 0 without deleting the DaemonSet or to chose which node will be running the daemonset pods, it is recommended to use Node Selectors. Here are two ways to implement this solution:

## Setting Labels to Kubernetes Nodes

1. Add a label to all existing nodes. In this example I will use "fluent-bit=false" to control how many FluentBit Daemonset pods will be running in my nodes. To add a label use this command:

    `kubectl get nodes -o name | xargs -I{} kubectl label {} fluent-bit=false --overwrite`

    **Note**: You may need to rerun this command if new nodes are added to the cluster.

2. Verify the label change:
   
    `kubectl get nodes --show-labels`

3. Modify your DaemonSet manifest adding a new selector:

    ```yaml
    nodeSelector:
    kubernetes.io/os: linux
    fluent-bit: "true"
    ```

    For example:

    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluent-bit
      namespace: amazon-cloudwatch
      labels:
        k8s-app: fluent-bit
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      selector:
        matchLabels:
          k8s-app: fluent-bit
      template:
        metadata:
          labels:
            k8s-app: fluent-bit
            version: v1
            kubernetes.io/cluster-service: "true"
        spec:
          containers:
            - name: fluent-bit
              image: public.ecr.aws/aws-observability/aws-for-fluent-bit:2.32.4
              imagePullPolicy: Always
          nodeSelector:
            kubernetes.io/os: linux
            fluent-bit: "true"
    ```
4. Apply the updated configuration: 
    
    `kubectl apply -f <manifest-name>.yaml`

5. Verify that your Daemonset pods are not running.
6. To re-enable Daemonset pods in the future, you can update the node labels to a desired label and value. e.g. "fluent-bit=true":


## Patching Node Selectors 

Patching on the fly. In this case, the command is setting the nodeSelector to include a label "non-existing": "true", which means that the fluent-bit pods will only be scheduled on nodes that have this label.

```
kubectl -n amazon-cloudwatch patch daemonset fluent-bit -p '{
  "spec": {
    "template": {
      "spec": {
        "nodeSelector": {
          "non-existing": "true"
        }
      }
    }
  }
}'
```