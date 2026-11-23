# AWS EKS Kubernetes Upgrade | v1.35 to v1.36



## Project Overview



This hands-on DevOps project demonstrates an end-to-end Amazon EKS Kubernetes upgrade from version 1.35 to 1.36 using AWS CLI, eksctl, and kubectl.



The project covers cluster provisioning, workload deployment, pre-upgrade health checks, Kubernetes control plane upgrades, EKS add-on upgrades, managed worker node replacement, and post-upgrade application validation.



\*\*AWS Region:\*\* ap-south-1 (Mumbai)  

\*\*Cluster Name:\*\* eks-upgrade-lab  

\*\*Node Group:\*\* eks-lab-nodes  

\*\*Operating System:\*\* Amazon Linux 2023  

\*\*Worker Node Instance Type:\*\* t3.medium  

\*\*Worker Nodes:\*\* 2  

\*\*Kubernetes Upgrade:\*\* 1.35 → 1.36



## Architecture



```text

AWS Cloud (ap-south-1)

&#x20;       |

&#x20;       v

Amazon EKS Cluster

&#x20;       |

&#x20;       +-- EKS Control Plane (v1.36)

&#x20;       |

&#x20;       +-- Managed Node Group

&#x20;       |      |

&#x20;       |      +-- Worker Node 1 (v1.36)

&#x20;       |      |       |

&#x20;       |      |       +-- Nginx Pod

&#x20;       |      |

&#x20;       |      +-- Worker Node 2 (v1.36)

&#x20;       |              |

&#x20;       |              +-- Nginx Pod

&#x20;       |

&#x20;       +-- Kubernetes Service (ClusterIP)

&#x20;       |

&#x20;       +-- Pod Disruption Budget

&#x20;       |

&#x20;       +-- EKS Add-ons

&#x20;              |

&#x20;              +-- VPC CNI

&#x20;              +-- CoreDNS

&#x20;              +-- kube-proxy

&#x20;              +-- Metrics Server

```



## Tools and Technologies



\- AWS EKS

\- AWS CLI

\- eksctl

\- Kubernetes and kubectl

\- Amazon EC2

\- Amazon VPC

\- AWS CloudFormation

\- Amazon Linux 2023

\- Nginx

\- Git and GitHub



## Step 1 – Provision EKS Cluster



The EKS cluster was provisioned using eksctl and a declarative `cluster.yaml` configuration.



```bash

eksctl create cluster -f cluster.yaml

```



Configure kubectl:



```bash

aws eks update-kubeconfig \\

&#x20; --region ap-south-1 \\

&#x20; --name eks-upgrade-lab

```



Verify cluster nodes:



```bash

kubectl get nodes -o wide

```



## Step 2 – Deploy Sample Application



Deploy the Nginx application with two replicas.



```bash

kubectl apply -f kubernetes/deployment.yaml

kubectl apply -f kubernetes/service.yaml

kubectl apply -f kubernetes/pdb.yaml

```



Verify:



```bash

kubectl get deployments

kubectl get pods -o wide

kubectl get svc

kubectl get pdb

```



The Pod Disruption Budget uses `minAvailable: 1` to protect at least one healthy Nginx pod during voluntary disruptions.



## Step 3 – Pre-upgrade Validation



Verify Kubernetes cluster health:



```bash

kubectl get nodes

kubectl get pods -A

kubectl get pdb -A

```



Review EKS upgrade insights and confirm add-on compatibility before starting the upgrade.



Export Kubernetes resource configurations for local backup:



```bash

mkdir -p eks-backup



kubectl get deployments -A -o yaml > eks-backup/deployments.yaml

kubectl get services -A -o yaml > eks-backup/services.yaml

kubectl get pdb -A -o yaml > eks-backup/pdb.yaml

kubectl get configmaps -A -o yaml > eks-backup/configmaps.yaml

```



These exports are not a complete cluster backup. Backup files are excluded from GitHub.



## Step 4 – Upgrade EKS Control Plane



Upgrade Kubernetes from 1.35 to 1.36:



```bash

aws eks update-cluster-version \\

&#x20; --name eks-upgrade-lab \\

&#x20; --kubernetes-version 1.36 \\

&#x20; --region ap-south-1

```



Verify:



```bash

aws eks describe-cluster \\

&#x20; --name eks-upgrade-lab \\

&#x20; --region ap-south-1 \\

&#x20; --query 'cluster.{Version:version,Status:status}' \\

&#x20; --output table

```



\*\*Result:\*\* Control plane successfully upgraded to Kubernetes 1.36.



