
# Kubernetes 기반 애플리케이션 배포 및 트래픽 라우팅 아키텍처 구축
<div align="center">
  <img width="500" alt="image" src="https://github.com/user-attachments/assets/7d6f8d77-2f4e-4b6d-bc35-58237828c979" />
</div>
Kubernetes 기반 애플리케이션 배포 환경을 직접 구성하며, 트래픽 라우팅 구조 전반을 설계·정리했습니다. 
<br>
특히 Service 유형(ClusterIP / NodePort)을 활용해 Pod 간 통신 및 외부 노출 방식을 비교·구축했고, Ingress 리소스와 NGINX Ingress Controller를 적용해 Hostname 기반 라우팅 구조를 구현했습니다.
<br>

## Kubernetes Service 유형
쿠버네티스에서 Pod는 동적으로 생성·삭제되므로 직접 IP로 접근하기 어렵습니다.
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

### 📝 `yaml` 파일로 리소스 관리하기
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
: 클러스터 외부에서도 접근할 수 있도록 각 노드(Node)의 특정 포트에서 서비스하는 유형입니다.

- yaml 파일 설명
  - 노드포트는 30000 ~ 32767의 범위로 고정
  - 외부에서 접근 시 http://<NodeIP>:<NodePort> 형식으로 접근 가능
```shell
type: NodePort
ports:
  - port: 80
    targetPort: 8900
    nodePort: 30080
```

- 📄 `nginx-nodeport.yaml`
```shell
apiVersion: apps/v1
kind: Deployment
metadata:
  name: giljar-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: giljar
  template:
    metadata:
      labels:
        app: giljar
    spec:
      containers:
      - name: giljar-container
        image: byeonggill/giljar1:1.0  
        ports:
        - containerPort: 8900       
---
apiVersion: v1
kind: Service
metadata:
  name: giljar-service
spec:
  selector:
    app: giljar
  ports:
    - protocol: TCP
      port: 80        
      targetPort: 8900
      nodePort: 30081  
  type: NodePort
```

- 해당 이미지 구동 및 실행 확인

| 사진 | 설명 |
| ---- | ----- |
| <img width="580" alt="image" src="https://github.com/user-attachments/assets/d9008d9c-d5b0-4e73-b1f8-fb86b4929df5" />| yaml 파일 적용 및 확인하기 |
|<img width="420" alt="image" src="https://github.com/user-attachments/assets/bee0dfdf-29a1-4c91-96dc-7aafacefef2e" />| curl로 jar 구동 확인 |

- 웹 접속 확인

| 사진 | 설명 |
| ---- | ----- |
| <img width="600" alt="image" src="https://github.com/user-attachments/assets/6ba68de0-2e20-4d4f-94e4-9ee9807323b8" /> | 사전 작업 : 포트 포워딩 |
| <img width="300" alt="image" src="https://github.com/user-attachments/assets/dab940b2-a18a-452d-a1bb-928b5b8fa2f1" /> | 웹 접속하기 |

---

<br>

## 🌐 Ingress로 Hostname 기반 라우팅
쿠버네티스에서 Ingress는 클러스터 외부에서 내부 서비스로 들어오는 HTTP/HTTPS 요청을 처리하는 규칙들의 집합입니다. 단순히 IP 기반 라우팅을 제공하는 Service와 달리, 도메인 기반 라우팅을 설정할 수 있습니다.
> Ingress를 통해 외부 트래픽을 라우팅할 때, Service는 굳이 외부 IP를 가질 필요가 없습니다.
> 모든 트래픽은 Ingress Controller가 받아서 내부적으로 처리하기 때문입니다.
> 따라서 외부 노출을 최소화하고 클러스터 내부 통신에 최적화된 ClusterIP 타입을 선택했습니다.
> 만약 NodePort를 사용했다면 불필요한 포트가 외부에 노출되어 보안적으로 취약해질 수 있었을 겁니다.

#### ➡️ Kubernetes Ingress Service Deployment Architecture Diagram
<div align="center">
<img width="800" height="900" alt="Image" src="https://github.com/user-attachments/assets/3c267e68-cb23-4acf-b933-b5977f289d7a" />
</div>

> [그림] Kubernetes 요청 처리 흐름
>
> 클라이언트가 http://spring.local로 요청을 보내면 Ingress가 이를 받아 nginx-service로 전달합니다.
> Service는 ClusterIP(80 → 8900)을 통해 Deployment에 연결된 Pod들(spring1, spring2)로 트래픽을 분산합니다.
> 같은 라벨을 가진 Pod들은 모두 containerPort: 8900에서 요청을 수신하며, ReplicaSet에 의해 다중화되어 부하 분산과 장애 복구가 가능합니다.

<br>

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

- 실행 확인

| curl 확인 | health 확인 |
| ---- | ---- | 
| <img width="600" alt="image" src="https://github.com/user-attachments/assets/18df8426-8a3a-446a-8fbc-fb65827f1d60" /> | <img width="300" alt="image" src="https://github.com/user-attachments/assets/954e7446-02cd-4455-9074-15433d1c4a6c" /> |

---


### 📚 학습 성과 및 통찰
**1. 서비스 유형의 차이를 명확히 이해했습니다.**
ClusterIP, NodePort, Ingress를 직접 구성해보면서 단순히 개념으로만 알고 있던 차이가 실제로 어떻게 동작하는지 체감할 수 있었습니다.

**2. 트래픽 라우팅 구조가 눈에 보였습니다.**
Ingress를 적용해 nginx.local, spring.local 같은 호스트 기반 라우팅을 테스트하면서, 단순 포트 기반 접근보다 훨씬 직관적인 구조를 만들 수 있다는 점이 인상 깊었습니다.

**3. YAML 기반 리소스 관리의 필요성을 느꼈습니다.**
단순히 kubectl expose 같은 명령으로 리소스를 만들 때보다, kubectl apply -f . 명령어로 2개의 애플리케이션(총 4개 Pod)을 5초 안에 배포할 수 있는 선언형 인프라 환경을 구축했습니다. yaml 파일로 명시적으로 관리하는 방식이 재현성과 버전 관리 측면에서 훨씬 안정적임을 확인했습니다.
