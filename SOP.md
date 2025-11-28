# 📝 SOP – Deployment Documentation for MEAN CRUD Application
## Discover Dollar – DevOps Assignment

This document explains the steps followed to successfully deploy a containerized **MEAN (MongoDB, Express, Angular, Node.js)** CRUD application using Docker, Docker Hub, AWS EC2, Nginx, and GitHub Actions.

---

## 1. Project Overview & Technology Stack

The objective of this assignment was to deploy a robust, multi-tier MEAN application incorporating containerization, cloud deployment, and CI/CD automation. The application is live at: `http://3.83.210.137`.

### Key Technologies

* **Application Stack:** MEAN (MongoDB, Express, Angular, Node.js)
* **Containerization:** Docker & Docker Compose
* **Cloud Provider:** Amazon Web Services (AWS)
* **Compute:** AWS EC2 (`t3.small`)
* **Reverse Proxy:** Nginx
* **CI/CD Automation:** GitHub Actions

---

## 2. Cloud Infrastructure Details

### 2.1 AWS EC2 Instance Details

The application foundation is an AWS EC2 instance configured as follows:

* **Instance Type:** `t3.small`
* **Operating System:** Linux OS
* **Pre-installed Tools:** Docker and Docker Compose installed
* **Elastic IP (EIP):** `3.83.210.137`

### 2.2 Network Security (Security Group Ports)

The AWS Security Group was configured to allow public access to the necessary application ports:

| Port | Protocol | Use | Source |
| :--- | :--- | :--- | :--- |
| **80** | TCP | **Nginx Reverse Proxy** (Public Application Access) | `0.0.0.0/0` |
| **22** | TCP | **SSH** (For management and CI/CD deployment) | `0.0.0.0/0` |
| 4200 | TCP | Frontend (Angular - *Internal container port*) | *Internal/Specific* |
| 3000 | TCP | Backend (Node/Express API - *Internal container port*) | *Internal/Specific* |
| 27017 | TCP | MongoDB (*Internal container port*) | *Internal/Specific* |

---

## 3. Docker Containerization

### 3.1 Frontend Dockerfile (./frontend/Dockerfile)

The Angular application is built and run using a Node base image.

> **Listing 1: Frontend Dockerfile**
>
> ```dockerfile
> FROM node:18-alpine
> 
> WORKDIR /app
> 
> COPY package*.json ./
> RUN npm install
> 
> COPY . .
> 
> EXPOSE 4200
> 
> CMD ["npm", "start", "--", "--host", "0.0.0.0"]
> ```

### 3.2 Backend Dockerfile (./backend/Dockerfile)

The Node.js/Express API container setup.

> **Listing 2: Backend Dockerfile**
>
> ```dockerfile
> FROM node:18-alpine
> 
> WORKDIR /app
> 
> COPY package*.json ./
> RUN npm install
> 
> COPY . .
> 
> EXPOSE 3000
> 
> CMD ["node", "server.js"]
> ```

---

## 4. Service Orchestration & Configuration

### 4.1 Docker Compose Configuration

This file defines the four-service MEAN stack, including container names, image sources from Docker Hub (`yaseen778/dd-*`), and the custom network (`mean-net`).

> **Listing 3: Docker Compose Configuration (docker-compose.yml)**
>
> ```yaml
> version: "3"
> 
> services:
>   mongo:
>     image: mongo:6
>     container_name: mongo
>     restart: always
>     ports:
>       - "27017:27017"
>     volumes:
>       - mongo_data:/data/db
>     networks:
>       - mean-net
> 
>   backend:
>     image: yaseen778/dd-backend:latest
>     container_name: mean-backend
>     restart: always
>     ports:
>       - "3000:3000"
>     depends_on:
>       - mongo
>     networks:
>       - mean-net
> 
>   frontend:
>     image: yaseen778/dd-frontend:latest
>     container_name: mean-frontend
>     restart: always
>     ports:
>       - "4200:4200"
>     depends_on:
>       - backend
>     networks:
>       - mean-net
> 
>   nginx:
>     image: nginx:latest
>     container_name: mean-nginx
>     ports:
>       - "80:80"
>     depends_on:
>       - frontend
>       - backend
>     volumes:
>       - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
>     networks:
>       - mean-net
> 
> networks:
>   mean-net:
> 
> volumes:
>   mongo_data:
> ```

### 4.2 Nginx Reverse Proxy Configuration

Nginx acts as the entry point on port 80, routing API calls to the backend and UI traffic to the frontend.

> **Listing 4: Nginx Configuration (nginx/nginx.conf)**
>
> ```nginx
> events {}
> 
> http {
>     server {
>         listen 80;
> 
>         # Frontend
>         location / {
>             proxy_pass http://frontend:4200;
>             proxy_set_header Host $host;
>         }
> 
>         # Backend API
>         location /api/ {
>             proxy_pass http://backend:3000/;
>             proxy_set_header Host $host;
>         }
>     }
> }
> ```

### 4.3 MongoDB Connection

The backend application connects to the MongoDB service using the Docker Compose service name:

> **Connection String:** `mongodb://mongo:27017/docker_db`

---

## 5. CI/CD with GitHub Actions

### 5.1 Pipeline Strategy

The pipeline automatically **Builds** and **Pushes** the frontend and backend images to Docker Hub, then uses **SSH** to deploy the latest versions to the EC2 instance.

### 5.2 GitHub Actions Workflow

The workflow is defined in `.github/workflows/deploy.yml`.

> **Listing 5: CI/CD Workflow File (deploy.yml)**
>
> ```yaml
> name: Deploy to EC2
> 
> on:
>   push:
>     branches:
>       - main
> 
> jobs:
>   deploy:
>     runs-on: ubuntu-latest
> 
>     steps:
>     - name: Checkout code
>       uses: actions/checkout@v3
> 
>     - name: Setup SSH
>       run: |
>         mkdir -p ~/.ssh
>         echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/id_rsa
>         chmod 600 ~/.ssh/id_rsa
>         ssh-keyscan -H ${{ secrets.EC2_HOST }} >> ~/.ssh/known_hosts
> 
>     - name: Login to Docker Hub
>       run: |
>         echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
> 
>     - name: Build & Push Backend
>       run: |
>         docker build -t ${{ secrets.DOCKER_USERNAME }}/dd-backend ./backend
>         docker push ${{ secrets.DOCKER_USERNAME }}/dd-backend:latest
> 
>     - name: Build & Push Frontend
>       run: |
>         docker build -t ${{ secrets.DOCKER_USERNAME }}/dd-frontend ./frontend
>         docker push ${{ secrets.DOCKER_USERNAME }}/dd-frontend:latest
> 
>     - name: Deploy on EC2
>       run: |
>         ssh -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} "
>           cd ~/discover-dollar-mean-devops-task &&
>           git pull &&
>           docker-compose pull &&
>           docker-compose up -d
>         "
> ```

---

## 6. Testing and Deliverables

The application is deployed and accessible at **http://3.83.210.137**.

---

## 7. Conclusion

This project successfully demonstrates full cloud deployment using Docker, AWS EC2, Nginx, and a robust CI/CD pipeline, fulfilling all requirements for the Discover Dollar – DevOps Assignment.