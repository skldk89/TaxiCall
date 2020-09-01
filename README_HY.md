# 4th-team3-health-screening

# screeningReservation (건강검진 예약 서비스)1

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

## 기능적 요구사항
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

## 비기능적 요구사항
1. 트랜잭션
    1. 고객의 예약에 따라서 해당 날짜 / 병원의 검진가능 인원이 감소한다. > Sync
    1. 고객의 취소에 따라서 해당 날짜 / 병원의 검진가능 인원이 증가한다. > Async
1. 장애격리
    1. 예약 관리 서비스에 장애가 발생하더라도 검진 예약은 정상적으로 처리 가능하다.  > Async (event-driven)
    1. 서킷 브레이킹 프레임워크 > istio-injection + DestinationRule
1. 성능
    1. 고객은 본인의 예약 상태 및 이력 정보를 확인할 수 있다. > CQRS


## 시나리오 테스트결과

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
|---|:---:|:---:|---|---|
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

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, 
pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
- CodeBuild 기반으로 CI/CD 파이프라인 구성
MSA 서비스별 CodeBuild 프로젝트 생성하여  CI/CD 파이프라인 구성

<img src="https://user-images.githubusercontent.com/67447253/91837032-92bf3a00-ec86-11ea-96e6-263a590cd849.JPG"/>

- Git Hook 연결
연결한 Github의 소스 변경 발생 시 자동으로 빌드 및 배포 되도록 Git Hook 연결 설정

<img src="https://user-images.githubusercontent.com/67447253/91837300-0103fc80-ec87-11ea-9698-fc1afb52893c.JPG" />


## 동기식 호출 / 서킷 브레이킹 / 장애격리

### 방식1) 서킷 브레이킹 프레임워크의 선택: istio-injection + DestinationRule

* istio-injection 적용 (기 적용완료)
```
kubectl label namespace mybnb istio-injection=enabled
```
* 예약, 결제 서비스 모두 아무런 변경 없음

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시
```
$ siege -v -c100 -t60S -r10 --content-type "application/json" 'http://booking:8080/bookings POST {"roomId":1, "name":"호텔", "price":1000, "address":"서울", "host":"Superman", "guest":"배트맨", "usedate":"20201230"}'

HTTP/1.1 201     2.19 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     3.91 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     2.22 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     2.30 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     2.23 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     2.06 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     0.11 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     2.02 secs:     321 bytes ==> POST http://booking:8080/bookings

```
* 서킷 브레이킹을 위한 DestinationRule 적용
```
cd mybnb/yaml
kubectl apply -f dr-pay.yaml

HTTP/1.1 500     0.28 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.35 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.28 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.29 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.41 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.15 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.24 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.41 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.21 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.33 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.43 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.34 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.32 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.33 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.36 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.33 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.34 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.43 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.46 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.38 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.33 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.39 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.49 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.21 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.32 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.42 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.38 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     0.28 secs:     250 bytes ==> POST http://booking:8080/bookings

Transactions:                   1986 hits
Availability:                  63.88 %
Elapsed time:                  47.75 secs
Data transferred:               0.88 MB
Response time:                  2.39 secs
Transaction rate:              41.59 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   99.57
Successful transactions:        1986
Failed transactions:            1123
Longest transaction:            7.53
Shortest transaction:           0.05
```

