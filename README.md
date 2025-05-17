# Business Scaling Operator

This project demonstrates a custom Kubernetes operator that uses business metrics to dynamically scale a target deployment. Unlike the standard HPA that relies on CPU and memory, this operator queries an external API (or any metric source) to adjust the replica count of a given deployment based on the number of active users (or other business-driven measurements).

The operator is implemented using [Kopf](https://kopf.readthedocs.io/en/stable/), the Kubernetes Operator Pythonic Framework, and is designed to run continuously inside a Kubernetes cluster.

---

## Use Case

**Problem:**  
Traditional autoscaling uses resource metrics (CPU/memory), but many applications benefit more from scaling based on real business events—such as the number of active users or transactions per second.

**Solution:**  
The operator watches a custom resource (`BusinessScaling`) that defines:
- Which deployment to target.
- The external metric source (e.g., a REST API returning active user counts).
- Thresholds to determine the minimum and maximum replica counts.

It then periodically queries the external metric source and patches the target deployment’s replica count accordingly.

---

## Prerequisites

- A running Kubernetes cluster (e.g., AKS)
- `kubectl` access configured
- Python 3.9+ and `pip`
- Docker (for containerizing the operator)
- Internet connectivity to your external metric API

---

## Components

1. **CRD Definition:** Defines the `BusinessScaling` custom resource.
2. **Scaling Rule:** A sample YAML instance of the `BusinessScaling` CRD.
3. **Operator Code:** Python script implemented with Kopf.
4. **Dockerfile:** To containerize the operator.
5. **Deployment YAML:** Deploy the operator as a Kubernetes Deployment.

---

## Step 1: Define the CRD

Create a file named `business_scaling_crd.yaml`:

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
