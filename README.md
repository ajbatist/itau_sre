# Rede SRE
## POC Rede SRE
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
~~~
terraform init
terraform apply
aws eks --region us-east-1 update-kubeconfig --name app-cluster
~~~


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

yaml
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

1. Dashboard OpenTelemetry (Traces)
otel-traces-dashboard.json:
~~~
{
  "annotations": {
    "list": []
  },
  "panels": [
    {
      "id": 1,
      "title": "Active Spans",
      "type": "timeseries",
      "targets": [
        {
          "expr": "otel_distribution_count{job=\"otelcol\",span_status_code=\"OK\"}",
          "legendFormat": "OK spans",
          "refId": "A"
        }
      ],
      "datasource": "Prometheus"
    },
    {
      "id": 2,
      "title": "Latency (p95)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(otel_exporter_latency_bucket[5m])) by (le))",
          "legendFormat": "p95",
          "refId": "B"
        }
      ],
      "datasource": "Prometheus"
    }
  ],
  "schemaVersion": 36,
  "title": "OpenTelemetry Traces",
  "uid": "otel-traces"
}

~~~

2. Dashboard Kubernetes (Cluster Metrics)
k8s-cluster-dashboard.json:
~~~
{
  "annotations": { "list": [] },
  "panels": [
    {
      "id": 1,
      "title": "CPU Usage",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(container_cpu_usage_seconds_total{image!=\"\",container!=\"POD\"}[5m])) by (pod)",
          "legendFormat": "{{pod}}",
          "refId": "A"
        }
      ],
      "datasource": "Prometheus"
    },
    {
      "id": 2,
      "title": "Memory Usage",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(container_memory_usage_bytes{image!=\"\",container!=\"POD\"}) by (pod)",
          "legendFormat": "{{pod}}",
          "refId": "B"
        }
      ],
      "datasource": "Prometheus"
    }
  ],
  "schemaVersion": 36,
  "title": "Kubernetes Cluster",
  "uid": "k8s-cluster"
}

~~~

3. Dashboard da Aplicação (Métricas de Request)
app-metrics-dashboard.json:
~~~
{
  "annotations": { "list": [] },
  "panels": [
    {
      "id": 1,
      "title": "HTTP Requests per Second",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count[1m])) by (method)",
          "legendFormat": "{{method}}",
          "refId": "A"
        }
      ],
      "datasource": "Prometheus"
    },
    {
      "id": 2,
      "title": "Error Rate (5xx)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100",
          "legendFormat": "5xx %",
          "refId": "B"
        }
      ],
      "datasource": "Prometheus"
    },
    {
      "id": 3,
      "title": "Latency (p95)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))",
          "legendFormat": "p95",
          "refId": "C"
        }
      ],
      "datasource": "Prometheus"
    }
  ],
  "schemaVersion": 36,
  "title": "App Metrics",
  "uid": "app-metrics"
}

~~~

#### Como usar
Abra o Grafana e vá em "Create" → "Import".

Cole o conteúdo de cada arquivo JSON ou selecione o arquivo.

Selecione a mesma fonte de dados Prometheus usada no seu cluster.

Esses dashboards fornecem um bom ponto de partida para:

Traces (OpenTelemetry)

Recursos do cluster (CPU / RAM)

Métricas de aplicação (requests, erros, latência)


### 7. Documento Final
Diagrama de Arquitetura
O diagrama deve incluir:
GitHub → Actions → ECR → GitOps repo
ArgoCD → EKS → Helm
OpenTelemetry → Collector → Datadog/Grafana

![Arquitetura](https://github.com/ajbatist/itau_sre/blob/main/docs/arquitetura/arquitetura.png)

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

### Estrutura do Projeto
~~~
infra/
  └── terraform/
      ├── main.tf
      ├── variables.tf
      └── outputs.tf

k8s/
  └── helm/
      └── app/
          ├── Chart.yaml
          ├── values.yaml
          └── templates/
              ├── deployment.yaml
              ├── service.yaml
              └── ingress.yaml

.github/
  └── workflows/
      └── deploy.yml

otel/
  └── collector-config.yaml

docs/
  └── arquitetura.png

~~~

Instalar com Helm:
bash
helm install otel open-telemetry/opentelemetry-collector   -f otel/collector-config.yaml


