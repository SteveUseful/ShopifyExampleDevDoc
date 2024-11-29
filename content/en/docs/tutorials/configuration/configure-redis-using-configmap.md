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

### Step 1: Create a ConfigMap for Redis Configuration

Let’s start by creating a ConfigMap that will hold Redis-specific settings.

1. In your terminal, create a file called `example-redis-config.yaml` with the following content:

**Terminal**
```shell
# Create a YAML file for Redis ConfigMap
cat <<EOF > ./example-redis-config.yaml

apiVersion: v1                 # API version
kind: ConfigMap                # Declares a Kubernetes ConfigMap
metadata:
  name: example-redis-config   # Name of the ConfigMap
data:
  redis-config: ""             # Placeholder for Redis configuration

EOF
```

### Step 2: Apply ConfigMap and Manifest

After defining the `ConfigMap`, apply it to your Kubernetes cluster, and deploy a `Redis Pod` configured to use it.

**Terminal**
```shell
# Apply the ConfigMap to your Kubernetes cluster
kubectl apply -f example-redis-config.yaml

# Deploy the Redis Pod using a predefined manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

> The first command applies the `example-redis-config` `ConfigMap`, adding it to your Kubernetes cluster’s configuration. The second command deploys a `Redis Pod`, referencing the `ConfigMap` to set up Redis with your specified configurations.

### Step 3: Examine the Redis Pod Manifest

In this step, you’ll review the `Redis Pod` manifest to understand how it references the `example-redis-config` ConfigMap, enabling dynamic configuration in Redis.

#### Key Components in the Redis Pod Manifest

| Component                   | Description                                                                                                                 |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **Volume Creation**         | A volume named `config` is created under `spec.volumes[1]`.                                                                |
| **ConfigMap Key Exposure**  | The `key` and `path` fields in `spec.volumes[1].configMap.items[0]` expose the `redis-config` key from `example-redis-config` as a file named `redis.conf` within the config volume. |
| **Volume Mounting**         | The `config` volume is mounted at `/redis-master` via `spec.containers[0].volumeMounts[1]`.                                 |


> This configuration exposes the `data.redis-config` key from `example-redis-config` as `/redis-master/redis.conf` inside the Pod.

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
    env:
    - name: MASTER
      value: "true"                
    ports:
    - containerPort: 6379          
    resources:
      limits:
        cpu: "0.1"                 # Limits CPU usage
    volumeMounts:
    - mountPath: /redis-master-data
      name: data                   # Mounts empty data directory
    - mountPath: /redis-master
      name: config                 # Mounts ConfigMap volume for Redis configuration
  volumes:
    - name: data                   
      emptyDir: {}
    - name: config                 # Volume referencing ConfigMap for Redis
      configMap:
        name: example-redis-config # Loads ConfigMap for Redis configuration
        items:
        - key: redis-config
          path: redis.conf         # Sets ConfigMap key path inside the container
```
  
> **Access Example Code**: View the full [`redis-pod.yaml`](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/redis-pod.yaml) file in the GitHub repository.

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

2. Describe the `ConfigMap` to verify contents:

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

> You should see an empty `redis-config` key.

**Data Section**
| Key          | Value           |
|--------------|-----------------|
| redis-config | (empty)         |
  
Ensure that the `redis.conf` file path and `volume mounts` in the Redis Pod manifest match your configuration. Any discrepancies might prevent Redis from loading the expected configuration. Double-check the `example-redis-config.yaml` file and reapply it if necessary.

### Step 5: Access the Redis CLI in the Pod
To verify that the Redis configuration values are set to their defaults, access the Redis CLI within the running Redis Pod.

1. Access the Redis CLI by running the following command:

**Terminal**
```shell
kubectl exec -it redis -- redis-cli
```

2. Run the following command in the Redis CLI to check the `maxmemory` configuration:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

**Expected Output**
| Key          | Value |
|--------------|-------|
| maxmemory    | 0     |

3. Now check `maxmemory-policy`:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Key              | Value      |
|------------------|------------|
| maxmemory-policy | noeviction |


> The `noeviction policy` means that Redis will not evict any data when it reaches its memory limit, which is the default setting. If the values do not display as shown, make sure the Redis Pod is running and accessible. You can check the Pod’s status with `kubectl get pod redis` and re-enter the Redis CLI with `kubectl exec -it redis -- redis-cli`.

## Step 6: Update the ConfigMap with Custom Configuration

Add specific Redis configuration values to `example-redis-config.yaml` to enable custom memory settings.

1. Edit the `example-redis-config.yaml` file and add the following configuration values:

    **Terminal**
    ```shell
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: example-redis-config
    data:
      redis-config: |
        maxmemory 2mb                       # Limit memory to 2MB
        maxmemory-policy allkeys-lru        # Set eviction policy to allkeys-lru
    ```

2. Apply the updated `ConfigMap`:

   **Terminal**
    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

3. Verify the update:

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


> If the custom values do not display as shown, double-check the `example-redis-config.yaml` file for accuracy and reapply it with `kubectl apply -f example-redis-config.yaml'. You can also view the full ConfigMap file [here](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/example-redis-config.yaml).

### Step 7: Verify redis configuration with redis-cli

To verify that the Redis configuration has been correctly applied, connect to the Redis CLI in your Redis Pod.

1. Access the `Redis CLI`:

   **Terminal**
    ```shell
    kubectl exec -it redis -- redis-cli
    ```

2. Verify the maxmemory configuration:

   **Terminal**
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    **Expected Output**
   
    | Key         | Value  |
    |-------------|--------|
    | maxmemory   | 0      |

3. Verify that `maxmemory-policy` remains at the `noeviction` default setting:

   **Terminal**
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected Output**
   
    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | noeviction  |

> The default policy of the `noeviction` setting prevents data eviction when memory limits are reached. The `ConfigMap` values are not yet reflected because the 'Redis Pod' must be restarted for updated values to take effect.

4. Restart the `redis pod` to apply updated `ConfigMap` values by deleting and redeploying:

   **Terminal**
    ```shell
    kubectl delete pod redis
    ```
   **Terminal**
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```
    
5. Verify the Configuration after Pod restart:

   **Terminal**
    ```shell
    kubectl exec -it redis -- redis-cli
    ```

6. Check the updated `maxmemory` configuration:

    **Terminal**
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    **Expected Output**

    | Key         | Value   |
    |-------------|---------|
    | maxmemory   | 2097152 |

7. Confirm `maxmemory-policy` update:

    **Terminal**

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected Output**

    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | allkeys-lru |
  
**Final Configuration Summary**

| Configuration Key   | Expected Value |
|---------------------|----------------|
| maxmemory           | 2097152        |
| maxmemory-policy    | allkeys-lru    |

### Step 8: Clean up resources
As the final step, we need to clean up the resources by deleting the Redis Pod and ConfigMap:

**Terminal**
```shell
kubectl delete pod/redis configmap/example-redis-config
```
> It's important to remove resources that are no longer needed to avoid unnecessary resource consumption in your Kubernetes cluster. Ensure that both the `Redis Pod` and `ConfigMap` are successfully deleted.


### Next Steps
Congratulations on completing this tutorial! Here are some recommended next steps to further your skills:

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
