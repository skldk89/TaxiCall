```
curl -o kafka2.5.tgz -l http://mirror.navercorp.com/apache/kafka/2.5.0/kafka_2.13-2.5.0.tgz
tar -xvf kafka2.5.tgz
zookeeper 실행
cd  kafka_2.13-2.5.0/bin
./zookeeper-server-start.sh ../config/zookeeper.properties 
kafka 실행
cd  kafka_2.13-2.5.0/bin
./kafka-server-start.sh ../config/server.properties





./kafka-topics.sh --zookeeper localhost:2181 --topic TaxiCall --create --partitions 1 --replication-factor 1

./kafka-topics.sh --zookeeper localhost:2181 --list

./kafka-console-consumer.sh --bootstrap-server http://localhost:9092 --topic RestaurantReservaiton --from-beginning





[1]AWS 설정
1. aws 설정
aws configure
 -액세스키 입력 : AWS콘솔에 발급받은 액세스키 입력
 -시크릿 액세스키 입력 : AWS콘솔에 발급받은 비밀 액세스키 입력
 -리전입력 : ap-northeast-2  
 -아웃풋 포맷 : json

2. Amazon EKS 생성
eksctl create cluster --name teama-cluster --version 1.15 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
eksctl create cluster --name admin03-sk-cluster --version 1.15 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3

3. AWS 콘솔에서 ECR 만들기  
https://A0E7900A0844E750540BC49E54CDCA81.sk1.ap-northeast-2.eks.amazonaws.com

4. AWS 컨테이너 레지스트리 로그인
aws ecr get-login-password --region (Region-Code) | docker login --username AWS --password-stdin (Account-Id).dkr.ecr.(Region-Code).amazonaws.com
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 271153858532.dkr.ecr.ap-northeast-2.amazonaws.com
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 271153858532.dkr.ecr.ap-northeast-2.amazonaws.com

오류(unknown flag: --password-stdin) 발생 시,
docker login --username AWS -p $(aws ecr get-login-password --region (Region-Code)) (Account-Id).dkr.ecr.(Region-Code).amazonaws.com/
(Account-Id)  AKIAT6IQHQ7SE745V4MN
docker login --username AWS -p $(aws ecr get-login-password --region ap-northeast-2) admin03.dkr.ecr.ap-northeast-2.amazonaws.com/
docker login --username AWS -p $(aws ecr get-login-password --region ap-northeast-1) 907740458419.dkr.ecr.ap-northeast-1.amazonaws.com/
docker login --username AWS -p $(aws ecr get-login-password --region ap-northeast-2) 271153858532.dkr.ecr.ap-northeast-2.amazonaws.com

5. AWS 클러스터 토큰 가져오기 (뭐 할때 쓰지??)
aws eks --region (Region-Code) update-kubeconfig --name (Cluster-Name)

aws eks --region (Region-Code) update-kubeconfig --name (Cluster-Name)

-------------------------------------------------------------------------------------------------------------------
[2]Kafka설치
(헬름이 설치가 안되어 있거나, 버전이 2.xx 일때 카프카 설치 방법)
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
kubectl --namespace kube-system create sa tiller      # helm 의 설치관리자를 위한 시스템 사용자 생성
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller

kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm repo update
helm install --name my-kafka --namespace kafka incubator/kafka  

kubectl get po -n kafka 확인                
(helm 으로 설치를 하게 되면 클러스터에 주키퍼가 3개 카프카가 3개가 올라가게 됩니다. 
약간의 시간이 흐른 후에 kubectl get po -n kafka 명령으로 6개의 pods 의 Status 가 Running 상태를 확인 합니다.)


카프카컨슈머	
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic local --from-beginning
카프카 토픽 확인	
kubectl -n kafka exec my-kafka-0 -- /usr/bin/kafka-topics --zookeeper my-kafka-zookeeper:2181 --list


---------------------------------------------------------------------------------------------------------------------------
[3] istio + DestinationRule 적용한 서킷브레이커 구현
1. Circuit Breaking  [Circuit breaker using istio]
(Istio가 활성화된 네임스페이스 생성) 
kubectl create namespace istio-cb-ns
kubectl get ns    [네임스페이스 생성 확인]

2. Istio 설치
[lab에서 하는 경우 home/project 경로에서 설치함 / 로컬PC는 root 경로에 설치]
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.4.5 sh -
cd istio-1.4.5   [설치한 경로로 이동]
export PATH=$PWD/bin:$PATH
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
kubectl apply -f install/kubernetes/istio-demo.yaml
(설치확인)
kubectl get pod -n istio-system
kubectl get svc -n istio-system

3. Sidecar Actvate
kubectl label namespace istio-cb-ns istio-injection=enabled  [네임스페이스에 Label로 우리가 할것]
kubectl describe ns istio-cb-ns 

4. nano설치 [vi 대신 nano를 사용할 경우  nano 설치]
apt-get update
apt-get install nano

5. CI/CD를 통한 소스배포/도커라이징/도커푸쉬/컨테이너라이징
(1)SA생성
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
EOF

(2)eks-admin-cluster-role-binding.yaml 파일 생성하여 롤바인딩
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EOF

(3) 만들어진 eks-admin SA 의 토큰 가져오기
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

결과:eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJla3MtYWRtaW4tdG9rZW4tdGs4c2YiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZWtzLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOTkyNjNmOTctZGY4Yy00N2U0LWJkY2EtMTk3ZjRkNjBiZjc3Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmVrcy1hZG1pbiJ9.nRn-gRQ1cF2XftjDtpo2HYmUtkl9yPsn7o3uosDzN9DHdWCw9pRomoSwcvfj6TaH9jrtSK023Gmg7PWT3wASrBJ91A5DUZ6Ex51Ue1CcWbhO5PKOieODZws5dBi3k3OIZZpflP33egSO7sJyDrgCFSjiAchkQCk9p0mEco0wIQgLeyyANWDn5Q5y0r9IaJmes1zjc4d6IcbmUHyq-4dH_2Z1N15Ppcg6hapcR4g1vaMmrYiGTF4uo8H2HE2OYde5KeI3raVDgVPd5szX9ynLRgTaJiyJl_lyfSnx3NuNYD_qCBagpHCJLlG3A22LO2zyirVFNerYV085H_601wzUyg

(4)AWS codebuild 생성
- 환경변수설정입력
AWS_ACCOUNT_ID : AWS ECR의 앞 숫자…
KUBE_URL : EKS 클러스터 : API서버 엔트포인트
KUBE_TOKEN : 토큰 입력

- 인라인 정책 추가
- JSON 탭에 아래 내용추가
{
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:GetAuthorizationToken",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "*",
      "Effect": "Allow"
 }

- Github에 buildspec.yml 파일 생성

(6)배포 후 생성된 Deploy/Replica/Service/pod를 확인)
kubectl get all -n istio-cb-ns
kubectl logs -f  pod명 -n istio-cb-ns  [log확인 : 해당 namesape안에  우리가 만든 서비스와 istio-proxy 2개가 생성된거 2개를 볼 수 있음 ]
kubectl logs -f  pod명 --container order -n istio-cb-ns   [우리가 만든 Container를 지정하여 보기]

6. Cloud로 올라간 서비스를 httpie를 통해 확인 : Cloud에 httpie설치
(1) 생성된 node에 httpie Pod 생성하기 
cat <<EOF | kubectl apply -f -
apiVersion: "v1"
kind: "Pod"
metadata: 
  name: httpie
  namespace: istio-cb-ns
  labels: 
    name: httpie
spec: 
  containers: 
    - name: httpie
      image: clue/httpie
      command:
        - sleep
        - "360000"
EOF

(2)MSA 서비스 확인
생성된 pod 안으로 들어가기  (httpie pod로 들어간거임 : 여기에서 테스트 하기 위해...)
root@labs-629359896:/home/project/cna-gateway# kubectl exec -it pod/httpie -n istio-cb-ns -c httpie -- /bin/bash

(kafka 이벤트 수신 설정하기)
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic TaxiCall --from-beginning

kubectl -n kafka exec -ti my-kafka-0 -- kafka-console-consumer --bootstrap-server my-kafka:9092 --topic TaxiCall --from-beginning

7. 서킷브레이커 설정 (동기호출을 당하는 서비스쪽에 설치)
5XX 오류에 대해 해당 서비스 차단 및 Thread 부하에 따른 Circuit Breaker 생성 (두꺼비 집 설치) Delivery 쪽
[DestinationRule로 설정]
-->/home/project#  경로에서 
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-driver
  namespace: istio-cb-ns
spec:
  host: a-driver
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 5
        maxRequestsPerConnection: 5
    outlierDetection:
      interval: 1s
      consecutiveErrors: 1
      baseEjectionTime: 10m
      maxEjectionPercent: 100
EOF

kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-driver
  namespace: istio-cb-ns
spec:
  host: a-driver
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1           # 목적지로 가는 HTTP, TCP connection 최대 값. (Default 1024)
      http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
        maxRetries: 3               # 기다리는 동안 최대 재시도 수(Default 1024)
    outlierDetection:
      consecutiveErrors: 5          # 5xx 에러가 5번 발생하면
      interval: 1s                  # 1초마다 스캔 하여
      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
      maxEjectionPercent: 100       # 100% 로 차단
EOF

*참고정보********************************************************************
        http1MaxPendingRequests: 3    (pending수 )
        maxRequestsPerConnection: 3   
        consecutiveErrors: 1    --> 1번이라도 500에러가 떨어지면
        baseEjectionTime: 10m     --> 10분동안 제외 (라우팅 타겟에서 제외)
*******************************************************************************
(생성확인방법 : 아래 명령어 실행 시 설정한 내용이 보여야함)
kubectl describe dr dr-driver -n istio-cb-ns

서킷브레이커 삭제 
kubectl delete DestinationRule a-driver


8. 부하발생 및 모니터링
부하발생기 생성방법 (1)
kubectl create deploy siege --image=apexacme/siege-nginx -n istio-cb-ns

부하발생기 생성방법 (2)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: siege
  namespace: istio-cb-ns
spec:
  containers:
  - name: siege
    image: apexacme/siege-nginx
EOF

kubectl get all -n istio-cb-ns
kubectl exec -it pod/siege-7df8f548c-pvv6w -n istio-cb-ns -c siege-nginx -- /bin/bash

부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
kubectl exec -it siege --container siege -n istio-cb-ns -- /bin/bash
siege -c1 -t3S -v  http://a-driver:8080     (c뒤는 사용자 수(current user)  t뒤에는 3초동안 : 1명의 사용자가 3초동안 타켓서비스에 부하를 발생 시킨다)
siege -c100 -t10S -r20 -v  http://a-driver:8080     (c뒤는 사용자 수(current user)  t뒤에는 3초동안 : 1명의 사용자가 3초동안 타켓서비스에 부하를 발생 시킨다)




siege -v -c1 -t240S --content-type "application/json" 'http://ab63205308df54474afe2cebc4e45bb5-1085189319.ap-northeast-2.elb.amazonaws.com:8080/a-driver POST {"id": "101","hospitalId":"2","hospitalNm":"bye","chkDate":"0909","pcnt":20}'



-- 서킷브레이커가 트랜잭션을 적절히 파란색(허용?) / 빨간색(차단?)함 -- 운영시스템이 죽지않고 지속적으로 CB에 의하여 자원을 보호함 대신 
    Availability(가용성)이 떨어짐, 이 HPA(오토스케일 아웃을 통한 가용성을 높여야함)

[모니터링]
kubectl get svc -n istio-system
 --> kiali가 ClusterIP로 되어 있어 이걸 LoadBalancer로 변경
kubectl edit svc kiali -n istio-system
:set nu
[편집화면에서 type: 을 LoadBalancer로 변경하고 저장]
:wq

kubectl get svc -n istio-system  [kiail  URL로 접속]
LoadBalancer로 변경 후 외부 url 정보로 kiali 접속  :  admin / admin
------------------------------------------------------------------------------------------------
[4] 오토 스케일링 설정, hpa: HorizontalPodAutoscaler (어느 시점에 해야 하는지???)
1. 오토스케일아아웃 설정 (cpu가 20% 이상 1분가 유지될 replica를 3개까지 늘려주라는 명령어) 
kubectl autoscale deploy a-driver --min=1 --max=10 --cpu-percent=10


kubectl autoscale deployment.apps/a-driver --cpu-percent=20 --min=1 --max=10

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

kubectl get deployment metrics-server -n kube-system


kubectl label namespace istio-cb-ns istio-injection=disabled --overwrite

2. 오토스케일 아웃 설정 후 모니터링 
(1) CB 에서 했던 방식대로 2분동안 부하를 걸어준다. 
siege -c100 -t120S -r20 -v  http://a-driver:8080

(2) 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
kubectl get deploy a-driver -w

스케일아웃이 벌어지는 것을 확인
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
delivery     1         1         1            1           17s
delivery     1         2         1            1           45s
delivery     1         4         1            1           1m

siege의 로그의 Availability가 높아짐을 확인
------------------------------------------------------------------------------------------------
[5] 무정지 배포 (Readness / Liveness)  CI/CD Buildspec.yml에 설정되어 있음
buildsepc.yaml 에서 RollingUpdate / Readness / Liveness 설정을 통해 진행

* RollingUpdate

우선 무정지 배포를 증명하기 위해 
------------------------------------------------------------------------------------------------
[6] config-map 이용  (????????????????)
-----------------------------------------------------------------------------------------------
참고 : kubectl 명령어

kubectl config set-context --current --namespace=istio-cb-ns   #istio-cb-ns를 기본네임스페이스로 지정
kubectl config view --minify | grep namespace:       #default Namespace 확인



kubectl -n kafka exec -ti my-kafka-0 -- kafka-console-consumer --bootstrap-server my-kafka:9092 --topic TaxiCall --from-beginning

kubectl -n kafka exec my-kafka-0 – /usr/bin/kafka-topics --zookeeper my-kafka-zookeeper:2181 --topic TaxiCall --create --partitions 1 --replication-factor 1

kubectl logs -f a-management-596c6799ff-285r6 -c a-management


```
