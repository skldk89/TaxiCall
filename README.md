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
| 관관리자가 병원 정보( 병원이름, 예약일, 가능인원수)를 등록한다. |![image](https://user-images.githubusercontent.com/25805562/91837451-3577b880-ec87-11ea-88b1-dc4e9d74790d.png)|
| 고객이 건강검진을 예약을 요청한다. (Sync)</bt>해당 병원의 검진가능 인원이 감소한다. (Sync)</br>예약 완료로 변경된다. (Sync)</br> 예약관리의 해당 내역의 상태가 등록된다. |![image](https://user-images.githubusercontent.com/25805562/91837806-7a035400-ec87-11ea-8966-09403bd5e7eb.png)|
| 고객이 건강검진 예약을 취소한다.</br>취소 시, 병원의 검진가능 인원이 증가한다. (Async)</br>예약관리의 해당 내역의 상태가 예약 취소로 변경된다. | ![image](https://user-images.githubusercontent.com/25805562/91837990-c2227680-ec87-11ea-9fb1-530410922532.png) |
| 관리자가 병원 정보를 삭제한다.</br>해당 병원에 예약한 예약자의 상태를 예약 강제 취소 변경한다.</br>예약관리의 해당 내역의 상태가 예약 강제 취소로 변경된다. | ![image](https://user-images.githubusercontent.com/25805562/91838119-f007bb00-ec87-11ea-9edd-38d9963f9ee0.png) | 
| 건강검진 예약내역 상태를 조회한다.| ![image](https://user-images.githubusercontent.com/25805562/91838415-6ad0d600-ec88-11ea-9df8-1c6895fe6d75.png) |

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

분석단계에서의 조건 중 하나로 고객검진요청(Screening)->병원정보관리(Hospital) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
고객검진요청 > 병원정보관리 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리
- FeignClient 서비스 구현

```
# HospitalService.java

@FeignClient(name="HospitalManage", url="${api.hospital.url}")
public interface HospitalService {

    @RequestMapping(method= RequestMethod.PUT, value="/hospitals/{hospitalId}", consumes = "application/json")
    public void screeningRequest(@PathVariable("hospitalId") Long hospitalId, @RequestBody Hospital hospital);

}
```

- 고객검진요청을 받은 직후(@PostPersist) 병원벙보를 요청하도록 처리
```
# Screening.java (Entity)

    @PostPersist
    public void onPostPersist(){;

        // 검진 요청함 ( Req / Res : 동기 방식 호출)
        local.external.Hospital hospital = new local.external.Hospital();
        hospital.setHospitalId(getHospitalId());
        ScreeningManageApplication.applicationContext.getBean(local.external.HospitalService.class)
            .screeningRequest(hospital.getHospitalId(),hospital);

        Requested requested = new Requested();
        BeanUtils.copyProperties(this, requested);
        requested.publishAfterCommit();
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 병원관리 시스템이 장애가 나면 검진요청 못받는다는 것을 확인


```
#병원정보관리(Hospital) 서비스를 잠시 내려놓음 (ctrl+c)

#고객검진요청
http POST localhost:8081/screenings hospitalId=1 hospitalNm="Samsung" custNm="MOON" chkDate="20200910" status="REQUESTED"   #Fail
http POST localhost:8081/screenings hospitalId=2 hospitalNm="SK" custNm="JUNG" chkDate="20200911" status="REQUESTED"   #Fail

#병원정보관리 재기동
cd HospitalManage
mvn spring-boot:run

#고객검진요청 처리
http POST localhost:8081/screenings hospitalId=1 hospitalNm="Samsung" custNm="MOON" chkDate="20200910" status="REQUESTED"   #Success
http POST localhost:8081/screenings hospitalId=2 hospitalNm="SK" custNm="JUNG" chkDate="20200911" status="REQUESTED"   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, Fallback 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


고객검진취소가 이루어진 후에 예약시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하였다.
 
- 이를 위하여 고객검진신청이력에 기록을 남긴 후에 곧바로 검진취소신청 되었다는 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package local;

@Entity
@Table(name="Screening_table")
public class Screening {

 ...
    @PostUpdate
    public void onPostUpdate(){

        System.out.println("#### onPostUpdate :" + this.toString());

        if("CANCELED".equals(this.getStatus())) {
            Canceled canceled = new Canceled();
            BeanUtils.copyProperties(this, canceled);
            canceled.publishAfterCommit();
        }
	}
}
```
- 예약관리 서비스에서는 검진취소신청 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
package local;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverCanceled_ReservationCancel(@Payload Canceled canceled){

        if(canceled.isMe()){
            //  검진예약 취소로 인한 취소
            Reservation temp = reservationRepository.findByScreeningId(canceled.getId());
            temp.setStatus("CANCELED");
            reservationRepository.save(temp);

        }
    }

}

```

예약관리 시스템은 병원/검진신청과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 예약관리시스템이 유지보수로 인해 잠시 내려간 상태라도 예약신청을 받는데 문제가 없다.

```
#예약관리 서비스 (Reservation) 를 잠시 내려놓음 (ctrl+c)

#검진신청취소처리
http PUT localhost:8081/screenings hospitalId=1 hospitalNm="Samsung" chkDate= "20200907" custNm= "Moon44" status= "Canceled"   #Success
http PUT localhost:8081/screenings hospitalId=2 hospitalNm="SK" chkDate= "20200908" custNm= "YOU" status= "Canceled"   #Success

#예약관리상태 확인
http localhost:8083/reservations     # 예약상태 안바뀜 확인

#예약관리 서비스 기동
cd ReservationManage
mvn spring-boot:run

#예약관리상태 확인
http localhost:8083/reservations     # 예약상태가 "취소됨"으로 확인
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

### 서킷 브레이킹 프레임워크의 선택 > istio-injection + DestinationRule

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
![Kaili_DR RUlE적용](https://user-images.githubusercontent.com/67453893/91850567-b7bca880-ec98-11ea-8f9b-59b4223fb046.png)

* 다시 부하 발생하여 DestinationRule 적용 제거하여 정상 처리 확인
```
cd mybnb/yaml
kubectl delete -f dr-pay.yaml
```

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

