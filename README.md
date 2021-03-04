
# 서비스 시나리오

## 기능적 요구사항
1. 고객이 필요한 규격과 대여기간을 선택하고 창고대여를 신청한다
2. 비어있는 창고를 확인하여 창고할당을 한다
3. 할당되면 고객에게 대여신청 확정정보를 전달한다.

4. 고객은 창고대여 신청을 취소할 수 있다.
5. 대여신청이 취소되면 할당된 창고도 취소된다
6. 고객이 창고대여신청 현황을 확인할 수 있다


## 비기능적 요구사항
1. 트랜잭션
- 창고 할당요청을 하지 않으면 창고 대여 신청을 할 수 없다 Sync 호출
2. 장애격리
- 대여 신청은 창고 할당 기능이 동작하지 않더라도, 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
- 창고 할당 요청이 과중되면 대여신청을 잠시동안 받지 않고 잠시 후에 하도록 유도한다 Circuit breaker, fallback
3. 성능
- 고객은 창고신청상태를 확인 할 수 있어야 한다. CQRS, Event driven



# 체크포인트

1. Saga
1. CQRS
1. Correlation
1. Req/Resp
1. Gateway
1. Deploy/ Pipeline
1. Circuit Breaker
1. Autoscale (HPA)
1. Zero-downtime deploy (Readiness Probe)
1. Config Map/ Persistence Volume
1. Polyglot
1. Self-healing (Liveness Probe)


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![조직구조](https://user-images.githubusercontent.com/78134019/109453964-977a7480-7a96-11eb-83cb-5445c363a9e8.jpg)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/vR9WqhlS6chqQTFb8irp9osAHFv2/mine/16d6bba6ce50b1ba59ad87967b6df21a


### 이벤트 도출
(![02](https://user-images.githubusercontent.com/78134087/109808762-dc570480-7c6a-11eb-94bf-acbbcfc8cb29.JPG)

### 부적격 이벤트 탈락
(![03](https://user-images.githubusercontent.com/78134087/109808788-e416a900-7c6a-11eb-9f07-4140f51f50f6.JPG)

과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
- 필요규격 선택됨, 대여기간 선택됨 :  UI의 이벤트이지 업무적인 의미의 이벤트가 아니라서 제외
- 빈창고 확인됨 :  계획된 사업 범위 및 프로젝트에서 벗어서난다고 판단하여 제외

	

### 액터, 커맨드 부착하여 읽기 좋게
![04액터](https://user-images.githubusercontent.com/78134087/109808951-16280b00-7c6b-11eb-8726-067ced0f05df.JPG)


### 어그리게잇으로 묶기
![05](https://user-images.githubusercontent.com/78134087/109808969-1b855580-7c6b-11eb-93fe-55a35a56b667.JPG)

창고신청, 창고관리, 창고할당 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌 


### 바운디드 컨텍스트로 묶기
![06바운디드](https://user-images.githubusercontent.com/78134087/109809053-335cd980-7c6b-11eb-8643-c908d29c6506.JPG)



### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![07](https://user-images.githubusercontent.com/78134087/109809089-3d7ed800-7c6b-11eb-95ae-b8d6785f3364.JPG)


### 폴리시의 이동

![08이동](https://user-images.githubusercontent.com/78134087/109809105-42dc2280-7c6b-11eb-9c1f-5821808e8587.JPG)


### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![09](https://user-images.githubusercontent.com/78134087/109809115-47084000-7c6b-11eb-9ed2-b73f100e53d2.JPG)




### 완성된 모형

![10완성](https://user-images.githubusercontent.com/78134087/109900159-3fcc4b00-7cda-11eb-8dfb-73dae0c302c9.JPG)



### 기능적 요구사항 검증
![11기능](https://user-images.githubusercontent.com/78134087/109900164-422ea500-7cda-11eb-8152-0eb228741831.JPG)


#### 신청case (red)
1. 고객이 창고를 신청한다.(ok)
2. 창고관리 시스템이 창고 할당을 요청한다.(ok)
3. 창고 자동 할당이 완료된다.(ok)
4. 신청상태 및 할당상태를 갱신한다.(ok)
5. 고객이 상태변경을 확인한다.(ok)



#### 취소case (blue)
1. 고객이 신청한 창고대여를 취소요청한다.(ok)
2. 창고관리 시스템이 창고 할당 취소를 요청한다.(ok)
3. 창고 할당이 취소된다.(ok)
4. 취소상태로 갱신한다.(ok)
5. 고객이 상태변경을 확인한다.(ok)


#### 확인case (black)
1. 고객이 신청 진행내역을 볼 수 있어야 한다. (ok) 



### 비기능 요구사항 검증
![12비기능](https://user-images.githubusercontent.com/78134087/109900192-4ce93a00-7cda-11eb-9bca-9fc019aed69c.JPG)


마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
 
1) 창고 할당요청이 완료되지 않은 신청요청 완료처리는 최종 할당이 되지 않는 경우 
  무한정 대기 등 대고객 서비스 및 신뢰도에 치명적 문제점이 있어 ACID 트랜잭션 적용. 
  신청요청 시 창고 할당요청에 대해서는 Request-Response 방식 처리 
2) 신청요청 완료시 할당확인 및 결과 전송: storagemanage service 에서
  storageassign 마이크로서비스로 창고할당 요청이 전달되는 과정에 있어서 
  해당 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함. 
3) 나머지 모든 inter-microservice 트랜잭션
  신청상태 등 이벤트에 대해 데이터 일관성 시점이 critical 하지 않은 경우가 대부분이라
  Eventual Consistency 를 기본으로 채택함. 




## 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![hsqldb](https://user-images.githubusercontent.com/78134087/109930212-bb45f080-7d0a-11eb-8420-e8a8db855068.JPG)






# 구현:

cd storagecall

mvn spring-boot:run  


cd storagemanage

mvn spring-boot:run


cd storageassign

mvn spring-boot:run 


cd gateway

mvn spring-boot:run  


cd customer

python policy-handler.py

## DDD 의 적용
msaez.io 를 통해 구현한 Aggregate 단위의 Entity 선언 후, 구현을 진행하였다.

Entity Pattern 과 Repository Pattern 을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

![ddd적용](https://user-images.githubusercontent.com/78134087/109931222-eaa92d00-7d0b-11eb-977c-234cc0030dab.JPG)




## 폴리글랏 퍼시스턴스

위치 : /storagerental>storagemanage>pom.xml

![hsqldb](https://user-images.githubusercontent.com/78134087/109928785-27bff000-7d09-11eb-9fdd-01da78503b3f.JPG)


## 마이크로 서비스 호출 흐름

### 신청시나리오 서비스 호출처리

http localhost:8081/storagecalls tel="01012345678" location="마포" status=호출 cost=25000

![1호출](https://user-images.githubusercontent.com/78134087/109931142-d107e580-7d0b-11eb-9d16-8259d8d384a6.JPG)
![2호출확정-manage](https://user-images.githubusercontent.com/78134087/109931153-d402d600-7d0b-11eb-991b-4c4ae84133d1.JPG)

### 취소시나리오 서비스 호출처리

http delete localhost:8081/storagecalls/1

![2-1호출취소](https://user-images.githubusercontent.com/78134087/109931164-d9602080-7d0b-11eb-925c-e6697a59ece5.JPG)
![2-2호출취소확정-mana](https://user-images.githubusercontent.com/78134087/109931171-db29e400-7d0b-11eb-8151-a47dffa50750.JPG)


## Gateway 적용

gateway > applitcation.yml 설정

![gw](https://user-images.githubusercontent.com/78134087/109932531-a9b21800-7d0d-11eb-993e-27659398806d.JPG)


gateway 테스트
```
http localhost:8088/storagecalls
-> gateway 를 호출하나 8081 로 호출됨

![gw-test](https://user-images.githubusercontent.com/78134087/109935954-41653580-7d11-11eb-8fe5-258679ee6c11.JPG)




```
# (app) 결제이력Service.java