## Step 5 – Upgrade EKS Add-ons



The following add-ons were reviewed for Kubernetes 1.36 compatibility:



| Add-on | Final Version |

|---|---|

| VPC CNI | v1.22.4-eksbuild.14 |

| CoreDNS | v1.14.7-eksbuild.35 |

| kube-proxy | v1.36.0-eksbuild.65 |

| Metrics Server | v0.9.1-eksbuild.2 |



Upgrade CoreDNS:



```bash

aws eks update-addon \\

&#x20; --cluster-name eks-upgrade-lab \\

&#x20; --addon-name coredns \\

&#x20; --addon-version v1.14.7-eksbuild.35 \\

&#x20; --region ap-south-1 \\

&#x20; --resolve-conflicts PRESERVE

```



Upgrade kube-proxy:



```bash

aws eks update-addon \\

&#x20; --cluster-name eks-upgrade-lab \\

&#x20; --addon-name kube-proxy \\

&#x20; --addon-version v1.36.0-eksbuild.65 \\

&#x20; --region ap-south-1 \\

&#x20; --resolve-conflicts PRESERVE

```



VPC CNI and Metrics Server were already compatible with Kubernetes 1.36 and did not require version changes in this lab.



## Step 6 – Upgrade Managed Node Group



Upgrade the worker nodes from Kubernetes 1.35 to 1.36:



```bash

aws eks update-nodegroup-version \\

&#x20; --cluster-name eks-upgrade-lab \\

&#x20; --nodegroup-name eks-lab-nodes \\

&#x20; --kubernetes-version 1.36 \\

&#x20; --region ap-south-1

```



Wait for the managed node group to become active:



```bash

aws eks wait nodegroup-active \\

&#x20; --cluster-name eks-upgrade-lab \\

&#x20; --nodegroup-name eks-lab-nodes \\

&#x20; --region ap-south-1

```



Verify:



```bash

kubectl get nodes -o wide

```



\*\*Result:\*\* Two Kubernetes 1.35 worker nodes were replaced with two Kubernetes 1.36.5 worker nodes.



Both new nodes reached Ready status.



## Step 7 – Post-upgrade Validation



Verify the Nginx deployment:



```bash

kubectl rollout status deployment/nginx-app

kubectl get deployments

kubectl get pods -o wide

```



Verify system components:



```bash

kubectl get pods -n kube-system

kubectl get pdb -A

```



Verify the managed node group:



```bash

aws eks describe-nodegroup \\

&#x20; --cluster-name eks-upgrade-lab \\

&#x20; --nodegroup-name eks-lab-nodes \\

&#x20; --region ap-south-1

```



### Validation Results



| Component | Result |

|---|---|

| EKS Control Plane | 1.36 – ACTIVE |

| Managed Node Group | 1.36 – ACTIVE |

| Worker Nodes | 2/2 Ready |

| Nginx Deployment | 2/2 Available |

| Nginx Pods | Running with zero restarts |

| CoreDNS | Healthy |

| kube-proxy | Healthy |

| VPC CNI | Healthy |

| Metrics Server | Healthy |

| Node Group Health Issues | None |



## Step 8 – Test Application Connectivity



Use Kubernetes port forwarding:



```bash

kubectl port-forward svc/nginx-service 8080:80

```



Open:



`http://localhost:8080`



Expected application page:



\*\*Welcome to nginx!\*\*



## Key Learnings



\- Performed an EKS Kubernetes minor-version upgrade.

\- Validated cluster health and upgrade prerequisites.

\- Reviewed Kubernetes add-on compatibility.

\- Upgraded CoreDNS and kube-proxy.

\- Upgraded an EKS managed node group.

\- Observed worker node cordoning, draining, and replacement.

\- Used Pod Disruption Budgets to manage voluntary disruptions.

\- Verified Nginx application health after the upgrade.

\- Practiced AWS CLI, eksctl, and kubectl operations.



## Cleanup



To avoid ongoing AWS charges, delete the lab cluster after completing validation.



```bash

eksctl delete cluster -f cluster.yaml --wait

```



Verify that the EKS cluster and associated CloudFormation stacks, EC2 instances, networking resources, and load balancers have been removed. Review AWS billing for remaining resources.



## Project Outcome



Successfully upgraded an Amazon EKS cluster from Kubernetes 1.35 to 1.36, including the control plane, managed add-ons, and worker nodes, while maintaining the Nginx application's desired availability during the observed upgrade.



\*\*Note:\*\* This lab used eksctl and AWS CloudFormation for infrastructure provisioning, not Terraform.

