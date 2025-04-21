  Inthe world of container orchestration, Kubernetes has emerged as the de facto standard. One of its powerful features is autoscaling, which helps maintain application performance and optimize resource usage. In this post, we’ll dive into two main types of autoscaling in Kubernetes: Horizontal Pod Autoscaling (HPA) and Vertical Pod Autoscaling (VPA).

Introduction to Kubernetes Autoscaling
======================================

Autoscaling in Kubernetes ensures that your application can handle varying loads by automatically adjusting the number of running pods or the resources allocated to them. This dynamic adjustment is crucial for maintaining application performance, optimizing resource usage, and reducing operational costs.

![1_0wJBUCAWTLAe62PHmhoLOQ](https://github.com/user-attachments/assets/f11e6800-9a2f-4bde-9b05-acfd32fa0874)

Metric Server Setup
===================

Before testing Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA), it’s essential to have the Metrics Server installed in your Kubernetes cluster. The Metrics Server collects resource usage metrics from the cluster’s nodes and pods, which are necessary for autoscaling decisions.

You can install the Metrics Server using either a YAML manifest or the official Helm chart. To install the latest release of the Metrics Server from the YAML manifest, follow these steps:

1.  **Download the Components Manifest:** Use kubectl apply to download and apply the Components manifest directly from the latest release of the Metrics Server:

``` 
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```
This command fetches the YAML manifest for the latest release of the Metrics Server from its GitHub repository and applies it to your Kubernetes cluster.

2. Verify Installation: After applying the manifest, verify that the Metrics Server pods are running successfully. You can check the pods in the kube-system namespace:

```
kubectl get pods -n kube-system | grep metrics-server
```

You should see pods related to the Metrics Server running and ready.

3. Confirm Metrics Collection: Once the Metrics Server is up and running, you can confirm that it’s collecting metrics by querying the API. For example, you can retrieve the CPU and memory usage metrics for nodes and pods:
```
$ kubectl top nodes kubectl top pods --all-namespaces
```

If the Metrics Server is properly installed and functioning, you should see CPU and memory usage metrics for nodes and pods in your cluster.

With the Metrics Server installed and collecting metrics, you can proceed to test the Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA) functionalities in your Kubernetes cluster.


Horizontal Pod Autoscaling (HPA)
================================

What is HPA?
------------

Horizontal Pod Autoscaling (HPA) automatically scales the number of pods in a replication controller, deployment, or replica set based on observed CPU utilization (or other select metrics). This allows your application to scale out (add more pods) or scale in (reduce the number of pods) in response to load changes.

How Does HPA Work?
------------------

HPA operates by:

1.  **Monitoring Metrics:** HPA continuously monitors the specified metrics (e.g., CPU usage, memory usage, custom metrics).
    
2.  **Evaluating Thresholds:** It compares the current metric values against predefined thresholds.
    
3.  **Scaling Pods:** If the metric values exceed the thresholds, HPA increases the number of pods. Conversely, if the values drop below the thresholds, it decreases the number of pods.

Advantages of HPA
-----------------

*   **Elasticity**: Automatically adjusts the number of pods to meet current demand.
    
*   **Cost-Efficiency:** Optimizes resource usage by adding or removing pods as needed.
    
*   **Performance**: Helps maintain application performance during high traffic periods.
    

HPA Practical Example
=====================

To set up HPA, you need to define an HPA object in your Kubernetes cluster.

Let’s deploy an example using HPA.

**Step 1: Create a Deployment**
-------------------------------

Create a file named hpa-deployment.yaml with the following content:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "25m"
          limits:
            cpu: "200m"

```

Apply the deployment:

```
kubectl apply -f hpa-deployment.yaml
```

Step 2: Create a Service
------------------------

Create a file named nginx-service.yaml with the following content:

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  selector:
    app: nginx
```
Apply the service:

```
kubectl apply -f nginx-service.yaml
```

Step 3: Create an HPA
---------------------

