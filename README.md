# workshop-cicd
2025.05.31 쿠버네티스 워크샵

## 0. 진행 과정
![cicd 아키텍처](./images/cicd-architecture.png)

이번 워크샵에서는 GitHub에 올라온 코드를 기반으로, 

Jenkins가 코드를 가져와 애플리케이션을 빌드하고, 

빌드한 이미지를 Nexus에 저장한 다음, 

ArgoCD가 그 이미지를 감지해서 Kubernetes 클러스터에 자동 배포하는 전체 과정을 실습 해보겠습니다.



## 1. GitOps CI/CD 파이프라인 구축
### 1) 





2025-05-23 정리

현재 생성 서버

ssh [root@27.96.145.28](mailto:root@27.96.145.28)


# 1. NKS 생성

1. 클러스터 22개 생성하기
    - vCPU 8EA, Memory 16GB
    - 100gb
    - 노드 1
2. acg 포트 허용(모든 포트) 1-65535 - 22개
TCP	59.10.110.164/32	1-65535
UDP	59.10.110.164/32	1-65535
ICMP	59.10.110.164/32

3. 각 클러스터 터미널로 접근(워커1로) - ssh 공인아이피
    - 관리자 비밀번호 확인
    서버 이름
    관리자 이름
    ncloud
    비밀번호
4. 루트 계정 허용
    - 루트 비밀번호 설정
    - sshd_config 수정
    - restart
    - 접근 확인
    
    ---
    
    >> 여기까지 전날에 준비
    
    alias k='kubectl --kubeconfig=/root/.kube/kubeconfig.yaml’
    

## 2. Jenkins 생성
**Jenkins**란?


### 0) Namespace 생성
```bash
k create namespace jenkins
```

jenkins manifest 파일을 모을 폴더 생성
```bash
mkdir -p ~/manifest/jenkins
```
```bash
cd ~/manifest/jenkins
```

### 1) Storage Class 생성
**Storage Class란?**

쿠버네티스에서 “어떤 성능과 방식의 저장공간을 자동으로 만들지” 정해주는 설정입니다.

```bash
vi sc.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jenkins-sc                         # PVC에서 사용할 StorageClass 이름
provisioner: blk.csi.ncloud.com            # Naver Cloud용 Block Storage CSI 드라이버
parameters:   
  type: SSD                                # 저장소 타입 : SSD
reclaimPolicy: Retain                      # PVC 삭제 시 볼륨을 남겨둠
volumeBindingMode: WaitForFirstConsumer    # 실제로 Pod가 만들어져야 볼륨도 생성됨
```

```bash
k apply -f sc.yaml
```

### 2) PersistentVolumeClaim 생성
**PersistentVolumeClaim이란?**

애플리케이션이 쿠버네티스에 “이만큼 저장공간이 필요하다”고 요청하는 자원 요청서입니다.

StorageClass를 참고하여 실제 볼륨이 생성됩니다.

```bash
vi pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc               # PVC 이름
  namespace: jenkins              # PVC가 속할 네임스페이스
spec:
  accessModes:
    - ReadWriteOnce               # 하나의 노드에서 읽기/쓰기 가능
  storageClassName: jenkins-sc    # 사용할 StorageClass 이름
  resources:
    requests:
      storage: 10Gi               # 요청할 저장공간 용량(10GiB)
```

```bash
k apply -f pvc.yaml
```

### 3) Deployment 생성
**Deployment**란?

애플리케이션을 몇 개의 Pod로 실행할지, 언제 재시작할지 등을 정의하는 실행 관리 설정입니다.

애플리케이션을 안정적으로 배포하고 운영하기 위한 핵심 구성 요소입니다.

```bash
vi deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: jenkins
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      securityContext:
        fsGroup: 1000
      containers:
      - image: kbsys9505/jenkins:2.492-jdk17
        imagePullPolicy: Always
        name: jenkins
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 50000
          protocol: TCP
        resources:
          limits:
            cpu: 2
            memory: 4Gi
          requests:
            cpu: 1
            memory: 2Gi
        securityContext:
          privileged: true
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
```

```bash
k apply -f deploy.yaml
```

### 4) Service 생성
**Service**란?

쿠버네티스에서 Pod에 안정적으로 접근할 수 있도록 IP와 포트를 제공해주는 네트워크 설정입니다.

외부 또는 클러스터 내부에서 애플리케이션에 접근할 때 사용됩니다.

```bash
vi svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-svc
  namespace: jenkins
  labels:
    app: jenkins
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
    name: jenkins-web
  selector:
    app: jenkins
```

```bash
k apply -f svc.yaml
```

### 5) Jenkins 접속

http://[서버공인IP]:30080/

ID : admin

PW : clush1234

## 3. Nexus 생성
**Nexus**란?


### 0) Namespace 생성
```bash
k create namespace nexus
```

jenkins manifest 파일을 모을 폴더 생성
```bash
mkdir -p ~/manifest/nexus
```
```bash
cd ~/manifest/nexus
```

### 1) Storage Class 생성
**Storage Class란?**

