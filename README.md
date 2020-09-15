# 5th-teamA-Taxi-Call

# TaxiCall (택시 호출 서비스)

# repo
 1. 호출관리 : https://github.com/rladutp/order.git
 1. 상태관리 : https://github.com/rladutp/management.git
 1. 기사관리 : https://github.com/rladutp/driver.git
 1. 예약관리 : https://github.com/rladutp/orderStatus.git
 1. 게이트웨이 : https://github.com/rladutp/gateway.git

A조 택시 호출 서비스 CNA개발 실습을 위한 프로젝트

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
1. 고객이 택시를 호출(장소, 고객명)한다.
1. Management 에서 호출을 받아서 택시 기사에서 체크할 것을 요청한다.(Sync)
1. 택시 기사는 받은 호출을 수락하거나 거절한다.(Async)
1. Management에서 변경사항을 접수 받는다.
1. 호출 수락 시 고객에게 호출되었음을 공유한다.(Async)
1. 호출 거절 시 고객에게 호출 거절되었음을 공유한다.(Async)
1. 고객이 Taxi 호출 예약을 취소한다.(Async)
1. 고객의 예약 취소에 따라서 Management 내역의 상태가 예약 취소로 변경된다.
1. Driver에게 예약취소 되었음을 공유 한다.(Async)
1. 호출된 현황을 조회한다.

## 비기능적 요구사항
1. 트랜잭션
    1. 고객의 예약을 기사가 수락/거절 가능하다. > Sync
    1. 고객의 취소에 따라서 요청 예약의 상태가 변경된다. > Async
1. 장애격리
    1. 기사 관리 서비스에 장애가 발생하더라도 고객 예약은 정상적으로 처리 가능하다.  > Async (event-driven)
    1. 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
1. 성능
    1. 고객은 본인의 예약 상태 및 이력 정보를 확인할 수 있다. > CQRS


# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
  ![1](https://user-images.githubusercontent.com/67453893/91927140-baa8af00-ed13-11ea-8ead-0d4616a6da56.png)

## TO-BE 조직 (Vertically-Aligned)
  ![#017](https://github.com/skldk89/TaxiCall/blob/master/Image/%23017.png)

## Event Storming 결과

![#000](https://github.com/skldk89/TaxiCall/blob/master/Image/%23000.png)

### 이벤트 도출
![#001](https://github.com/skldk89/TaxiCall/blob/master/Image/%23001.png)

### 부적격 이벤트 탈락
![#002](https://github.com/skldk89/TaxiCall/blob/master/Image/%23002.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함

### 액터, 커맨드 부착하여 읽기 좋게
![#003](https://github.com/skldk89/TaxiCall/blob/master/Image/%23003.png)

### 어그리게잇으로 묶기
![#004](https://github.com/skldk89/TaxiCall/blob/master/Image/%23004.png)


### 바운디드 컨텍스트로 묶기

![#005](https://github.com/skldk89/TaxiCall/blob/master/Image/%23005.png)

    - 도메인 서열 분리 
        - Core Domain:  Management : 없어서는 안될 핵심 서비스이며, 연결 Up-time SLA 수준을 99.999% 목표, 배포주기는  1주일 1회 미만
        - Supporting Domain:   Order : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 80% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   Driver : Order의 상태를 수신하고, 요청에 대한 승인/거절을 진행하는 서비스이며, SLA 수준은 연간 80% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함. 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![#006](https://github.com/skldk89/TaxiCall/blob/master/Image/%23006.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![#007](https://github.com/skldk89/TaxiCall/blob/master/Image/%23007.png)

```
# 도메인 서열
- Core : Management
- Supporting : Order
- General : Driver
```

### 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![#011](https://github.com/skldk89/TaxiCall/blob/master/Image/%23011.png)
![#012](https://github.com/skldk89/TaxiCall/blob/master/Image/%23012.png)
![#013](https://github.com/skldk89/TaxiCall/blob/master/Image/%23013.png)
![#014](https://github.com/skldk89/TaxiCall/blob/master/Image/%23014.png)
![#015](https://github.com/skldk89/TaxiCall/blob/master/Image/%23015.png)

## 헥사고날 아키텍처 다이어그램 도출

* CQRS 를 위한 orderStatus 서비스만 DB를 구분하여 적용
![#018](https://github.com/skldk89/TaxiCall/blob/master/Image/%23018.png)


# 구현

## 시나리오 테스트결과

| 기능 | 이벤트 Payload |
|---|:---:|
| 1. 고객이 택시를 호출(장소, 고객명)한다. |![image](https://user-images.githubusercontent.com/25805562/91837451-3577b880-ec87-11ea-88b1-dc4e9d74790d.png)|
| 2. Management 에서 호출을 받아서 택시 기사에서 체크할 것을 요청한다.(Sync)</br>3. 택시 기사는 받은 호출을 수락하거나 거절한다.(Async)</br>4. Management에서 변경사항을 접수 받는다.</br>5. 호출 수락 시 고객에게 호출되었음을 공유한다.(Async)</br>6. 호출 거절 시 고객에게 호출 거절되었음을 공유한다.(Async) |![image](https://user-images.githubusercontent.com/25805562/91837806-7a035400-ec87-11ea-8966-09403bd5e7eb.png)|
| 7. 고객이 Taxi 호출 예약을 취소한다.(Async)</br>8. 고객의 예약 취소에 따라서 Management 내역의 상태가 예약 취소로 변경된다.</br>9. Driver에게 예약취소 되었음을 공유 한다.(Async) | ![image](https://user-images.githubusercontent.com/25805562/91837990-c2227680-ec87-11ea-9fb1-530410922532.png) |
| 10. 호출된 현황을 조회한다.| ![image](https://user-images.githubusercontent.com/25805562/91838415-6ad0d600-ec88-11ea-9df8-1c6895fe6d75.png) |


## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* MyPage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| Order | 호출 관리 | 8081 | http://localhost:8081/orders | http://Order:8080/orders |
| Management | 상태 관리 | 8082 | http://localhost:8082/managements | http://Management:8080/managements |
| Driver | 기사 관리 | 8083 | http://localhost:8083/drivers | http://Driver:8080/drivers |
| OrderStatus | 예약 관리 | 8084 | http://localhost:8084/orderStatuses | http://OrderStatus:8080/orderStatuses |


## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: Order
          uri: http://Order:8080
          predicates:
            - Path=/orders/** 
        - id: Management
          uri: http://Management:8080
          predicates:
            - Path=/managements/** 
        - id: Driver
          uri: http://Driver:8080
          predicates:
            - Path=/drivers/** 
        - id: orderStatus
          uri: http://orderStatus:8080
          predicates:
            - Path= /orderStatuses/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 orderStatus 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

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

분석단계에서의 조건 중 하나로 고객 호출(Order) -> 상태 관리(Management) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
Order 상태 관리 > 상태 관리 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리
- FeignClient 서비스 구현

```
# DriverService.java

@FeignClient(name="Driver", url= "${api.url.driver}")
public interface DriverService {

    @RequestMapping(method= RequestMethod.GET, path="/drivers/check")
    public void checkOrder(@RequestBody Driver param);

}
```

- Order 호출을 받은 직후(@PostPersist) Management에서 요청하도록 처리
```
# Management.java (Entity)
    
    @PostPersist
    public void onPostPersist(){
        System.out.println("## seq 0 :  "+this.getStatus());
        System.out.println("## seq 1 ");

        if(this.getStatus().equals("Ordered")) {
            TaxiCall.external.Driver driver = new TaxiCall.external.Driver();
            driver.setStatus("Ordered");
            driver.setDriverId(this.getDriverId());
            driver.setOrderId(this.getOrderId());
            driver.setLocation(this.getLocation());

            System.out.println(this.getDriverId() + "TEST 1 : " + this.getOrderId());

            ManagementApplication.applicationContext.getBean(TaxiCall.external.DriverService.class)
                    .checkOrder(driver);

            System.out.println("AAAA");
        }

        else if(this.getStatus().equals("OrderCanceled")) {
            CancelOrderRequested cancelOrderRequested = new CancelOrderRequested();

            cancelOrderRequested.setOrderId(this.getOrderId());
            cancelOrderRequested.setStatus(this.getStatus());
            cancelOrderRequested.setDriverId(this.getDriverId());

            BeanUtils.copyProperties(this, cancelOrderRequested);
            cancelOrderRequested.publishAfterCommit();
        }
    }
    
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 상태 관리 시스템이 장애가 나면 호출요청을 못 받는다는 것을 확인


```
#상태 관리(Management) 서비스를 잠시 내려놓음 (ctrl+c)

#고객호출요청
http POST localhost:8081/orders driverId=1 customerName="customer1" location="seoul1" status="Ordered"   #Fail
http POST localhost:8081/orders driverId=2 customerName="customer2" location="seoul2" status="Ordered"   #Fail

#상태 관리 재기동
cd Management
mvn spring-boot:run

#고객검진요청 처리
http POST localhost:8081/orders driverId=1 customerName="customer1" location="seoul1" status="Ordered"   #Success
http POST localhost:8081/orders driverId=2 customerName="customer2" location="seoul2" status="Ordered"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


고객호출취소가 이루어진 후에 예약시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하였다.
 
- 이를 위하여 고객호출신청 상태관리에 기록을 남긴 후에 곧바로 호출취소신청 되었다는 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
@Entity
@Table(name="Management_table")
public class Management {

    @PostPersist
    public void onPostPersist(){
        System.out.println("## seq 0 :  "+this.getStatus());
        System.out.println("## seq 1 ");

        if(this.getStatus().equals("Ordered")) {
            TaxiCall.external.Driver driver = new TaxiCall.external.Driver();
            driver.setStatus("Ordered");
            driver.setDriverId(this.getDriverId());
            driver.setOrderId(this.getOrderId());
            driver.setLocation(this.getLocation());

            System.out.println(this.getDriverId() + "TEST 1 : " + this.getOrderId());

            ManagementApplication.applicationContext.getBean(TaxiCall.external.DriverService.class)
                    .checkOrder(driver);

            System.out.println("AAAA");
        }
        else if(this.getStatus().equals("OrderCanceled")) {
            CancelOrderRequested cancelOrderRequested = new CancelOrderRequested();

            cancelOrderRequested.setOrderId(this.getOrderId());
            cancelOrderRequested.setStatus(this.getStatus());
            cancelOrderRequested.setDriverId(this.getDriverId());

            BeanUtils.copyProperties(this, cancelOrderRequested);
            cancelOrderRequested.publishAfterCommit();
        }
    }
```
- 상태관리 서비스에서는 호출취소신청 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }
    @Autowired
    DriverRepository driverRepository;
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverCancelOrderRequested_ReceiveCancelOrder(@Payload CancelOrderRequested cancelOrderRequested){

        if(cancelOrderRequested.isMe()){
            System.out.println("##### listener ReceiveCancelOrder : " + cancelOrderRequested.toJson());

            Driver driver = new Driver();
            driverRepository.findById(Long.valueOf(cancelOrderRequested.getOrderId())).ifPresent((Driver)->{
                Driver.setDriverId(cancelOrderRequested.getDriverId());
                Driver.setStatus("OrderCanceled");
                driverRepository.save(Driver);
            });

        }
    }

}
```

기사관리 시스템은 Taxi 호출과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 기사 관리시스템이 유지보수로 인해 잠시 내려간 상태라도 호출 신청을 받는데 문제가 없다.

```
#기사관리 서비스 (Driver) 를 잠시 내려놓음 (ctrl+c)

#호출취소처리
http PATCH localhost:8081/orders driverId=1 status= "OrderCanceled"   #Success

#예약관리상태 확인
http localhost:8083/drivers     # 예약상태 안바뀜 확인

#예약관리 서비스 기동
cd Driver
mvn spring-boot:run

#예약관리상태 확인
http localhost:8083/drivers     # 예약상태가 "취소됨"으로 확인
```

# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, 
pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
- CodeBuild 기반으로 CI/CD 파이프라인 구성
MSA 서비스별 CodeBuild 프로젝트 생성하여  CI/CD 파이프라인 구성

![#019](https://github.com/skldk89/TaxiCall/blob/master/Image/%23019.png)

- Git Hook 연결
연결한 Github의 소스 변경 발생 시 자동으로 빌드 및 배포 되도록 Git Hook 연결 설정

![#020](https://github.com/skldk89/TaxiCall/blob/master/Image/%23020.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

### 서킷 브레이킹 istio-injection + DestinationRule

* istio-injection 적용 (기 적용완료)
```
kubectl label namespace skcc-ns istio-injection=enabled
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시
```
$siege -c100 -t60S -r10  -v http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals 
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.00 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
```
* 서킷 브레이킹을 위한 DestinationRule 적용
```
#dr-hospital.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-hospital
  namespace: skcc-ns
spec:
  host: hospitalmanage
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      interval: 1s
      consecutiveErrors: 2
      baseEjectionTime: 10s
      maxEjectionPercent: 100
```


```
$kubectl apply -f dr-hospital.yaml

$siege -c100 -t60S -r10  -v http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals 
HTTP/1.1 200   0.03 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.04 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.01 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      81 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      95 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      95 bytes ==> GET  /hospitals
HTTP/1.1 200   0.03 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 200   0.02 secs:    5650 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.02 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.01 secs:      19 bytes ==> GET  /hospitals
HTTP/1.1 503   0.00 secs:      19 bytes ==> GET  /hospitals

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

* DestinationRule 적용되어 서킷 브레이킹 동작 확인 (kiali 화면)
![Kaili_DR RUlE적용](https://user-images.githubusercontent.com/67453893/91850567-b7bca880-ec98-11ea-8f9b-59b4223fb046.png)

* 다시 부하 발생하여 DestinationRule 적용 제거하여 정상 처리 확인
```
kubectl delete -f dr-hospital.yaml
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
kubectl label namespace skcc-ns istio-injection=disabled --overwrite
```

- Deployment 배포시 resource 설정 적용
```
    spec:
      containers:
          ...
          resources:
            limits:
              cpu: 500m 
            requests:
              cpu: 200m 
```

- replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy hospitalmanage -n skcc-ns --min=1 --max=10 --cpu-percent=15

# 적용 내용
$kubectl get all -n skcc-ns
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)          AGE
service/gateway             LoadBalancer   10.100.95.162    a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com   8080:30387/TCP   19h
service/hospitalmanage      ClusterIP      10.100.5.64      <none>                                                                   8080/TCP         19h
service/mypage              ClusterIP      10.100.240.169   <none>                                                                   8080/TCP         19h
service/reservationmanage   ClusterIP      10.100.232.233   <none>                                                                   8080/TCP         19h
service/screeningmanage     ClusterIP      10.100.101.120   <none>                                                                   8080/TCP         19h

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gateway             1/1     1            1           19h
deployment.apps/hospitalmanage      1/1     1            1           11h
deployment.apps/mypage              1/1     1            1           19h
deployment.apps/reservationmanage   1/1     1            1           19h
deployment.apps/screeningmanage     1/1     1            1           19h

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/gateway-5d58bbcb67             1         1         1       19h
replicaset.apps/gateway-db44fcf75              0         0         0       19h
replicaset.apps/hospitalmanage-8658bbbb6f      1         1         1       11h
replicaset.apps/mypage-567c4b57ff              1         1         1       18h
replicaset.apps/mypage-f5486756b               0         0         0       19h
replicaset.apps/reservationmanage-6f47749879   0         0         0       18h
replicaset.apps/reservationmanage-c96669994    1         1         1       18h
replicaset.apps/reservationmanage-f74d47f65    0         0         0       19h
replicaset.apps/screeningmanage-56ff67c8cf     0         0         0       18h
replicaset.apps/screeningmanage-598b5f9767     0         0         0       17h
replicaset.apps/screeningmanage-645c457774     0         0         0       19h
replicaset.apps/screeningmanage-6485bb9857     0         0         0       17h
replicaset.apps/screeningmanage-6865764467     0         0         0       19h
replicaset.apps/screeningmanage-78984d5dc8     0         0         0       17h
replicaset.apps/screeningmanage-9498f6bdc      1         1         1       17h

NAME                                                 REFERENCE                   TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hospitalmanage   Deployment/hospitalmanage   2%/15%   1         10        0          7s
```

- siege로 워크로드를 1분 동안 걸어준다.
```
$  siege -c100 -t60S -r10  -v http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy hospitalmanage -n skcc-ns -w 
```

- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hospitalmanage   1/1     1            1           11h
hospitalmanage   1/4     1            1           11h
hospitalmanage   1/4     1            1           11h
hospitalmanage   1/4     1            1           11h
hospitalmanage   1/4     4            1           11h
hospitalmanage   1/5     4            1           11h
hospitalmanage   1/5     4            1           11h
hospitalmanage   1/5     4            1           11h
hospitalmanage   1/5     5            1           11h
```

- kubectl get으로 HPA을 확인하면 CPU 사용률이 64%로 증가됐다.
```
$kubectl get hpa hospitalmanage -n skcc-ns 
NAME                                                 REFERENCE                   TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hospitalmanage   Deployment/hospitalmanage   64%/15%   1         10        5          2m54s
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
$kubectl kubectl delete hpa hospitalmanage  -n skcc-ns
```


## 무정지 재배포

먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, CB 설정을 제거함
Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
$ siege -v -c1 -t240S --content-type "application/json" 'http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals POST {"id": "101","hospitalId":"2","hospitalNm":"bye","chkDate":"0909","pcnt":20}'

** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 200     0.48 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 200     0.49 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 200     0.63 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 200     0.48 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
:

```

- CI/CD 파이프라인을 통해 새버전으로 재배포 작업함
Git hook 연동 설정되어 Github의 소스 변경 발생 시 자동 빌드 배포됨
재배포 작업 중 서비스 중단됨 (503 오류 발생)
```
HTTP/1.1 200     0.48 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 200     0.55 secs:       0 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.47 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.48 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.51 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.47 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.48 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.53 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.50 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
HTTP/1.1 503     0.45 secs:      95 bytes ==> POST http://a67fdf8668e5d4b518f8ac2a62bd4b45-334568913.us-east-2.elb.amazonaws.com:8080/hospitals
:

```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:                    372 hits
Availability:                  90.29 %
Elapsed time:                 205.09 secs
Data transferred:               0.00 MB
Response time:                  0.55 secs
Transaction rate:               1.81 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    1.00
Successful transactions:         372
Failed transactions:              40
Longest transaction:            1.50
Shortest transaction:           0.43

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
    initialDelaySeconds: 30
    timeoutSeconds: 2
    periodSeconds: 5
    failureThreshold: 10
    
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:                    234 hits
Availability:                 100.00 %
Elapsed time:                 119.04 secs
Data transferred:               0.00 MB
Response time:                  0.51 secs
Transaction rate:               1.97 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    1.00
Successful transactions:         234
Failed transactions:               0
Longest transaction:            1.57
Shortest transaction:           0.41

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


## ConfigMap 사용

시스템별로 또는 운영중에 동적으로 변경 가능성이 있는 설정들을 ConfigMap을 사용하여 관리합니다.
Application에서 특정 도메일 URL을 ConfigMap 으로 설정하여 운영/개발등 목적에 맞게 변경가능합니다.  

* my-config.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: skcc-ns
data:
  api.hospital.url: http://HospitalManage:8080
```
my-config라는 ConfigMap을 생성하고 key값에 도메인 url을 등록한다. 

* ScreeningManage/buildsepc.yaml (configmap 사용)
```
 cat  <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: $_PROJECT_NAME
          namespace: $_NAMESPACE
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
                  image: $AWS_ACCOUNT_ID.dkr.ecr.$_AWS_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                  ports:
                    - containerPort: 8080
                  env:
                    - name: api.hospital.url
                      valueFrom:
                        configMapKeyRef:
                          name: my-config
                          key: api.hospital.url
                  imagePullPolicy: Always
                
        EOF
```
Deployment yaml에 해단 configMap 적용

* HospitalService.java
```
@FeignClient(name="HospitalManage", url="${api.hospital.url}")//,fallback = HospitalServiceFallback.class)
public interface HospitalService {

    @RequestMapping(method= RequestMethod.PUT, value="/hospitals/{hospitalId}", consumes = "application/json")
    public void screeningRequest(@PathVariable("hospitalId") Long hospitalId, @RequestBody Hospital hospital);

}
```
url에 configMap 적용

* kubectl describe pod screeningmanage-9498f6bdc-qtclh  -n skcc-ns
```
Containers:
  screeningmanage:
    Container ID:   docker://8415f0125bac0264b5f77d14ed8ee7c28bc177e2cce9141a4c36e076c7920971
    Image:          052937454741.dkr.ecr.us-east-2.amazonaws.com/screeningmanage:f8102f4078683bdbf345cc5cae7983b1cb8ea                                                                      668
    Image ID:       docker-pullable://052937454741.dkr.ecr.us-east-2.amazonaws.com/screeningmanage@sha256:ebc8945df607                                                                      acc63d87e20d345e17245e3472fec43a9690e8ab9ca959573c9b
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 01 Sep 2020 07:55:29 +0000
    Ready:          True
    Restart Count:  0
    Liveness:       http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:      http-get http://:8080/actuator/health delay=30s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.hospital.url:  <set to the key 'api.hospital.url' of config map 'my-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-xw8ld (ro)

```
kubectl describe 명령으로 컨테이너에 configMap 적용여부를 알 수 있다. 

