<!-- overview -->

<div style="text-align: center;">
  
# Configure Redis using a ConfigMap
In this guide, you’ll learn to set up and configure a Redis instance within a Kubernetes environment using ConfigMaps. With ConfigMaps, you can manage dynamic, environment-specific settings, enhancing flexibility and control over your Redis configurations in Kubernetes. 

![Diagram showing the relationship between Kubernetes Node, Redis Pod, and ConfigMap](../../../images/RedisCMChart.png)

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

To manage Redis configurations externally, create a ConfigMap to hold Redis-specific settings.

Create a file named `example-redis-config.yaml` with the following content:

**Terminal**
```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

## Step 2: Apply ConfigMap and Manifest
Apply the ConfigMap created above, along with a Redis pod manifest:

**Terminal**
```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

## Step 3: Review the Redis Pod Manifest

In this step we will take a look at the Redis pod manifest to understand how it’s configured to use the ConfigMap. The Redis Pod manifest is configured to use the example-redis-config ConfigMap. Here’s what each part of the manifest does:

- `Volume Creation:` A volume named config is created by spec.volumes[1].
- `ConfigMap Key Exposure:` The `key` and `path` under `spec.volumes[1].configMap.items[0]` expose the redis-config key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the config volume.
- The `config` volume is mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

**Note**: This setup exposes the data in `data.redis-config` from the `example-redis-config` ConfigMap as `/redis-master/redis.conf` inside the Pod.

**Terminal**

```shell
pods/config/redis-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf"
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
```

{{% code_sample file="pods/config/redis-pod.yaml" %}}


### Examine the created objects
**Terminal**
```shell
kubectl get pod/redis configmap/example-redis-config 
```
### Expected Output

| NAME                             | READY | STATUS  | RESTARTS | AGE |
|----------------------------------|-------|---------|----------|-----|
| pod/redis                        | 1/1   | Running | 0        | 8s  |
| configmap/example-redis-config   | 1     |         |          | 14s |


We've left the `redis-config` key in the `example-redis-config` ConfigMap blank:

**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key:

### Key

| Field        | Value           |
|--------------|-----------------|
| Name         | example-redis-config |
| Namespace    | default         |
| Labels       | \<none>         |
| Annotations  | \<none>         |

### Data

| Key          | Value           |
|--------------|-----------------|
| redis-config | (empty)         |

> **Note:** Ensure that the `redis.conf` file path and volume mounts in the Redis Pod manifest align with your configuration. Any discrepancies may prevent Redis from loading the expected configuration. Double-check the `example-redis-config.yaml` file and reapply it if needed.

## Step 4: Access the Redis CLI in the Pod
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

## Step 5: Update `example-redis-config` ConfigMap with Custom Configuration
Now we will add configuration values to the `example-redis-config` ConfigMap.

> Example: {{% code_sample file="pods/config/example-redis-config.yaml" %}}

1. Edit the `example-redis-config.yaml` file and add the following configuration values:

```shell 
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
```

2. Run the following command to apply your changes to the cluster:
```shell
kubectl apply -f example-redis-config.yaml
```

3. Verify the ConfigMap Update

```shell
kubectl describe configmap/example-redis-config
```

Now you should see the configuration values we just added:

| Field           | Value                        |
|-----------------|------------------------------|
| Name            | example-redis-config         |
| Namespace       | default                      |
| Labels          | <none>                       |
| Annotations     | <none>                       |
| `Data`          |                              |
| redis-config    | maxmemory 2mb                |
|                 | maxmemory-policy allkeys-lru |


> **Note:** If these values do not display as shown, please review your ConfigMap configuration file (`example-redis-config.yaml`) and ensure it was applied correctly. You can reapply the configuration with `kubectl apply -f example-redis-config.yaml` and recheck with `kubectl describe configmap/example-redis-config`.

## Step 6: Verify redis configuration with `redis-cli`

To confirm the Redis configuration values in the Redis Pod, access the Redis CLI.

1. Run the following command in your terminal to access the Redis CLI:

    ```shell
    kubectl exec -it redis -- redis-cli
    ```

2. Check the `maxmemory` configuration by running the following command in the Redis CLI:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory
    ```

    **Expected output:**

    | Key         | Value  |
    |-------------|--------|
    | maxmemory   | 0      |

3. Verify that `maxmemory-policy` remains at the `noeviction` default setting:

    ```shell
    127.0.0.1:6379> CONFIG GET maxmemory-policy
    ```

    **Expected output:**

    | Key                | Value       |
    |--------------------|-------------|
    | maxmemory-policy   | noeviction  |

> **Note:** The default policy of `noeviction` means Redis will not evict any data when it reaches its memory limit. The configuration values have not changed because the Pod needs to be restarted to apply updated values from associated ConfigMaps.

4. Delete and recreate the Redis Pod to apply the updated ConfigMap values:

    **Terminal**
    ```shell
    kubectl delete pod redis
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
    ```

5. Verify the configuration values after restarting the Pod:

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


## Step 7: Clean up resources
To conclude, clean up the resources created during this tutorial by deleting the Redis Pod and ConfigMap:

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
