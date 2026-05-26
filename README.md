
Docker-Desktop - Networking MaP
Component Breakdown -
• Edge Entry Point: Nginx accepts external traffic and bridges the public-to-private boundary by forwarding requests to the inner cluster network.
• Private Ingress: An isolated Istio ingress gateway that listens only to internal cluster communication. It uses app-gateway and app-routes to filter traffic for the /api prefix.
• Network Isolation: The allow-only-istio-ingress policy blocks all traffic to the secure-apps namespace unless it originates explicitly from istio-system.
Mesh Sidecar: The destination pod intercepts incoming traffic via an injected Envoy proxy sidecar before delivering the payload to your application container on port 5678.

[ PUBLIC USER TRAFFIC ]
         │
         ▼ (Port 80 / NodePort 30080)
┌────────────────────────────────────────────────────────────────────────┐
│ ISTIO-SYSTEM NAMESPACE                                                 │
│                                                                        │
│  ┌───────────────────────────┐                                         │
│  │ Service: public-edge      │                                         │
│  └─────────────┬─────────────┘                                         │
│                │                                                       │
│                ▼                                                       │
│  ┌───────────────────────────┐                                         │
│  │ Pod: edge-public-proxy    │ (Nginx reverse proxy)                   │
│  │   └─► proxy_pass          │                                         │
│  └─────────────┬─────────────┘                                         │
│                │                                                       │
│                ▼ (http://cluster.local)                                │
│  ┌───────────────────────────┐                                         │
│  │ Service: private-gateway  │ (ClusterIP Internal Service)            │
│  └─────────────┬─────────────┘                                         │
│                │                                                       │
│                ▼ (Port 8080)                                           │
│  ┌───────────────────────────┐                                         │
│  │ Pod: private-ingress      │ ◄── Binds To ── Gateway: app-gateway   │
│  └─────────────┬─────────────┘                                         │
└────────────────┼───────────────────────────────────────────────────────┘
                 │
                 │ ◄─── Allowed By ── NetworkPolicy: allow-only-istio-ingress
                 │
┌────────────────▼───────────────────────────────────────────────────────┐
│ SECURE-APPS NAMESPACE                                                  │
│                                                                        │
│  ┌───────────────────────────┐                                         │
│  │ Service: backend-service  │ ◄── Routed By ── VirtualService: /api   │
│  └─────────────┬─────────────┘                                         │
│                │                                                       │
│                ▼ (Port 80)                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Pod: secure-api-deployment                                       │  │
│  │                                                                  │  │
│  │  ┌────────────────────────┐       ┌───────────────────────────┐  │  │
│  │  │ Envoy Sidecar Proxy    │ ────► │ Container: my-app         │  │  │
│  │  │ (istio-proxy)          │       │ (hashicorp/http-echo:5678)│  │  │
│  │  └────────────────────────┘       └───────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘

[ PRIVATE VPC TRAFFIC ] (VPN, Direct Connect, or internal VPC subnets)
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│ AWS INFRASTRUCTURE LAYER                                               │
│                                                                        │
│  ┌────────────────────────────────────────┐                            │
│  │ AWS Internal NLB                       │                            │
│  │ (Target Type: "ip" / Direct Routing)   │                            │
│  └───────────────────┬────────────────────┘                            │
└──────────────────────┼─────────────────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐ (Bypasses kube-proxy entirely)
         ▼ (Port 80/443 -> 8080/8443)▼
┌────────────────────────────────────────────────────────────────────────┐
│ KUBERNETES CLUSTER (ISTIO-SYSTEM NAMESPACE)                            │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Pods: istio-private-ingressgateway (Deployment)                 │  │
│  │                                                                  │  │
│  │   [Port 15021] Health Checks                                     │  │
│  │   [Port 8080]  HTTP ◄─── Binds To ─── Gateway: aws-app-gateway   │  │
│  │   [Port 8443]  HTTPS                                             │  │
│  └─────────────────────────────────┬────────────────────────────────┘  │
└────────────────────────────────────┼───────────────────────────────────┘
                                     │
                                     │ ◄── NetworkPolicy: allow-only-istio-ingress
                                     │     (Blocks non-istio-system traffic)
                                     │
┌────────────────────────────────────▼───────────────────────────────────┐
│ KUBERNETES CLUSTER (SECURE-APPS NAMESPACE)                             │
│                                                                        │
│  ┌────────────────────────────────────────┐                            │
│  │ Service: backend-service               │ ◄── VirtualService: /api   │
│  └───────────────────┬────────────────────┘                            │
│                      │                                                 │
│            ┌─────────┴─────────┐ (Load balanced traffic distribution)  │
│            ▼                   ▼                                       │
│  ┌──────────────────┐ ┌──────────────────┐                             │
│  │ Pod 1: secure-api│ │ Pod 2: secure-api│                             │
│  │                  │ │                  │                             │
│  │ ┌──────────────┐ │ │ ┌──────────────┐ │                             │
│  │ │ Envoy Sidecar│ │ │ │ Envoy Sidecar│ │ (istio-proxy)               │
│  │ └──────┬───────┘ │ │ └──────┬───────┘ │                             │
│  │        ▼         │ │        ▼         │                             │
│  │ ┌──────────────┐ │ │ ┌──────────────┐ │                             │
│  │ │ App Container│ │ │ │ App Container│ │ (hashicorp/http-echo:5678)  │
│  │ └──────────────┘ │ │ └──────────────┘ │                             │
│  └──────────────────┘ └──────────────────┘                             │
└────────────────────────────────────────────────────────────────────────┘

Here is the architectural network map for your AWS-native EKS and Istio configuration.
AWS & Kubernetes Hybrid Network Map
text
[ PRIVATE VPC TRAFFIC ] (VPN, Direct Connect, or internal VPC subnets)
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│ AWS INFRASTRUCTURE LAYER                                               │
│                                                                        │
│  ┌────────────────────────────────────────┐                            │
│  │ AWS Internal NLB                       │                            │
│  │ (Target Type: "ip" / Direct Routing)   │                            │
│  └───────────────────┬────────────────────┘                            │
└──────────────────────┼─────────────────────────────────────────────────┘
                       │
         ┌─────────────┴─────────────┐ (Bypasses kube-proxy entirely)
         ▼ (Port 80/443 -> 8080/8443)▼
┌────────────────────────────────────────────────────────────────────────┐
│ KUBERNETES CLUSTER (ISTIO-SYSTEM NAMESPACE)                            │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Pods: istio-private-ingressgateway (Deployment)                 │  │
│  │                                                                  │  │
│  │   [Port 15021] Health Checks                                     │  │
│  │   [Port 8080]  HTTP ◄─── Binds To ─── Gateway: aws-app-gateway   │  │
│  │   [Port 8443]  HTTPS                                             │  │
│  └─────────────────────────────────┬────────────────────────────────┘  │
└────────────────────────────────────┼───────────────────────────────────┘
                                     │
                                     │ ◄── NetworkPolicy: allow-only-istio-ingress
                                     │     (Blocks non-istio-system traffic)
                                     │
┌────────────────────────────────────▼───────────────────────────────────┐
│ KUBERNETES CLUSTER (SECURE-APPS NAMESPACE)                             │
│                                                                        │
│  ┌────────────────────────────────────────┐                            │
│  │ Service: backend-service               │ ◄── VirtualService: /api   │
│  └───────────────────┬────────────────────┘                            │
│                      │                                                 │
│            ┌─────────┴─────────┐ (Load balanced traffic distribution)  │
│            ▼                   ▼                                       │
│  ┌──────────────────┐ ┌──────────────────┐                             │
│  │ Pod 1: secure-api│ │ Pod 2: secure-api│                             │
│  │                  │ │                  │                             │
│  │ ┌──────────────┐ │ │ ┌──────────────┐ │                             │
│  │ │ Envoy Sidecar│ │ │ │ Envoy Sidecar│ │ (istio-proxy)               │
│  │ └──────┬───────┘ │ │ └──────┬───────┘ │                             │
│  │        ▼         │ │        ▼         │                             │
│  │ ┌──────────────┐ │ │ ┌──────────────┐ │                             │
│  │ │ App Container│ │ │ │ App Container│ │ (hashicorp/http-echo:5678)  │
│  │ └──────────────┘ │ │ └──────────────┘ │                             │
│  └──────────────────┘ └──────────────────┘                             │
└────────────────────────────────────────────────────────────────────────┘

Infrastructure Differences & Flow Highlights
• AWS Native Bypass: The AWS Load Balancer Controller targets your network interface directly using target-type: "ip". Traffic completely skips the standard Kubernetes node-routing layers (kube-proxy), streaming directly from the AWS NLB into the Istio ingress pods.
• Security Isolation: AWS security groups protect the entry path at the network perimeter, while the Kubernetes NetworkPolicy stops cross-namespace lateral movement inside the cluster.
• Load Balancing: Traffic routing scales across two target pod replicas under the secure-apps namespace via Istio's service discovery mechanisms.
<img width="925" height="1853" alt="image" src="https://github.com/user-attachments/assets/57237a41-6f4f-4b83-b686-32542da9b5a1" />