Create a file named hpa.yaml with the following content:

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-deploy
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply hpa file:
```
kubectl apply -f hpa.yaml
```
With this setup, the HPA will automatically scale the number of nginx pods between 1 and 10 based on the CPU utilization, aiming to keep it around 50%.

Step 4: Testing
----------------
Now, let’s test the HPA by increasing the load on the hpa-deploy deployment. We'll use the following command to create a new pod that generates load on the target deployment and observe if the number of pods increases accordingly.

Pod status before generate load:
```
$ kubectl top po
NAME                          CPU(cores)   MEMORY(bytes)   
hpa-deploy-695d6d995c-lb7f2   0m           2Mi
```
Run this command to generate load:

```
$ kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx-svc; done"
```
This command will continuously generate requests to the nginx-svc service, thereby increasing the CPU utilization of the nginx pods.

To observe the CPU utilization and the status of the HPA, use the following commands:

Check the CPU utilization of the pods:

```
$ kubectl top po
NAME                          CPU(cores)   MEMORY(bytes)   
hpa-deploy-695d6d995c-5smnw   13m          3Mi             
hpa-deploy-695d6d995c-lb7f2   17m          2Mi             
load-generator                133m         0Mi
```
Monitor hpa status
```
$ kubectl get hpa -w
NAME        REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx-hpa   Deployment/hpa-deploy   0%/50%    1         10        1          101s
nginx-hpa   Deployment/hpa-deploy   80%/50%   1         10        1          105s
nginx-hpa   Deployment/hpa-deploy   68%/50%   1         10        2          2m
nginx-hpa   Deployment/hpa-deploy   54%/50%   1         10        2          2m15s
nginx-hpa   Deployment/hpa-deploy   42%/50%   1         10        2          2m30s
```

These commands will help you monitor the CPU usage and see if the HPA adjusts the number of pods in response to the increased load.

By following these steps, you can validate the functionality of the HPA and ensure it dynamically scales your application based on CPU utilization.


Vertical Pod Autoscaling (VPA)
==============================

What is VPA?
------------

Vertical Pod Autoscaling (VPA) adjusts the CPU and memory resource requests and limits for your pods. Instead of adding more pods, VPA modifies the resources allocated to existing pods.

How Does VPA Work?
------------------

VPA operates by:

1.  **Monitoring Resource Usage:** VPA continuously monitors the CPU and memory usage of the pods.
    
2.  **Recommending Resources:** Based on the usage patterns, VPA recommends resource adjustments.
    
3.  **Applying Changes:** The VPA can automatically apply the recommended changes to the pod specifications, typically requiring a pod restart.
    

Advantages of VPA
-----------------

*   **Resource Optimization:** Ensures pods have the right amount of resources, preventing over-provisioning or under-provisioning.
    
*   **Simplified Resource Management:** Reduces the need to manually adjust resource requests and limits.
    
*   **Improved Performance:** Helps maintain application performance by dynamically adjusting resources based on actual usage.
    

How VPA Adjusts Resources
=========================

1.  **Initial Adjustment:** When the VPA first starts managing a pod, it ensures that the pod’s resource requests are within the specified minAllowed and maxAllowed ranges. In your case, the minAllowed CPU is set to 100m, so the VPA initially adjusts the pod's CPU request to at least 100m to ensure it has a minimum baseline of resources.
    
2.  O**ngoing Monitoring and Adjustment:** After the initial adjustment, the VPA continues to monitor the CPU and memory usage of the pods. It collects data over time to understand the resource utilization patterns and make more informed recommendations.
    
3.  **Recommendation Based on Observed Usage:** The VPA makes recommendations based on observed usage. It doesn’t immediately jump to the maximum allowed (maxAllowed) value unless the observed usage justifies it. Instead, it incrementally adjusts the resource requests to align closely with actual usage patterns.
    

Practical Example of VPA
========================

Let’s deploy an example using VPA.

