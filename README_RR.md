# admin03-RestaurantReservation

# RestaurantReservation (식당 예약 서비스)

# repo
 1. 예약관리 : https://github.com/skldk89/Reservation.git
 1. 식당관리 : https://github.com/skldk89/Owner.git
 1. 상태관리 : https://github.com/skldk89/Management.git
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
  - [무정지 재배포](#무정지-재배포)
  

# 서비스 시나리오

## 기능적 요구사항
1. 고객이 식당을 예약(고객명, 일자)한다.
1. 상태 관리에서 요청된 예약을 받아서 식당 주인에게 전달한다.(Sync)
1. 식당 주인은 받은 예약 정보를 수락하거나 거절한다.(Async)
1. 상태 관리에서 변경사항을 접수 받는다.
1. 예약 수락 시 고객에게 수락 되었음을 공유한다.(Async)
1. 예약 거절 시 고객에게 거절되었음을 공유한다.(Async)
1. 현황을 조회한다.

## 비기능적 요구사항
1. 트랜잭션
    1. 고객의 예약을 식당 주인이 수락/거절 가능하다. > Sync
    1. 식당 주인이 거절하면 요청 예약의 상태가 변경된다. > Async
1. 장애격리
    1. 고객 예약 서비스에서 장애가 발생하더라도 식당 관리 서비스에서 주인이 승인/거절(취소) 가능하다  > Async (event-driven)
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
| 2. 상태 관리에서 요청된 예약을 받아서 식당 주인에게 전달한다.(Sync)</br>3. 식당 주인은 받은 예약 정보를 수락하거나 거절한다.(Async)</br>4. 식당 주인은 받은 예약 정보를 수락하거나 거절한다.(Async)</br>5. 상태 관리에서 변경사항을 접수 받는다.</br>6. 예약 수락 시 고객에게 수락 되었음을 공유한다.(Async)</br>7. 예약 거절 시 고객에게 거절되었음을 공유한다.(Async) |![#27](https://github.com/skldk89/TaxiCall/blob/master/Image/%2327.png)</br>![#28](https://github.com/skldk89/TaxiCall/blob/master/Image/%2328.png)|
| 8. 현황을 조회한다.| ![#29](https://github.com/skldk89/TaxiCall/blob/master/Image/%2329.png) |

## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* MyPage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| Reservation | 예약 관리 | 8081 | http://localhost:8081/reservations | http://Reservation:8080/reservations |
| Owner | 식당 관리 | 8082 | http://localhost:8082/owners | http://Owner:8080/owners |
| Management | 상태 관리 | 8083 | http://localhost:8083/managements | http://Management:8080/managements |
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
http localhost:8081/reservations     # 예약상태 안바뀜 확인

#예약관리 서비스 기동
cd Reservation
mvn spring-boot:run

#예약관리상태 확인
http localhost:8081/reservations     # 예약상태가 "승인/취소됨"으로 확인
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


## 무정지 재배포

먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, CB 설정을 제거함
Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
$ siege -c1 -t300S -r20 -v  http://owner:8080

The server is now under siege...

HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.09 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
:

```

- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함
Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
재배포 작업 중 서비스 중단됨 (503 오류 발생)
```
HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.00 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.01 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.02 secs:     206 bytes ==> GET  /
HTTP/1.1 200     0.06 secs:     206 bytes ==> GET  /
HTTP/1.1 503     0.05 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.07 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.03 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.09 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.03 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.06 secs:      91 bytes ==> GET  /
HTTP/1.1 503     0.07 secs:      91 bytes ==> GET  /
:

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                  77349 hits
Availability:                  97.28 %
Elapsed time:                 400.06 secs
Data transferred:              14.37 MB
Response time:                  0.00 secs
Transaction rate:             194.43 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    0.92
Successful transactions:       75245
Failed transactions:             925
Longest transaction:            1.95
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
Transactions:                  71843 hits
Availability:                 100.00 %
Elapsed time:                 298.18 secs
Data transferred:              14.21 MB
Response time:                  0.00 secs
Transaction rate:             241.12 trans/sec
Throughput:                     0.04 MB/sec
Concurrency:                    0.95
Successful transactions:       71843
Failed transactions:               0
Longest transaction:            0.50
Shortest transaction:           0.00

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


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
  host: owner
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
kubectl delete -f dr-owner.yaml
```