* DestinationRule 적용되어 서킷 브레이킹 동작 확인 (kiali 화면)
![슬라이드4](https://user-images.githubusercontent.com/61722732/89362532-167a1b00-d709-11ea-8981-07bf788080b5.JPG)


* 다시 부하 발생하여 DestinationRule 적용 제거하여 정상 처리 확인
```
cd mybnb/yaml
kubectl delete -f dr-pay.yaml
```

### 방식2) 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약-->결제 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

* Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
- kubectl apply -f booking_cb.yaml 실행
```
# application.yml

feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

* 피호출 서비스(결제) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
- kubectl apply -f pay_cb.yaml 
```
# Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -v -c100 -t60S -r10 --content-type "application/json" 'http://booking:8080/bookings POST {"roomId":1, "name":"호텔", "price":1000, "address":"서울", "host":"Superman", "guest":"배트맨", "usedate":"20201230"}'

** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     4.75 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.65 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.80 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     0.49 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.40 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     0.48 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.51 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.29 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     4.46 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.84 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     4.15 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.32 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     4.43 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     4.31 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.32 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.29 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.38 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.22 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.43 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.31 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.17 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     4.45 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     4.18 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.18 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.10 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     3.54 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.59 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.48 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     4.14 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     3.48 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.74 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.47 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     4.27 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     8.48 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     3.98 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     3.13 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.21 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     0.56 secs:     321 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     3.12 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.14 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.04 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.16 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.89 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.09 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     3.19 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.77 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     4.15 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     3.66 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     0.65 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     4.33 secs:     321 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     1.74 secs:     323 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     0.53 secs:     323 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     1.87 secs:     323 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     1.16 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.16 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     2.85 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     1.28 secs:     323 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     1.23 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.22 secs:     250 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 201     1.86 secs:     323 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 201     1.87 secs:     323 bytes ==> POST http://booking:8080/bookings

HTTP/1.1 500     1.21 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.22 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.24 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.17 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.23 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.12 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.08 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.16 secs:     250 bytes ==> POST http://booking:8080/bookings
HTTP/1.1 500     1.16 secs:     250 bytes ==> POST http://booking:8080/bookings

Lifting the server siege...siege aborted due to excessive socket failure; you
can change the failure threshold in $HOME/.siegerc

Transactions:                    796 hits
Availability:                  42.98 %
Elapsed time:                  59.06 secs
Data transferred:               0.50 MB
Response time:                  7.32 secs
Transaction rate:              13.48 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   98.61
Successful transactions:         796
Failed transactions:            1056
Longest transaction:           10.77
Shortest transaction:           0.08
```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 42.985% 가 성공하였고, 67%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

* (istio injection 적용한 경우) istio injection 적용 해제
```
kubectl label namespace mybnb istio-injection=disabled --overwrite

kubectl apply -f booking.yaml
kubectl apply -f pay.yaml
```

* (Spring FeignClient + Hystrix 적용한 경우) 위에서 설정된 CB는 제거해야함.
```
kubectl apply -f booking.yaml
kubectl apply -f pay.yaml
```

- 결제서비스 배포시 resource 설정 적용되어 있음
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

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 3개까지 늘려준다:
```
kubectl autoscale deploy pay -n mybnb --min=1 --max=3 --cpu-percent=15

# 적용 내용
NAME                           READY   STATUS    RESTARTS   AGE
pod/alarm-bc469c66b-nn7r9      2/2     Running   0          25m
pod/booking-6f85b67876-rhwl2   2/2     Running   0          25m
pod/gateway-7bd59945-g9hdq     2/2     Running   0          25m
pod/html-78f648d5b-zhv2b       2/2     Running   0          25m
pod/mypage-7587b7598b-l86jl    2/2     Running   0          25m
pod/pay-755d679cbf-7l7dq       2/2     Running   0          8m58s
pod/room-6c8cff5b96-78chb      2/2     Running   0          25m
pod/siege                      2/2     Running   0          25m

NAME              TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE
service/alarm     ClusterIP      10.100.36.234    <none>                                                                        8080/TCP         25m
service/booking   ClusterIP      10.100.19.222    <none>                                                                        8080/TCP         25m
service/gateway   LoadBalancer   10.100.195.171   a59f2304940914b7ca3875b12e62e321-738700923.ap-northeast-2.elb.amazonaws.com   8080:31754/TCP   25m
service/html      ClusterIP      10.100.19.81     <none>                                                                        8080/TCP         25m
service/mypage    ClusterIP      10.100.134.37    <none>                                                                        8080/TCP         25m
service/pay       ClusterIP      10.100.97.43     <none>                                                                        8080/TCP         8m58s
service/room      ClusterIP      10.100.78.233    <none>                                                                        8080/TCP         25m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/alarm     1/1     1            1           25m
deployment.apps/booking   1/1     1            1           25m
deployment.apps/gateway   1/1     1            1           25m
deployment.apps/html      1/1     1            1           25m
deployment.apps/mypage    1/1     1            1           25m
deployment.apps/pay       1/1     1            1           8m58s
deployment.apps/room      1/1     1            1           25m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/alarm-bc469c66b      1         1         1       25m
replicaset.apps/booking-6f85b67876   1         1         1       25m
replicaset.apps/gateway-7bd59945     1         1         1       25m
replicaset.apps/html-78f648d5b       1         1         1       25m
replicaset.apps/mypage-7587b7598b    1         1         1       25m
replicaset.apps/pay-755d679cbf       1         1         1       8m58s
replicaset.apps/room-6c8cff5b96      1         1         1       25m

NAME                                      REFERENCE        TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/pay   Deployment/pay   <unknown>/15%   1         3         0          7s
```

- CB 에서 했던 방식대로 워크로드를 3분 동안 걸어준다.
```
$ siege -v -c100 -t180S -r10 --content-type "application/json" 'http://booking:8080/bookings POST {"roomId":1, "name":"호텔", "price":1000, "address":"서울", "host":"Superman", "guest":"배트맨", "usedate":"20201230"}'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -n mybnb -w 
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
pay    1/1     1            1           4m21s
pay    1/2     1            1           4m28s
pay    1/2     1            1           4m28s
pay    1/2     1            1           4m28s
pay    1/2     2            1           4m28s
pay    1/3     2            1           4m43s
pay    1/3     2            1           4m43s
pay    1/3     2            1           4m43s
pay    1/3     3            1           4m43s
pay    2/3     3            2           5m53s
pay    3/3     3            3           5m59s
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
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

* configmap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mybnb-config
  namespace: mybnb
data:
  api.url.payment: http://pay:8080
  alarm.prefix: Hello
```
* booking.yaml (configmap 사용)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: booking
  namespace: mybnb
  labels:
    app: booking
spec:
  replicas: 1
  selector:
    matchLabels:
      app: booking
  template:
    metadata:
      labels:
        app: booking
    spec:
      containers:
        - name: booking
          image: 496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/mybnb-booking:latest
          ports:
            - containerPort: 8080
          env:
            - name: api.url.payment
              valueFrom:
                configMapKeyRef:
                  name: mybnb-config
                  key: api.url.payment
          resources:
```
* kubectl describe pod/booking-588cb89c6b-gmw8h -n mybnb
```
Containers:
  booking:
    Container ID:   docker://0b90fe0d06629fc367fa83273abecba2724958a0b838c058553d193a86c3e0fe
    Image:          496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/mybnb-booking:latest
    Image ID:       docker-pullable://496278789073.dkr.ecr.ap-northeast-2.amazonaws.com/mybnb-booking@sha256:59abe6ec02e165fda1c8e3dbf3e8bcedf7fb5edc53fcffca5f708a70969452f3
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 03 Aug 2020 16:48:56 +0900
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:  500m
    Requests:
      cpu:      200m
    Liveness:   http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:  http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:
      api.url.payment:  <set to the key 'api.url.payment' of config map 'mybnb-config'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-mrczz (ro)
```

# 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## 마케팅팀의 추가
    - KPI: 신규 고객의 유입률 증대와 기존 고객의 충성도 향상
    - 구현계획 마이크로 서비스: 기존 customer 마이크로 서비스를 인수하며, 고객에 음식 및 맛집 추천 서비스 등을 제공할 예정

## 이벤트 스토밍 
    ![image](https://user-images.githubusercontent.com/487999/79685356-2b729180-8273-11ea-9361-a434065f2249.png)


## 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/487999/79685243-1d704100-8272-11ea-8ef6-f4869c509996.png)

## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 마이크로서비스의 경우 API 변화나 Retire 시에 app(주문) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

                --> 

        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제2(pay);

    }
```

예) Retire 시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        /**
        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

        **/
    }
```

