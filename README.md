# itau_sre
## POC Itau SRE
### 1. Provisionar Cluster EKS com Terraform

   Ferramentas necessárias:
    Terraform
    AWS CLI configurado
    kubectl

  Passos principais:
    Criar main.tf com os módulos aws, eks, e vpc.
    Usar o módulo oficial: terraform-aws-modules/eks
    Configurar roles, security groups e outputs.

ˋˋˋ
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "app-cluster"
  cluster_version = "1.29"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id
}
ˋˋˋ

Comandos:
bash
terraform init
terraform apply
aws eks --region us-east-1 update-kubeconfig --name app-cluster

### 2. Instalar e Configurar ArgoCD
