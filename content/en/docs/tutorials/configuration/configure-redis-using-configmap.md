<!-- overview -->

## Configure Redis Using a ConfigMap

A [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) is a [Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) object that separates app configurations from code. This allows you to dynamically manage environment-specific settings, such as Redis memory limits or eviction policies, without rebuilding your container.

> Imagine scaling a web app where Redis handles session caching. As traffic increases, you need to adjust memory settings dynamically without downtime. ConfigMaps make this seamless.

![Diagram showing the relationship between Kubernetes Node, Redis Pod, and ConfigMap](/content/en/docs/images/RedisCMChart.png)

### What You'll Learn
1. **Set up a ConfigMap:** Dynamically manage Redis configurations.
2. **Deploy a Redis Pod:** Link Redis to your ConfigMap.
3. **Verify Configurations:** Confirm settings via the Redis CLI.
4. **Update Redis Settings:** Modify configurations dynamically.
5. **Clean up Resources:** Maintain a clean Kubernetes cluster.

---

### Requirements

| Requirement           | Description                                                                                                     |
|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| **Kubernetes Cluster** | Access to a Kubernetes cluster with `kubectl` installed (version 1.14 or higher).                              |
| **Redis Basics**       | Familiarity with Redis commands and deployment.                                                                |
| **ConfigMaps**         | Understanding of [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/). |
> If you don’t have a Kubernetes cluster, use [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or explore playgrounds like:
> - [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
> - [Play with Kubernetes](https://labs.play-with-k8s.com/)

---

### Step 1: Set Up Your Environment

Before deploying Redis, ensure your environment is prepared.

#### 1. Verify Kubernetes Setup

Check your Kubernetes CLI (`kubectl`) version to ensure compatibility:

**Terminal**
```shell
kubectl version --client

```
> Ensure kubectl version 1.14 or higher is installed.

#### 2. Verify Namespace Context
Ensure your commands are targeting the correct namespace. If no namespace is set, the default is used.

**Terminal**
```shell
kubectl config view --minify | grep namespace
```
> Use `kubectl config set-context` to switch namespaces if needed.



### Step 2: Create a ConfigMap
A ConfigMap stores Redis-specific settings. Let’s create one for dynamic configuration.

#### 1. Create a file named `example-redis-config.yaml` with the following content:

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
> The `redis-config` field will hold dynamic Redis settings. This modular approach allows dynamic updates to configurations without altering container code.

#### 2. Validate the ConfigMap
Always verify the YAML syntax to prevent errors when deploying to Kubernetes. 

**Terminal**
```shell
kubectl apply --dry-run=client -f example-redis-config.yaml
```
> This command performs a dry run to validate the file without applying it.

#### 3. Apply the ConfigMap to your cluster:
Use the following command to apply the ConfigMap to your Kubernetes cluster:

**Terminal**
```shell
kubectl apply -f example-redis-config.yaml
```

### 3. Deploy the Redis Pod
Now, deploy a Redis Pod that uses the ConfigMap for configuration.

#### 1. Download the Redis Pod manifest:

**Terminal**
```shell
curl -O https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

#### 2. Apply the Redis Pod manifest:
**Terminal**
```shell
kubectl apply -f redis-pod.yaml
```

#### 3. Verify Deployment
Ensure both the Redis Pod and ConfigMap are deployed successfully:

**Terminal**
```shell
kubectl get pods
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

> The first command applies the `example-redis-config` `ConfigMap`, adding it to your Kubernetes cluster’s configuration. The second command deploys a `Redis Pod`, referencing the `ConfigMap` to set up Redis with your specified configurations.









### Step 4: Verify Redis Configuration
To confirm that the ConfigMap is working correctly, access Redis CLI within the Pod and verify the settings.

#### Key Components in the Redis Pod Manifest

| Component                   | Description                                                                                                                 |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **Volume Creation**         | A volume named `config` is created under `spec.volumes[1]`.                                                                |
| **ConfigMap Key Exposure**  | The `key` and `path` fields in `spec.volumes[1].configMap.items[0]` expose the `redis-config` key from `example-redis-config` as a file named `redis.conf` within the config volume. |
| **Volume Mounting**         | The `config` volume is mounted at `/redis-master` via `spec.containers[0].volumeMounts[1]`.                                 |
> This step maps the `redis-config` key from the ConfigMap as `/redis-master/redis.conf` inside the Redis Pod.


#### 1. Check the Redis Pod and ConfigMap:

**Terminal**
```shell
kubectl get pod redis configmap/example-redis-config
```

**Expected Output**
| NAME                          | READY | STATUS  | RESTARTS | AGE  |
|-------------------------------|-------|---------|----------|------|
| pod/redis                     | 1/1   | Running | 0        | 10s  |
| configmap/example-redis-config |       |         |          | 10s  |


#### 2. Describe the ConfigMap to confirm its contents:

**Terminal**
```shell
kubectl describe configmap/example-redis-config
```

#### 3. Confirm Manifest configuration

**Expected Output**
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
```

> View the full [`redis-pod.yaml`](https://github.com/SteveUseful/ShopifyExampleDevDoc/blob/shopify-style-updates/content/en/examples/pods/config/redis-pod.yaml) file in the GitHub repository.
> **Pro Tip**: Always review your manifest files to ensure they align with your intended deployment setup.




### Step 5: Verify Redis Settings with Redis CLI
After applying, verify that both the `Redis Pod` and the `ConfigMap` are configured correctly in your Kubernetes cluster.

#### 1. Connect to the Redis CLI:

**Terminal**
```shell
kubectl get pod/redis configmap/example-redis-config 
```

#### 2. Check the `maxmemory` setting:
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

#### 3. Check the eviction policy:

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Key              | Value      |
|------------------|------------|
| maxmemory-policy | noeviction |


### Step 6: Update the ConfigMap Dynamically
Modify the ConfigMap to enable custom Redis configurations.

#### 1. Edit the example-redis-config.yaml file:

**Terminal**
```shell
cat <<EOF > ./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
EOF
```

#### 2. Apply the updated ConfigMap:

**Terminal**
```shell
kubectl apply -f example-redis-config.yaml 
```

#### 3. Restart the Redis Pod:
**Terminal**
```shell
kubectl delete pod redis
```
**Terminal**
```shell
kubectl apply -f redis-pod.yaml
```

### Step 7: Verify Updated Settings
#### 1. Access the Redis CLI and check updated configurations:

**Terminal**
```shell
kubectl exec -it redis -- redis-cli
```

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

**Terminal**
```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

**Expected Output**
| Configuration Key   | Expected Value |
|---------------------|----------------|
| maxmemory           | 2097152        |
| maxmemory-policy    | allkeys-lru    |


### Step 8: Clean up resources
After completing the tutorial, clean up the resources to avoid unnecessary resource consumption in your Kubernetes cluster. Cleaning up resources ensures your cluster remains efficient and avoids potential conflicts with future deployments.

#### 1. Delete the Redis Pod and ConfigMap

**Terminal**
```shell
kubectl delete pod/redis configmap/example-redis-config
```
> It's important to remove resources that are no longer needed to avoid unnecessary resource consumption in your Kubernetes cluster. Ensure that both the `Redis Pod` and `ConfigMap` are successfully deleted.

#### 2. Verify Deletion
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

### Quick Recap
| Action                     | Command                                  | Output Validation                  |
|----------------------------|------------------------------------------|-------------------------------------|
| Create ConfigMap           | `kubectl apply -f example-redis-config.yaml` | ConfigMap appears in `kubectl get configmaps`. |
| Deploy Redis Pod           | `kubectl apply -f redis-pod.yaml`       | Pod status is `Running`.           |
| Verify Redis Configurations | `kubectl exec -it redis -- redis-cli`   | `maxmemory` reflects updates.      |
| Clean Up Resources         | `kubectl delete pod redis configmap/example-redis-config` | No resources found in namespace.   |


### Next Steps
Congratulations on completing this tutorial! You’ve successfully deployed Redis using a ConfigMap in Kubernetes and verified its configuration. Here are some ways to build on this experience:

- **Configuration Example**: Review a practical example of [updating configurations via a ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/), showing how changes can be applied dynamically within a live Kubernetes environment.
- **ConfigMaps in Depth**: Gain a deeper understanding of [ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) and how they enable flexible configurations for containerized applications.

### Learn more with Community Resources!
The Kubernetes community offers resources where developers and users share insights and solve challenges together.

- **Kubernetes Support Resources**: Visit the [Kubernetes Community Support page](https://kubernetes.io/community/) to access Youtube tutorials, live examples, and guides designed to help you succeed.
- **Kubernetes Forum**: Join the discussion on the [Kubernetes Forum](https://discuss.kubernetes.io/)—an ideal place for sharing ideas, asking questions, and finding support from other Kubernetes users and developers.
---

***Was this page helpful?**
**Yes** | **No**

---
