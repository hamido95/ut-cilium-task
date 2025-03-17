# Cilium Configuration for Maximum Stability in a Production Kubernetes Environment 

## Introduction 
This document explains the configuration for Cilium (version 1.17.2) to ensure maximum stability in a production Kubernetes environment. The purpose is to optimize Cilium for high workloads by focusing on resource management, high availability, network and security optimization, and monitoring. These changes are informed by best practices from the Cilium documentation and tailored to meet the requirements of a stable production deployment. 

## Analysis and Adjustments 
1. **Resource Management**

    To maintain stability under high load, resource requests and limits were added to key Cilium components. This ensures the Kubernetes scheduler allocates enough CPU and memory, preventing interruptions and performance issues. 
   - **Cilium Agent:**
     - **Requests:** 100m CPU, 256Mi memory
     - **Limits:** 1 CPU, 1Gi memory 
     - **Rationale:** The agent runs on every node, managing networking and security policies. Adequate resources are critical for handling high traffic volumes.
   - **Cilium Operator:**  
     - **Requests:** 100m CPU, 128Mi memory 
     - **Limits:** 500m CPU, 512Mi memory 
     - **Rationale:** The operator oversees cluster-wide operations and requires resources for efficiency, especially with high availability (replicas: 2). 
   - **Hubble Relay:** 
     - **Requests:** 50m CPU, 64Mi memory 
     - **Limits:** 200m CPU, 256Mi memory 
     - **Rationale:** The relay aggregates observability data and needs resources to process metrics reliably under load. 

   - **Hubble UI Backend and Frontend:**  
     - **Requests:** 50m CPU, 64Mi memory (each)
     - **Limits:** 200m CPU, 256Mi memory (each) 
     - **Rationale:** These components provide network visibility and require resources to serve requests consistently.
  
    **Note:** These values are baselines and should be adjusted based on cluster size, workload, and performance testing in your specific environment.

2. **Replica Settings and High Availability**

    High availability ensures continuous operation during failures. 
    - **Cilium Operator:** Configured with replicas: 2 for redundancy. 
    - **Cilium Agent:** Runs as a DaemonSet (one instance per node), inherently providing high availability. 
    - **Hubble Relay and UI:** Configured as single replica deployments. Given their roles, single instances with sufficient resources works well for most production scenarios. 

3. **Network and Security Optimization** 

    The existing network settings were reviewed and found to be optimized for stability and performance: 

    - **BPF Masquerade:** Enabled for pod-to-external traffic. 
    - **IPAM Mode:** Set to kubernetes for native IP allocation. 
    - **Kube-Proxy Replacement:** Enabled for better performance. 
    - **Routing Mode:** Set to native with ipv4NativeRoutingCIDR aligned to the cluster’s pod CIDR (By default "10.244.0.0/24” in Minikube). 
    - **Auto Direct Node Routes:** Enabled for efficient node-to-node routing. 

4. **Logging and Monitoring Enhancements** 

    monitoring is essential for quick issue resolution. We enable it in the values.yaml file and then we should scrape them with our monitoring stack.
    - **Prometheus:** Enabled with ServiceMonitor for metrics collection. 
    - **Hubble:** Enabled with detailed metrics (DNS, drops, TCP, etc.) for network observability. 
    - **Debug Logging:** Disabled to reduce log noise in production.

## Testing and Validation
The configuration was validated on a local cluster using Minikube, followed by a load test.

### Cluster Setup
- **Minikube:**
  - **Download Minikube:**
    ```
    curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
    sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
    ```
  - **Start the Cluster**
    ```
    minikube start --cpus=4 --memory=8192 --disk-size=40g
    ```

### Cilium Installation
- **Add the Cilium Helm repository:**
  ```
  helm repo add cilium https://helm.cilium.io/
  helm repo update
  ```
- **Pull the repository**
  ```
  git clone https://github.com/hamido95/ut-cilium-task.git
  ```
- **Installing Cilium with our values.yaml:**
  ```
  helm install cilium cilium/cilium --version 1.17.2 --values values.yaml --namespace kube-system
  ```
- **Verify Cilium Installation**

  Check that all Cilium pods are running:
  ```
  kubectl get pods -n kube-system -l k8s-app=cilium
  ```

### Cilium Connectivity Check
once cilium installed successfully we can ensure our configuration is correct by using a connectivity check:
  ```
  kubectl create namespace cilium-test
  kubectl apply -f connectivity-check.yaml -n cilium-test
  ```
and then:
```
  watch kubectl get pods -n cilium-test
```
All pods should show a Running status.

**Congradulations! Your test cluster is now ready for the sample load test!**

## Sample Load Test
To test your cluster’s networking and performance, we’ll deploy a sample application (httpbin) and generate load using the k6 tool.

1. **Deploy the Sample Application**
    ```
    kubectl apply -f httpbin.yaml
    ```

    This deploys httpbin with 3 replicas and exposes it as a service.

2. **Run a Load Test with k6**

    We’ll use k6 inside the cluster to generate HTTP requests to the httpbin service.
    ```
    kubectl apply -f k6-pod.yaml
    ```
    Attach to the k6 Pod and Run the Test:
    ```
    kubectl attach k6-load-tester -i
    ```
    Once attached, paste this k6 script into the terminal and press Ctrl+D to execute:
    ```
    import http from 'k6/http';
    import sleep from 'k6';

    export let options = {
      vus: 100,
      duration: '30s',
    };

    export default function () {
      http.get('http://httpbin.default.svc.cluster.local/get');
      sleep(1);
    }
    ```
    This script simulates 100 users sending HTTP GET requests to httpbin every second for 30 seconds, generating a moderate load on your cluster.
3. **Observe Network Traffic with Hubble**
  
    After the test runs, k6 will output statistics like request rate, latency, and errors. Now we can check our monitoring stack for tracking or we can also use Cilium’s Hubble to observe network traffic by port forwarding it.
    ```
    kubectl port-forward -n kube-system svc/hubble-ui 12000:8081
    ```
    open the page http://localhost:12000 in your browser. You should see a screen with an invitation to select a namespace. Look for dropped packets or errors to assess how your cluster handled the load. 

