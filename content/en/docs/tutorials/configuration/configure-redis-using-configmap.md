<!-- overview -->

<div style="text-align: center;">
  
# Configure Redis using a ConfigMap
It's time to set up and configure a Redis instance within a Kubernetes environment using ConfigMaps! With ConfigMaps, you can manage dynamic, environment-specific settings, enhancing flexibility and control over your Redis configurations in Kubernetes. 

![Diagram showing the relationship between Kubernetes Node, Redis Pod, and ConfigMap](/content/en/docs/images/RedisCMChart.png)

</div>


## What you'll learn
In this tutorial, you'll learn how to...

- **Create** a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) with Redis configuration values.
- **Deploy** a Redis Pod that uses the created ConfigMap.
- **Verify** that the configuration was applied successfully.

## Requirements

| Requirement           | Description                                                                                                     |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| Kubernetes cluster    | Access to a Kubernetes cluster with `kubectl` installed (version 1.14 or higher).                              |
| Familiarity           | Familiarity with [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). |
| **Redis Basics**      | Basic knowledge of Redis setup and configurations.                                                              |

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
Now we will apply the ConfigMap and deploy a Redis Pod configured to use it.

**Terminal**
```shell
# Apply the ConfigMap to your Kubernetes cluster
kubectl apply -f example-redis-config.yaml

# Deploy the Redis Pod using a predefined manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

## Step 3: Review the Redis Pod Manifest

In this step, you’ll review the Redis Pod manifest to understand how it integrates the example-redis-config ConfigMap.

### Redis Pod Manifest Highlights
- `Volume Creation:` A volume named config is created by spec.volumes[1].
- `ConfigMap Key Exposure:` The `key` and `path` under `spec.volumes[1].configMap.items[0]` expose the redis-config key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the config volume.
- The `config` volume is mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

> **Note**: This setup exposes the data in `data.redis-config` from the `example-redis-config` ConfigMap as `/redis-master/redis.conf` inside the Pod.

**Terminal**
```shell
# Redis Pod manifest configuration (pods/config/redis-pod.yaml)

apiVersion: v1                     # Kubernetes API version
kind: Pod                          # Declares this object as a Pod
metadata:
  name: redis                      # Pod name
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf" # Uses ConfigMap for Redis configuration
    env:
    - name: MASTER
      value: "true"                # Sets Redis instance role as master
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"                 # Limits CPU usage
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config                 # Mounts ConfigMap volume for configuration
  volumes:
    - name: data                   # Ephemeral data storage
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config # Loads ConfigMap for Redis configuration
        items:
        - key: redis-config
          path: redis.conf         # ConfigMap key path inside the container

