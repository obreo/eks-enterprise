# EKS Enterprise

Setup and manage an enterprise-grade Kubernetes cluster on AWS EKS, that follows DevOps and GitOps practices with advanced features and integrations.

This project is designed to elaborate how to deploy a scalable, cost-effective, and secured EKS cluster, following the best practices while implementing automation in every stage using DevOps concepts and GitOps workflows.

## Overview

Running a Kubernetes cluster is relatively straightforward, but managing and scaling it in a production environment is a much bigger challenge.

This project demonstrates a multi-stage process of building and deploying applications to an **EKS cluster**. It begins with setting up a **custom Terraform modules** backed by a centralized state for multi environment management that deploys the infrastructure through GitOps practices, and extends to managing Kubernetes deployments through with **ArgoCD**, which continuously fetches and applies containerized applications from a registry.

The goal is to **automate the entire workflow** across all stages while ensuring security, scalability, and reliability for production workloads: 
- **Containerization**: Containerize applications using Docker and push them to Amazon ECR.
- **Infrastructure as Code (IaC)**: Provision AWS infrastructure with Terraform, delivered through Continuous Delivery pipelines. Setting up a multi-environment architecture (development, staging, production) with isolated state files, for an automated, ready with core plugins EKS cluster.
- **Application Delivery**: Manage Kubernetes manifests with ArgoCD, triggered automatically when new container images are built and pushed to the registry.  

Together, these components form an integrated **DevOps + GitOps workflow** that enables multiple teams to manage both their infrastructure and applications in a secure, scalable, and production-ready manner.

## Architecture
![Architecture](./architecture.png)

The EKS cluster will be deployed as IPv6 only version, on two private subnets in a dual stack VPC. Although the cluster uses IPv6 only, some of the EKS API components and most AWS services support IPv4 or dual stack (IPv4 and IPv6 should exist), hence Egress-Only Gateway and NAT gateway are mapped to the private subnets to allow internet access.

EKS IPv6 mode allows solving ENI pod exhaustion, means we can launch **smaller EC2 instances with more pods**, scaling out nodes based on CPU/memory usage rather than being limited by [ENI max pods per instance](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/misc/eni-max-pods.txt).

Clients can access the application based on dual stack application load balancer that routes traffic to IPv6 pods.

It worths noting that it is possible to replace the NAT gateway with multiple VPC endpoints to allow private access to AWS services, this wouldn't be a problem considering that the subnets support dual stack. However, using multiple endpoints will increase the cost and complexity of the architecture.

This architecture is based on three custom Terraform modules:

### VPC Module

**Related Repository:**
[eks-enterprise-terraform-vpc](https://github.com/obreo/eks-enterprise-terraform-vpc.git)

It creates four subnets based on two availability zones, two are private where EKS cluster resides, while worker node resides in a single availability zone subnet, and two public where NAT gateway resides. In addition to Internet and Egress-Only gateways.

The module creates security groups of EKS cluster and ALB. The module exports the subnets and security groups as outputs that will be used by the EKS module.

###  EKS Module

**Related Repository:**
[eks-enterprise-terraform-eks](https://github.com/obreo/eks-enterprise-terraform-eks.git)

It creates an EKS cluster and t3.medium type worker node with IPv6 only in the private subnets created by the VPC. It deploys the AWS basic Addons like CoreDNS, Pod identity association agent, cloudwatch, EBS driver.

Additionally, it uses a secondary custom module that installs core plugins for EKS, including:

1. S3 Gateway Endpoint for private access to S3 buckets.
2. EBS CSI Driver
3. Cluster Autoscaler
4. Metrics Server
5. Cert Manager
6. ArgoCD
7. Nginx Controller (IPv6)
8. ALB Controller (Dual-stack)
9. External Secrets Operator
10. ECR repositories

All these plugins are deployed with the serviceAccount and IRSA permissions required.

## GitOps Workflow

The Terraform VPC and EKS modules are deployed using continues delivery (CD) where every environment uses a seperate state file that is stored on S3 bucket.

The workflow runs as following:
1. Clones the source repository.
2. Retrieves updated Terraform modules.
3. Plans and applies the deployment per environment.
   * **Staging:** replicates production.
   * **Development:** verifies the plan before promotion.
   * **Production:** main environment.

The workflow uses **GitHub Actions** using **OpenID Connect (OIDC)** to authenticate with AWS.

### EKS Applications

**Related Repository:**
[eks-enterprise-kubernetes-manifests](https://github.com/obreo/eks-enterprise-kubernetes)

The applications are deployed using ArgoCD, which is installed as part of the EKS module. The applications are defined as Kubernetes manifests in a separate Git repository. The ArgoCD is configured to watch the repository and deploy the applications automatically.

Every environment has its own namespace, and the applications are deployed based on the environment directory.

For more information, see the [repository]()

## Application CI/CD

**Related Repository:**
[eks-enterprise-application-backend](https://github.com/obreo/eks-enterprise-application-backend) | [eks-enterprise-application-frontend](https://github.com/obreo/eks-enterprise-application-frontend)

The applications are containerized and pushed to ECR using GitHub Actions. The workflow builds the Docker image, tags it, and pushes it to the ECR repository based on the branch environment.

Then ArgoCD detects the new image tag and deploys it to the EKS cluster.

## Autoscalability and Load Balancing

The EKS cluster is designed to be highly scalable. The following features are implemented to achieve this:

1. **Cluster Autoscaler**: Automatically adjusts the number of EC2 instances in the cluster based on the resource requests of the pods. This plugin is installed as part of the EKS module using Helm chart.

2. **Horizontal Pod Autoscaler**: Automatically scales the number of pod replicas based on CPU/memory usage. This is achieved by monitoring the resource usage of the pods and adjusting the replica count accordingly.

3. **Load Balancing**: This is based on two controllers:
   * **Nginx Ingress Controller**: Manages internal load balancing for the applications within the cluster. It exposes a single endpoint using ALB load balancer and routes traffic to the appropriate services based on the defined ingress rules.
   * **AWS Application Load Balancer (ALB)**: Manages external load balancing for the applications. It routes traffic to the Nginx Ingress Controller based on the defined rules and supports dual-stack (IPv4 and IPv6) access.


# Resources
- [Terraform Custom Modules](https://github.com/obreo/iac-modules.git)
- [AWS VPC CNI](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html)
- [Running EKS with IPv6](https://docs.aws.amazon.com/eks/latest/best-practices/ipv6.html)
- [EKS Best Practices](https://docs.aws.amazon.com/eks/latest/best-practices/)
- [AWS IPv6 Services Support List](https://docs.aws.amazon.com/vpc/latest/userguide/aws-ipv6-support.html)
