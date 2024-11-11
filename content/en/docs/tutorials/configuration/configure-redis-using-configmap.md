<!-- overview -->

<div style="text-align: center;">
  
# Configure Redis using a ConfigMap
In this guide, you’ll set up and configure a Redis instance within a Kubernetes environment using a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)! With ConfigMaps, you can manage dynamic, environment-specific settings, enhancing flexibility and control over your Redis configurations in Kubernetes. 


</div>

## Redis Configuration Flow
The diagram below illustrates how Redis is configured within a Kubernetes environment using ConfigMaps.
![Diagram showing the relationship between Kubernetes Node, Redis Pod, and ConfigMap](/content/en/docs/images/RedisCMChart.png)

## What you'll learn
In this tutorial, you will:

- **Create a ConfigMap**: Learn how to configure Redis values for a flexible setup.
- **Deploy a Redis Pod**: Use the ConfigMap to manage configurations dynamically.
- **Verify Configuration**: Confirm that Redis is running with the correct settings.

## Requirements

| Requirement           | Description                                                                                                     |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| **Kubernetes Cluster**    | Access to a Kubernetes cluster with `kubectl` installed (version 1.14 or higher).                              |
| **ConfigMaps**            | Familiarity with [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). |
| **Redis Basics**          | Basic knowledge of Redis setup and configurations.                                                              |

<!-- lessoncontent -->

## Step 1: Create a ConfigMap for Redis Configuration

A ConfigMap allows you to manage Redis configurations externally. Let’s start by creating one that will hold Redis-specific settings.

1. Open your terminal and create a file called example-redis-config.yaml with the following content:

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

## Step 2: Apply ConfigMap and Manifest
With the ConfigMap created, it’s time to apply it to your Kubernetes cluster and deploy a Redis Pod configured to use this ConfigMap.

**Terminal**
```shell
# Apply the ConfigMap to your Kubernetes cluster
kubectl apply -f example-redis-config.yaml

# Deploy the Redis Pod using a predefined manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
> **Note**: The first command applies the example-redis-config ConfigMap, adding it to your Kubernetes cluster’s configuration. The second command deploys a Redis Pod, referencing the ConfigMap to set up Redis with your specified configurations.

## Step 3: Review the Redis Pod Manifest

In this step, we’ll examine the Redis Pod manifest to understand how it references the `example-redis-config` ConfigMap, connecting it to the Redis container.

### Key Components of the Redis Pod Manifest
- **Volume Creation:** A volume named `config` is created under `spec.volumes[1]`.
- **ConfigMap Key Exposure:** The `key` and `path` values in `spec.volumes[1].configMap.items[0]` expose the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` within the config volume.
- **Volume Mounting:** The `config` volume is mounted at `/redis-master` via `spec.containers[0].volumeMounts[1]`.

> **Note**: This configuration exposes the `data.redis-config` key from `example-redis-config` as `/redis-master/redis.conf` inside the Pod.

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

## Step 4: Verify the Redis Pod and ConfigMap
Confirm that the Redis Pod and ConfigMap were applied successfully by checking the resources in your Kubernetes cluster.

1. **List the Redis Pod and ConfigMap**

   **Terminal**
   ```shell
      kubectl get pod/redis configmap/example-redis-config 
   ```
**Expected Output**

| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| pod/redis                        | 1/1   | Running | 0        | 8s  |
| configmap/example-redis-config   | 1     |         |          | 14s |

> **Note**: This confirms the Redis Pod is running and the ConfigMap has been created.

2. **Describe the ConfigMap to verify contents**
The next command provides more details on the example-redis-config ConfigMap, allowing you to confirm that the redis-config key is currently empty as expected.

**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

**Expected Output**
You should see an empty redis-config key:
| Field        | Value           |
|--------------|-----------------|
| Name         | example-redis-config |
| Namespace    | default         |
| Labels       | \<none>         |
| Annotations  | \<none>         |

**Data Section**
| Key          | Value           |
|--------------|-----------------|
| redis-config | (empty)         |

> **Note:** Ensure that the 'redis.conf' file path and 'volume mounts' in the Redis Pod manifest match your configuration. Any discrepancies might prevent Redis from loading the expected configuration. Double-check the 'example-redis-config.yaml' file and reapply it if necessary.


