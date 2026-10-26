
# 🚀 Laboratorio #11 - Estrategias de Despliegue Avanzado (Blue/Green & Canary) con GitFlow de 3 Entornos en Jenkins y Kubernetes

Este repositorio contiene una arquitectura completa de Integración y Despliegue Continuo (CI/CD) de nivel empresarial. Automatiza el ciclo de vida de una API REST desarrollada en **FastAPI**, utilizando infraestructura como código con **Terraform**, contenedores gestionados en **AWS ECR**, y estrategias avanzadas de despliegue en un clúster de **Kubernetes (Minikube)**.

## 🎯 Finalidad Principal

Automatizar al 100% el ciclo de vida del software (SDLC) mediante un pipeline de CI/CD profesional basado en Jenkins Multibranch y GitFlow, implementando estrategias de despliegue avanzadas (Canary y Blue/Green) en un clúster de Kubernetes para garantizar entregas continuas sin interrupciones (zero-downtime) y un aislamiento estricto entre los entornos de desarrollo, pruebas y producción.

## 🎯 Lo que aprenderás e implementarás

    • Estrategia Blue/Green pura en Kubernetes: Manejo de selectores de etiquetas (color: blue / color: green) en los Services para switch instantáneo (Cero Downtime).

    • Jenkins Multibranch Pipeline: Jenkins leerá tu repositorio y creará automáticamente trabajos (jobs) separados para cada rama (develop, staging, main).

    • Sondeos y Aprobaciones Manuales en Jenkins: Uso del paso input para pausar el pipeline en la etapa de producción hasta que un administrador valide el entorno Green.


## 🛠️ Tecnologías y Herramientas Utilizadas

    • Control de Versiones: Git & GitHub (GitFlow).

    • CI/CD: Jenkins (Multibranch Pipeline as Code).

    • Contenedorización: Docker & Docker Compose.

    • Orquestación: Kubernetes (Minikube).

    • Cloud (Free Tier): AWS Elastic Container Registry (ECR).

    • Infraestructura como Código: Terraform.

    • Seguridad (DevSecOps): Trivy (Escaneo de vulnerabilidades en imágenes).

    • Backend: Python (FastAPI & Uvicorn).

## 📐 Arquitectura y GitFlow (3 Entornos)
    
    El flujo de trabajo sigue el estándar **GitFlow**, donde cada rama principal está estrictamente vinculada a un entorno y una estrategia de despliegue automatizada en Jenkins:

```text
 feature/*   ──(PR/Merge)──>   develop   ──(PR/Merge)──>   staging   ──(PR/Merge)──>   main
                                  │                           │                           │
                                  ▼                           ▼                           ▼
                           Namespace: dev            Namespace: staging           Namespace: prod
                           (Rolling Update)          (Estrategia Canary 20%)      (Blue/Green Zero-Downtime)

```
## 📁 Estructura del Proyecto
```
CI-CD.GitFlow/
├── README.md                           # Documentación de estrategias GitFlow, Blue/Green y Canary
├── Jenkinsfile                         # Multibranch Pipeline Declarativo (Branch-based logic)
├── terraform/                          # Infraestructura como Código (AWS ECR & IAM)
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── k8s/                                # Manifiestos de Kubernetes para Entornos Múltiples
│   ├── namespaces.yaml                 # Creación de namespaces: dev, staging, prod
│   ├── configmaps/
│   │   ├── configmap-dev.yaml          # ConfigMap para Entorno Dev
│   │   ├── configmap-staging.yaml      # ConfigMap para Entorno Staging
│   │   └── configmap-prod.yaml         # ConfigMap para Entorno Producción
│   ├── canary/                         # Estrategia para Entorno Staging (Rama: staging)
│   │   ├── deployment-stable.yaml      # Versión Estable (80% tráfico)
│   │   ├── deployment-canary.yaml      # Versión Canary (20% tráfico)
│   │   └── service-canary.yaml         # Service balanceando entre ambas versiones
│   └── blue-green/                     # Estrategia para Entorno Producción (Rama: main)
│       ├── deployment-blue.yaml        # Despliegue Actual (Producción Activo)
│       ├── deployment-green.yaml       # Despliegue Nuevo (Pruebas antes de Switch)
│       └── service-prod.yaml           # Service con selector dinámico (version: blue | green)
├── jenkins/
│   └── values.yaml                     # Helm values (soporte para Jenkins Multibranch & RBAC)
└── app/                                # Código Fuente de la API REST (Python FastAPI)
    ├── app/
    │   └── main.py                     # API con endpoint /version para validar el switch
    ├── tests/
    │   └── test_main.py                # Pruebas unitarias ejecutadas con Pytest
    ├── Dockerfile                      # Imagen multi-stage optimizada
    └── requirements.txt                # Dependencias de la aplicación
```

## 🚀 Guía Rápida de Configuración y Ejecución

### Paso 1: Iniciar el Clúster Local

Bash

minikube start

### Paso 2. Clonar e Inicializar Ramas en Git

Bash

git remote add origin [URL](Repositorio)
git checkout -b develop
git add .
git commit -m "chore: estructura inicial del proyecto GitFlow"
git push -u origin develop

git checkout -b staging
git push -u origin staging

git checkout -b main
git push -u origin main

### Paso 3. Configurar Infraestructura con Terraform

Bash

cd terraform/
terraform init
terraform apply -auto-approve

## 🧹 Limpieza de Recursos

Para evitar cargos innecesarios en AWS o consumo de memoria local:

Bash

# 1. Destruir infraestructura en AWS
cd terraform/
terraform destroy -auto-approve

# 2. Eliminar Jenkins y la app en Kubernetes
helm uninstall jenkins -n jenkins
kubectl delete namespace dev jenkins

# 3. Detener Minikube
minikube stop