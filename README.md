# PROJETO DEVOPS LABORATÓRIO

<img width="1536" height="1024" alt="fluxograma" src="https://github.com/user-attachments/assets/95986203-cd84-41b7-97db-78084ca6e9fd" />


[![Docker](https://img.shields.io/badge/docker-ready-blue)](#)
[![Kubernetes](https://img.shields.io/badge/kubernetes-manifests-blue)](#)
[![Observability](https://img.shields.io/badge/metrics-jolokia%20%2B%20prometheus-brightgreen)](#)

## Sumário
- [Arquitetura](#laborat%C3%B3rio-devops--jenkins--docker--kubernetes--prometheus-jolokia)
- [Etapa 1 — Docker (Jenkins Oficial + Jolokia)](#etapa-1--docker-jenkins-oficial--jolokia)
- [Etapa 2 — Kubernetes (Deployment + Service)](#etapa-2--kubernetes-deployment--service)
- [Etapa 3 — Monitoramento (Prometheus + Jolokia + Node Exporter)](#etapa-3--monitoramento-prometheus--jolokia--node-exporter)
- [Segurança (boas práticas aplicadas)](#boas-pr%C3%A1ticas-de-seguran%C3%A7a-aplicada)


---

# Laboratório DevOps — Jenkins + Docker + Kubernetes + Prometheus (Jolokia)

Este repositório documenta a montagem de um laboratório em **3 etapas**:
1) **Docker**: Jenkins (imagem oficial) + **Jolokia Agent** (métricas JVM/Jenkins) com segurança  
2) **Kubernetes (kind)**: implantação via **Deployment + Service**  
3) **Monitoramento**: **Prometheus + Node Exporter + Jolokia Exporter** (com **Auth Proxy** para Jolokia)

> Ambiente:VM Ubuntu 22.04 server (VirtualBox) + Docker + kind + kubectl

## Estrutura do repositório
```bash
projeto-jenkins/
├─ docker/
│  ├─ Dockerfile
│  ├─ docker-entrypoint.sh
│  └─ jolokia-access.xml
└─ k8s/
   ├─ jenkins.yaml
   ├─ jenkins-mon.yaml
   ├─ prometheus.yaml
   ├─ node-exporter.yaml
   └─ grafana.yaml
```

## Etapa 1 — Docker (Jenkins Oficial + Jolokia)

## 1) Pré-requisitos
- Docker instalado e funcionando

## 2) Arquivos da etapa
Crie/garanta estes arquivos em `docker/`:

### `docker/jolokia-access.xml`
Policy mínima (somente leitura):
```xml
<?xml version="1.0" encoding="utf-8"?>
<restrict>
  <commands>
    <command>read</command>
    <command>list</command>
    <command>version</command>
  </commands>
</restrict>
```
- Criação do arquivo no diretório docker/docker-entrypoint.sh
- Criação do arquivo no diretório docker/Dockerfile

## 3) Build da imagem
```bash
cd docker
docker build -t esig/jenkins-jolokia:2.528.3 .
```
## 4) Subir o container (porta 8778 restrita ao localhost)
```bash
docker run -d --name jenkins \
  -p 8080:8080 \
  -p 127.0.0.1:8778:8778 \
  -e JOLOKIA_USER=admin \
  -e JOLOKIA_PASSWORD='<SUA_SENHA>' \
  esig/jenkins-jolokia:2.528.3
```
## 5) Validar Jenkins 

- Acesse: http://localhost:8080
  
```bash
docker exec -it jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
## 6) Validar Jolokia (com autenticação)

- Sem auth → deve retornar 401:
  
```bash
curl -i http://127.0.0.1:8778/jolokia/version
```
- Com auth → deve retornar 200:
```bash
curl -u admin:'<SUA_SENHA>' http://127.0.0.1:8778/jolokia/version
```

## Etapa 2 — Kubernetes (Deployment + Service)

- Cluster Local com Kind

### 1) Criar cluster 
```bash
kind create cluster --name esig
```

### 2) Carregar a imagem Docker no kind
```bash
kind load docker-image esig/jenkins-jolokia:2.528.3 --name esig
```

### 3) Aplicar manifestos do Jenkins no cluster
```bash
kubectl apply -f k8s/jenkins.yaml
kubectl -n cicd get pods -w
kubectl -n cicd get svc
```
### 4) Acessar Jenkins no cluster (port-forward)

- Se a porta local 8080 estiver ocupada, use 18080.

```bash
kubectl -n cicd port-forward svc/jenkins 18080:8080
```
Acesse:
- http://localhost:18080


## Etapa 3 — Monitoramento (Prometheus + Jolokia + Node Exporter)

Nesta etapa:

Node Exporter coleta métricas do nó (CPU, memória, disco, rede)

Jolokia Exporter coleta métricas do Jenkins/JVM via Jolokia

Auth Proxy (Nginx sidecar) injeta Authorization no Jolokia (evita falhas e mantém Jolokia protegido)

Prometheus faz scrape de:

node-exporter:9100/metrics

jenkins-metrics:9422/metrics

### 1) Implementar Prometheus + Node Exporter
```bash
kubectl apply -f k8s/prometheus.yaml
kubectl -n monitoring get pods -w
```
### 2) Implantar o monitoramento do Jenkins (jenkins-mon + jenkins-metrics)
```bash
kubectl apply -f k8s/jenkins-mon.yaml
kubectl -n cicd get pods -w
kubectl -n cicd get svc jenkins-metrics
```
### 3) Validar métricas do Jenkins (exporter)

- Port-forward do service de métricas:
```bash
kubectl -n cicd port-forward svc/jenkins-metrics 19422:9422
```
- Testar:
```bash
curl -s http://127.0.0.1:19422/metrics | head -n 30
```
### 4) Validar Prometheus e targets UP
Port-forward do Prometheus:
```bash
kubectl -n monitoring port-forward svc/prometheus 19090:9090
```

Acessar:

- http://localhost:19090/targets

Você deve ver como UP:

- jenkins-jolokia (scrape em jenkins-metrics:9422/metrics)
- node-exporter (scrape em node-exporter:9100/metrics)

<img width="1780" height="895" alt="image" src="https://github.com/user-attachments/assets/c1d60d6a-f716-42b9-a44c-ce43655bd2d8" />



### 5) Grafana

<img width="1893" height="907" alt="image" src="https://github.com/user-attachments/assets/496a7a66-2293-4c88-8c58-0e3cc1b1723a" />

---

## Boas Práticas de Segurança Aplicada

Porta 8778 protegida: no Docker foi mapeada apenas para 127.0.0.1

Basic Auth habilitado no Jolokia

PolicyRestrictor com comandos mínimos (read, list, version)

No Kubernetes, Prometheus não acessa 8778 diretamente:

usa jenkins-metrics:9422 (exporter)

Jolokia fica interno no Pod

Auth Proxy Nginx: injeta Authorization e evita expor credenciais em URL