쿠버네티스에서 “어떤 성능과 방식의 저장공간을 자동으로 만들지” 정해주는 설정입니다.

```bash
vi sc.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nexus-sc                           # PVC에서 사용할 StorageClass 이름
provisioner: blk.csi.ncloud.com            # Naver Cloud용 Block Storage CSI 드라이버
parameters:   
  type: SSD                                # 저장소 타입 : SSD
reclaimPolicy: Retain                      # PVC 삭제 시 볼륨을 남겨둠
volumeBindingMode: WaitForFirstConsumer    # 실제로 Pod가 만들어져야 볼륨도 생성됨
```

```bash
k apply -f sc.yaml
```

### 2) PersistentVolumeClaim 생성
**PersistentVolumeClaim이란?**

애플리케이션이 쿠버네티스에 “이만큼 저장공간이 필요하다”고 요청하는 자원 요청서입니다.

StorageClass를 참고하여 실제 볼륨이 생성됩니다.

```bash
vi pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-pvc                 # PVC 이름
  namespace: nexus                # PVC가 속할 네임스페이스
spec:
  accessModes:
    - ReadWriteOnce               # 하나의 노드에서 읽기/쓰기 가능
  storageClassName: nexus-sc      # 사용할 StorageClass 이름
  resources:
    requests:
      storage: 10Gi               # 요청할 저장공간 용량(10GiB)
```

```bash
k apply -f pvc.yaml
```

### 3) Deployment 생성
**Deployment**란?

애플리케이션을 몇 개의 Pod로 실행할지, 언제 재시작할지 등을 정의하는 실행 관리 설정입니다.

애플리케이션을 안정적으로 배포하고 운영하기 위한 핵심 구성 요소입니다.

```bash
vi deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nexus
  name: nexus
  namespace: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
    spec:
      securityContext:
        fsGroup: 1000
      containers:
      - image: sonatype/nexus3:3.52.0
        imagePullPolicy: Always
        name: nexus
        ports:
        - containerPort: 8081
          protocol: TCP
          name: nexus-web
        - containerPort: 5000
          protocol: TCP
          name: nexus-docker
        resources:
          limits:
            cpu: 2
            memory: 4Gi
          requests:
            cpu: 1
            memory: 2Gi
        volumeMounts:
        - name: nexus-data
          mountPath: /nexus-data
      volumes:
      - name: nexus-data
        persistentVolumeClaim:
          claimName: nexus-pvc
```

```bash
k apply -f deploy.yaml
```

### 4) Service 생성
**Service**란?

쿠버네티스에서 Pod에 안정적으로 접근할 수 있도록 IP와 포트를 제공해주는 네트워크 설정입니다.

외부 또는 클러스터 내부에서 애플리케이션에 접근할 때 사용됩니다.

```bash
vi svc.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nexus-svc
  namespace: nexus
  labels:
    app: nexus
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8081
    nodePort: 30081
    name: nexus-web
  - port: 5000
    targetPort: 5000
    nodePort: 30500
    name: nexus-docker
  selector:
    app: nexus
```

```bash
k apply -f svc.yaml
```

### 5) Nexus 접속

http://[서버공인IP]:30081/

**초기 비밀번호 조회**

Pod 내부의 /nexus-data/admin.password에 위치

- Pod 조회
```bash
k get pod -n nexus
```
- Pod 접속
```bash
k exec -it <pod명> -n nexus -- /bin/bash 
```
- 폴더 이동
```bash
cd /nexus-data
```
- 파일 조회
```bash
ls -al
```
- 비밀번호 조회
```bash
cat admin.password
```
ID : admin
PW : 초기 비밀번호

# 4. ArgoCD 생성
----------
### 0) Namespace 생성
```bash
k create namespace argocd
```

### 1) ArgoCD 배포
```bash
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2) NodePort로 수정
```bash
k patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":8080,"nodePort":30082}]}}'
```

### 3) ArgoCD 접속
[서버공인IP]:30082

ID : admin

**초기 비밀번호 조회**
```bash
k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```


# 5. Prometheus + Grafana 생성
```bash
# helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 버전 확인
helm version

# repo 등록
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# repo 업데이트
helm repo update

# namespace 생성
k create namespace monitoring

# 다운로드
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack   -n monitoring   --set prometheus.prometheusSpec.maximumStartupDurationSeconds=300

# 비밀번호 조회
k --namespace monitoring get secrets kube-prometheus-stack-grafana   -o jsonpath="{.data.admin-password}" | base64 -d && echo

# Grafana Pod의 이름을 자동으로 찾아서 POD_NAME이라는 변수에 저장하는 작업
export POD_NAME=$(kubectl -n monitoring get pod \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus-stack" \
  -o name)
  
# NodePort로 변경
kubectl patch svc kube-prometheus-stack-grafana -n monitoring \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 3000, "nodePort": 30083}]}}' 
```

접속 URL

<서버공인IP>:31103/

ID : admin

PW : 초기 비밀번호

# 추가적으로 한 부분

**젠킨스 크레덴셜(nexus) 생성**
