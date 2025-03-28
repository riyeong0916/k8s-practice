# Kubernetes NodePort & LoadBalancer

### 📌 목적 
- Kubernetes에서 **NodePort와 LoadBalancer 서비스 타입**을 활용하여 **Spring Boot 애플리케이션을 외부에서 접속**
- 로컬 쿠버네티스 환경인 Minikube 기반 실습

<br>

### 📌 관련 개념 정리
#### ✅ Kubernetes Service
- Kubernetes에서 Pod들을 하나의 논리적 단위로 묶어 네트워크 서비스를 제공하는 리소스
- 클러스터 내부 또는 외부에서 **Pod에 접근할 수 있도록 하는 라우팅** 담당

#### ✅ NodePort
- 클러스터의 노드 IP + NodePort를 통해 외부에서 서비스에 접근
- 기본 포트 범위: 30000 ~ 32767
- 로컬 개발 환경이나 테스트에서 주로 사용

#### ✅ LoadBalancer
- 클라우드 환경에서 주로 사용되는 서비스 타입으로, 외부 로드밸런서를 자동으로 생성하여 외부 접근 가능
- Minikube에서는 로컬 환경이므로 `minikube tunnel` 명령어를 통해 LoadBalancer처럼 동작


#### 🖥️ NodePort vs LoadBalancer 비교
| 항목                 | NodePort                                                | LoadBalancer                                                   |
|----------------------|----------------------------------------------------------|----------------------------------------------------------------|
| **외부 접근 방식**     | `NodeIP:NodePort`로 접근                                  | 클라우드 로드밸런서 또는 `minikube tunnel`을 통해 `EXTERNAL-IP`로 접근 |
| **기본 포트 범위**     | 30000 ~ 32767                                           | 사용자가 지정한 port (예: 80, 85 등)                          |
| **EXTERNAL-IP**      | 없음 (`<none>`)                                          | 클라우드 환경에서는 Public IP / Minikube에서는 tunnel 필요     |
| **클러스터 외부 노출** | 가능                                                   | 가능                                                          |
| **주로 사용되는 환경** | 테스트, 로컬 개발 환경                                     | 클라우드 환경, 운영 환경                                      |

<br>

### 📌 실습 
#### 🛠️ 실습 환경
- OS: Ubuntu 24.04.2 LTS
- Minikube v1.35.0
- Docker 26.1.3
- Spring Boot JAR (포트: 85)

#### 1️⃣ JAR 파일 빌드 


#### 2️⃣ Docker 이미지 생성  
1. Dockerfile 생성 
```
FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY step07_cicd-0.0.1-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

2. Docker 이미지 빌드를 위한 파일 구조
```
00.test/
├── Dockerfile                    # Docker 빌드 정의 파일
└── step07_cicd-0.0.1-SNAPSHOT.jar  # Spring Boot 실행용 JAR 파일
```

3. Docker 이미지 Build
```
docker build -t springboot-app .
```

4. Docker 이미지 Push
```
docker login

docker tag springboot-app:latest <DockerHubID>/springboot-app:latest
docker push <DockerHubID>/springboot-app:latest
```

   
#### 3️⃣ NodePort 구동을 위한 yaml 파일 (`springboot-nodeport.yaml`)
- `springboot-nodeport.yaml` : NodePort 방식으로 외부에 노출하기 위한 설정 파일
- `Deployment` : Spring Boot 앱을 담은 파드를 2개 실행합니다 (replicas: 2)
- `Service (NodePort)` : 클러스터 외부에서 `minikube IP:30085`로 접속가능하도록 설정 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-nodeport-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springboot-node
  template:
    metadata:
      labels:
        app: springboot-node
    spec:
      containers:
      - name: springboot
        image: kimleeyoung/springboot-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 85

---
apiVersion: v1
kind: Service
metadata:
  name: springboot-nodeport-service
spec:
  type: NodePort
  selector:
    app: springboot-node
  ports:
    - protocol: TCP
      port: 85
      targetPort: 85
      nodePort: 30085
```

- 적용
```
kubectl apply -f springboot-nodeport.yaml
```


#### 4️⃣  LoadBalancer 구동을 위한 yaml 파일 (`springboot-loadbalancer.yaml`)
- `springboot-loadbalancer.yaml` : LoadBalancer 서비스 타입으로 외부에 노출하기 위한 설정 파일
- `Deployment` : Spring Boot 앱을 담은 파드를 2개 실행합니다 (replicas: 2)
- `Service (LoadBalancer)` : 클라우드 환경 또는 `Minikube + minikube tunnel`에서 외부 IP로 접근 가능하게 설정
```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-lb-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: springboot-lb
  template:
    metadata:
      labels:
        app: springboot-lb
    spec:
      containers:
      - name: springboot
        image: kimleeyoung/springboot-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 85

---
apiVersion: v1
kind: Service
metadata:
  name: springboot-lb-service
spec:
  type: LoadBalancer
  selector:
    app: springboot-lb
  ports:
    - protocol: TCP
      port: 85
      targetPort: 85
```

- 적용
```
kubectl apply -f springboot-loadbalancer.yaml

```

- `minikube tunnel` 🌟
  - 로컬 Minikube 환경에서 LoadBalancer 타입의 서비스를 외부에서 접근할 수 있도록 가짜(가상) 로드밸런서를 만들어주는 명령어
  - `minikube tunnel`를 통해 Minikube에서 LoadBalancer 서비스 외부 노출 가능하게 함
```
 minikube tunnel
```

<br>

### 📌 실행 및 접속 확인
![image](https://github.com/user-attachments/assets/7ab5dbdb-847b-4d51-bc4d-6fe2d91863d5)
#### 1️⃣ Nodeport Service 확인 
- `springboot-nodeport-service` : NodePort 방식으로 외부 노출한 서비스
  - 외부 포트 `30085` / 내부 포트 `85`


#### 2️⃣ LoadBalancer Service 확인 
- `springboot-lb-service` : LoadBalancer 방식으로 노출한 서비스
  - `EXTERNAL-IP`: 10.98.99.124
  - 외부 포트 `85` / 클러스터 내부 포트 `31877`
- 



