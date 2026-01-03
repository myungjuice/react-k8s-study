# 🚀 React App CI/CD Pipeline on AWS with k3s

본 프로젝트는 **React 애플리케이션을 AWS EC2 인스턴스에 구축된 k3s (Lightweight Kubernetes) 클러스터에 배포**하고,  
**GitHub Actions를 통해 자동화된 CI/CD 파이프라인을 구성**하는 실습 목적의 프로젝트입니다.

---

## 🛠 Tech Stack

### Frontend

- React (Vite)

### Containerization

- Docker (Multi-stage Build)

### Orchestration

- k3s (Lightweight Kubernetes)

### Cloud Infrastructure

- AWS EC2 (Ubuntu 24.04 LTS)

### CI/CD

- GitHub Actions

### Container Registry

- Docker Hub

---

## 🧐 Why k3s? (vs Kubernetes)

| 구분   | Kubernetes (k8s)                            | k3s                             |
| ------ | ------------------------------------------- | ------------------------------- |
| 목적   | 대규모 엔터프라이즈, 클라우드 네이티브 환경 | 엣지 컴퓨팅, IoT, CI, 개발/학습 |
| 리소스 | 메모리/CPU 요구량 높음                      | 초경량, 단일 바이너리           |
| 구성   | 설치 및 운영 복잡                           | One-line 설치                   |

### 💡 k3s 선택 이유

AWS `t3.medium` 같은 소규모 인스턴스 환경에서  
표준 Kubernetes는 리소스 오버헤드가 큽니다.

k3s는 **CNCF 인증 Kubernetes 배포판**으로,

- 가볍지만 핵심 기능을 모두 제공
- 비용 효율적인 학습 및 소규모 배포 환경에 적합

---

## 📅 Implementation Steps

### Step 1. Dockerization

React 애플리케이션을 어디서든 실행 가능하도록 컨테이너화했습니다.

- Multi-stage Build 적용
- 빌드(Node) / 실행(Nginx) 단계 분리
- 이미지 용량 최소화

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
# ... build steps

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

### Step 2. AWS EC2 & k3s 설치

- **Instance**: AWS EC2 `t3.medium`
- **OS**: Ubuntu 24.04 LTS
- **Security Group**
  - SSH: 22
  - HTTP: 80
  - HTTPS: 443

```bash
# k3s 설치
curl -sfL https://get.k3s.io | sh -
```

---

### Step 3. Kubernetes Manifests

#### Deployment

- Replica 수 정의
- Docker Hub 이미지 기반 배포

#### Service

- 외부 접근을 위해 `LoadBalancer` 타입 사용
- 80번 포트 노출

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-k8s-app-service
spec:
  selector:
    app: my-k8s-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

---

### Step 4. CI/CD Pipeline (GitHub Actions)

`main` 브랜치에 push 시 자동 실행

1. Checkout
2. Docker Image Build
3. Docker Hub Push
4. EC2 SSH 접속
5. `kubectl apply`
6. `rollout restart`

---

## ⚡ Troubleshooting: Port 80 Conflict (Traefik)

### 🔴 문제 상황

- NodePort에서는 정상 접속
- LoadBalancer(80) 변경 후 **404 Page Not Found 발생**

### 🔍 원인 분석

- k3s 기본 구성:
  - **Traefik Ingress Controller**가 기본 설치됨
  - 80 / 443 포트 선점
- Service LoadBalancer가 80 포트 바인딩 실패
- Traefik이 Ingress 규칙 없이 요청을 받아 404 반환

### ✅ 해결 방법

본 프로젝트는 **단일 앱 배포 구조**이므로 Ingress 불필요 → Traefik 비활성화

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

➡ 이후 Service가 정상적으로 80 포트 바인딩 성공
