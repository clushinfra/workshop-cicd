# workshop-cicd
2025.05.31 쿠버네티스 워크샵

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
    

# 2. Jenkins 생성

<aside>
💡

참고
https://kbsyscloud.atlassian.net/wiki/spaces/CLOUD/pages/208994305/HL+DEV-EKS-Jenkins

</aside>

```bash
k create namespace jenkins
```

## 1) sc.yaml apply

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jenkins-sc
provisioner: blk.csi.ncloud.com
parameters:
  type: SSD
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

```bash
k apply -f sc.yaml
```

## 2) pvc.yaml apply

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: jenkins-sc
  resources:
    requests:
      storage: 10Gi
```

```bash
k apply -f pvc.yaml
```

## 3) deploy.yaml apply

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

## 4) svc.yaml apply

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

젠킨스 접속 URL

http://27.96.145.28:30080/

ID : test

PW : test

# 3. Nexus 생성

<aside>
💡

참고

</aside>

```bash
k create namespace nexus
```

## 1) sc.yaml apply

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nexus-sc
provisioner: blk.csi.ncloud.com
parameters:
  type: SSD
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

```bash
k apply -f sc.yaml
```

## 2) pvc.yaml apply

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-pvc
  namespace: nexus
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nexus-sc
  resources:
    requests:
      storage: 10Gi
```

```bash
k apply -f pvc.yaml
```

## 3) deploy.yaml apply

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

## 4) svc.yaml apply

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

Nexus 접속 URL

http://27.96.145.28:30081/

ID : admin

PW : clush1234

# 4. ArgoCD 생성

<aside>
💡

참고
https://potato-yong.tistory.com/137

</aside>

```bash
# argocd namespace 생성
k create namespace argocd

# argocd 배포
k apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
 
```

```bash
# NodePort로 수정
k edit svc argocd-server -n argocd

k patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
# 이렇게 하는게 실수없을듯
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":8080,"nodePort":30082}]}}'

# secret 암호 정보를 평문으로 가져옴
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

```

# 5. Prometheus + Grafana 생성

챗지피티의 추천 방법으로 진행해보았습니다……

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

# 
export POD_NAME=$(kubectl -n monitoring get pod \
  -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=kube-prometheus-stack" \
  -o name)
  
# NodePort로 변경
kubectl patch svc kube-prometheus-stack-grafana -n monitoring   -p '{"spec": {"type": "NodePort"}}' 
```

접속 URL

http://27.96.145.28:31103/

ID : admin

PW : clush1234

**~~질문 >> yaml 파일 서버에 넣어놓고 진행해도 될까요?~~**

# 추가적으로 한 부분

**젠킨스 크레덴셜(nexus) 생성**

![image.png](attachment:981238ae-c800-423e-8778-ddd1b13c6900:image.png)

**Github test 레포생성**

https://github.com/clushinfra/test

cicd부분 챗지피티의 추천방법으로 진행해보았는데

docker 부분(jenkins 파드에서 도커 빌드 안되지않나요?) 에서 막혔습니다..

nks로 생성한 서버에서 진행하면 될까요…?

잘모르겠습니다,,,,,,

# 해야되는 부분

- cicd 시나리오 어떻게 진행해야될까요..
- 모니터링 세션때 어떤거를 체크하고 봐야되는지? 연결할게 따로 있는지..?
- 모니터링(prometheus+grafana) 관련 개념 공부 → 1도 모름.. 세미나때 설명할수있을정도로 공부필요
- 워크샵때 github 레포에 올라간 코드에 삽입되는 공인아이피가 다 다를텐데(nks 서버 공인아이피), 이거 어떻게 진행하면 좋을까요
- argocd는 어떻게 연결해야될지 찾아봐야됨..
- nks 22개 생성(워크샵 전날)
- vpn 발급 .. 안해도될듯? 사내깃랩 안쓰니까
-
