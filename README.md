# AWS VPC + EKS Cluster (Terraform)

Infrastructure-as-code for a production-shaped AWS foundation: a multi-AZ VPC
with public/private subnet separation, and an EKS cluster running in the
private subnets behind a NAT gateway.

## Architecture

```
                         Internet Gateway
                                |
                +---------------+---------------+
                |                               |
        Public Subnet (AZ-a)            Public Subnet (AZ-b)
          - NAT Gateway                   - (optional 2nd NAT)
        Private Subnet (AZ-a)           Private Subnet (AZ-b)
          - EKS worker nodes               - EKS worker nodes
```

- **VPC** with DNS support enabled, CIDR `10.0.0.0/16`.
- **2 public subnets** (one per AZ) for the NAT gateway and any public-facing
  load balancers.
- **2 private subnets** (one per AZ) for EKS worker nodes — no direct
  inbound internet access.
- **1 NAT Gateway** in the public subnet so private-subnet nodes can reach
  the internet (pulling images, calling AWS APIs) without being reachable
  from it.
- **EKS cluster** provisioned via the community
  [`terraform-aws-modules/eks`](https://github.com/terraform-aws-modules/terraform-aws-eks)
  module, with a managed node group.
- Subnets are tagged (`kubernetes.io/role/elb`, `kubernetes.io/role/internal-elb`)
  so the AWS Load Balancer Controller can auto-discover them later.

## Why this structure

Everything is split into a `network` module and an `eks` module so either
piece can be reused or swapped independently (e.g. pointing the EKS module
at an existing VPC). Root-level `main.tf` wires them together and passes
outputs from `network` into `eks`.

## Usage

```bash
terraform init
terraform plan -var="cluster_name=demo-cluster" -var="region=us-east-1"
terraform apply
```

After apply, configure `kubectl`:

```bash
aws eks update-kubeconfig --region us-east-1 --name demo-cluster
kubectl get nodes
```

## Files

| Path | Purpose |
|---|---|
| `main.tf` | Root module wiring network + EKS together |
| `variables.tf` | Input variables (region, CIDR, cluster name, node sizing) |
| `outputs.tf` | Cluster endpoint, VPC id, subnet ids |
| `providers.tf` | AWS + Kubernetes provider configuration |
| `modules/network/` | VPC, subnets, IGW, NAT gateway, route tables |
| `modules/eks/` | EKS cluster + managed node group |

## Notes

This is a portfolio/demo project meant to demonstrate IaC structure and AWS
networking fundamentals, not a drop-in production template — review sizing,
NAT redundancy (a single NAT gateway is a single point of failure across
AZs), and IAM least-privilege before using it as-is.

## License

MIT
