# PROJETO DEVOPS LABORATÓRIO ESIG

# Jenkins (WAR) em Tomcat/JBoss + Jolokia + Kubernetes + Prometheus

[![Docker](https://img.shields.io/badge/docker-ready-blue)](#)
[![Kubernetes](https://img.shields.io/badge/kubernetes-manifests-blue)](#)
[![Observability](https://img.shields.io/badge/metrics-jolokia%20%2B%20prometheus-brightgreen)](#)

## Sumário
- [Objetivo](#objetivo)
- [Arquitetura](#arquitetura)
- [Pré-requisitos](#pré-requisitos)
- [Etapa 1 — Docker (Tomcat/JBoss + Jenkins WAR + Jolokia)](#etapa-1--docker-tomcatjboss--jenkins-war--jolokia)
- [Etapa 2 — Kubernetes](#etapa-2--kubernetes)
- [Etapa 3 — Monitoramento (Prometheus + Node Exporter)](#etapa-3--monitoramento-prometheus--node-exporter)
- [Segurança (boas práticas aplicadas)](#segurança-boas-práticas-aplicadas)
- [Troubleshooting](#troubleshooting)
- [Evidências](#evidências)
- [Roadmap / Melhorias](#roadmap--melhorias)

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

> Diagrama: `images/architecture.png`

---

## Pré-requisitos
- Docker e Docker Compose
- Kubernetes (minikube, kind, k3d, ou cluster real)
- kubectl
- (Opcional) Helm

---

## Etapa 1 — Docker (Tomcat/JBoss + Jenkins WAR + Jolokia)

### 1. Baixar Jenkins WAR
- Baixe o `jenkins.war` (versão mais recente) e coloque em `docker/<tomcat|jboss>/jenkins.war`.

### 2. Build e execução (local)
```bash
cd docker
./../scripts/build.sh
./../scripts/run-local.sh
