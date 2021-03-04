![TheStorage](https://user-images.githubusercontent.com/78134087/110009013-c2e3b480-7d5f-11eb-869e-56b89433f159.JPG)


# 서비스 시나리오

## 기능적 요구사항
1. 고객이 필요한 규격과 대여기간을 선택하고 창고대여를 신청한다
2. 비어있는 창고를 확인하여 창고할당을 한다
3. 할당되면 고객에게 대여신청 확정정보를 전달한다.
4. 고객은 창고대여 신청을 취소할 수 있다.
5. 대여신청이 취소되면 할당된 창고도 취소된다



## 비기능적 요구사항
1. 트랜잭션
- 창고 할당요청을 하지 않으면 창고 대여 신청을 할 수 없다 Sync 호출
2. 장애격리
- 대여 신청은 창고 할당 기능이 동작하지 않더라도, 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
- 창고 할당 요청이 과중되면 대여신청을 잠시동안 받지 않고 잠시 후에 하도록 유도한다 Circuit breaker, fallback
3. 성능
- 고객은 창고신청상태를 확인할 수 있어야 한다. CQRS, Event driven



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
![04](https://user-images.githubusercontent.com/78134087/110018426-9d0fdd00-7d6a-11eb-8596-99d020006329.JPG)



### 어그리게잇으로 묶기
![05어그리](https://user-images.githubusercontent.com/78134087/110018446-a1d49100-7d6a-11eb-8b3f-4bceeee475e5.JPG)


창고신청, 창고관리, 창고할당 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어줌 



### 바운디드 컨텍스트로 묶기

![06바운디드](https://user-images.githubusercontent.com/78134087/110019083-5f5f8400-7d6b-11eb-8081-bcdd87f54f5d.JPG)



### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![07](https://user-images.githubusercontent.com/78134087/110018510-b1ec7080-7d6a-11eb-9fd9-b8f84a5e412a.JPG)



### 폴리시의 이동

![08](https://user-images.githubusercontent.com/78134087/110019298-a3eb1f80-7d6b-11eb-8497-6bf1ca982efc.JPG)



### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![09](https://user-images.githubusercontent.com/78134087/110018547-bdd83280-7d6a-11eb-958f-4df9706800bc.JPG)





### 완성된 모형

![10완성](https://user-images.githubusercontent.com/78134087/110018567-c2045000-7d6a-11eb-95cb-975b3e120848.jpg)




### 기능적 요구사항 검증
![11기능검증](https://user-images.githubusercontent.com/78134087/110018593-c6c90400-7d6a-11eb-9c30-431a3e3a970f.JPG)


#### 신청case (red)
1. 고객이 창고를 신청한다.(ok)
2. 창고관리 시스템이 창고 할당을 요청한다.(ok)
3. 창고 자동 할당이 완료된다.(ok)
4. 신청상태 및 할당상태를 갱신한다.(ok)


#### 취소case (blue)
1. 고객이 신청한 창고대여를 취소요청한다.(ok)
2. 창고관리 시스템이 창고 할당 취소를 요청한다.(ok)
3. 창고 할당이 취소된다.(ok)
4. 취소상태로 갱신한다.(ok)



### 비기능 요구사항 검증
![13비기능](https://user-images.githubusercontent.com/78134087/110018777-05f75500-7d6b-11eb-836b-6f89175fd4fb.JPG)



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

![15헥사](https://user-images.githubusercontent.com/78134087/110018791-0a237280-7d6b-11eb-8c9f-b585f7e34fee.JPG)






# 구현:

cd storagecall

mvn spring-boot:run  


cd storagemanage

mvn spring-boot:run


cd storageassign

mvn spring-boot:run 


cd gateway

mvn spring-boot:run  



## DDD 의 적용
msaez.io 를 통해 구현한 Aggregate 단위의 Entity 선언 후, 구현을 진행하였다.

Entity Pattern 과 Repository Pattern 을 적용하기 위해 Spring Data REST 의 RestRepository 를 적용하였다.

![ddd적용](https://user-images.githubusercontent.com/78134087/109931222-eaa92d00-7d0b-11eb-977c-234cc0030dab.JPG)

![ddd2](https://user-images.githubusercontent.com/78134087/110033551-e537fb00-7d7c-11eb-9902-adec71234e8c.JPG)



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

http localhost:8088/storagecalls
-> gateway 를 호출하나 8081 로 호출됨






![gw-test](https://user-images.githubusercontent.com/78134087/109935954-41653580-7d11-11eb-8fe5-258679ee6c11.JPG)



## 동기식 호출과 Fallback 처리

창고신청(storagecall)->창고관리(storagemanage) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하였습니다.
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 


```
# external > StoragemanageService.java

package storagerental.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name="storagemanage", url="http://localhost:8082", fallback = StoragemanageServiceFallback.class)
//@FeignClient(name="storagemanage", url="http://drivermanage:8080", fallback = StoragemanageServiceFallback.class)
public interface StoragemanageService {

    @RequestMapping(method= RequestMethod.POST, path="/storagemanages")
    public void reqDriverassign(@RequestBody Storagemanage storagemanage);

}

```

StoragemanageService 인터페이스를 구현한 StoragemanageServiceFallback

```
# external > StoragemanageServiceFallback.java


package storagerental.external;

import org.springframework.stereotype.Component;

@Component
public class StoragemanageServiceFallback implements StoragemanageService {
		
	@Override
	public void reqStorageassign(Storagemanage storagemanage) {
		System.out.println("Circuit breaker has been opened. Fallback returned instead. " + storagemanage.getId());
	}

}

```
창고신청을 하면 창고관리에 창고 할당 요청을 동기적으로 진행

동기식 호출 적용으로 창고 관리 시스템이 정상적이지 않으면, 창고대여 신청할 수 없음

![reqres](https://user-images.githubusercontent.com/78134087/110003958-426e8500-7d5a-11eb-9071-06c3c72868aa.JPG)


```
창고관리 (storagemanage) 재기동 후 호출

http localhost:8082/storagemanages 

http localhost:8081/storagecalls tel="01011115678" location="파주" status="호출" cost=50000

```
![reqres2](https://user-images.githubusercontent.com/78134087/110005260-a2b1f680-7d5b-11eb-8daa-b35831931f09.JPG)



## 비동기식 호출 / 장애격리  / 성능

창고 관리 (storage manage) 이후 창고 할당(storage assign) 은 비동기식 처리이므로, 
창고 신청 (storage call) 의 서비스 호출에는 영향이 없도록 구성

Storagecall 내에서 서비스 Pub 구현
```
package storagecall;

import javax.persistence.PostPersist;
import javax.persistence.PreRemove;
import javax.persistence.Table;

import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Storagecall_table")
public class Storagecall {

    @PostPersist
    public void onPostPersist(){

    	System.out.println("tel " + getTel());
        System.out.println("startdate " + getStartdate());
        System.out.println("status " + getStatus());
        System.out.println("cost " + getCost());

    	if(getTel() != null)
		{
    		System.out.println("SEND###############################" + getId());
			Storagemanage storagemanage = new Storagemanage();

```

Storagemanage 내 Policy Handler 에서 Sub 구현

```
package storagemanage;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import storagemanage.config.kafka.KafkaProcessor;

@Service
public class StoragemanagePolicyHandler {
	@Autowired
	StoragemanageRepository storagemanageRepository;

	@StreamListener(KafkaProcessor.INPUT)
	public void onStringEventListener(@Payload String eventString) {

	}

	@StreamListener(KafkaProcessor.INPUT)
	public void whenever호출취소됨_(@Payload StoragecallCancelled storagecallCancelled) {
		System.out.println("##### EVT TYPE[StoragecallCancelled]  : " + storagecallCancelled.getEventType());
		if (storagecallCancelled.isMe()) {
			System.out.println("##### listener  : " + storagecallCancelled.toJson());

```


<창고 신청 Storage call>


창고할당(storage assign) 서비스 중단 후 창고신청 실행해도 이상 없이 동작

![pubsub1](https://user-images.githubusercontent.com/78134087/110007201-ca09c300-7d5d-11eb-9e12-1ac9cfab8b85.JPG)



# 클라우드 배포/운영 파이프라인

1. git에서 소스 가져오기


  git clone http://github.com/younis21c/storage-rental


2. Build
```
cd gateway
mvn clean && mvn package
cd ..
cd storagecall
mvn clean && mvn package
cd ..
cd storagemanage
mvn clean && mvn package
cd ..
cd storageassign
mvn clean && mvn package
```

3. Dockerlizing, ACR(Azure Container Registry에 Docker Image Push
```
cd gateway
az acr build --registry skuser17 --image skuser17.azurecr.io/gateway:v1 .
 
cd ..
cd storagecall
az acr build --registry skuser17 --image skuser17.azurecr.io/storagecall:v1 .
cd ..
cd storagemanage
az acr build --registry skuser17 --image skuser17.azurecr.io/storagemanage:v1 .
cd ..
cd storageassign
az acr build --registry skuser17 --image skuser17.azurecr.io/storageassign:v1 .
```

ACR에 정상적으로 push되었음을 확인


![ACR](https://user-images.githubusercontent.com/78134087/109987442-f87da300-7d49-11eb-8345-5bba6f3fcca2.JPG)


4. Kubernetes에서 Deploy
```
cd gateway/kubernetes
kubectl apply -f deployment.yml --namespace=skuser17ns
kubectl apply -f service.yaml --namespace=skuser17ns

cd ../../
cd storagecall/kubernetes
kubectl apply -f deployment.yml --namespace=skuser17ns
kubectl apply -f service.yaml --namespace=skuser17ns

cd ../../
cd storagemanage/kubernetes
kubectl apply -f deployment.yml --namespace=skuser17ns
kubectl apply -f service.yaml --namespace=skuser17ns

cd ../../
cd storageassign/kubernetes
kubectl apply -f deployment.yml --namespace=skuser17ns
kubectl apply -f service.yaml --namespace=skuser17ns
```

Kubectl 결과 확인


![배포확인](https://user-images.githubusercontent.com/78134087/109987592-1ba85280-7d4a-11eb-8e39-ed2af488f677.JPG)





## 동기식 호출 / 서킷 브레이킹 / 장애격리
서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현
시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

Hystrix 를 설정: 요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

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

![캡처1](https://user-images.githubusercontent.com/78134087/110044222-1d930580-7d8c-11eb-8161-272390761097.JPG)


kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash


siege 종료: Ctrl + C -> exit
 
부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
동시사용자 100명 60초 동안 실시


siege -c100 -t60S -r10 -v --content-type "application/json" 'storagecall:8080/storagecalls/ POST {"tel": "01023456789", "cost":30000}'

부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 pay에서 처리되면서 다시 order를 받기 시작

![캡처1](https://user-images.githubusercontent.com/78134087/110044269-300d3f00-7d8c-11eb-8ea3-c60ca8587f42.JPG)


report

![캡처2](https://user-images.githubusercontent.com/78134087/110044285-356a8980-7d8c-11eb-850b-05d823e76f5f.JPG)


CB 잘 적용됨을 확인


### 오토스케일 아웃



## 무정지 재배포

- deployment.yml에 readiness 옵션을 추가 

![무정지배포1](https://user-images.githubusercontent.com/78134087/110012102-8023db80-7d63-11eb-9d15-c366ec82d181.JPG)



- seige 실행


- Availability: 100.00 % 확인


## Config Map