Step 1: Install VPA
-------------------

Before creating VPA objects first we need to install VPA in our cluster using the below steps:

Clone the VPA Source Code: Use Git to clone the VPA source code repository to your local machine. Run the following command:

```
git clone https://github.com/kubernetes/autoscaler.git
```
Navigate to the VPA Directory: Change your current directory to the autoscaler directory, which contains the VPA source code:

```
cd autoscaler
```
Run the Installation Script: Inside the autoscaler directory, run the installation script vpa-up.sh. This script sets up the necessary components for VPA in your Kubernetes cluster:

```
./hack/vpa-up.sh
```
This script will install the Vertical Pod Autoscaler components, including custom resources, controllers, and other required resources, into your Kubernetes cluster.

Verify Installation: Once the installation script has completed successfully, verify that the VPA components are installed and running in your cluster:

```
kubectl get pods -n kube-system | grep vpa
```
You should see pods related to Vertical Pod Autoscaler running in the kube-system namespace.

Step 2: Create a Deployment
----------------------------
Create a file named vpa-deployment.yaml with the following content:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: high-cpu-utilization-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cpu-utilization-app
  template:
    metadata:
      labels:
        app: cpu-utilization-app
    spec:
      containers:
      - name: cpu-utilization-container
        image: ubuntu
        command: ["/bin/sh", "-c", "apt-get update && apt-get install -y stress-ng && while true; do stress-ng --cpu 1; done"]
        resources:
          limits:
            cpu: "0.05"
          requests:
            cpu: "0.05"
```

This deployment repeatedly runs a CPU stress test using the `stress-ng` tool, consuming a small but continuous amount of CPU (limited to 0.05 cores) to simulate high CPU utilization.

Apply the deployment:

```
$ kubectl apply -f vpa-deployment.yaml
```

Step 3: Create a VPA
--------------------
Create a file named vpa.yaml with the following content:

```
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: stress-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: high-cpu-utilization-deployment
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 200m  #maximum vpa will be allocating this many cpus even if demand is higher.
          memory: 500Mi
        controlledResources: ["cpu", "memory"]
```
There are multiple valid options for updateMode in VPA. They are:

*   **Off** — VPA will only provide the recommendations, and it will not automatically change resource requirements.
    
*   **Initial** — VPA only assigns resource requests on pod creation and never changes them later.
    
*   **Recreate** — VPA assigns resource requests on pod creation time and updates them on existing pods by evicting and recreating them.
    
*   **Auto mode** — It recreates the pod based on the recommendation.
    

We increased the CPU metrics in the above demo and then manually applied the changes to scale the pod. We can do this automatically by using the **updateMode: “Auto”** parameter.

This Vertical Pod Autoscaler (VPA) automatically adjusts the CPU and memory requests and limits for the \`high-cpu-utilization-deployment\` to ensure efficient resource usage, within specified bounds (100m to 200m CPU and 50Mi to 500Mi memory). It targets all containers in the deployment for resource management.

Apply the VPA:

```
$ kubectl apply -f vpa.yaml
```
Step 4: Testing
--------------
First, let’s check the initial CPU utilization of the target pods:

```
$ kubect top po
NAME                                               CPU(cores)   MEMORY(bytes)   
high-cpu-utilization-deployment-78cc948dfb-fqbq9   50m          13Mi            
high-cpu-utilization-deployment-78cc948dfb-qtbt8   50m          9Mi
```

Verify the CPU requests for the pods:

```
$ kubectl get po -o jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}'
50m 50m
```

As seen, the two target pods are consuming their maximum CPU request since we are generating CPU load on them.

Next, check the VPA status:
```
$ kubectl get vpa     
NAME         MODE   CPU    MEM       PROVIDED   AGE
stress-vpa   Auto   100m   262144k   True       2m5s
```

Wait for some time and then check the pod status again:

```
$ kubectl get po                                                                    
NAME                                               READY   STATUS    RESTARTS   AGE
high-cpu-utilization-deployment-78cc948dfb-wvfkm   1/1     Running   0          66s
high-cpu-utilization-deployment-78cc948dfb-z7f87   1/1     Running   0
```
Check the updated CPU requests:

```
$ kubectl get po -o jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}'
100m 100m%
```
Monitor the updated CPU utilization:
```
$ kubectl top po                                                                    
NAME                                               CPU(cores)   MEMORY(bytes)   
high-cpu-utilization-deployment-78cc948dfb-wvfkm   101m         47Mi            
high-cpu-utilization-deployment-78cc948dfb-z7f87   100m         49Mi
```
As observed, the pods were restarted and their CPU request values increased from 50m to 100m due to the VPA’s minAllowed setting. The pods will continue generating load until they reach their limits.

The VPA will then increase the CPU request values to the recommended value of i.e., max of 200m as needed.
```
$ kubectl get po -o jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}'
126m 126m

