Kustomize Configuration Management for Resume Matcher
Declarative Kubernetes configuration management using bases and overlays. Simplifying environment‑specific deployments while maintaining reusable manifests.

## Access the walkthrough

[![Kustomize walkthrough](https://img.youtube.com/vi/AGKW1t756Fw/0.jpg)](https://www.youtube.com/embed/AGKW1t756Fw?si=gcPf2MJCwWkJAqZo)

[Watch the Kustomize walkthrough on YouTube](https://www.youtube.com/embed/AGKW1t756Fw?si=gcPf2MJCwWkJAqZo)

🛠 Deployment Strategy
<div align="center">
<img src="images/kustomize/kustomize.gif" width="1000"/>
</div>

Step 1: Base Layer Setup
A multi‑node Kind cluster was created using kind-node.yaml configuration. The config defined one control‑plane and three worker nodes, with custom networking subnets:


kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: false
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
    
Cluster creation was triggered with:

`kind.exe create cluster --name kind-node \
  --config kind-node.yaml \
  --image kindest/node:v1.27.3`
  
Observation: Initially, all nodes appeared in NotReady state. Within ~1 minute, they transitioned to Ready, confirming a healthy 4‑node cluster.

<div align="center">
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.1.png" width="250"/>
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.2.png" width="250"/>
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.3.png" width="250"/>
</div>

Step 2: Base Manifests
Inside the kustomize/base directory, YAML manifests (Secrets, ConfigMaps, Deployments, Services, PVCs) were placed. A kustomization.yaml declared these resources:

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - db-secret.yaml
  - google-api-key.yaml
  - google-cx-id.yaml
  - postgres-pvc.yaml
  - resume-matcher-deployment.yaml
  - resume-matcher-service.yaml

Applied with:

`kubectl kustomize ./base
kubectl apply -k ./base`

Result: All pods transitioned to Running. Default configuration included two pods for the application and one pod for PostgreSQL.

<div align="center">
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.4.png" width="250"/>
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.5.png" width="250"/>
<img src="images/kustomize/S1 KUSTOMIZE/kustomize1.6.png" width="250"/>
</div>

Step 3: Overlay Creation – Dev Environment
To support environment‑specific customization, overlays were created for dev and prod. Each overlay references the base configuration and applies patches.

Development Overlay Structure
deployment-patch.yaml — overrides replica count and imagePullSecrets

service-patch.yaml — changes service type from NodePort to ClusterIP

kustomization.yaml — references ../base and applies patches

Applied with:

`kubectl apply -k  ./overlays/dev`

Verification:

Pods: Only 1 pod for resume-matcher-ghcr was running (down from 2 in base).

Service: Type successfully changed from NodePort → ClusterIP.

Endpoints: Available and mapped correctly.

Port forwarding:

bash
`kubectl port-forward service/resume-matcher-service 3100:3000`
<div align="center">
<img src="images/kustomize/S2 KUSTOMIZE/kustomize2.1.png" width="250"/>
<img src="images/kustomize/S2 KUSTOMIZE/kustomize2.2.png" width="250"/>
<img src="images/kustomize/S3 KUSTOMIZE/kustomize3.1.png" width="250"/>
</div>

Step 4: Overlay Creation – Prod Environment
The prod overlay references the base configuration but applies stronger scaling and service exposure changes.

Replicas increased to 3

CPU/memory requests and limits raised

Service type changed to LoadBalancer

Applied with:

`kubectl apply -k ./overlays/prod`
Result: Production overlay scaled the application to 3 replicas and exposed the service via LoadBalancer.

<div align="center">
<img src="images/kustomize/S4 KUSTOMIZE/kustomize4.1.png" width="250"/>
<img src="images/kustomize/S4 KUSTOMIZE/kustomize4.2.png" width="250"/>
</div>

Step 5: LoadBalancer Support with MetalLB
Since Kind does not support LoadBalancer services natively, MetalLB was installed.

bash
`kubectl create namespace metallb-system
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml`
Configured IP pools via cluster-addons/metallb-config.yaml:

yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: kind-pool
spec:
  addresses:
    - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  namespace: metallb-system
  name: kind-advertisement
spec:
  ipAddressPools:
    - kind-pool
Result: Services previously stuck in Pending were now assigned external IPs.

<div align="center">
<img src="images/kustomize/S5 KUSTOMIZE/kustomize5.1.png" width="250"/>
<img src="images/kustomize/S5 KUSTOMIZE/kustomize5.2.png" width="250"/>
</div>

Step 6: Deployment Verification
Applied with:

`kubectl apply -k overlays/prod`
Verification:

Deployment: Replicas scaled to 3, all ready.

Service: Type successfully changed to LoadBalancer.

Endpoints: Available and mapped correctly.

Port forwarding:

bash
`kubectl port-forward service/resume-matcher-service 3100:3000`
<div align="center">
<img src="images/kustomize/S6 KUSTOMIZE/kustomize6.1.png" width="250"/>
<img src="images/kustomize/S6 KUSTOMIZE/kustomize6.2.png" width="250"/>
</div>

📝 Notes
Kustomize: Declarative overlays for environment‑specific deployments.

Infrastructure as Code: Clean separation of base and overlays.

MetalLB: Enabled LoadBalancer support in Kind.

Outcome: Successfully deployed a production‑ready application architecture using Kustomize.

MetalLB: Enabled LoadBalancer support in Kind.

Outcome: Successfully deployed a production‑ready application architecture using Kustomize.
