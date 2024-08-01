# GPU Slicing

Research an option to cut GPU costs for AI workload by enabling GPU Slicing

GPU slicing is a technique that allows multiple AI workloads to share a single GPU, thereby optimizing resource utilization and reducing costs. By partitioning a GPU into smaller, independent slices, we can run multiple containers or pods concurrently on the same GPU without the need for each workload to have a dedicated GPU. This is particularly beneficial in environments with GPU-intensive AI workloads, as it maximizes the efficiency of GPU usage and reduces overall infrastructure costs.

For more information about GPU Slicing, refer to the [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html), which is built on the [Kubernetes device plugin framework](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/).

## Calculation Example

### Without GPU Slicing:
Assume we have 4 AI workloads, each requiring a full GPU. If we use an instance type like [p3.2xlarge (with one GPU)](https://aws.amazon.com/ec2/instance-types/p3/), we would need 4 such instances.
- Instance type: p3.2xlarge
- Hourly cost per instance: $3.06 (on-demand EC2 instance)
- Total instances needed: 4
- Total hourly cost: 4 * $3.06 (on-demand EC2 instance) = $12.24

### With GPU Slicing:
By enabling GPU slicing, we can run multiple workloads on a single GPU. Let’s assume each workload now uses 25% of a GPU.
- Instance type: p3.2xlarge
- Hourly cost per instance: $3.06 (on-demand EC2 instance)
- Total instances needed: 1 (as each GPU can handle 4 workloads)
- Total hourly cost: 1 * $3.06 (on-demand EC2 instance) = $3.06

### Cost Savings Calculation:
- Without GPU Slicing: $12.24 per hour
- With GPU Slicing: $3.06 per hour
- Savings: $12.24 - $3.06 = $9.18 per hour
- Percentage Savings: (9.18 / 12.24) * 100 = 75%

By enabling GPU slicing, we can potentially save 75% on GPU costs, which is significant for long-running or large-scale deployments.

## Performance Impact of GPU Slicing

However, introducing GPU Slicing may potentially affect performance depending on the workload’s GPU usage pattern of ML models.

### Workload Characteristics:
- If the model consistently requires high GPU resources, slicing may lead to performance degradation.
- If the model uses GPU resources in bursts, slicing can be efficient without significant performance loss.

### Resource Contention:
- Multiple workloads sharing GPU memory bandwidth can lead to contention, affecting performance.
- Sharing GPU compute units may lead to reduced performance if all workloads require peak compute power simultaneously.

We should keep in mind all these potential risks. 

### Predict Performance Impact and Obtain Existing Performance Metrics

If the analysis of existing performance shows the possibility to introduce GPU Slicing, the next step is to analyze current performance and resource requirements of ML models to predict performance impact and obtain some current performance metrics. 

This will help compare performance after introducing GPU Slicing to have a real picture of results and make a final decision on whether to expand GPU Slicing configuration outside of the test environment.

### Tools for Analyzing ML Models: 
- [NVIDIA Nsight](https://developer.nvidia.com/nsight-systems)
- [TensorFlow Profiler](https://www.tensorflow.org/tensorboard/tensorboard_profiling_keras)

### Testing from QA side

Run QA performance tests before and after introducing GPU Slicing to obtain information about performance degradation. Monitor key metrics before and after introducing GPU Slicing.

These steps can provide insightful information about the right configuration for improvement, and stakeholders can make decisions based on these results and financial reports.

## PoC of GPU Slicing Implementation

The next step is to implement GPU Slicing in the test environment to obtain real results and compare them with current results. Ideally, this should reduce costs without impacting performance. Provide the testing results to stakeholders so they have all the information needed to make a final decision about implementing GPU Slicing on the whole system or specific parts of it.


### Steps to Enable GPU Slicing on AWS EKS 

To enable GPU Slicing on your AWS EKS clusters with specific configurations for p3.2xlarge and p3.8xlarge Instances, as an example, follow next steps:

1. **Create the Namespace for GPU Operator**
   
   ```sh
   kubectl create namespace gpu-operator
   ```

2. **Add NVIDIA Helm Repository and Install GPU Operator**

   ```sh
   helm repo add nvidia https://nvidia.github.io/gpu-operator
   helm repo update
   ```

3. **Create a ConfigMap for Time-Slicing Configuration**

   Create a file named `time-slicing-config.yaml` with the following content:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: time-slicing-config
   data:
     p3.2xlarge: |-
       version: v1
       flags:
         migStrategy: none
       sharing:
         timeSlicing:
           resources:
           - name: nvidia.com/gpu
             replicas: 4
     p3.8xlarge: |-
       version: v1
       flags:
         migStrategy: none
       sharing:
         timeSlicing:
           resources:
           - name: nvidia.com/gpu
             replicas: 8
   ```

   Apply the ConfigMap:

   ```sh
   kubectl create -n gpu-operator -f time-slicing-config.yaml
   ```

4. **Install the GPU Operator with the ConfigMap**

   ```sh
   helm install gpu-operator nvidia/gpu-operator \
       -n gpu-operator \
       --set devicePlugin.config.name=time-slicing-config
   ```

Next, we have 2 options - manual configure labels for GPU nodes or configure Node Templates in [Karpenter](https://karpenter.sh/).  

5. **Option 1. Apply Node-Specific Labels manually**

   Label our nodes to apply the specific time-slicing configurations.

   ```sh
   # Label for p3.2xlarge nodes
   kubectl label nodes <node-name-1> <node-name-2> nvidia.com/device-plugin.config=p3.2xlarge

   # Label for p3.8xlarge nodes
   kubectl label nodes <node-name-3> nvidia.com/device-plugin.config=p3.8xlarge
   ```

5. **Option 2. Configure Node Templates in Karpenter**

This is just a simple example because Karpenter has different features which we can utilize for cost optimization. 

Create a file named `karpenter-node-template-p3-2xlarge.yaml` with the following content:

```
apiVersion: karpenter.sh/v1beta1
kind: Provisioner
metadata:
  name: gpu-provisioner-p3-2xlarge
spec:
  constraints:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["p3.2xlarge"]
  labels:
    karpenter.sh/capacity-type: "ON_DEMAND"
    nvidia.com/device-plugin.config: "p3.2xlarge"  
  limits:
    resources:
      cpu: 1000
      memory: 2000Gi
  provider:
    amiFamily: "AL2"
    tags:
      karpenter.sh/provisioner-name: "gpu-provisioner-p3-2xlarge"
      Name: "gpu-provisioner-p3-2xlarge"
```

and a file named `karpenter-node-template-p3-8xlarge.yaml` with the following content:

```
apiVersion: karpenter.sh/v1beta1
kind: Provisioner
metadata:
  name: gpu-provisioner-p3-8xlarge
spec:
  constraints:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["p3.8xlarge"]
  labels:
    karpenter.sh/capacity-type: "ON_DEMAND"
    nvidia.com/device-plugin.config: "p3.8xlarge"  
  limits:
    resources:
      cpu: 1000
      memory: 2000Gi
  provider:
    amiFamily: "AL2"
    tags:
      karpenter.sh/provisioner-name: "gpu-provisioner-p3-8xlarge"
      Name: "gpu-provisioner-p3-8xlarge"
```

Apply the node templates:

```
kubectl apply -f karpenter-node-template-p3-2xlarge.yaml
kubectl apply -f karpenter-node-template-p3-8xlarge.yaml
```

6. **Verify the GPU Time-Slicing Configuration**

   To ensure the configuration has been applied correctly, check the node descriptions:

   ```sh
   kubectl describe node <node-name>
   ```

   We should see the `nvidia.com/gpu.replicas` and `nvidia.com/gpu.product` labels reflecting the correct configuration.

7. **Deploy and Test a Workload**

   Create a test deployment to verify that GPU time-slicing is working:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: time-slicing-verification
     labels:
       app: time-slicing-verification
   spec:
     replicas: 5
     selector:
       matchLabels:
         app: time-slicing-verification
     template:
       metadata:
         labels:
           app: time-slicing-verification
       spec:
         tolerations:
           - key: nvidia.com/gpu
             operator: Exists
             effect: NoSchedule
         containers:
           - name: cuda-sample-vector-add
             image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
             command: ["/bin/bash", "-c", "--"]
             args:
               - while true; do /cuda-samples/vectorAdd; done
             resources:
               limits:
                 nvidia.com/gpu: 1
   ```

   Apply the deployment:

   ```sh
   kubectl apply -f time-slicing-verification.yaml
   ```

   Verify that the pods are running and the logs indicate successful GPU usage:

   ```sh
   kubectl get pods
   kubectl logs deploy/time-slicing-verification
   ```

   Clean up the deployment after verification:

   ```sh
   kubectl delete -f time-slicing-verification.yaml
   ```

## References 
- [Time-Slicing GPUs in Kubernetes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html)
- [Supporting Multi-Instance GPUs (MIG) in Kubernetes](https://docs.google.com/document/d/1mdgMQ8g7WmaI_XVVRrCvHPFPOMCm5LQD5JefgAh6N8g/edit?usp=sharing)
- [NVIDIA Kubernetes Device Plugin](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/k8s-device-plugin)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Time-Slicing GPUs in Kubernetes GitHub repository](https://github.com/yandex-cloud-examples/yc-mk8s-gpu-time-slicing)


