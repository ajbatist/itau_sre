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

~~~
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "app-cluster"
  cluster_version = "1.29"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id
}
~~~

Comandos:
bash
terraform init
terraform apply
aws eks --region us-east-1 update-kubeconfig --name app-cluster

### 2. Instalar e Configurar ArgoCD

Passos:
1- Aplicar o manifest oficial:
bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

2- Expor o ArgoCD com LoadBalancer ou Ingress.
3- Configurar um Application apontando para o repositório GitOps:
~~~
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/seu-usuario/app-gitops.git
    targetRevision: HEAD
    path: helm/app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
~~~

### 3. Criar Helm Chart da Aplicação
Estrutura do chart:
bash
~~~
helm/app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml

~~~

Exemplo values.yaml:
yaml
~~~
image:
  repository: <ECR_URL>/app
  tag: "latest"
service:
  type: ClusterIP
  port: 80
~~~

 ### 4. Criar Pipeline GitHub Actions
.github/workflows/deploy.yml
yaml
~~~
name: CI/CD

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build and push image
      run: |
        docker build -t ${{ secrets.ECR_URI }}:latest .
        docker push ${{ secrets.ECR_URI }}:latest

    - name: Update GitOps Repo
      run: |
        git clone https://github.com/seu-usuario/app-gitops.git
        cd app-gitops/helm/app
        yq e -i '.image.tag = "latest"' values.yaml
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git commit -am "Update image tag"
        git push

 ~~~

### 5. Instrumentar Aplicação com OpenTelemetry

Back-end (ex: Node.js):
 - Integrar SDK do OpenTelemetry.
 - Exportar métricas e traces para Datadog ou Collector.

Exemplo Node.js:
~~~
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: '<collector-endpoint>' }),
  instrumentations: [getNodeAutoInstrumentations()]
});
sdk.start();
~~~

Deploy do OpenTelemetry Collector no cluster:
bash
~~~
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install otel open-telemetry/opentelemetry-collector -f custom-values.yaml
~~~

### 6. Dashboard no Datadog ou Grafana

#### Datadog
Enviar traces e métricas para o agente Datadog com o token da conta.
Visualizar painéis no APM e Logs.

#### Grafana
Instalar Grafana via Helm.
Configurar o Prometheus como data source.
Importar dashboards via JSON (OpenTelemetry, Kubernetes, App).

### 7. Documento Final
Diagrama de Arquitetura
Pode ser feito com Lucidchart, draw.io, ou PlantUML. O diagrama deve incluir:
GitHub → Actions → ECR → GitOps repo
ArgoCD → EKS → Helm
OpenTelemetry → Collector → Datadog/Grafana

 ### SLIs e SLOs Propostos

| **SLI**         | **Descrição**           | **SLO**   |
| --------------- | ----------------------- | --------- |
| Latência HTTP   | Tempo médio de resposta | < 300ms   |
| Disponibilidade | Uptime da aplicação     | 99.9%     |
| Taxa de erro    | Percentual de erros 5xx | < 1%      |
| Throughput      | Requisições por segundo | ≥ 100 rps |

### Estratégia de Rollout Seguro

 * Usar Argo Rollouts ou Helm com canary:
  ** Dividir o tráfego em 10%, 30%, 100%
* Healthchecks com readinessProbe/livenessProbe
* Monitoramento com métricas + rollback automático no ArgoCD



