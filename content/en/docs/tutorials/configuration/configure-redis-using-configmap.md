<!-- overview -->

## Configure Redis using a ConfigMap
A [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) is a [Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) object that separates app configurations from code, allowing you to manage environment-specific settings dynamically. By using ConfigMaps, you can update Redis configurations like memory limits or eviction policies without downtime or rebuilding your container. A ConfigMap is a Kubernetes object that lets you separate app configurations from code and ensures your Redis Pod can be updated dynamically. 

![Diagram showing the relationship between Kubernetes Node, Redis Pod, and ConfigMap](/content/en/docs/images/RedisCMChart.png)
> This illustration shows how Redis is configured in a Kubernetes environment using `ConfigMaps`. Imagine scaling a web app and needing Redis for session caching. As user traffic increases, you must adjust memory settings dynamically without downtime. ConfigMaps makes this seamless.

### What you'll learn
- **Create a ConfigMap**: Set up dynamic Redis configurations.
- **Deploy a Redis Pod**: Link the ConfigMap to your Redis instance.
- **Verify Configuration**: Use Redis CLI to ensure settings are applied.
- **Update Redis Settings**: Adjust Redis settings in real-time.

### Requirements

| Requirement           | Description                                                                                                     |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| **Kubernetes Cluster**    | Access to a Kubernetes cluster with `kubectl` installed (version 1.14 or higher).                              |
| **ConfigMaps**            | Familiarity with [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). |
| **Redis Basics**          | Basic knowledge of Redis setup and commands (`kubectl` and `redis-cli`).                                 |
> If you do not have a cluster, you can create one by using [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or use one of these Kubernetes playgrounds:

> [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
> [Play with Kubernetes](https://labs.play-with-k8s.com/)

#### ConfigMaps are essential for dynamic and scalable deployments. Whether you’re adjusting Redis caching for a growing app or managing multi-environment setups, this tutorial will help you configure Redis efficiently in Kubernetes.

<!-- lessoncontent -->

### Step 1: Set Up Your Environment

Before creating a ConfigMap, ensure your environment is ready to support Redis in Kubernetes. This involves verifying tools, access, and prerequisites.

#### 1. Verify Kubernetes Setup:
Ensure you have access to a Kubernetes cluster and that `kubectl` is installed and configured:

**Terminal**
```shell
kubectl version --client
```
> This command confirms your Kubernetes CLI version. You'll need version 1.14 or higher.

#### 2. Verify Namespace Context
It's a best practice to confirm or switch your namespace.

**Terminal**
```shell
kubectl config view --minify | grep namespace
```
> If no namespace is set, the default will be used. Use `kubectl config set-context` to switch namespaces if needed.

### Step 2: Create a ConfigMap

ConfigMaps allow you to dynamically manage configurations without embedding them into your container. After defining the `ConfigMap`, apply it to your Kubernetes cluster, and deploy a `Redis Pod` configured to use it.

1. Open your terminal and create a YAML file named `example-redis-config.yaml` with the following content:

**Terminal**
```shell
cat <<EOF > ./example-redis-config.yaml
apiVersion: v1                 # Kubernetes API version
kind: ConfigMap                # Declares this object as a ConfigMap
metadata:
  name: example-redis-config   # ConfigMap name
data:
  redis-config: ""             # Placeholder for Redis configuration
EOF
```
> The redis-config field is a placeholder that you will update later with specific Redis settings. This modular approach allows dynamic updates to configurations without altering container code.

2. Validate the ConfigMap
Always verify the YAML syntax to prevent errors when deploying to Kubernetes. Run the following command in Terminal:

**Terminal**
```shell
kubectl apply --dry-run=client -f example-redis-config.yaml
```
> This command performs a dry run to validate the file without applying it.

3. Apply the ConfigMap and Deploy Redis Pod
After validation, apply the ConfigMap to your cluster and deploy the Redis Pod using the following commands:

**Terminal**
```shell
# Apply the ConfigMap to your Kubernetes cluster
kubectl apply -f example-redis-config.yaml

# Deploy the Redis Pod using a predefined manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
> The first command applies the `example-redis-config` `ConfigMap`, adding it to your Kubernetes cluster’s configuration. The second command deploys a `Redis Pod`, referencing the `ConfigMap` to set up Redis with your specified configurations.

4. Confirm Successful Deployment:

**Terminal**
```shell
kubectl get pod redis
```

**Terminal**
```shell
kubectl get configmap example-redis-config
```

**Expected Output**

| NAME                          | READY | STATUS  | RESTARTS | AGE  |
|-------------------------------|-------|---------|----------|------|
| pod/redis                     | 1/1   | Running | 0        | 10s  |
| configmap/example-redis-config |       |         |          | 10s  |

> This manifest references the example-redis-config ConfigMap and dynamically applies its settings to Redis. Using a predefined manifest simplifies deployment and ensures consistency.


### Step 3: Examine the Manifest and Deploy the Redis Pod

After creating the ConfigMap, the next step is deploying a Redis Pod that uses the ConfigMap for dynamic configuration. This ensures Redis settings can be managed without rebuilding your container.

#### Key Components in the Redis Pod Manifest

| Component                   | Description                                                                                                                 |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **Volume Creation**         | A volume named `config` is created under `spec.volumes[1]`.                                                                |
| **ConfigMap Key Exposure**  | The `key` and `path` fields in `spec.volumes[1].configMap.items[0]` expose the `redis-config` key from `example-redis-config` as a file named `redis.conf` within the config volume. |
| **Volume Mounting**         | The `config` volume is mounted at `/redis-master` via `spec.containers[0].volumeMounts[1]`.                                 |
> This step maps the `redis-config` key from the ConfigMap as `/redis-master/redis.conf` inside the Redis Pod.

1. Apply the Redis Pod Manifest
Download and apply the Redis Pod manifest using the following commands:

**Terminal**
```shell
# Download the Redis Pod manifest
curl -O https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
**Terminal**
```shell
# Apply the Redis Pod manifest to your cluster
kubectl apply -f redis-pod.yaml
```

2. Examine the Redis Pod Manifest
Review the key configurations in the redis-pod.yaml file:

**Terminal**
```shell
# Redis Pod manifest configuration (pods/config/redis-pod.yaml)

apiVersion: v1                     # Specifies Kubernetes API version
kind: Pod                          # Declares this object as a Pod
metadata:
  name: redis                      # Defines Pod name
spec:
  containers:
  - name: redis                    # Container name
    image: redis:5.0.4             # Specifies Redis image
    command:
      - redis-server
      - "/redis-master/redis.conf" # Uses ConfigMap for Redis configuration
    volumeMounts:
    - mountPath: /redis-master
      name: config                 # Mounts ConfigMap volume for Redis configuration
  volumes:
    - name: config                 # Volume referencing ConfigMap for Redis
      configMap:
        name: example-redis-config # Loads ConfigMap for Redis configuration
        items:
        - key: redis-config
          path: redis.conf         # Sets ConfigMap key path inside the container

3. Verify Deployment
Run the following command to check if the Redis Pod is successfully deployed:

**Terminal**
```shell
kubectl get pods
```

**Expected Output**
| NAME     | READY | STATUS  | RESTARTS | AGE  |
|----------|-------|---------|----------|------|
| redis    | 1/1   | Running | 0        | 10s  |

> View the full [`redis-pod.yaml`](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/redis-pod.yaml) file in the GitHub repository.
> **Pro Tip**: Always review your manifest files to ensure they align with your intended deployment setup.

### Step 4: Verify the Redis Pod and ConfigMap
After applying, verify that both the `Redis Pod` and the `ConfigMap` are configured correctly in your Kubernetes cluster.

1. List the `Redis Pod` and `ConfigMap`:

   **Terminal**
   ```shell
      kubectl get pod/redis configmap/example-redis-config 
   ```
**Expected Output**

| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| pod/redis                        | 1/1   | Running | 0        | 8s  |
| configmap/example-redis-config   | 1     |         |          | 14s |

> This confirms the Redis Pod is running and the ConfigMap has been created.

2. Verify the contents of the `ConfigMap`

**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

**Expected Output**
| Field        | Value           |
|--------------|-----------------|
| Name         | example-redis-config |
| Namespace    | default         |
| Labels       | \<none>         |
| Annotations  | \<none>         |

> The Data section should display the redis-config key. If the Data section is missing or incorrect, double-check the example-redis-config.yaml file for errors.

**Data Section**
| Key          | Value           |
|--------------|-----------------|
| redis-config | (empty)         |

3. Check Redis Pod Logs (Optional)
If the Pod is not running or behaving unexpectedly, inspect the logs for debugging:

**Terminal**
``shell
kubectl logs redis
```
> Look for any errors indicating issues with configuration or dependencies.
> Ensure that the `redis.conf` file path and `volume mounts` in the Redis Pod manifest match your configuration. Any discrepancies might prevent Redis from loading the expected configuration. Double-check the `example-redis-config.yaml` file and reapply it if necessary.


> **Pro Tip:** Use `kubectl describe` regularly to validate the state of Kubernetes objects and ensure consistency in your deployment.


### Step 5: Access the Redis CLI in the Pod
To verify that the Redis configuration values are set to their defaults, access the Redis CLI within the running Redis Pod.

1. Access the Redis CLI by running the following command:

**Terminal**
```shell
kubectl exec -it redis -- redis-cli
```

2. Verify `maxmemory` Configuration
Once inside the Redis CLI, check the maxmemory configuration by running:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```
> This command opens an interactive shell session within the Redis Pod, allowing you to run Redis commands directly.

**Expected Output**
| Key          | Value |
|--------------|-------|
| maxmemory    | 0     |

3. Verify `maxmemory-policy`
   Check the maxmemory-policy configuration to confirm the eviction policy:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Key              | Value      |
|------------------|------------|
| maxmemory-policy | noeviction |

> The `noeviction policy` means Redis will not evict data when memory limits are reached, which is the default setting. Check the Pod’s status with `kubectl get pod redis` and re-enter the Redis CLI with `kubectl exec -it redis -- redis-cli` in your Terminal.

4. Troubleshooting Tips
If the values do not display as expected, ensure:

The Redis Pod is running: 
**Terminal**
```shell
kubectl get pod redis
```

The ConfigMap is correctly applied:
**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

Connecting to the Redis CLI and verifying configurations ensures that Redis is running as expected and is prepared for dynamic updates. This step provides hands-on confirmation of your deployment's success.

## Step 6: Update the ConfigMap with Custom Configuration

Add specific Redis configuration values to `example-redis-config.yaml` to enable custom memory settings.

1. Edit the `example-redis-config.yaml` file and add the following configuration values:

**Terminal**
```shell
cat <<EOF > ./example-redis-config.yaml
apiVersion: v1                 # Kubernetes API version
kind: ConfigMap                # Declares this object as a ConfigMap
metadata:
  name: example-redis-config   # ConfigMap name
data:
  redis-config: |
    maxmemory 2mb                       # Limit memory to 2MB
    maxmemory-policy allkeys-lru        # Set eviction policy to allkeys-lru
EOF
```

2. Apply the updated `ConfigMap` to your Kubernetes cluster:

**Terminal**
```shell
kubectl apply -f example-redis-config.yaml
```
> This command updates the configuration stored in your cluster with the new values.

3. Verify the Updated ConfigMap:

**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

  **Expected Output**
  | Field           | Value                        |
  |-----------------|------------------------------|
  | Name            | example-redis-config         |
  | Namespace       | default                      |
  | Labels          | <none>                       |
  | Annotations     | <none>                       |
    
  **Data Section**
  | Key             | Value                        |
  |-----------------|------------------------------|
  | redis-config    | maxmemory 2mb                |
  |                 | maxmemory-policy allkeys-lru |


> If the values don’t match the expected output, double-check the `example-redis-config.yaml` file for accuracy and reapply it with `kubectl apply -f example-redis-config.yaml'. You can also view the full ConfigMap file [here](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/example-redis-config.yaml).

By updating the ConfigMap, you can dynamically adjust Redis configurations without requiring a container rebuild. 

**Memory Optimization:** Limiting memory usage ensures Redis doesn't exceed available system resources.
**Eviction Policies:** Setting an eviction policy like allkeys-lru ensures that the least recently used keys are removed first when memory is full, maintaining optimal performance.


### Step 7: Verify redis configuration with redis-cli

To verify that the Redis configuration has been correctly applied, connect to the Redis CLI in your Redis Pod.

1. Access the `Redis CLI`:

**Terminal**
```shell
kubectl exec -it redis -- redis-cli
```

2. Verify the `maxmemory` configuration:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

**Expected Output**
| Key         | Value  |
|-------------|--------|
| maxmemory   | 0      |
> A value of 0 indicates no memory limit is currently set, which is the default setting.

4. Verify the Current Eviction Policy
Check the `maxmemory-policy` to confirm the default eviction behavior:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Key                | Value       |
|--------------------|-------------|
| maxmemory-policy   | noeviction  |

> The default policy of the `noeviction` setting prevents data eviction when memory limits are reached.
> **Note:** The `ConfigMap` values are not yet reflected because the 'Redis Pod' must be restarted for updated values to take effect.

4. Restart the `redis pod` to apply updated `ConfigMap` values by deleting and redeploying:

**Terminal**
```shell
kubectl delete pod redis
```

**Terminal**
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
    
5. Verify the updated configuration after restart
Reconnect to the `Redis CLI` and check the `maxmemory` setting again:

**Terminal**
```shell
kubectl exec -it redis -- redis-cli
```

7. Check the updated `maxmemory` configuration:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

**Expected Output**

| Key         | Value   |
|-------------|---------|
| maxmemory   | 2097152 |

8. Verify the `maxmemory-policy` reflects the updated `eviction policy`:

**Terminal**
```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

**Expected Output**
| Key                | Value       |
|--------------------|-------------|
| maxmemory-policy   | allkeys-lru |
  
**Final Configuration Summary**
After applying the updated ConfigMap and restarting the Redis Pod, the settings should align with the following values:

| Configuration Key   | Expected Value |
|---------------------|----------------|
| maxmemory           | 2097152        |
| maxmemory-policy    | allkeys-lru    |



### Step 8: Clean up resources
After completing the tutorial, clean up the resources to avoid unnecessary resource consumption in your Kubernetes cluster. Cleaning up resources ensures your cluster remains efficient and avoids potential conflicts with future deployments.

1. Delete the Redis Pod and ConfigMap

**Terminal**
```shell
kubectl delete pod/redis configmap/example-redis-config
```
> It's important to remove resources that are no longer needed to avoid unnecessary resource consumption in your Kubernetes cluster. Ensure that both the `Redis Pod` and `ConfigMap` are successfully deleted.

2. Verify Deletion
**Terminal**
```shell
kubect1 get pods
``

**Terminal**
```shell
kubect1 get configmaps
```

**Expected Output**
```shell
No resources found in default namespace.
```

### Next Steps
Congratulations on completing this tutorial! You’ve successfully deployed Redis using a ConfigMap in Kubernetes and verified its configuration. Here are some ways to build on this experience:

- **Configuration Example**: Review a practical example of [updating configurations via a ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/), showing how changes can be applied dynamically within a live Kubernetes environment.
- **ConfigMaps in Depth**: Gain a deeper understanding of [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and how they enable flexible configurations for containerized applications.

### Learn more with Community Resources!
The Kubernetes community offers resources where developers and users share insights and solve challenges together.

- **Kubernetes Support Resources**: Visit the [Kubernetes Community Support page](https://kubernetes.io/community/) to access Youtube tutorials, live examples, and guides designed to help you succeed.
- **Kubernetes Forum**: Join the discussion on the [Kubernetes Forum](https://discuss.kubernetes.io/)—an ideal place for sharing ideas, asking questions, and finding support from other Kubernetes users and developers.
  
> **Pro Tip**: Engaging with the Kubernetes community is a powerful way to enhance your skills and build professional connections.

---

***Was this page helpful?**
**Yes** | **No**

---
