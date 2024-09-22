### Exercise: Understanding Kubernetes Pods
**Objective**: Create a Kubernetes Pod that runs a simple web server and verify its functionality. You will also modify the Pod to use a ConfigMap for configuration.
#### Part 1: Create a Pod
1. **Create a Pod YAML definition** named `web-server-pod.yaml` that:
    - Uses the `nginx` image.
    - Exposes port `80`.
    - Has a label `app: web-server`.
1. **Apply the Pod definition** to create the Pod in your Kubernetes cluster.
2. **Verify** that the Pod is running.
3. **Access** the Nginx web server to confirm it's working.
4. #### Part 2: Use a ConfigMap for Custom Configuration
1. **Create a ConfigMap** named `nginx-config` that contains a custom `index.html` file with the following content:
```html
    <html>
    <head><title>Welcome to My Nginx Server</title></head>
    <body><h1>Hello from Nginx!</h1></body>
    </html>
```
1. **Modify the original Pod definition** to mount the ConfigMap as a volume so that Nginx serves the custom `index.html`.
2. **Reapply** the modified Pod definition to update the Pod.
3. **Access** the Nginx server again and verify that it serves the custom content
