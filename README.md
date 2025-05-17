
# Business Scaling Operator

This project demonstrates a custom Kubernetes operator that uses business metrics to dynamically scale a target deployment. Unlike the standard HPA that relies solely on CPU and memory, this operator queries an external API (or any metric source) to adjust the replica count of a given deployment based on real-world business metrics (for example, the number of active users). The operator is implemented using [Kopf](https://kopf.readthedocs.io/en/stable/), the Kubernetes Operator Pythonic Framework, and is designed to run continuously inside a Kubernetes cluster.

## Use Case

Traditional autoscaling uses resource metrics (CPU/memory), but many applications require scaling based on actual business events—such as a surge in active users or transactions per second. In this implementation, the operator watches a custom resource named `BusinessScaling` that defines:
- **Target Deployment:** The deployment that needs to be scaled.
- **Metric Source:** An external endpoint (e.g., a REST API returning active user counts).
- **Scale Thresholds:** The minimum and maximum user count thresholds along with the corresponding minimum and maximum replica counts.

Based on these parameters, the operator continuously polls the given metric source and adjusts the target deployment’s replica count accordingly.

## Prerequisites

- A running Kubernetes cluster (e.g., AKS)
- `kubectl` access configured
- Python 3.9+ and `pip`
- Docker (for containerizing the operator)
- Internet connectivity to your external metric API

## Components Overview

1. **CRD Definition:** Defines the `BusinessScaling` custom resource.
2. **Scaling Rule Instance:** A sample YAML instance of the `BusinessScaling` resource.
3. **Operator Code:** A Python script implemented with Kopf.
4. **Dockerfile:** Contains instructions to package the operator as a Docker image.
5. **Deployment YAML:** Deploys the operator inside the Kubernetes cluster.

## Step-by-Step Setup

### 1. Define the CRD

Create a file named `business_scaling_crd.yaml` with the following content:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: businessscalings.example.com
spec:
  group: example.com
  names:
    kind: BusinessScaling
    plural: businessscalings
    singular: businessscaling
  scope: Cluster
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
```

Apply the CRD with:

```bash
kubectl apply -f business_scaling_crd.yaml
```

### 2. Create a Scaling Rule Instance

Create a file named `scaling_rule.yaml` with the following content:

```yaml
apiVersion: example.com/v1
kind: BusinessScaling
metadata:
  name: scale-app-based-on-users
spec:
  targetDeployment: "payment-service"
  metricSource: "https://api.example.com/active-users"
  scaleThresholds:
    minUsers: 1000
    maxUsers: 5000
    minReplicas: 2
    maxReplicas: 10
```

Apply this scaling rule with:

```bash
kubectl apply -f scaling_rule.yaml
```

### 3. Write the Operator Using Kopf

Create a file named `operator.py` with the following content:

```python
import kopf
import kubernetes.client
import requests
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)

# Load Kubernetes configuration (when running inside the cluster, use load_incluster_config())
kubernetes.config.load_kube_config()
apps_v1 = kubernetes.client.AppsV1Api()

@kopf.timer("example.com", "v1", "businessscalings", interval=60)
def dynamic_scaling(spec, name, namespace, **kwargs):
    deployment_name = spec.get("targetDeployment")
    metric_url = spec.get("metricSource")
    thresholds = spec.get("scaleThresholds", {})
    min_users = thresholds.get("minUsers", 1000)
    max_users = thresholds.get("maxUsers", 5000)
    min_replicas = thresholds.get("minReplicas", 2)
    max_replicas = thresholds.get("maxReplicas", 10)

    try:
        response = requests.get(metric_url)
        active_users = response.json().get("active_users", 0)
    except Exception as e:
        logging.error(f"Error fetching metrics: {e}")
        return

    # Calculate the desired replica count based on the active user metric.
    if active_users < min_users:
        replicas = min_replicas
    elif active_users > max_users:
        replicas = max_replicas
    else:
        # Linear scaling calculation between min and max thresholds.
        replicas = int((active_users - min_users) / (max_users - min_users) * (max_replicas - min_replicas) + min_replicas)

    try:
        deployment = apps_v1.read_namespaced_deployment(deployment_name, namespace)
        deployment.spec.replicas = replicas
        apps_v1.patch_namespaced_deployment(deployment_name, namespace, deployment)
        logging.info(f"Scaled {deployment_name} to {replicas} replicas (active users: {active_users}).")
    except Exception as e:
        logging.error(f"Error scaling deployment {deployment_name}: {e}")
```

This operator uses Kopf's `@kopf.timer` decorator to execute the scaling logic every 60 seconds, ensuring continuous monitoring of the business metric.

### 4. Containerize the Operator

Create a file named `Dockerfile` with the following content:

```dockerfile
FROM python:3.9

WORKDIR /app
COPY operator.py .

# Install dependencies
RUN pip install kopf kubernetes requests

# Run the operator using Kopf when the container starts
CMD ["kopf", "run", "operator.py"]
```

Build and push your Docker image (replace `myrepo` with your container registry):

```bash
docker build -t myrepo/business-scaling-operator:latest .
docker push myrepo/business-scaling-operator:latest
```

### 5. Deploy the Operator in Kubernetes

Create a file named `operator_deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: business-scaling-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: business-scaling-operator
  template:
    metadata:
      labels:
        app: business-scaling-operator
    spec:
      containers:
        - name: operator
          image: myrepo/business-scaling-operator:latest
          resources:
            limits:
              memory: "512Mi"
              cpu: "500m"
```

Apply the deployment with:

```bash
kubectl apply -f operator_deployment.yaml
```

This deploys the operator into your cluster as a pod that continuously runs the scaling logic.

### 6. Testing and Monitoring

1. **Check that the operator pod is running:**

   ```bash
   kubectl get pods
   ```

2. **View the operator logs to monitor scaling events:**

   ```bash
   kubectl logs -f deployment/business-scaling-operator
   ```

3. **Update or reapply the scaling rule (if necessary):**

   ```bash
   kubectl apply -f scaling_rule.yaml
   ```

4. **Monitor the target deployment (in this example, `payment-service`) to verify that its replica count adjusts based on the active user metric.**

## Additional Enhancements

- When running inside your cluster, switch to using `kubernetes.config.load_incluster_config()` for improved security.
- Integrate with Prometheus or another monitoring solution to fetch advanced metrics.
- Enhance error handling, logging, and possibly introduce distributed tracing.
- Refine the scaling logic to implement more sophisticated algorithms as per your business needs.

## Conclusion

This operator demonstrates how you can extend Kubernetes' native autoscaling capabilities by leveraging custom business metrics. By using Kopf, you can rapidly iterate on operator functionality to meet your specific application requirements.

For more information, check out:
- [Kopf Documentation](https://kopf.readthedocs.io/en/stable/)
- [Kubernetes CRD Documentation](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Docker Documentation](https://docs.docker.com/)

