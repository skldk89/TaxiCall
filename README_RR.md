# admin03-RestaurantReservation

# RestaurantReservation (식당 예약 서비스)

# repo
 1. 예약관리 : https://github.com/skldk89/Reservation.git
 1. 상태관리 : https://github.com/skldk89/Management.git
 1. 식당관리 : https://github.com/skldk89/Owner.git
 1. 현황관리 : https://github.com/rladutp/reservationStatus.git
 1. 게이트웨이 : https://github.com/rladutp/gateway.git

식당 에약 서비스 CNA개발 실습을 위한 프로젝트

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
  - [시나리오 테스트결과](#시나리오-테스트결과)
- [분석/설계](#분석설계)
- [구현](#구현)
  - [DDD 의 적용](#ddd-의-적용)
  - [Gateway 적용](#Gateway-적용)
  - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
  - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
  - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
- [운영](#운영)
  - [CI/CD 설정](#cicd설정)
  - [서킷 브레이킹 / 장애격리](#서킷-브레이킹-/-장애격리)
  - [오토스케일 아웃](#오토스케일-아웃)
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 식당을 예약(고객명, 일자)한다.
1. Management 에서 호출을 받아서 식당 주인에게 체크할 것을 요청한다.(Sync)
1. 식당 주인은 받은 호출을 수락하거나 거절한다.(Async)
1. Management에서 변경사항을 접수 받는다.
1. 예약 수락 시 고객에게 수락 되었음을 공유한다.(Async)
1. 예약 거절 시 고객에게 거절되었음을 공유한다.(Async)
1. 현황을 조회한다.

## 비기능적 요구사항
1. 트랜잭션
    1. 고객의 예약을 식당 주인이 수락/거절 가능하다. > Sync
    1. 식당 주인이 거절하면 요청 예약의 상태가 변경된다. > Async
1. 장애격리
    1. 식당 관리 서비스에 장애가 발생하더라도 고객 예약은 정상적으로 처리 가능하다.  > Async (event-driven)
    1. 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
1. 성능
    1. 고객은 본인의 예약 상태 및 이력 정보를 확인할 수 있다. > CQRS


# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
  ![1](https://user-images.githubusercontent.com/67453893/91927140-baa8af00-ed13-11ea-8ead-0d4616a6da56.png)

## TO-BE 조직 (Vertically-Aligned)
  ![#17](https://github.com/skldk89/TaxiCall/blob/master/Image/%2317.png)

## Event Storming 결과
![#00](https://github.com/skldk89/TaxiCall/blob/master/Image/%2300.png)

## 헥사고날 아키텍처 다이어그램 도출

* CQRS 를 위한 reservationStatus 서비스만 DB를 구분하여 적용
![#18](https://github.com/skldk89/TaxiCall/blob/master/Image/%2318.png)

# 구현

## 시나리오 테스트결과

| 기능 | 이벤트 Payload |
|---|:---:|
| 1. 고객이 식당을 예약(고객명, 일자)한다. |![#24](https://github.com/skldk89/TaxiCall/blob/master/Image/%2324.png)</br>![#25](https://github.com/skldk89/TaxiCall/blob/master/Image/%2325.png)|
| 2. Management 에서 호출을 받아서 식당 주인에게 체크할 것을 요청한다.(Sync)</br>3. 식당 주인은 받은 호출을 수락하거나 거절한다.(Async)</br>4. Management에서 변경사항을 접수 받는다.</br>5. 예약 수락 시 고객에게 수락 되었음을 공유한다.(Async)</br>6. 예약 거절 시 고객에게 거절되었음을 공유한다.(Async) |![#27](https://github.com/skldk89/TaxiCall/blob/master/Image/%2327.png)</br>![#28](https://github.com/skldk89/TaxiCall/blob/master/Image/%2328.png)|
| 7. 현황을 조회한다.| ![#29](https://github.com/skldk89/TaxiCall/blob/master/Image/%2329.png) |


## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* MyPage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| Reservation | 예약 관리 | 8081 | http://localhost:8081/reservations | http://Reservation:8080/reservations |
| Management | 상태 관리 | 8083 | http://localhost:8083/managements | http://Management:8080/managements |
| Owner | 식당 관리 | 8082 | http://localhost:8082/owners | http://Owner:8080/owners |
| OrderStatus | 현황 관리 | 8084 | http://localhost:8084/reservationStatuses | http://ReservationStatus:8080/reservationStatuses |

## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: reservation
          uri: http://reservation:8080
          predicates:
            - Path=/reservations/** 
        - id: owner
          uri: http://owner:8080
          predicates:
            - Path=/owners/** 
        - id: management
          uri: http://management:8080
          predicates:
            - Path=/managements/** 
        - id: reservationstatus
          uri: http://reservationstatus:8080
          predicates:
            - Path= /reservationStatuses/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 reservationStatus 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
pom.xml 에 적용
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 고객 예약(Reservation) -> 상태 관리(Management) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
Reservation 상태 관리 > 상태 관리 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리
- FeignClient 서비스 구현

```
# OwnerService.java

@FeignClient(name="Owner", url="http://Owner:8080")
public interface OwnerService {

    @RequestMapping(method= RequestMethod.GET, path="/owners")
    public void checkReservation(@RequestBody Owner owner);

}
```

- Reservation 예약을 받은 직후(@PostPersist) Management에서 요청하도록 처리
```
# Management.java (Entity)
    
    @PostPersist
    public void onPostPersist(){
        System.out.println("## seq 0 ");
        System.out.println("## seq 1 ");

        if(this.getStatus().equals("RequestedReservation")) {
            RestaurantReservation.external.Owner owner = new RestaurantReservation.external.Owner();

            owner.setStatus("Requested");
            owner.setOwnerId(this.getOwnerId());
            owner.setReservationId(this.getReservationId());
            owner.setReservationDate(this.getReservationDate());

            System.out.println(this.getOwnerId() + "TEST 1");

            ManagementApplication.applicationContext.getBean(RestaurantReservation.external.OwnerService.class)
                    .checkReservation(owner);

            System.out.println("AAAA");
        }
    }
    
```

- 상태관리 서비스에서는 예약 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다
```
@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    ManagementRepository managementRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReservationRequested_RequestConfirmReservation(@Payload ReservationRequested reservationRequested){

        if(reservationRequested.isMe()){
            System.out.println("##### listener RequestConfirmReservation : " + reservationRequested.toJson());
            Management management = new Management();

            management.setStatus("RequestedReservation");
            management.setOwnerId(reservationRequested.getOwnerId());
            management.setReservationId(reservationRequested.getReservationId());
            management.setReservationDate(reservationRequested.getReservationDate());
            management.setId(reservationRequested.getId());

            managementRepository.save(management);
        }
    }

}
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 상태 관리 시스템이 장애가 나면 호출요청을 못 받는다는 것을 확인


```
#상태 관리(Management) 서비스를 잠시 내려놓음 (ctrl+c)

#고객 예약 요청
http POST localhost:8081/reservations orderId=1 driverId=1 customerName="customer1" reservationDate="20200916" status="SSS"   #Fail
http POST localhost:8081/reservations orderId=2 driverId=2 customerName="customer2" reservationDate="20200916" status="SSS"   #Fail

#상태 관리 재기동
cd Management
mvn spring-boot:run

#고객 예약 요청 처리
http POST localhost:8081/reservations orderId=1 driverId=1 customerName="customer1" reservationDate="20200916" status="SSS"   #Success
http POST localhost:8081/reservations orderId=2 driverId=2 customerName="customer2" reservationDate="20200916" status="SSS"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


식당에서 승인/취소로 이루어진 후에 예약시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하였다.
 
- 이를 위하여 예약 상태관리에 기록을 남긴 후에 곧바로 호출 승인/취소신청 되었다는 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
@Entity
@Table(name="Management_table")
public class Management {

   @PostUpdate
    public void onPostUpdate(){
        System.out.println("START1");
        if(this.getStatus().equals("Approved")) {
            System.out.println("test approved");
            ReservationApporved reservationApporved = new ReservationApporved();

            reservationApporved.setReservationId(this.getReservationId());
            reservationApporved.setStatus(this.getStatus());
            reservationApporved.setOwnerId(this.getOwnerId());

            BeanUtils.copyProperties(this, reservationApporved);
            reservationApporved.publishAfterCommit();
        }

        else if(this.getStatus().equals("Declined")) {
            System.out.println("test declined");
            ReservationCanceled reservationCanceled = new ReservationCanceled();

            reservationCanceled.setId(this.getId());
            reservationCanceled.setReservationId(this.getReservationId());
            reservationCanceled.setStatus(this.getStatus());
            reservationCanceled.setOwnerId(this.getOwnerId());

            BeanUtils.copyProperties(this, reservationCanceled);
            reservationCanceled.publishAfterCommit();
        }
    }

```
- 상태관리 서비스에서는 예약 승인/취소 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    ManagementRepository managementRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReservationApproved_StatusChange(@Payload ReservationApproved reservationApproved){


        if(reservationApproved.isMe()){
            System.out.println("##### listener StatusChange : " + reservationApproved.toJson());

            managementRepository.findById(Long.valueOf(reservationApproved.getReservationId())).ifPresent((Management)->{
                Management.setOwnerId(reservationApproved.getOwnerId());
                Management.setReservationDate(reservationApproved.getReservationDate());
                Management.setStatus("Approved");
                managementRepository.save(Management);
            });

            System.out.println("reservationrequested end1");
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReservationDeclined_StatusChange(@Payload ReservationDeclined reservationDeclined){

        if(reservationDeclined.isMe()){
            System.out.println("##### listener StatusChange : " + reservationDeclined.toJson());

            System.out.println(reservationDeclined.toJson());
            managementRepository.findById(Long.valueOf(reservationDeclined.getReservationId())).ifPresent((Management)->{
                Management.setOwnerId(reservationDeclined.getOwnerId());
                Management.setReservationDate(reservationDeclined.getReservationDate());
                Management.setStatus("Declined");
                managementRepository.save(Management);
            });

            System.out.println("reservationrequested end1");
        }
    }
}

```

예약 시스템은 식당 관리와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 예약 관리시스템이 유지보수로 인해 잠시 내려간 상태라도 승인/취소 신청을 받는데 문제가 없다.

```
#예약 관리 서비스 (Reservation) 를 잠시 내려놓음 (ctrl+c)

#예약 승인/거절
http PATCH http://localhost:8082/owners/check?reservationId=1  #Success

#예약관리상태 확인
http localhost:8081/resrvations     # 예약상태 안바뀜 확인

#예약관리 서비스 기동
cd Reservation
mvn spring-boot:run

#예약관리상태 확인
http localhost:8081/resrvations     # 예약상태가 "승인/취소됨"으로 확인
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, 
pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
- CodeBuild 기반으로 CI/CD 파이프라인 구성
MSA 서비스별 CodeBuild 프로젝트 생성하여  CI/CD 파이프라인 구성

![#19](https://github.com/skldk89/TaxiCall/blob/master/Image/%2319.png)

- Git Hook 연결
연결한 Github의 소스 변경 발생 시 자동으로 빌드 및 배포 되도록 Git Hook 연결 설정

![#20](https://github.com/skldk89/TaxiCall/blob/master/Image/%2320.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

### 서킷 브레이킹 istio-injection + DestinationRule

* istio-injection 적용 (기 적용완료)
```
# Sidecar Actvate
kubectl label namespace istio-cb-ns istio-injection=enabled 
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시
```
$siege -c100 -t60S -r10  -v http://a9ef33930883f4c65a6c67007ec48a17-176394376.ap-northeast-2.elb.amazonaws.com:8080/owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /owners
```

* 서킷 브레이킹을 위한 DestinationRule 적용
```
#dr-owner.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-owner
  namespace: istio-cb-ns
spec:
  host: a-owner
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 5
        maxRequestsPerConnection: 5
    outlierDetection:
      interval: 1s
      consecutiveErrors: 1
      baseEjectionTime: 3s
      maxEjectionPercent: 100
```


```
$kubectl apply -f dr-owner.yaml
$siege -c100 -t60S -r10  -v http://a9ef33930883f4c65a6c67007ec48a17-176394376.ap-northeast-2.elb.amazonaws.com:8080/owners

HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /owners
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /owners
HTTP/1.1 503   0.02 secs:      95 bytes ==> GET  /owners
HTTP/1.1 503   0.01 secs:      95 bytes ==> GET  /owners
HTTP/1.1 200   0.03 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /owners
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /owners

Transactions:                    194 hits
Availability:                  16.68 %
Elapsed time:                  59.76 secs
Data transferred:               1.06 MB
Response time:                  0.03 secs
Transaction rate:               3.25 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                    0.10
Successful transactions:         194
Failed transactions:             969
Longest transaction:            0.04
Shortest transaction:           0.00
```

* 다시 부하 발생하여 DestinationRule 적용 제거하여 정상 처리 확인
```
kubectl delete -f dr-driver.yaml
```


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

* Metric Server 설치(CPU 사용량 체크를 위해)
```
$kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
$kubectl get deployment metrics-server -n kube-system
```

* (istio injection 적용한 경우) istio injection 적용 해제
```
kubectl label namespace istio-cb-ns istio-injection=disabled --overwrite
```

- replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy a-driver -n istio-cb-ns --min=1 --max=10 --cpu-percent=20

# 적용 내용
$kubectl get all -n istio-cb-ns
NAME                                READY   STATUS    RESTARTS   AGE
pod/a-driver-7d64667f79-ghspc       1/1     Running   1          5m14s
pod/a-driver-7d64667f79-lsvc6       0/1     Running   0          7s
pod/a-driver-7d64667f79-nk7gg       0/1     Running   0          7s
pod/a-driver-7d64667f79-tcssl       0/1     Running   0          7s
pod/a-gateway-64c49fb9c-fvxs5       2/2     Running   0          5h15m
pod/a-management-5565fcf6b7-vmjlf   2/2     Running   1          101m
pod/a-order-86f4f96986-x2q8b        2/2     Running   2          6h14m
pod/a-orderstatus-8c867d5df-5sfww   2/2     Running   3          6h18m
pod/httpie                          2/2     Running   0          9h
pod/siege-7df8f548c-pvv6w           2/2     Running   0          9h

NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)          AGE
service/a-driver        ClusterIP      10.100.195.28    <none>                                                                         8080/TCP         9h
service/a-gateway       LoadBalancer   10.100.30.17     ab63205308df54474afe2cebc4e45bb5-1085189319.ap-northeast-2.elb.amazonaws.com   8080:31850/TCP   9h
service/a-management    ClusterIP      10.100.205.161   <none>                                                                         8080/TCP         26h
service/a-order         ClusterIP      10.100.59.26     <none>                                                                         8080/TCP         25h
service/a-orderstatus   ClusterIP      10.100.33.227    <none>                                                                         8080/TCP         9h

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/a-driver        1/4     4            1           9h
deployment.apps/a-gateway       1/1     1            1           9h
deployment.apps/a-management    1/1     1            1           26h
deployment.apps/a-order         1/1     1            1           25h
deployment.apps/a-orderstatus   1/1     1            1           9h
deployment.apps/siege           1/1     1            1           9h

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/a-driver-6b69686948       0         0         0       118m
replicaset.apps/a-driver-7d64667f79       4         4         1       5m15s
replicaset.apps/a-driver-86567d5564       0         0         0       74m
replicaset.apps/a-gateway-64c49fb9c       1         1         1       5h15m
replicaset.apps/a-management-5565fcf6b7   1         1         1       101m
replicaset.apps/a-management-5d6cbb5568   0         0         0       125m
replicaset.apps/a-management-657c879dff   0         0         0       130m
replicaset.apps/a-management-6949d6d89f   0         0         0       119m
replicaset.apps/a-management-6d7b5b64f    0         0         0       6h23m
replicaset.apps/a-management-7b76f45765   0         0         0       4h7m
replicaset.apps/a-management-9f68d559     0         0         0       25h
replicaset.apps/a-order-66dcd46648        0         0         0       25h
replicaset.apps/a-order-86f4f96986        1         1         1       6h14m
replicaset.apps/a-orderstatus-6cd5fcc9b   0         0         0       9h
replicaset.apps/a-orderstatus-8c867d5df   1         1         1       6h18m
replicaset.apps/siege-7df8f548c           1         1         1       9h

NAME                                           REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/a-driver   Deployment/a-driver   12%/20%   1         10        1          39m
```

- siege로 워크로드를 2분 동안 걸어준다.
```
$  siege -c100 -t120S -v  http://a-driver:8080
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy a-driver -w 
```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
a-driver   1/1     1            1           9h
a-driver   2/1     1            2           9h
a-driver   1/1     1            1           9h
a-driver   0/1     1            0           9h
a-driver   1/1     1            1           9h
a-driver   1/4     1            1           9h
a-driver   1/4     1            1           9h
a-driver   1/4     1            1           9h
a-driver   1/4     4            1           9h
a-driver   1/7     4            1           9h
a-driver   1/7     4            1           9h
a-driver   1/7     4            1           9h
a-driver   1/7     7            1           9h
a-driver   2/7     7            2           9h
a-driver   3/7     7            3           9h
a-driver   4/7     7            4           9h
a-driver   5/7     7            5           9h
a-driver   6/7     7            6           9h
a-driver   7/7     7            7           9h
```

- kubectl get으로 HPA을 확인하면 CPU 사용률이 136%로 증가됐다.
```
$kubectl get hpa a-driver 
NAME       REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
a-driver   Deployment/a-driver   136%/20%    1         10        7          41m
```

- siege 의 로그를 보면 Availability가 100%로 유지된 것을 확인 할 수 있다.  
```
Lifting the server siege...
Transactions:                  26446 hits
Availability:                 100.00 %
Elapsed time:                 179.76 secs
Data transferred:               8.73 MB
Response time:                  0.68 secs
Transaction rate:             147.12 trans/sec
Throughput:                     0.05 MB/sec
Concurrency:                   99.60
Successful transactions:       26446
Failed transactions:               0
Longest transaction:            5.85
Shortest transaction:           0.00
```

- HPA 삭제 
```
$kubectl kubectl delete hpa a-driver
```


## 무정지 재배포

먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, CB 설정을 제거함
Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
$ siege -c1 -t300S -r20 -v  http://a-driver:8080

The server is now under siege...

HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.09 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
:

```

- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함
Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
재배포 작업 중 서비스 중단됨 (503 오류 발생)
```
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.06 secs:     206 bytes ==> GET  /
HTTP/1.1 503     0.07 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.04 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.03 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.09 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.03 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.02 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.05 secs:      91 bytes ==> GET  /
:

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                  77349 hits
Availability:                  98.82 %
Elapsed time:                 400.06 secs
Data transferred:              15.27 MB
Response time:                  0.00 secs
Transaction rate:             193.34 trans/sec
Throughput:                     0.04 MB/sec
Concurrency:                    0.94
Successful transactions:       77349
Failed transactions:             926
Longest transaction:            1.96
Shortest transaction:           0.00


```
- 배포기간중 Availability 가 평소 100%에서 90% 대로 떨어지는 것을 확인. 
원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문으로 판단됨. 
이를 막기위해 Readiness Probe 를 설정함 (buildspec.yml의 Readiness Probe 설정)
```
# buildspec.yaml 의 Readiness probe 의 설정:
- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함

readinessProbe:
    httpGet:
      path: '/actuator/health'
      port: 8080
    initialDelaySeconds: 10
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 10
    
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:                  71859 hits
Availability:                 100.00 %
Elapsed time:                 299.15 secs
Data transferred:              14.12 MB
Response time:                  0.00 secs
Transaction rate:             240.21 trans/sec
Throughput:                     0.05 MB/sec
Concurrency:                    0.96
Successful transactions:       71859
Failed transactions:               0
Longest transaction:            0.50
Shortest transaction:           0.00

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* a-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: a-config
  namespace: istio-cb-ns
data:
  api.url.driver: http://a-driver:8080
```
a-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* Management/buildsepc.yaml (configmap 사용)
```
cat  <<EOF | kubectl apply -f -
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: $_PROJECT_NAME
            namespace: istio-cb-ns
            labels:
              app: $_PROJECT_NAME
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: $_PROJECT_NAME
            template:
              metadata:
                labels:
                  app: $_PROJECT_NAME
              spec:
                containers:
                  - name: $_PROJECT_NAME
                    image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                    ports:
                      - containerPort: 8080
                    env:
                      - name: api.url.driver
                        valueFrom:
                          configMapKeyRef:
                            name: a-config
                            key: api.url.driver
                    readinessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 10
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 10
                    livenessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 120
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 5
        EOF
```
Deployment yaml에 해단 configMap 적용

* DriverService.java
```
@FeignClient(name="Driver", url= "${api.url.driver}")
public interface DriverService {

    @RequestMapping(method= RequestMethod.GET, path="/drivers/check")
    public void checkOrder(@RequestBody Driver param);

}
```
url에 configMap 적용

* kubectl describe pod/a-management-5565fcf6b7-vmjlf -n istio-cb-ns
```
Containers:
  a-management:
    Container ID:   docker://1bd0c96dc3ab4ee0d080104952c21e6e49ca9d7711d47b157ef3d3520febfa26
    Image:          271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/a-management:198772e0ab0fb240f7f05d0d0c32deb03ffe8b03
    Image ID:       docker-pullable://271153858532.dkr.ecr.ap-northeast-2.amazonaws.com/a-management@sha256:ebabfd754f645d3e0a8b9d5653634a1e5bb153ec15bb614edb75b0f5f2cbaad8
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 15 Sep 2020 09:12:40 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:      http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.url.driver:  <set to the key 'api.url.driver' of config map 'a-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-zcgdh (ro)
  istio-proxy:
    Container ID:  docker://bdb76ddb2a6243617094b905fd76dc4954caf49d4dc56412536d9dff35eff8e4
    Image:         docker.io/istio/proxyv2:1.4.5
    Image ID:      docker-pullable://istio/proxyv2@sha256:fc09ea0f969147a4843a564c5b677fbf3a6f94b56627d00b313b4c30d5fef094
    Port:          15090/TCP
    Host Port:     0/TCP
    Args:
      proxy
      sidecar
      --domain
      $(POD_NAMESPACE).svc.cluster.local
      --configPath
      /etc/istio/proxy
```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 

