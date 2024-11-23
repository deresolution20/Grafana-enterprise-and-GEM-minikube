
# Setup Minikube + Grafana Enterprise and GEM

This guide explains how to deploy Grafana Enterprise on a Minikube cluster. It includes all required setup steps, license management, and accessing Grafana via port forwarding. This was done on a Mac using the Homebrew package manager.

---

## Supporting Documentation

- [Generate Tokens for GEM](https://grafana.com/docs/enterprise-metrics/latest/tokengen/)
- [Set up GEM with Helm](https://grafana.com/docs/enterprise-metrics/latest/set-up/helm/)
- [Grafana Helm Chart](https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md)

---

## Prerequisites

Ensure the following are installed on your system:

1. **Minikube**
   ```bash
   brew install minikube
   ```

2. **kubectl**
   Kubernetes CLI for managing resources.
   ```bash
   brew install kubectl
   ```

3. **Helm**
   The package manager for Kubernetes.
   ```bash
   brew install helm
   ```

Ensure `kubectl` is configured to point to Minikube:
```bash
kubectl config use-context minikube
```

Start Minikube with enough resources:
```bash
minikube start --cpus=4 --memory=8192 --kubernetes-version=v1.28.3
```

---

## Step 1: Create Kubernetes Namespaces

Create namespaces for Grafana:
```bash
kubectl create namespace grafana
```

---

## Step 2: Create Grafana Enterprise License Secret

1. Save the Grafana Enterprise license to a file named `license_grafanaenterprise.jwt`.
2. Create a Kubernetes secret using this file:
   ```bash
   kubectl create secret generic grafana-license \
       --from-file=license.jwt=license_grafanaenterprise.jwt \
       -n grafana
   ```

---

## Step 3: Add the Grafana Helm Repository

Add and update the Grafana Helm chart repository:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## Step 4: Deploy Grafana Enterprise

Deploy Grafana Enterprise using Helm:
```bash
helm install grafana grafana/grafana \
   --namespace grafana \
   --set enterprise.enabled=true \
   --set enterprise.license.secretName=grafana-license \
   --set enterprise.license.secretKey=license.jwt \
   --set adminPassword=admin \
   --set service.type=NodePort \
   --set service.nodePort=32000 \
   --set image.repository=grafana/grafana-enterprise \
   --set persistence.enabled=true
```

---

## Step 5: Access Grafana

### Port Forwarding
Port-forward the Grafana service to localhost:
```bash
kubectl port-forward -n grafana svc/grafana 3000:80
```

Access Grafana in your browser:  
[http://localhost:3000](http://localhost:3000)

Default credentials:
- **Username:** `admin`
- **Password:** `admin`

---

## Step 6: Verify Deployment

Verify that the Grafana pod is running:
```bash
kubectl get pods -n grafana
```

Expected output:
```
NAME                       READY   STATUS    RESTARTS   AGE
grafana-7998d77455-7mc77   1/1     Running   0          2m
```

Verify the service is exposed:
```bash
kubectl get svc -n grafana
```

Expected output:
```
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
grafana   NodePort   10.98.240.39   <none>        80:32000/TCP   2m
```

---

## Step 7: Clean Up

To remove Grafana and its resources:
```bash
helm uninstall grafana -n grafana
kubectl delete namespace grafana
```

If needed, stop Minikube:
```bash
minikube stop
```

To delete the Minikube cluster entirely:
```bash
minikube delete
```

---

## Adding Grafana Enterprise Metrics (GEM) on Minikube

### Prerequisites

1. Ensure you have the following ready:
   - A running Minikube cluster.
   - `kubectl` and `helm` installed and configured.
   - GEM license saved to a file (`GEM_license.jwt`).

   **Note:** The license (in the Grafana admin page) must have your cluster name as the URL. Minikube defaults to `minikube`. If you set it to something else, you'll need to adjust accordingly.

2. Create a namespace for GEM:
   ```bash
   kubectl create namespace gem
   ```

3. Create the GEM license secret:
   ```bash
   kubectl create secret generic gem-license \
       --from-file=license.jwt=GEM_license.jwt \
       -n gem
   ```

Add and update the Grafana Helm repository if not already done:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## Step 2: Install GEM with Helm

Use the following Helm command to deploy GEM, ensuring adjustments for Minikube:
```bash
helm install gem grafana/mimir-distributed \
  --namespace gem \
  --set enterprise.enabled=true \
  --set enterprise.legacyLabels=true \
  --set license.contents=null \
  --set license.external=true \
  --set license.secretName=gem-license \
  --set license.secretKey=license.jwt \
  --set mimir.structuredConfig.cluster_name=minikube \
  --set gateway.service.type=NodePort \
  --set gateway.service.nodePort=32001
```

---

## Step 3: Verify GEM Deployment

### Check the Pods:
```bash
kubectl get pods -n gem
```

Ensure all GEM-related pods are in the `Running` or `Completed` state.

### Check the Services:
```bash
kubectl get svc -n gem
```

Example output:
```
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       
gem-enterprise-metrics-gateway    NodePort    10.96.6.227      <none>        80:32001/TCP,8080:31455/TCP   5m
```

---

## Testing the GEM Gateway

Port-forward the GEM Gateway service to localhost:
```bash
kubectl port-forward -n gem svc/gem-enterprise-metrics-gateway 8080:80
```

Access the gateway locally:
```bash
curl -v http://localhost:8080/ready
```

Example output:
```
ready
```

---

## Step 4: Retrieve GEM Token

1. List all completed pods:
   ```bash
   kubectl get pods -n gem --field-selector=status.phase=Succeeded
   ```

2. Get the logs of the token generation pod:
   ```bash
   kubectl logs -n gem gem-enterprise-metrics-tokengen-abcdef1234
   ```

3. Look for a line in the logs like:
   ```
   Token: X19hZG1pbl9fLTMxZTRiZGRkZDk1MTRmZWY6OC8vezIzNTU1WH4/bzlONjU6XX51flw3
   ```

Copy the token for later use.

---



## Issues with GEM URL

### Problem Explanation

- Grafana and GEM are running inside Kubernetes pods: When you use `localhost` in Grafana's data source configuration, Grafana tries to connect to `localhost` inside its own pod, not your local machine.
- Port-forwarding is local to your machine: The `kubectl port-forward` command forwards a port from your local machine to the Kubernetes service, but this doesn't affect how pods within the cluster communicate with each other.
- Grafana can't reach GEM via `localhost:8080`: Because GEM is running in a different pod (and possibly a different namespace), Grafana needs to connect to GEM using the internal Kubernetes service DNS name or ClusterIP.

[Learn more about Kubernetes networking](https://kubernetes.io/docs/concepts/services-networking/).

### Solution Steps

1. **Use the Internal Service URL in Grafana**

   Instead of using `localhost:8080`, configure Grafana to connect to GEM using the internal Kubernetes service DNS name.

   - **Get the GEM Gateway Service Name**:
     - The GEM Gateway service is typically named `gem-enterprise-metrics-gateway`.
     - Since it's in the `gem` namespace, the full DNS name is:
       ```
       gem-enterprise-metrics-gateway.gem.svc.cluster.local
       ```

   - **Configure the Data Source in Grafana**:
     - In Grafana's UI, go to `Configuration > Data Sources > Add data source`.
     - Choose **Prometheus** (since GEM is compatible with Prometheus APIs).
     - Set the URL to:
       ```
       http://gem-enterprise-metrics-gateway.gem.svc.cluster.local:80
       ```
       **Note:** Use `http` and port `80` because that's the port exposed by the GEM Gateway service internally.

[More details on Kubernetes service networking](https://kubernetes.io/docs/concepts/services-networking/service/).

2. **Set the Grafana Enterprise Metrics URL**

   In the Grafana UI under the plugin section, set up the Grafana Enterprise Metrics URL:
   ```
   http://gem-enterprise-metrics-gateway.gem.svc.cluster.local:80
   ```

   Paste the token from earlier into the **Access Token** box.

![image](https://github.com/user-attachments/assets/f1912523-96c4-489b-90b2-0f00dec609f3)


Check the dashboards to verify metrics are visible:

![image](https://github.com/user-attachments/assets/f1a321f1-cdb6-4b0c-8354-487872aaf838)

![image](https://github.com/user-attachments/assets/9b24cb9a-2031-4db0-a1cc-925d29eea141)



## Step 5: Clean Up

To remove Grafana Enterprise Metrics and its resources:
```bash
helm uninstall gem -n gem
kubectl delete namespace gem
```

If needed, stop Minikube:
```bash
minikube stop
```

To delete the Minikube cluster entirely:
```bash
minikube delete
```
---
