# 4th-team3-health-screening

# screeningReservation (건강검진 예약 서비스)

# repo
 1. 병원관리 : https://github.com/byounghoonmoon/HospitalManage.git
 1. 검진관리 : https://github.com/byounghoonmoon/ScreeningManage.git
 1. 예약관리 : https://github.com/byounghoonmoon/ReservationManage.git
 1. 마이페이지 : https://github.com/byounghoonmoon/MyPage.git
 1. 게이트웨이 : https://github.com/byounghoonmoon/gateway.git

3조 건강검진 예약 서비스 CNA개발 실습을 위한 프로젝트

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

기능적 요구사항
1. 관리자가 병원 정보( 병원이름, 예약일, 가능인원수)를 등록한다.
1. 고객이 건강검진을 예약을 요청한다. (Sync)
1. 고객의 검진 요청에 따라서 해당 병원의 검진가능 인원이 감소한다. (Sync) 
1. 고객의 건강검진 예약 상태가 예약 완료로 변경된다. (Sync) 
1. 고객의 검진 예약 완료에 따라서 예약관리의 해당 내역의 상태가 등록된다.
1. 고객이 건강검진 예약을 취소한다.
1. 고객의 예약 취소에 따라서 병원의 검진가능 인원이 증가한다. (Async)
1. 고객의 예약 취소에 따라서 예약관리의 해당 내역의 상태가 예약 취소로 변경된다.
1. 관리자가 병원 정보를 삭제한다.
1. 관리자의 병원 정보 삭제에 따라서 해당 병원에 예약한 예약자의 상태를 변경한다.
1. 관리자의 병원 정보 삭제에 따라서 예약관리의 해당 내역의 상태가 예약 강제 취소로 변경된다.
1. 사용자가 건강검진 예약내역 상태를 조회한다.

비기능적 요구사항
1. 트랜잭션
    1. 고객의 예약에 따라서 해당 날짜 / 병원의 검진가능 인원이 감소한다. > Sync
    1. 고객의 취소에 따라서 해당 날짜 / 병원의 검진가능 인원이 증가한다. > Async
1. 장애격리
    1. 예약 관리 서비스에 장애가 발생하더라도 검진 예약은 정상적으로 처리 가능하다.  > Async (event-driven)
    1. 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
1. 성능
    1. 고객은 본인의 예약 상태 및 이력 정보를 확인할 수 있다. > CQRS


##시나리오 테스트결과

| 기능 | 이벤트 Payload |
|---|:---:|
| 관관리자가 병원 정보( 병원이름, 예약일, 가능인원수)를 등록한다. | 제일 아래와 같은 형태로 kafka 메시지를 찍어준다. |
| 고객이 건강검진을 예약을 요청한다. (Sync)</bt>해당 병원의 검진가능 인원이 감소한다. (Sync)</br>예약 완료로 변경된다. (Sync)</br> 예약관리의 해당 내역의 상태가 등록된다. | 제일 아래와 같은 형태로 kafka 메시지를 찍어준다. |
| 고객이 건강검진 예약을 취소한다.</br>취소 시, 병원의 검진가능 인원이 증가한다. (Async)</br>예약관리의 해당 내역의 상태가 예약 취소로 변경된다. | 제일 아래와 같은 형태로 kafka 메시지를 찍어준다. |
| 관리자가 병원 정보를 삭제한다.</br>해당 병원에 예약한 예약자의 상태를 예약 강제 취소 변경한다.</br>예약관리의 해당 내역의 상태가 예약 강제 취소로 변경된다. | 제일 아래와 같은 형태로 kafka 메시지를 찍어준다. | 
| 건강검진 예약내역 상태를 조회한다.| 제일 아래와 같은 형태로 kafka 메시지를 찍어준다. |
| 사용자가 콘서트 예약내역 상태를 조회한다. | [{"id":1,"bookingId":6659,"concertId":1,"userId":1,"status":"BookingRequested"},</br> {"id":2,"bookingId":6660,"concertId":3,"userId":1,"status":"PaymentCanceled"}] |

# 분석/설계

## Event Storming 결과

