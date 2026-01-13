# PROJETO DEVOPS LABORATÓRIO ESIG

<img width="1536" height="1024" alt="projeto-devops" src="https://github.com/user-attachments/assets/de84183a-bbfc-4086-8041-2cd8ffb8f80c" />

# Jenkins (WAR) em Tomcat/JBoss + Jolokia + Kubernetes + Prometheus

[![Docker](https://img.shields.io/badge/docker-ready-blue)](#)
[![Kubernetes](https://img.shields.io/badge/kubernetes-manifests-blue)](#)
[![Observability](https://img.shields.io/badge/metrics-jolokia%20%2B%20prometheus-brightgreen)](#)

## Sumário
- [Objetivo](#objetivo)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Etapa 1 — Docker (Tomcat/JBoss + Jenkins WAR + Jolokia)](#etapa-1--docker-tomcatjboss--jenkins-war--jolokia)
- [Etapa 2 — Kubernetes](#etapa-2---kubernetes-deployment--service)
- [Etapa 3 — Monitoramento (Prometheus + Node Exporter)](#3-monitoramento-prometheus--jolokia--node-exporter)
- [Segurança (boas práticas aplicadas)](#segurança-boas-práticas-aplicadas)


---

## Objetivo
Este repositório implementa um Jenkins atualizado via **WAR** rodando em um servidor de aplicação (**Tomcat ou JBoss**),
containerizado em **Docker** e com métricas expostas via **Jolokia**. Em seguida, o mesmo deploy é realizado em **Kubernetes**
com coleta de métricas por **Prometheus** e **Node Exporter**.

> Requisitos do desafio: Jenkins WAR em JBoss/Tomcat, métricas via Jolokia, Kubernetes (Deployment/Service) e monitoramento com Prometheus + Node Exporter.  

---

## Arquitetura
**Local (Docker):**
- Container: Tomcat/JBoss + Jenkins WAR
- Jolokia agent exposto em `/jolokia` (porta definida no compose)
- Acesso Jenkins via `http://localhost:8080`

**Kubernetes:**
- Deployment + Service para app
- Prometheus scraping do endpoint Jolokia
- Node Exporter para métricas dos nós


---

## Pré-requisitos
- Docker e Docker Compose
- Kubernetes (minikube, kind, k3d, ou cluster real)
- kubectl
- (Opcional) Helm

---

## Etapa 1 — Docker (Tomcat/JBoss + Jenkins WAR + Jolokia)

### 1. Estrutura de Pastas
```bash

docker/
  Dockerfile
  docker-compose.yml
  jenkins.war
  jolokia-jvm.jar
  jolokia.properties

```

### 2. Baixar os Artefatos (Fontes confiáveis)

- Baixe jenkins.war (última versão) e salve em: docker/jenkins.war
- Baixe o Jolokia JVM agent (jolokia-jvm.jar) e salve em: docker/jolokia-jvm.jar

https://hub.docker.com/r/jolokia/java-jolokia?utm_source=chatgpt.com

### 3. Configurar Jolokia

- Crie docker/jolokia.properties:
```bash
host=127.0.0.1
port=8778
agentId=jenkins
discoveryEnabled=false
```
Se for LAB local: o mínimo seguro é “não publicar” a 8778

- Você pode manter Jolokia rodando, mas não expor a porta fora do container

- E quando precisar acessar o Jolokia, você faz via host usando docker exec ou publica temporariamente.

- Se quiser acessar do host sem publicar, dá pra usar docker exec + curl de dentro do container:
```bash
docker exec -it jenkins sh -lc 'curl -s http://127.0.0.1:8778/jolokia/version'
```

### 4. Dockerfile

- Use imagem com Java 17
- Rodar como usuário não-root
- Não expor Jolokia publicamente

### 5. Docker-compose

- Mapeie portas assim:

Jenkins/Tomcat: 127.0.0.1:8080:8080 /
Jolokia: 127.0.0.1:8778:8778

### 6. Subir e validar
```bash
cd docker
docker compose up -d --build
docker compose logs -f
```
Validar Jenkins:
- http://localhost:8080

Validar Jolokia:
```bash
curl -s http://127.0.0.1:8778/jolokia/version | jq
```

---

## Etapa 2 — Kubernetes (Deployment + Service)

### 1. Criar namespaces
```bash
kubectl create namespace jenkins-lab
kubectl create namespace monitoring
```
### 2. Publicar a imagem no cluster
```bash
cd docker
docker build -t jenkins-jolokia:latest .
kind load docker-image jenkins-jolokia:latest
```

### 3. Manifestos (base)

Crie em k8s/base/:

- deployment.yaml (com securityContext, liveness/readiness, volume para Jenkins home)
- service.yaml ClusterIP (nada exposto externamente)

Boas práticas de segurança no Deployment (recomendadas):

- runAsNonRoot: true
- allowPrivilegeEscalation: false
- readOnlyRootFilesystem: false (Jenkins precisa escrever; em vez disso, monte volumes apenas onde precisa)
- resources (requests/limits)
- imagePullPolicy: IfNotPresent

Aplicar:
```bash
kubectl apply -f k8s/base/ -n jenkins-lab
kubectl get pods -n jenkins-lab
kubectl get svc -n jenkins-lab
```
### 4. Acessar Jenkins com port-forward (mais seguro)
```bash
kubectl -n jenkins-lab port-forward svc/jenkins 8080:8080
```
- Acesse: http://localhost:8080

### 5. Validar Jolokia no cluster

Se o Jolokia estiver no Pod, valide com port-forward temporário:
```bash
kubectl -n jenkins-lab port-forward svc/jenkins 8778:8778
curl -s http://127.0.0.1:8778/jolokia/version | jq
```

---

## Etapa 3 — Monitoramento (Prometheus + Jolokia + Node Exporter)

### 1. Node Exporter
Aplicar:
```bash
kubectl apply -f k8s/monitoring/node-exporter/ -n monitoring
kubectl get pods -n monitoring -l app=node-exporter
```

### 2. Prometheus (Deployment + ConfigMap + Service)
Aplicar:
```bash
kubectl apply -f k8s/monitoring/prometheus/ -n monitoring
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```
Acessar Prometheus (port-forward):
```bash
Acessar Prometheus (port-forward):
```
Abrir:
- http://localhost:9090/targets

### 3. Garantir que “Targets” estão UP

Você precisa ver:

- job do node-exporter como UP
- job do jenkins/jolokia como UP












