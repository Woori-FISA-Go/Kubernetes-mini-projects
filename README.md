
# Kubernetes 미니 프로젝트
<div align="center">
  <img width="500" alt="image" src="https://github.com/user-attachments/assets/7d6f8d77-2f4e-4b6d-bc35-58237828c979" />
</div>
쿠버네티스를 공부하면서 직접 실습해본 내용들을 정리했습니다.
특히 Service 유형 (ClusterIP / NodePort) 과 Ingress를 활용한 외부 접근 방식을 다뤘습니다.

## Kubernetes Service 유형
쿠버네티스에서 Pod은 동적으로 생성·삭제되므로 직접 IP로 접근하기 어렵습니다.
이 문제를 해결하기 위해 Service를 사용하며, 유형별 특징은 아래와 같습니다.

## 📢 **쿠버네티스 서비스 유형별 설명**

| 서비스 유형 | 간단한 설명 | 사용 사례 |
| :--- | :--- | :--- |
| **ClusterIP** | 클러스터 내부에서만 접근 가능한 IP를 할당합니다. | 내부 서비스 간 통신 (기본 옵션) |
| **NodePort** | 각 노드의 특정 포트를 열어 외부에서 서비스에 접근할 수 있도록 합니다. | 테스트, 간단한 외부 접근 |
| **LoadBalancer** | 클라우드 플랫폼의 로드 밸런서를 사용하여 외부 트래픽을 서비스로 분산합니다. | 클라우드 환경에서 외부 서비스 노출 |
| **ExternalName** | 외부 서비스를 DNS CNAME 레코드로 매핑하여 클러스터 내부에서 별칭으로 사용할 수 있게 합니다. | 외부 서비스에 내부에서 접근 |

<br>

### 🔵 ClusterIP 구성하기
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


### 🟣 NodePort 구성하기
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

### 🌐 Ingress로 Hostname 기반 라우팅