```

> **Access Example Code**: View the full [`redis-pod.yaml`](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/redis-pod.yaml) file in the GitHub repository.

## Step 4: Verify the Redis Pod and ConfigMap
To confirm that the Redis Pod and ConfigMap are correctly applied, follow these commands and review the expected output.

**Terminal**
```shell
kubectl get pod/redis configmap/example-redis-config 
```
### Expected Output

| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| pod/redis                        | 1/1   | Running | 0        | 8s  |
| configmap/example-redis-config   | 1     |         |          | 14s |

#

The next command provides more detailed information on the ConfigMap’s contents, allowing us to confirm the redis-config key is empty as expected.


**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key:

| Field        | Value           |
|--------------|-----------------|
| Name         | example-redis-config |
| Namespace    | default         |
| Labels       | \<none>         |
| Annotations  | \<none>         |

We've left the `redis-config` key in the `example-redis-config` ConfigMap blank:
### Data
| Key          | Value           |
|--------------|-----------------|
| redis-config | (empty)         |

---

> **Note:** Ensure that the `redis.conf` file path and volume mounts in the Redis Pod manifest align with your configuration. Any discrepancies may prevent Redis from loading the expected configuration. Double-check the `example-redis-config.yaml` file and reapply it if needed.


## Step 5: Access the Redis CLI in the Pod
In this step, we will access the Redis CLI within the Pod to verify that the configuration values are set to their defaults before applying our custom settings.

1. To verify the current Redis configuration, access the Redis CLI by running the following command:

```shell
kubectl exec -it redis -- redis-cli
```

2. Run the following command in the Redis CLI to check the `maxmemory` configuration:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

**Expected output:**

```shell
1) "maxmemory"
2) "0"
```

3. Run the following command in the Redis CLI to check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected output:**

```shell
1) "maxmemory-policy"
2) "noeviction"
```
> **Note:** If the values do not display as shown, make sure the Redis Pod is running and accessible. You can check the Pod’s status with `kubectl get pod redis` and re-enter the Redis CLI with `kubectl exec -it redis -- redis-cli`.

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

    The output should show the configuration values added:

    | Field           | Value                        |
    |-----------------|------------------------------|
    | Name            | example-redis-config         |
    | Namespace       | default                      |
    | Labels          | <none>                       |
    | Annotations     | <none>                       |
    
    **Data**

    | Key             | Value                        |
    |-----------------|------------------------------|
    | redis-config    | maxmemory 2mb                |
    |                 | maxmemory-policy allkeys-lru |

> **Note:** If the values do not display as shown, double-check the `example-redis-config.yaml` file for accuracy and reapply it with `kubectl apply -f example-redis-config.yaml`. You can also view the full ConfigMap file [here](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/example-redis-config.yaml).


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

    **Expected output:**

    | Key         | Value  |
    |-------------|--------|
    | maxmemory   | 0      |

3. Now we need to verify that `maxmemory-policy` remains at the `noeviction` default setting:

   **Terminal**
    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected output:**

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

    **Expected output:**

    | Key         | Value   |
    |-------------|---------|
    | maxmemory   | 2097152 |

7. Confirm that `maxmemory-policy` is set to the desired `allkeys-lru` value:

    **Terminal**

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected output:**

    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | allkeys-lru |


## Step 8: Clean up resources
As the final step, we need to clean up the resources by deleting the Redis Pod and ConfigMap:

**Terminal**
```shell
kubectl delete pod/redis configmap/example-redis-config
```
> Note: It's important to remove resources that are no longer needed to avoid unnecessary resource consumption in your Kubernetes cluster. Ensure that both the Redis Pod and ConfigMap are successfully deleted.


## Next Steps
Congratulations! You’ve successfully configured Redis using a ConfigMap in Kubernetes. Take a look at the articles below to further expand your skills and understanding:

- **Configuration Example**: Review a practical example of [updating configurations via a ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/), showing how changes can be applied dynamically within a live Kubernetes environment.
- **ConfigMaps in Depth**: Gain a deeper understanding of [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and how they enable flexible configurations for containerized applications.
- **Performance Optimization**: Improve your Kubernetes clusters by learning about [resource management strategies](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) to ensure Redis runs efficiently in diverse environments.

## Community Resources
The Kubernetes community is a vibrant and collaborative ecosystem where users, developers, and contributors come together to share knowledge, solve problems, and contribute to the growth of Kubernetes. Here are some ways to connect and continue learning:

- **Kubernetes Support Resources**: Visit the [Kubernetes Community Support page](https://kubernetes.io/community/) to access our Youtube channel, tutorials, examples, and guides designed to help you succeed in using Kubernetes effectively.
- **Kubernetes Forum**: Join the discussion on the [Kubernetes Forum](https://discuss.kubernetes.io/)—an ideal place for sharing ideas, asking questions, and finding support from other Kubernetes users and developers.
- **Kubernetes GitHub Repository**: Dive into the [Kubernetes GitHub repository](https://github.com/kubernetes/kubernetes) to keep up with the latest development efforts, view source code, and participate in issue discussions or code contributions.

By engaging with these resources, you’ll continue to build your Kubernetes expertise and connect with a supportive network of professionals and enthusiasts from around the world!