$ kubectl top po                                                                    
NAME                                               CPU(cores)   MEMORY(bytes)   
high-cpu-utilization-deployment-78cc948dfb-7pqln   127m         48Mi            
high-cpu-utilization-deployment-78cc948dfb-8tn82   127m         47Mi

```

Why the CPU Request Increased to 126m Instead of 200m
-----------------------------------------------------

*   **Observed Usage:** The VPA’s recommendation of 126m for the CPU is based on the actual usage observed from the container. The VPA aims to provide just enough resources to meet the demand without over-provisioning. The 126m recommendation suggests that the current usage pattern of the container indicates that 126m is sufficient for its needs at this time.
    
*   **Incremental Adjustments:** The VPA makes incremental adjustments to avoid sudden large changes that might disrupt the application. It gradually increases the resource requests based on the monitored usage, ensuring stability and efficiency.
    
*   **Efficiency and Limits:** While the maxAllowed is set to 200m, this value is a ceiling rather than a target. The VPA only increases the CPU request to 200m if the observed usage indicates that such an increase is necessary. In above case, the observed usage suggested that 126m is adequate, so the VPA set the request to this value.
    

After some time:

```
$ kubectl get po -o jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}'
163m 163m
```
With this setup, the VPA will automatically adjust the CPU and memory requests for the high-cpu-utilization-deployment deployment, aiming to keep the resource usage within the specified limits.

**Exclude scaling for a specific container**
To exclude scaling for a specific container within a pod managed by the VerticalPodAutoscaler (VPA), you can set the mode for that container to “Off”. This ensures that the VPA does not adjust the resource requests for that particular container. Here’s an example configuration:

```
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
  .
  .
  .
  resourcePolicy:
    containerPolicies:
    - containerName: <container_name>
      mode: "Off"
```
In this example:

*    refers to the name of the container within the pod for which you want to exclude scaling.
    
*   mode: "Off" specifies that the VPA should not apply any autoscaling actions to this particular container.
    

By configuring the VPA in this way, you can ensure that only specific containers within your pods are subject to autoscaling, while others remain unaffected. This flexibility allows you to fine-tune the scaling behavior according to the requirements of your workload.

HPA vs. VPA: Which One to Use?
==============================

Choosing between HPA and VPA depends on your application’s requirements:

*   **HPA** is ideal for stateless applications that can easily scale horizontally by adding more pods. It’s useful when you need to handle variable workloads and ensure high availability.
    
*   **VPA** is suitable for applications where scaling horizontally is not feasible or when you need to optimize resource usage for individual pods. It’s beneficial for stateful applications or those with varying resource needs.
    

Conclusion
==========

Kubernetes autoscaling, through HPA and VPA, provides robust mechanisms to manage application scalability and resource optimization. By understanding and implementing these autoscaling strategies, you can ensure that your applications are resilient, performant, and cost-effective.

**Credits:**
[Anvesh Muppeda](https://github.com/anveshmuppeda)
