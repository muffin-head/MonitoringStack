### **Wiki: Post-Deployment Steps for Prometheus and Grafana**

---

### **Purpose**
After deploying Prometheus and Grafana, some additional steps are required to ensure that:
1. **Prometheus is accessible externally**: Allows for external monitoring and management.
2. **Application metrics (`current_prediction`) appear in Prometheus and Grafana**: Ensures metrics are generated, scraped, and visualized.

---

### **Steps to Perform After Deployment**

#### **1. Expose Prometheus Externally**
By default, Prometheus is deployed as a `ClusterIP` service, which restricts access to within the Kubernetes cluster. To make Prometheus accessible externally:
1. **Edit the Prometheus Service**:
   ```bash
   kubectl edit svc prometheus-server -n monitoring
   ```
2. **Modify the `spec` Section**:
   Update the service type to `LoadBalancer` and optionally assign an existing static public IP:
   ```yaml
   spec:
     type: LoadBalancer
     loadBalancerIP: <YOUR_EXISTING_PUBLIC_IP>  # Optional: Use if you have an unused IP
   ```
3. **Save and Exit**:
   - In `vi`, press `Esc`, type `:wq`, and press `Enter`.
   - In `nano`, press `Ctrl+O`, `Enter`, and then `Ctrl+X`.

4. **Verify the Service**:
   Check the status of the Prometheus service:
   ```bash
   kubectl get svc prometheus-server -n monitoring
   ```

   You should see an `EXTERNAL-IP` assigned. Use this IP to access Prometheus, e.g., `http://<EXTERNAL-IP>:80`.

---

#### **2. Trigger Metrics in the Flask Application**
Prometheus requires your application to expose metrics to be scraped. The `current_prediction` metric is generated only after your application processes a prediction request. To trigger this:
1. **Send a POST Request to the Application**:
   Use a `curl` command or similar tool to send sample data to the `/predict` endpoint:
   ```bash
   curl -X POST http://<APPLICATION-IP>:5000/predict \
   -H "Content-Type: application/json" \
   -d '[{"MDC_PULS_OXIM_PULS_RATE_Result_min": 20, "MDC_TEMP_Result_mean": 16.5, "MDC_PULS_OXIM_PULS_RATE_Result_mean": 105, "HR_to_RR_Ratio": 3.2}]'
   ```
   Replace `<APPLICATION-IP>` with the IP of your application.

2. **Verify Metrics in the Application**:
   Check the `/metrics` endpoint to ensure `current_prediction` is being exposed:
   ```bash
   curl http://<APPLICATION-IP>:5000/metrics
   ```

   You should see:
   ```plaintext
   # HELP current_prediction Real-time prediction value
   # TYPE current_prediction gauge
   current_prediction{timestamp="1234567890"} 1.0
   ```

---

#### **3. Add the Metric in Grafana**
1. Open Grafana at its external IP.
2. Add Prometheus as a data source:
   - Go to **Configuration > Data Sources > Add data source**.
   - Select **Prometheus** and provide the URL:
     ```
     http://<PROMETHEUS-EXTERNAL-IP>:80
     ```
   - Save and test the connection.

3. Add a new panel:
   - Query for `current_prediction`.
   - Customize the visualization.

---

### **Why These Steps Are Required**

1. **Exposing Prometheus**:
   - Allows external tools like Grafana or a browser to access Prometheus for monitoring and management.
   - Without this step, Prometheus is accessible only inside the Kubernetes cluster.

2. **Triggering Metrics in Flask**:
   - The metric `current_prediction` is not static; it is generated dynamically when a prediction request is made.
   - Sending a POST request ensures that Prometheus can scrape meaningful metrics from the application.

3. **Configuring Grafana**:
   - Grafana requires Prometheus as a data source to fetch and display metrics.
   - Once connected, you can visualize `current_prediction` and other metrics in real time.

---

### **Wiki: Manifest Files for Prometheus and Grafana Deployment**

---

### **Purpose**
These Kubernetes manifest files configure and deploy Prometheus and Grafana for monitoring your Flask application. Below is an explanation of each file, its role, and why it's necessary.

---

### **1. Grafana Configuration (ConfigMap)**

#### **File Content**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: monitoring
data:
  grafana.ini: |
    [server]
    http_port = 3000

    [auth.anonymous]
    enabled = true
    org_role = Viewer

    [auth.basic]
    enabled = true
```

#### **What It Does**
- **Purpose**: This file configures Grafana's settings.
- **Key Configurations**:
  - `http_port`: Grafana will listen on port `3000`.
  - `auth.anonymous.enabled`: Enables anonymous access to Grafana. Anyone can view dashboards without logging in.
  - `auth.basic.enabled`: Enables basic authentication for added security.

#### **Why It's Needed**
- This configuration makes Grafana accessible and user-friendly by allowing anonymous users to view dashboards.
- Ensures Grafana is correctly configured to listen on the right port (`3000`).

---

### **2. Monitoring Namespace**

#### **File Content**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
```

#### **What It Does**
- **Purpose**: Creates a dedicated namespace called `monitoring` to isolate Prometheus and Grafana resources from other Kubernetes resources.

#### **Why It's Needed**
- Namespaces help organize and separate resources in Kubernetes. This ensures that Prometheus and Grafana resources don’t interfere with your application or other deployments.

---

### **3. Prometheus Configuration (ConfigMap)**

#### **File Content**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 1s

    scrape_configs:
      - job_name: 'flask-app'
        metrics_path: '/metrics'
        static_configs:
          - targets: ['ml-model-deployment.default.svc.cluster.local:5000']
```

#### **What It Does**
- **Purpose**: Configures Prometheus to scrape metrics from your Flask app every second.
- **Key Configurations**:
  - `scrape_interval`: Prometheus scrapes metrics every 1 second for real-time monitoring.
  - `scrape_configs`: Defines the metrics source:
    - `job_name`: A name for the scrape job (`flask-app`).
    - `metrics_path`: The path where metrics are exposed (`/metrics`).
    - `static_configs.targets`: Specifies the location of your Flask app within the Kubernetes cluster (`ml-model-deployment.default.svc.cluster.local:5000`).

#### **Why It's Needed**
- This configuration tells Prometheus where to find your Flask app and how often to scrape its metrics.
- Ensures Prometheus collects data from your app's `/metrics` endpoint.

---

### **Summary**

| **Manifest File**       | **Purpose**                                                                                   | **Why It's Needed**                                                                                                     |
|--------------------------|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **Grafana ConfigMap**    | Configures Grafana (port, authentication)                                                     | Ensures Grafana is accessible and user-friendly.                                                                        |
| **Monitoring Namespace** | Creates a namespace to group monitoring resources                                             | Keeps Prometheus and Grafana resources organized and isolated.                                                          |
| **Prometheus ConfigMap** | Configures Prometheus to scrape metrics from the Flask app (`/metrics`)                       | Ensures Prometheus collects data from the Flask app every second for real-time monitoring.                               |

---

### **How These Work Together**
1. The **namespace** groups the monitoring stack (Prometheus and Grafana).
2. **Prometheus** collects metrics from the Flask app based on the `prometheus-config`.
3. **Grafana** visualizes these metrics using its configured settings (`grafana-config`).

This setup ensures a seamless monitoring solution for your application. Let me know if you need further clarification! 🚀