## Step 5: Access the Redis CLI in the Pod
To verify that the Redis configuration values are set to their defaults, access the Redis CLI within the running Redis Pod.

1. To verify the current Redis configuration, access the Redis CLI by running the following command:

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

3. Run the following command in the Redis CLI to check `maxmemory-policy`:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Key              | Value      |
|------------------|------------|
| maxmemory-policy | noeviction |

> **Note:** The 'noeviction policy' means that Redis will not evict any data when it reaches its memory limit, which is the default setting. If the values do not display as shown, make sure the Redis Pod is running and accessible. You can check the Pod’s status with `kubectl get pod redis` and re-enter the Redis CLI with `kubectl exec -it redis -- redis-cli`.

## Step 6: Update `example-redis-config` ConfigMap with Custom Configuration

In this step, you’ll add specific configuration values to the `example-redis-config` ConfigMap to customize Redis settings.

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

2. Apply the updated ConfigMap to your Kubernetes cluster:

   **Terminal**
    ```shell
    kubectl apply -f example-redis-config.yaml
    ```

3. Verify the ConfigMap update to confirm your changes:

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

> **Note:** If the custom values do not display as shown, double-check the 'example-redis-config.yaml' file for accuracy and reapply it with 'kubectl apply -f example-redis-config.yaml'. You can also view the full ConfigMap file [here](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/example-redis-config.yaml).


## Step 7: Verify redis configuration with `redis-cli`

To confirm the applied Redis configuration in the Redis Pod, connect to the Redis CLI.

1. Access the Redis CLI:

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

4. Now we need to verify that `maxmemory-policy` remains at the `noeviction` default setting:

   **Terminal**
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected Output**
   
    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | noeviction  |

> **Note:** The default policy of the `noeviction` setting prevents data eviction when memory limits are reached. Since the Pod must be restarted to apply updated ConfigMap values, configuration values may initially display as defaults.

4. Delete and recreate the Redis Pod to apply the updated ConfigMap values:

   **Terminal**
    ```shell
    kubectl delete pod redis
    ```
   **Terminal**
    ```shell
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```
    
5. Confirm the applied configuration after the restart:

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

7. Confirm that `maxmemory-policy` is set to the desired `allkeys-lru` value:

    **Terminal**

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected Output**

    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | allkeys-lru |

### Final Configuration Summary

| Configuration Key   | Expected Value |
|---------------------|----------------|
| maxmemory           | 2097152        |
| maxmemory-policy    | allkeys-lru    |

## Step 8: Clean up resources
As the final step, we need to clean up the resources by deleting the Redis Pod and ConfigMap:

**Terminal**
```shell
kubectl delete pod/redis configmap/example-redis-config
```
> Note: It's important to remove resources that are no longer needed to avoid unnecessary resource consumption in your Kubernetes cluster. Ensure that both the Redis Pod and ConfigMap are successfully deleted.


## Next Steps
Great work on completing this tutorial! Now that you’ve configured Redis using a ConfigMap in Kubernetes, here are some next steps to explore and enhance your skills.

- **Configuration Example**: Review a practical example of [updating configurations via a ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/), showing how changes can be applied dynamically within a live Kubernetes environment.
- **ConfigMaps in Depth**: Gain a deeper understanding of [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and how they enable flexible configurations for containerized applications.

## Learn more with Community Resources!
The Kubernetes community is a vibrant and collaborative ecosystem where users, developers, and contributors come together to share knowledge, solve problems, and contribute to the growth of Kubernetes. 

- **Kubernetes Support Resources**: Visit the [Kubernetes Community Support page](https://kubernetes.io/community/) to access our Youtube channel, tutorials, examples, and guides designed to help you succeed in using Kubernetes effectively.
- **Kubernetes Forum**: Join the discussion on the [Kubernetes Forum](https://discuss.kubernetes.io/)—an ideal place for sharing ideas, asking questions, and finding support from other Kubernetes users and developers.

> **Tip:** Engaging with these resources helps you build Kubernetes expertise and connect with a supportive network of professionals and enthusiasts worldwide!
