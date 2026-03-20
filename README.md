# Deploying Applications Into Kubernetes Cluster-101

## Project Overview

This project builds on your manually built Kubernetes cluster from the [k8s-cluster-from-ground-up](https://github.com/Munyat/k8s-cluster-from-ground-up) project. You will now learn how to deploy containerized applications on Kubernetes using essential objects: **Pods**, **Services**, **ReplicaSets**, and **Deployments**. You'll also explore different ways to expose applications (ClusterIP, NodePort) and understand how Kubernetes ensures desired state.

---

## Prerequisites

- A working Kubernetes cluster (3 masters, 3 workers) from the previous project.
- `kubectl` configured on your master node with the `admin.kubeconfig` file.
- Basic understanding of YAML syntax.
- (Optional) Docker Hub account and a custom image for the Tooling app.

---

## 1. Choosing the Right Kubernetes Cluster

For this project we are using the **from‑ground‑up cluster** you built earlier. That cluster is ideal for learning because it exposes every component and forces you to understand networking, certificates, and configuration. In production, you would likely use a managed service like **Amazon EKS**, but the concepts are identical.

---

## 2. Deploying a Single Pod (Nginx)

### 2.1 Create a Pod Manifest

On your **master node** (or any machine with `kubectl` pointing to your cluster), create a file `nginx-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx-pod # label used later by the service
spec:
  containers:
    - image: nginx:latest
      name: nginx-pod
      ports:
        - containerPort: 80
          protocol: TCP
```

Apply it:

```bash
kubectl apply -f nginx-pod.yaml
```

![Create a Pod Manifest for Nginx – kubectl get pods -o wide](screenshots/Create_a_Pod_Manifest_for_Nginxkubectl_get_pods-o_wide.png)

### 2.2 Access the Pod Internally

The pod gets an IP inside the cluster. Use a temporary `curl` pod to test it:

```bash
kubectl run curl --image=dareyregistry/curl -it --rm --restart=Never -- sh
```

Inside the container, curl the pod's IP (from `kubectl get pod nginx-pod -o wide`):

```bash
curl <pod-ip>:80
```

You should see the Nginx welcome page.

![Step 1.2 – Access the Pod Internally](screenshots/Step1_2Access_the_Pod_Internally.png)

### 2.3 Create a Service (ClusterIP)

A service provides a stable endpoint for the pod, even if the pod is recreated with a new IP. Create `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-pod # must match pod label
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-service.yaml
```

Check the service:

```bash
kubectl get svc nginx-service
```

![Step 1.3 – kubectl get service nginx-service](screenshots/Step_1.3kubectlgetservicenginx-service.png)

### 2.4 Access via Port‑Forward

From your master, forward traffic from your local port 8089 to the service:

```bash
kubectl port-forward svc/nginx-service 8089:80
```

Now open your browser: `http://localhost:8089`. You'll see the Nginx page.

![Port‑Forward to Access from Your Local Machine](screenshots/Port‑ForwardtoAccessfromYourLocalMachine.png)
![Browser showing Nginx page via port-forward](screenshots/Browser_showing_Nginx_page_via_port_forward.png)

---

## 3. Using ReplicaSet to Maintain Multiple Pods

Pods are ephemeral; if a pod dies, we need another to take its place. A **ReplicaSet** ensures a specified number of identical pods are always running.

### 3.1 Delete the Single Pod

```bash
kubectl delete -f nginx-pod.yaml
```

### 3.2 Create a ReplicaSet Manifest (`nginx-rs.yaml`)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - image: nginx:latest
          name: nginx-pod
          ports:
            - containerPort: 80
```

Apply:

```bash
kubectl apply -f nginx-rs.yaml
```

![Part 3 – nginx replica yaml file](screenshots/part3_nginx_replica_yamlfile.png)

### 3.3 Verify Pods

```bash
kubectl get pods
```

You should see three pods with random suffixes (e.g., `nginx-pod-j784r`).

![Part 3 – kubectl get pods 3 replicas](screenshots/part3_kubectl_get_pods_3_replicas.png)

### 3.4 Scale the ReplicaSet Imperatively

Increase replicas to 5:

```bash
kubectl scale rs nginx-rs --replicas=5
```

![Part 3 – kubectl scale rs nginx-rs replicas 5](screenshots/part3_kubectl_scale_rs_nginx_rs_replicas_5.png)

### 3.5 Describe the ReplicaSet

```bash
kubectl describe rs nginx-rs
```

![Part 3 – kubectel describe rs nginx-rs](screenshots/part3_kubectel_describe_rs_nginx_rs.png)

---

## 4. Deployments – The Recommended Way

A **Deployment** adds declarative updates, rollbacks, and rolling updates to ReplicaSets.

### 4.1 Delete the ReplicaSet

```bash
kubectl delete rs nginx-rs
```

### 4.2 Create a Deployment Manifest (`nginx-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

![Part 4 – deployment manifest yaml file](screenshots/part4deploymenymanifestyamlfile.png)

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

### 4.3 Inspect Resources

```bash
kubectl get deployments
kubectl get rs
kubectl get pods
```

![Part 4 – kubectl get deployments, rs, pods](screenshots/part4ubectlgetdeploymentskubectlgetrskubectlgetpods.png)

### 4.4 Scale the Deployment

```bash
kubectl scale deployment nginx-deployment --replicas=15
```

![Part 4 – scale deployment to 15](screenshots/part4scaledeploymentsto15.png)

### 4.5 Exec into a Pod

```bash
kubectl exec -it nginx-deployment-56466d4948-78j9c bash   # use your pod name
```

Inside, you can explore the nginx filesystem:

```bash
ls -ltr /etc/nginx/
cat /etc/nginx/conf.d/default.conf
```

![Part 4 – kubectl exec into nginx-deployment showing nginx files](screenshots/part4kubectlexec-itnginx-deployment_showing_nginx_files.png)

---

## 5. Exposing Services with NodePort

The ClusterIP service we used earlier is only reachable inside the cluster. To expose an application to the outside world, we can use a **NodePort** service.

### 5.1 Update the Service to NodePort

Modify `nginx-service.yaml` (or create a new one) to use `type: NodePort`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx-pod # or tier: frontend if using deployment
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply it:

```bash
kubectl apply -f nginx-nodeport.yaml
```

### 5.2 Verify the Service

```bash
kubectl get svc nginx-nodeport
```

![Part 4 – kubectl get svc nginx-nodeport service](screenshots/part4kubectgetscvnginxnodeportservice.png)

### 5.3 Access via Worker Node Public IP

- Get the public IP of any worker node (e.g., `54.172.207.31`).
- Ensure your security group allows inbound TCP on port `30080` from anywhere.
- Open a browser to `http://<worker-public-ip>:30080`.

You should see the Nginx page.

---

## 6. Tooling App (Self Task)

As a self study, containerize the Tooling PHP application and deploy it similarly:

- Build a Docker image of the Tooling app and push to Docker Hub.
- Write a Pod manifest for the Tooling frontend (e.g., `tooling-pod.yaml`).
- Write a Service manifest (ClusterIP) for it.
- Use port‑forward to access it.

![Tooling pod yaml file](screenshots/tooling_pod_yaml-file.png)
![Tooling service file](screenshots/tooling_service_file.png)

---

## 7. Persistence Demonstration

Deployments are stateless. To see this:

- Scale the deployment down to 1 replica.
- Exec into the remaining pod and edit `/usr/share/nginx/html/index.html`.
- Delete that pod – a new pod will be created with the original content.

![Part 4 – nginx page updated before pod deletion](screenshots/paart4nginxpageupdated.png)

After pod recreation, the custom content is gone.

---

## 8. Clean Up

```bash
kubectl delete deployment nginx-deployment
kubectl delete svc nginx-service nginx-nodeport
kubectl delete pod tooling-pod  # if created
```

![Cleanup](screenshots/lastpartcleanup.png)

---

## Troubleshooting: Pod-to-Pod Networking Issues on AWS

When building your own cluster from scratch, you may encounter pod‑to‑pod communication failures. The most common cause on AWS is the **lack of a self‑referencing security group rule**. Here is a condensed runbook from the previous project.

### Root Cause

AWS security groups drop traffic between instances in the same group unless explicitly allowed. Without a self‑referencing rule, even if iptables and routes are correct, packets are dropped at the hypervisor level.

### The Fix

```bash
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=k8s-cluster-from-ground-up" \
  --query 'SecurityGroups[0].GroupId' --output text)

# Allow ALL traffic between instances in the same security group
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol -1 \
  --source-group $SECURITY_GROUP_ID
```

### Complete Security Group Rules

| Port/Protocol   | Source         | Purpose                   |
| --------------- | -------------- | ------------------------- |
| ALL traffic     | Self (same SG) | Node‑to‑node + pod‑to‑pod |
| TCP 22          | 0.0.0.0/0      | SSH                       |
| TCP 6443        | 0.0.0.0/0      | Kubernetes API            |
| TCP 2379-2380   | 172.31.0.0/24  | etcd                      |
| TCP 10250       | 172.31.0.0/24  | kubelet API               |
| TCP 30000-32767 | 0.0.0.0/0      | NodePort services         |
| ICMP -1         | 0.0.0.0/0      | Ping                      |

### Other Required Configurations

1. **Disable Source/Destination Check** on all worker nodes.
2. **Add Pod CIDR routes** to the VPC route table.
3. **Add OS‑level routes** on each worker for other workers' pod CIDRs.
4. **Enable IP forwarding** (`net.ipv4.ip_forward=1`).
5. **Add iptables FORWARD rules** for the pod CIDR.

For full details, refer to the [Kubernetes From-Ground-Up Project](https://github.com/Munyat/k8s-cluster-from-ground-up).

---

## Key Concepts Learned

- **Pod**: smallest deployable unit; encapsulates one or more containers.
- **Service**: stable network endpoint; abstracts pod IPs.
- **Label & Selector**: how services identify pods.
- **ReplicaSet**: ensures a specified number of pod replicas.
- **Deployment**: higher‑level controller for rolling updates, scaling, and rollbacks.
- **ClusterIP**: internal service reachable only inside the cluster.
- **NodePort**: exposes service on a static port on every worker node.

---

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Previous Project: k8s-cluster-from-ground-up](https://github.com/Munyat/k8s-cluster-from-ground-up)
