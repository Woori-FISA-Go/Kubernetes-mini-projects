
# Kubernetes 미니 프로젝트
<div align="center">
  <img width="500" alt="image" src="https://github.com/user-attachments/assets/7d6f8d77-2f4e-4b6d-bc35-58237828c979" />
</div>
쿠버네티스를 공부하면서 직접 실습해본 내용들을 정리했습니다.
특히 Service 유형 (ClusterIP / NodePort) 과 Ingress를 활용한 외부 접근 방식을 다뤘습니다.
<br>

## Kubernetes Service 유형
쿠버네티스에서 Pod은 동적으로 생성·삭제되므로 직접 IP로 접근하기 어렵습니다.
이 문제를 해결하기 위해 Service를 사용하며, 유형별 특징은 아래와 같습니다.
<br>

## 📢 **쿠버네티스 서비스 유형별 설명**

| 서비스 유형 | 간단한 설명 | 사용 사례 |
| :--- | :--- | :--- |
| **ClusterIP** | 클러스터 내부에서만 접근 가능한 IP를 할당합니다. | 내부 서비스 간 통신 (기본 옵션) |
| **NodePort** | 각 노드의 특정 포트를 열어 외부에서 서비스에 접근할 수 있도록 합니다. | 테스트, 간단한 외부 접근 |
| **LoadBalancer** | 클라우드 플랫폼의 로드 밸런서를 사용하여 외부 트래픽을 서비스로 분산합니다. | 클라우드 환경에서 외부 서비스 노출 |
| **ExternalName** | 외부 서비스를 DNS CNAME 레코드로 매핑하여 클러스터 내부에서 별칭으로 사용할 수 있게 합니다. | 외부 서비스에 내부에서 접근 |

<br>

### 📝 `yaml` 파일로 명시적으로 리소스 관리하기
- 보통 `kubectl expose` 명령어를 쓰면 간단하게 Service를 만들 수 있습니다.
  - 하지만 이렇게 하면 kubectl이 자동으로 Service 리소스를 생성합니다.
- 반면에 `yaml` 파일(nginx-clusterip.yaml)을 작성해서 `kubectl apply -f`로 적용하는 방법은,<br>
  → 명시적으로 **Service 리소스를 정의하고, 원하는 설정을 버전 관리(Git 등)와 함께 기록**할 수 있다는 장점이 있습니다.

> 이에 따라, 💫**운영 및 재현 가능한 인프라 관리 방식을 익히기 위해**💫 yaml 파일 기반의 리소스 관리 방법을 선택했습니다.
<br>

---

## 🔵 ClusterIP 구성하기
: 클러스터 내부에서만 접근 가능한 내부 IP 제공합니다. (사용 예시: 내부 DB, 내부 API 서버)

- 📄 `nginx-clusterip.yaml`
```shell
apiVersion: v1
kind: Service
metadata:
  name: my-clusterip-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

---

## 🟣 NodePort 구성하기
: 클러스터 외부 접근을 가능하게 합니다. (사용 예시: 로드밸런서 없이 테스트 환경에서 외부 접근)

- 📄 `nginx-clusterip.yaml`
```shell
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP  
      port: 80          # 서비스가 내부 Pod와 소통하는 포트
      targetPort: 8080  # Pod 내부에서 컨테이너가 열어둔 포트
      nodePort: 30007   # 외부 접근 포트 (30000~32767 범위)
  type: NodePort
```

---

## 🌐 Ingress로 Hostname 기반 라우팅
쿠버네티스에서 Ingress는 클러스터 외부에서 내부 서비스로 들어오는 HTTP/HTTPS 요청을 처리하는 규칙들의 집합입니다.
단순히 IP 기반 라우팅을 제공하는 Service와 달리, 도메인 기반 라우팅을 설정할 수 있습니다.

- 🔑 Ingress Controller
  - Ingress 리소스가 실제로 동작하려면 Ingress Controller가 필요합니다.
  - Ingress Controller는 정의된 **Ingress 규칙을 읽고, 실제로 트래픽을 라우팅하는 리버스 프록시(예: NGINX)를 실행**합니다.

### 📝 기능별 yaml 파일 구현

<details>
<summary> <h4>🟠 Ingress 구성 - nginx-ingress.yaml</h4> </summary>

```shell
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default  
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service # nginx.local로 들어온 요청은 nginx-service의 port 80으로 포워딩
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spring-ingress
  namespace: default  
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: spring.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spring-service # spring.local로 들어온 요청은 spring-service의 port 88로 포워딩
            port:
              number: 88
```
</details>

<details>
<summary> <h4>🟡 Service 구성 - service.yaml</h4> </summary>

- 📄 `service.yaml`
```shell
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: spring-service
spec:
  selector:
    app: spring
  ports:
    - port: 88
      targetPort: 8900
  type: ClusterIP
```
</details>

<details>
<summary> <h4>🟢 Deployment 구성 - deployment.yaml </h4> </summary>
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring
  template:
    metadata:
      labels:
        app: spring
    spec:
      containers:
      - name: spring
        image: byeonggill/giljar1:1.0   
        ports:
        - containerPort: 8900
```
</details>

### ⚙️ 동작 설명
- nginx
  - containerPort: 80에서 HTTP 요청을 수신합니다.
  - registry.k8s.io/nginx:latest 이미지를 사용합니다.
  - Pod 2개로 실행되어 부하 분산이 가능합니다.

- spring
  - containerPort: 8900에서 요청을 수신합니다.
  - byeonggill/giljar1:1.0 이미지를 사용합니다.
  - Pod 2개로 실행되어 고가용성이 보장됩니다.

#### ▶️ Ingress 실행 방법

- Ingress 활성화 및 yaml 실행
```shell
minikube addons enable ingress
kubectl apply -f nginx-ingressdepsvc.yaml 
kubectl apply -f nginx-ingress.yaml 
```

| 사진 | 설명 |
| ---- | ---- |
| <img width="500" alt="image" src="https://github.com/user-attachments/assets/bf221b49-d370-4926-b857-aa54336c5e94" />| service, deployment, pod 확인 |
| <img width="380"  alt="image" src="https://github.com/user-attachments/assets/acd48438-5eae-435e-9a0f-445f90311bfb" />| /etc/hosts 파일에 DNS 등록 |
| <img width="500" alt="image" src="https://github.com/user-attachments/assets/d0ec434f-156a-429a-8c17-b6466bc017f2" />| curl 명령어로 실행 확인 |