![eventstorming](https://user-images.githubusercontent.com/67453893/91806738-06514f00-ec67-11ea-9226-a5cd28ec1163.png)

```
# 도메인 서열
- Core : screening
- Supporting : Hospital
- General : Reservation
```


## 헥사고날 아키텍처 다이어그램 도출

* CQRS 를 위한 Mypage 서비스만 DB를 구분하여 적용
    
![kafka](https://user-images.githubusercontent.com/67453893/91807070-1cf7a600-ec67-11ea-9e5e-f085f5904d5b.png)


# 구현

## DDD 의 적용

분석/설계 단계에서 도출된 MSA는 총 5개로 아래와 같다.
* MyPage 는 CQRS 를 위한 서비스

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|:---:|:---:|
| Screening | 콘서트 관리 | 8081 | http://localhost:8081/screenings | http://ScreeningManage:8080/screenings |
| Hospital | 예약 관리 | 8082 | http://localhost:8082/hospitals | http://HospitalManage:8080/hospitals |
| Reservation | 결제 관리 | 8083 | http://localhost:8083/reservations | http://ReservationManage:8080/reservations |
| MyPage | 사용자 관리 | 8084 | http://localhost:8084/myPages | http://MyPage:8080/myPages |



## Gateway 적용

```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: ScreeningManage
          uri: http://ScreeningManage:8080
          predicates:
            - Path=/screenings/**
        - id: HospitalManage
          uri: http://HospitalManage:8080
          predicates:
            - Path=/hospitals/** 
        - id: ReservationManage
          uri: http://ReservationManage:8080
          predicates:
            - Path=/reservations/** 
        - id: MyPage
          uri: http://MyPage:8080
          predicates:
            - Path= /myPages/**
```


## 폴리글랏 퍼시스턴스

CQRS 를 위한 Mypage 서비스만 DB를 구분하여 적용함. 인메모리 DB인 hsqldb 사용.

```
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

예약 > 결제 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리

- FeignClient 서비스 구현

```
# PaymentService.java

@FeignClient(name="payment", url="${feign.payment.url}", fallback = PaymentServiceFallback.class)
public interface PaymentService {
    @PostMapping(path="/payments")
    public void requestPayment(Payment payment);
}
```


- 동기식 호출

```
# Booking.java

@PostPersist
public void onPostPersist(){
    BookingRequested bookingRequested = new BookingRequested();
    BeanUtils.copyProperties(this, bookingRequested);
    bookingRequested.setStatus(BookingStatus.BookingRequested.name());
    bookingRequested.publishAfterCommit();

    Payment payment = new Payment();
    payment.setBookingId(this.id);

    Application.applicationContext.getBean(PaymentService.class).requestPayment(payment);
}
```


- Fallback 서비스 구현

```
# PaymentServiceFallback.java

@Component
public class PaymentServiceFallback implements PaymentService {

	@Override
	public void enroll(Payment payment) {
		System.out.println("Circuit breaker has been opened. Fallback returned instead.");
	}

}
```


## 비동기식 호출 과 Fallback 처리

- 비동기식 발신 구현

```
# Booking.java

@PostUpdate
public void onPostUpdate(){
    if (BookingStatus.BookingApproved.name().equals(this.getStatus())) {
        BookingApproved bookingApproved = new BookingApproved();
        BeanUtils.copyProperties(this, bookingApproved);
        bookingApproved.publishAfterCommit();
    }
}
```

- 비동기식 수신 구현

```
# PolicyHandler.java

@StreamListener(KafkaProcessor.INPUT)
public void paymentApproved(@Payload PaymentApproved paymentApproved){
    if(paymentApproved.isMe()){
	bookingRepository.findById(paymentApproved.getBookingId())
	    .ifPresent(
			booking -> {
				booking.setStatus(BookingStatus.BookingApproved.name());;
				bookingRepository.save(booking);
		    }
	    )
	;
    }
}
```


# 운영

## CI/CD 설정

- CodeBuild 기반으로 파이프라인 구성

<img src="https://user-images.githubusercontent.com/62231786/85087121-927ed900-b217-11ea-8f57-bbd4efc25997.JPG"/>

- Git Hook 연경

<img src="https://user-images.githubusercontent.com/62231786/85087123-93b00600-b217-11ea-90b3-4de01d03583a.JPG" />


## 서킷 브레이킹 / 장애격리

* Spring FeignClient + Hystrix 구현
* Booking 서비스 내 PaymentService FeignClient에 적용

- Hystrix 설정

```
# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610
```

- 서비스 지연 설정

```
//circuit test
try {
    Thread.currentThread().sleep((long) (400 + Math.random() * 220));
} catch (InterruptedException e) { }
```

- 부하 테스트 수행

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8082/bookings/ POST {"concertId":1, "userId":1, "qty":5}'
```

- 부하 테스트 결과

```
2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[container-0-C-1][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m preSend on channel 'event-in', message: GenericMessage [payload=byte[142], headers={kafka_offset=4013, scst_nativeHeadersPresent=true, kafka_consumer=org.apache.kafka.clients.consumer.KafkaConsumer@5775c5aa, deliveryAttempt=1, kafka_timestampType=CREATE_TIME, kafka_receivedMessageKey=null, kafka_receivedPartitionId=0, contentType=application/json, kafka_receivedTopic=sts, kafka_receivedTimestamp=1592499287785}]
Circuit breaker has been opened. Fallback returned instead.
Circuit breaker has been opened. Fallback returned instead.
[2m2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[o-8082-exec-153][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m postSend (sent=true) on channel 'event-out', message: GenericMessage [payload=byte[142], headers={contentType=application/json, id=cbdf4d07-547d-5dbe-80a1-659a0e00b607, timestamp=1592499291969}]
[2m2020-06-19 01:54:52.576[0;39m [32mDEBUG[0;39m [35m6600[0;39m [2m---[0;39m [2m[o-8082-exec-166][0;39m [36mo.s.c.s.m.DirectWithAttributesChannel   [0;39m [2m:[0;39m postSend (sent=true) on channel 'event-out', message: GenericMessage [payload=byte[142], headers={contentType=application/json, id=3a646994-497f-717a-cb13-443133007248, timestamp=1592499291969}]
```

```
defaulting to time-based testing: 30 seconds

{	"transactions":			         447,
	"availability":			      100.00,
	"elapsed_time":			       29.92,
	"data_transferred":		        0.10,
	"response_time":		        6.11,
	"transaction_rate":		       14.94,
	"throughput":			        0.00,
	"concurrency":			       91.21,
	"successful_transactions":	         447,
	"failed_transactions":		           0,
	"longest_transaction":		       17.07,
	"shortest_transaction":		        0.00
}
```


### 오토스케일 아웃

- 현재 상태 확인

```
  Namespace                   Name                        CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                        ------------  ----------  ---------------  -------------  ---
  default                     booking-7764c68d4b-27jrp    0 (0%)        0 (0%)      0 (0%)           0 (0%)         9h
  default                     concert-6b54bd565c-grvnf    0 (0%)        0 (0%)      0 (0%)           0 (0%)         15h
  default                     payment-684fd5785c-67ptn    0 (0%)        0 (0%)      0 (0%)           0 (0%)         9h
  kafka                       my-kafka-0                  0 (0%)        0 (0%)      0 (0%)           0 (0%)         144m
  kafka                       my-kafka-zookeeper-2        0 (0%)        0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                 aws-node-5rd64              10m (0%)      0 (0%)      0 (0%)           0 (0%)         16h
  kube-system                 coredns-555b56bfbb-mj2pk    100m (5%)     0 (0%)      70Mi (2%)        170Mi (6%)     16h
  kube-system                 kube-proxy-bmc2z            100m (5%)     0 (0%)      0 (0%)           0 (0%)         16h
```

- 오토스케일 설정
```
kubectl autoscale deploy booking --min=1 --max=3 --cpu-percent=1
```

- 부하 수행

```
siege -c100 -t60S -r10 --content-type "application/json" 'http://aa8dc72fe9cbb4ba0ba62c5720326102-1685876144.ap-northeast-2.elb.amazonaws.com:8080/bookings/ POST {"concertId":1, "userId":1, "qty":5}' -v
```

- 모니터링

```
kubectl get deploy booking -w
```

- 스케일 아웃 확인

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
booking   1/1     1            1           5h9m
```

```
defaulting to time-based testing: 60 seconds

{	"transactions":			        6316,
	"availability":			      100.00,
	"elapsed_time":			       60.00,
	"data_transferred":		        1.43,
	"response_time":		        0.94,
	"transaction_rate":		      105.27,
	"throughput":			        0.02,
	"concurrency":			       99.46,
	"successful_transactions":	        6316,
	"failed_transactions":		           0,
	"longest_transaction":		        6.22,
	"shortest_transaction":		        0.05
}
```


## 무정지 재배포

<img src="https://user-images.githubusercontent.com/62231786/85087355-497b5480-b218-11ea-804c-6e884f60c92f.JPG" />
