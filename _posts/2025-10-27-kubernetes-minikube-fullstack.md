---
title: "Building a Full-Stack App on a Single-Node Kubernetes Cluster"
date: 2025-10-27
layout: post
permalink: /kubernetes-minikube-fullstack/
categories: [DevOps, Kubernetes, Full-Stack]
tags: [Kubernetes, Minikube, Docker, Node.js, React, MongoDB, Cloud-Native, CI/CD]
description: "Learn how to deploy a full-stack (React + Node.js + MongoDB) app on a single-node Kubernetes cluster using Minikube — from containerization to deployment, monitoring, and best practices."
image: /assets/images/kubernetes-minikube-cover.png
author: "Md. Shihabuddin Sadi"
---
 
*Software Engineer · DevOps & Cloud Native Engineer · Aspiring Platform Engineer*  
*October 26, 2025*  

<br>

### **Summary**

This project showcases how I built a complete **full-stack application** (React frontend, Node.js backend, MongoDB database) inside a **single-node Kubernetes cluster** using **Minikube**.  
It demonstrates real-world DevOps practices such as **Deployments, Services, Ingress, ConfigMaps, Secrets**, and **Persistent Volumes**, all tied together with a custom automation script that simplifies cluster management.  
The goal was to **replicate a production-like Kubernetes setup locally**, helping me deepen my understanding of **cloud-native application deployment and orchestration**.  
<br>
🔗 **GitHub Repository:** [sadishihab/kubernetes-minikube-fullstack](https://github.com/sadishihab/Single-Node-Kubernetes-Cluster)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. Project Overview](#1-project-overview)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. Setup Instructions](#3-setup-instructions)
- [4. Automation Script Instructions](#4-automation-script-instructions)
- [5. Common Errors & Fixes](#5-common-errors--fixes)
- [6. Learning Outcomes](#6-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`Kubernetes` · `Minikube` · `Docker` · `Node.js` · `React` · `MongoDB` · `NGINX Ingress` · `ConfigMaps` · `Secrets` · `Persistent Volume` · `Shell scripting`  
<br>

### **1. Project Overview**

The project demonstrates a **three-tier architecture** deployed in Minikube using the Docker driver:

- **Frontend** – React-based UI  
- **Backend** – Node.js API service  
- **Database** – MongoDB with persistent storage  
- **Ingress** – For routing and browser access  
- **PVC (Persistent Volume Claim)** – To ensure MongoDB data survives restarts  
- **ConfigMaps & Secrets** – For configuration and environment management  
<br>

#### **Architecture Diagram**
```pgsql
               ┌────────────────────────────┐
               │      Browser / User        │
               └──────────────┬─────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │     Ingress      │
                    │ (NGINX / Rules)  │
                    └───────┬──────────┘
                            │
            ┌───────────────┴────────────────┐
            │                                │
            ▼                                ▼
 ┌───────────────────┐             ┌───────────────────┐
 │   Frontend Pod    │             │   Backend Pod     │
 │ (React / Node.js) │◄──────────►│ (Node.js / API)   │
 │   ClusterIP SVC   │             │   ClusterIP SVC   │
 └────────┬──────────┘             └────────┬──────────┘
          │                                 │
          │                                 ▼
          │                     ┌───────────────────┐
          │                     │   MongoDB Pod     │
          │                     │ (Database Layer)  │
          │                     │ Persistent Volume │
          │                     └────────┬──────────┘
          │                              │
          │                     ┌───────────────────┐
          │                     │ PVC + PV (Storage)│
          │                     └───────────────────┘
          │
   ┌──────────────────────────┐
   │ ConfigMaps & Secrets     │
   │ (Env vars, credentials)  │
   └──────────────────────────┘

```
<br>

### **2. Folder Structure**  
```bash
├── frontend/               # Frontend (React)
├── backend/                # Backend (Node.js)
├── k8s/                    # Kubernetes manifests
│   ├── deployments        # Deployment YAMLs
│   ├── services           # Service YAMLs
│   ├── ingress.yaml        # Ingress config
│   ├── pvc.yaml            # MongoDB PVC
│   └── configmaps-secrets  # ConfigMaps & Secrets
└── README.md

```
<br>

### **3. Setup Instructions**  
**Start Minikube**  
```bash
minikube start --driver=docker
eval $(minikube docker-env)

```
<br>

**Apply Configurations**  
```bash
kubectl apply -f k8s/app-configmap.yaml
kubectl apply -f k8s/app-secret.yaml
kubectl apply -f k8s/mongo-pv.yaml
kubectl apply -f k8s/mongo-pvc.yaml
kubectl apply -f k8s/mongo.yaml
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml

```
<br>

**Verify**
```bash
kubectl get pods
kubectl get svc
kubectl get ingress

```
<br>

**Access the App**  
```bash
minikube tunnel

```
Then open the browser and visit the **Ingress host URL.**  
<br>
### **4. Automation Script Instructions**  
To simplify cluster operations, I created a script that can:

- Stop all deployments (scale to 0)
- Restart deployments from saved replicas
- Backup all manifests and services
- Store from backups
- Operate by namespace
<br>

**Examples:**  
```bash
./k8s-manage.sh --stop
./k8s-manage.sh --start
./k8s-manage.sh --save
./k8s-manage.sh --restore

```
<br>

### **5. Common Errors & Fixes**  

**❌ Image not updating after rebuild**


If you rebuild Docker images but pods still use the old one:
```bash
eval $(minikube docker-env)
kubectl rollout restart deployment <deployment-name>

```
**❌ Ingress not working**

Ensure Ingress addon is enabled:
```bash
minikube addons enable ingress

```
**❌ MongoDB Pod in CrashLoopBackOff**

Check logs:
```bash
kubectl logs <pod-name>

```
Possible causes:

- PVC path not writable → adjust permissions
- Missing environment variables → verify ConfigMap and Secret

**❌ Port not accessible**

Use:
```bash
minikube tunnel
kubectl get ingress

```
Then open the listed **ADDRESS** in your browser.
<br>

### **6. Learning Outcomes**
- Hands-on experience with single-node Kubernetes
- Understanding multi-service orchestration
- Learned PVC, ConfigMap, Secret, and Ingress usage
- Practiced pod scaling, rolling updates, and backup/restore workflows
- Deepened understanding of DevOps & Cloud Native principles
<br>

### **References**
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Minikube Official Docs](https://minikube.sigs.k8s.io/docs/)
- [Docker Official Docs](https://docs.docker.com/)
<br>

### **License**
This project is open-source and available under the MIT License
