Bare-Metal Kubernetes Ingress Guide: From NodePort to MetalLB
Date: November 27, 2025 Environment: On-Premise / Local VMs (VirtualBox/KVM) Cluster: Kubernetes v1.34.2 (3 Masters, 3 Workers)

1. Introduction & Architecture
In a standard Cloud environment (AWS, GCP, Azure), creating a Service of type LoadBalancer automatically spins up a cloud load balancer with a public IP.

In a Bare-Metal or VM-based environment (like this laptop setup), there is no cloud provider to assign IPs. Without a load balancer implementation, the EXTERNAL-IP field stays stuck in <pending>.

To solve this, we use MetalLB in Layer 2 mode. MetalLB runs inside the cluster, "claims" an IP address from your home network range, and announces to the network (via ARP) that the Kubernetes node running the Ingress Controller "owns" that IP.

Goal

To access services (like Trino) via a clean URL on Port 80 (e.g., http://trino.dev.clemlab.com) without using port forwarding or ugly ports like :32080.

2. Prerequisites
Kubernetes Cluster: Running and healthy.

Network Range: A reserved range of IPs on the VM network (LAN) that DHCP will not assign to other devices.

Our Network: 192.168.50.0/24

Our Reserved Pool: 192.168.50.30 to 192.168.50.60

3. Installation & Configuration
Step 1: Install MetalLB

Deploy the MetalLB controller and speaker pods (DaemonSet).

Bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
Verification: Ensure pods are running.

Bash
kubectl get pods -n metallb-system
# You should see 'controller' and 'speaker' pods.
Step 2: Configure the IP Address Pool

We must tell MetalLB which IPs it is allowed to hand out.

File: metallb-config.yaml

YAML
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.50.30-192.168.50.60
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
Apply:

Bash
kubectl apply -f metallb-config.yaml
Step 3: Configure Ingress Nginx

The Ingress Controller is the gateway. It must request a LoadBalancer IP from MetalLB.

The Crucial Change: We switch the Service Type from NodePort (which ignores MetalLB) to LoadBalancer (which triggers MetalLB).

Helm/Ansible Configuration:

YAML
ingress_nginx_values:
  controller:
    service:
      # CHANGED FROM NodePort TO LoadBalancer
      type: LoadBalancer
      
      # Standard ports (no high ports needed)
      ports:
        http: 80
        https: 443
        
      # REMOVED: nodePorts section (allow K8s to manage internal routing)
4. Verification
Run the following command to check if an IP was assigned:

Bash
kubectl get svc -n ingress-nginx
Success Output:

Plaintext
NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
nginx-ingress-ingress-nginx-controller   LoadBalancer   10.96.139.124   192.168.50.30   80:30287/TCP...
TYPE: Must be LoadBalancer.

EXTERNAL-IP: Must be an IP from your pool (e.g., 192.168.50.30).

5. DNS Configuration
To access the service by name from the host machine (laptop), map the new MetalLB IP to the hostname.

File: /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows)

Add Line:

Plaintext
192.168.50.30   trino-test-1.dev.clemlab.com
Test: Open http://trino-test-1.dev.clemlab.com in your browser. It should load without requiring a port number.

6. Troubleshooting Guide
Issue 1: EXTERNAL-IP is stuck at <pending>

Symptoms: kubectl get svc shows <pending> under EXTERNAL-IP indefinitely.

Checklist:

Is MetalLB running? Check kubectl get pods -n metallb-system.

Is the Pool configured? Run kubectl get ipaddresspools -n metallb-system. If it returns "No resources found", apply the config from Step 2.

Are there IPs available? Ensure the pool range is valid and not exhausted.

Issue 2: EXTERNAL-IP is <none>

Symptoms: The service is running, but it has no external IP at all.

Cause: The Service type is likely set to NodePort or ClusterIP. MetalLB ignores these types. Fix: Edit the service or Helm values to ensure type: LoadBalancer.

Issue 3: Connection Refused / Timeout

Symptoms: You have an IP, but cannot ping or curl it from the laptop.

Cause:

Virtualization Networking: Ensure the VMs are in "Bridged" mode or on a host-only network reachable by the laptop.

Firewalls: Check firewalld or iptables on the Kubernetes nodes. MetalLB uses ARP; firewalls blocking ARP or port 7946 (MetalLB internal comms) will break it.

7. Summary of Success
We successfully transitioned from a restricted NodePort setup (requiring high ports like :32080) to a production-like LoadBalancer setup using MetalLB. This allows services to be accessed on standard HTTP/HTTPS ports using "real" LAN IP addresses.

Final Configuration:

Ingress IP: 192.168.50.30

URL: http://trino-test-1.dev.clemlab.com