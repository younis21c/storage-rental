
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

http localhost:8088/storagecalls
-> gateway 를 호출하나 8081 로 호출됨






![gw-test](https://user-images.githubusercontent.com/78134087/109935954-41653580-7d11-11eb-8fe5-258679ee6c11.JPG)



## 동기식 호출과 Fallback 처리

호출(taxicall)->택시관리(taximanage) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하였습니다.
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

로컬 테스트를 위한 파일은 다음과 같이 구현 하였으며,
```
# external > 택시관리Service.java


package taxiguider.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

//@FeignClient(name="taximanage", url="http://localhost:8082")
@FeignClient(name="taximanage", url="http://localhost:8082", fallback = 택시관리ServiceFallback.class)
public interface 택시관리Service {

    @RequestMapping(method= RequestMethod.POST, path="/택시관리s")
    public void 택시할당요청(@RequestBody 택시관리 택시관리);

}

```

클라우드 배포시 구현은 영문 클래스 해당 URL 호출은 다음과 같이 구현 하였습니다.

```
# external > TaximanageService.java


package taxiguider.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

//@FeignClient(name="taximanage", url="http://localhost:8082")
//@FeignClient(name="taximanage", url="http://localhost:8082", fallback = TaximanageServiceFallback.class)
@FeignClient(name="taximanage", url="http://taximanage:8080", fallback = TaximanageServiceFallback.class)
public interface TaximanageService {

    @RequestMapping(method= RequestMethod.POST, path="/taximanages")
    public void requestTaxiAssign(@RequestBody Taximanage txMange);

}

```

다음은 택시관리Service 인터페이스를 구현한 택시관리ServiceFallback 클래스이며, 클라우드 배포용 영문과 따로 구현 되었습니다.

```
# external > 택시관리ServiceFallback.java


package taxiguider.external;

import org.springframework.stereotype.Component;

@Component
public class 택시관리ServiceFallback implements 택시관리Service {
	 
	
	@Override
	public void 택시할당요청(택시관리 택시관리) {
		// TODO Auto-generated method stub
		System.out.println("Circuit breaker has been opened. Fallback returned instead. " + 택시관리.getId());
	}

}

```

![동기식](https://user-images.githubusercontent.com/78134019/109463569-97837000-7aa8-11eb-83c4-6f6eff1594aa.jpg)


![2021-03-04_004922](https://user-images.githubusercontent.com/7607807/109832226-80e54080-7c83-11eb-9526-e1820a60c938.png)


- 로컬 택시 할당요청

택시호출을 하면 택시관리에 택시 할당 요청을 다음과 같이 동기적으로 진행 합니다.
```
# 택시호출.java

 @PostPersist
    public void onPostPersist(){    	
    	System.out.println("휴대폰번호 " + get휴대폰번호());
        System.out.println("호출위치 " + get호출위치());
        System.out.println("호출상태 " + get호출상태());
        System.out.println("예상요금 " + get예상요금());
        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.   	
    	if(get휴대폰번호() != null)
		{
    		System.out.println("SEND###############################" + getId());
			택시관리 택시관리 = new 택시관리();
	        
			택시관리.setOrderId(String.valueOf(getId()));
	        택시관리.set고객휴대폰번호(get휴대폰번호());
	        if(get호출위치()!=null) 
	        	택시관리.set호출위치(get호출위치());
	        if(get호출상태()!=null) 
	        	택시관리.set호출상태(get호출상태());
	        if(get예상요금()!=null) 
	        	택시관리.set예상요금(get예상요금());
	        
	        // mappings goes here
	        TaxicallApplication.applicationContext.getBean(택시관리Service.class).택시할당요청(택시관리);
		}
```

- 클라우드 배포시 택시 할당요청(영문)

택시호출을 하면 택시관리에 택시 할당 요청을 다음과 같이 동기적으로 진행 합니다.
```
# 택시호출.java

@PostPersist
public void onPostPersist(){
	System.out.println("휴대폰번호 " + getTel());
	System.out.println("호출위치 " + getLocation());
	System.out.println("호출상태 " + getStatus());
	System.out.println("예상요금 " + getCost());
	//Following code causes dependency to external APIs
	// it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.   	
	if(getTel() != null)
	{
		System.out.println("SEND###############################" + getId());
		Taximanage txMgr = new Taximanage();
		txMgr.setId(getId());
		txMgr.setOrderId(String.valueOf(getId()));
		txMgr.setTel(getTel());
		if(getLocation()!=null) 
			txMgr.setLocation(getLocation());
		if(getStatus()!=null) 
			txMgr.setStatus(getStatus());
		if(getCost()!=null) 
			txMgr.setCost(getCost());
		
		// mappings goes here
		TaxicallApplication.applicationContext.getBean(TaximanageService.class)
			.requestTaxiAssign(txMgr);;
	}

}
```

![동기식2](https://user-images.githubusercontent.com/78134019/109463985-47f17400-7aa9-11eb-8603-c1f83e17951d.jpg)


![2021-03-04_005205](https://user-images.githubusercontent.com/7607807/109832649-e6393180-7c83-11eb-822f-bd41957e7a65.png)

- 동기식 호출 적용으로 택시 관리 시스템이 정상적이지 않으면 , 택시콜도 접수될 수 없음을 다음과 같이 확인 할 수 있습니다.

```
- 택시 관리 시스템 down 후 taxicall 호출 
#taxicall

C:\Users\Administrator>http localhost:8081/택시호출s 휴대폰번호="01012345678" 호출상태="호출"
```

![택시관리죽으면택시콜놉](https://user-images.githubusercontent.com/78134019/109464780-905d6180-7aaa-11eb-9c90-e7d1326deea1.jpg)


```
# 택시 관리 (taximanage) 재기동 후 호출

http localhost:8081/택시호출s 휴대폰번호="01012345678" 호출상태="호출"
```

![택시관리재시작](https://user-images.githubusercontent.com/78134019/109464984-e5997300-7aaa-11eb-9363-b7bfe15de105.jpg)

-fallback 

![fallback캡쳐](https://user-images.githubusercontent.com/78134019/109480299-b5f46600-7abe-11eb-906e-9e1e6da245b2.png)


## 비동기식 호출 / 장애격리  / 성능

택시 관리 (Taxi manage) 이후 택시 할당(Taxi Assign) 은 비동기식 처리이므로 , 
택시 호출(Taxi call) 의 서비스 호출에는 영향이 없도록 구성 합니다.
 
고객이 택시 호출(Taxicall) 후 상태가 [호출]->[호출중] 로 변경되고 할당이 완료되면 [호출확정] 로 변경이 되지만 , 
택시 할당(TaxiAssign)이 정상적이지 않으므로 [호출중]로 남게 됩니다. 

<고객 택시 호출 Taxi call>
![비동기_호출2](https://user-images.githubusercontent.com/78134019/109468467-f4365900-7aaf-11eb-877a-049637b5ee6a.png)

<택시 할당이 정상적이지 않아 호출중으로 남아있음>
![택시호출_택시할당없이_조회](https://user-images.githubusercontent.com/78134019/109471791-99ebc700-7ab4-11eb-924f-03715de42eba.png)



## 정보 조회 / View 조회
고객은 택시가 할당되는 동안의 내용을 조회 할 수 있습니다.

![고객View](https://user-images.githubusercontent.com/78134019/109483385-80ea1280-7ac2-11eb-9419-bf3ff5a0dbbc.png)


## 소스 패키징

- 클라우드 배포를 위해서 다음과 같이 패키징 작업을 하였습니다.
```
cd gateway
mvn clean && mvn package
cd ..
cd taxicall
mvn clean && mvn package
cd ..
cd taximanage
mvn clean && mvn package
cd ..
cd taxiassign
mvn clean && mvn package
cd ..
```
	
<taxicall>
	
![mvn_taxicall](https://user-images.githubusercontent.com/78134019/109744165-31682b80-7c15-11eb-9d94-7bc23efca6b6.png)

<taximanage>
	
![mvn_taximanage](https://user-images.githubusercontent.com/78134019/109744195-3b8a2a00-7c15-11eb-9554-1c3ba088af52.png)

<taxiassign>
	
![mvn_taxiassign](https://user-images.githubusercontent.com/78134019/109744226-46dd5580-7c15-11eb-8b47-5100ed01e3ae.png)


# 클라우드 배포/운영 파이프라인

- 애저 클라우드에 배포하기 위해서 다음과 같이 주요 정보를 설정 하였습니다.

```
리소스 그룹명 : skccteam03-rsrcgrp
클러스터 명 : skccteam03-aks
레지스트리 명 : skccteam03
```

- az login
우선 애저에 로그인 합니다.
```
{
    "cloudName": "AzureCloud",
    "homeTenantId": "6011e3f8-2818-42ea-9a63-66e6acc13e33",
    "id": "718b6bd0-fb75-4ec9-9f6e-08ae501f92ca",
    "isDefault": true,
    "managedByTenants": [],
    "name": "2",
    "state": "Enabled",
    "tenantId": "6011e3f8-2818-42ea-9a63-66e6acc13e33",
    "user": {
      "name": "skTeam03@gkn2021hotmail.onmicrosoft.com",
      "type": "user"
    }
  }
```

- 토큰 가져오기
```
az aks get-credentials --resource-group skccteam03-rsrcgrp --name skccteam03-aks
```

- aks에 acr 붙이기
```
az aks update -n skccteam03-aks -g skccteam03-rsrcgrp --attach-acr skccteam03
```

![aks붙이기](https://user-images.githubusercontent.com/78134019/109653395-540e2c00-7ba4-11eb-97dd-2dcfdf5dc539.jpg)

- 네임스페이스 만들기

```
kubectl create ns team03
kubectl get ns
```
![image](https://user-images.githubusercontent.com/78134019/109776836-5cb73e80-7c46-11eb-9562-d462525d6dab.png)

* 도커 이미지 만들고 레지스트리에 등록하기
```
cd taxicall_eng
az acr build --registry skccteam03 --image skccteam03.azurecr.io/taxicalleng:v1 .
az acr build --registry skccteam03 --image skccteam03.azurecr.io/taxicalleng:v2 .
cd ..
cd taximanage_eng
az acr build --registry skccteam03 --image skccteam03.azurecr.io/taximanageeng:v1 .
cd ..
cd taxiassign_eng
az acr build --registry skccteam03 --image skccteam03.azurecr.io/taxiassigneng:v1 .
cd ..
cd gateway_eng
az acr build --registry skccteam03 --image skccteam03.azurecr.io/gatewayeng:v1 .
cd ..
cd customer_py
az acr build --registry skccteam03 --image skccteam03.azurecr.io/customer-policy-handler:v1 .
```

![docker_gateway](https://user-images.githubusercontent.com/78134019/109777813-76a55100-7c47-11eb-8d8d-59eaabefab54.png)

![docker_taxiassign](https://user-images.githubusercontent.com/78134019/109777820-77d67e00-7c47-11eb-9d77-85403dcf2da4.png)

![docker_taxicall](https://user-images.githubusercontent.com/78134019/109777826-786f1480-7c47-11eb-9992-41f75907d16f.png)

![docker_taximanage](https://user-images.githubusercontent.com/78134019/109777827-786f1480-7c47-11eb-9c9b-d3357eda0bd5.png)

![docker_customer](https://user-images.githubusercontent.com/78134019/109777829-7907ab00-7c47-11eb-936f-723396cb272a.png)


-각 마이크로 서비스를 yml 파일을 사용하여 배포 합니다.


![deployment_yml](https://user-images.githubusercontent.com/78134019/109652001-9171ba00-7ba2-11eb-8c29-7128ceb4ec97.jpg)

- deployment.yml로 서비스 배포
```
cd ../../
cd customer_py/kubernetes
kubectl apply -f deployment.yml --namespace=team03
kubectl apply -f service.yaml --namespace=team03
cd ../../
cd taxicall_eng/kubernetes
kubectl apply -f deployment.yml --namespace=team03
kubectl apply -f service.yaml --namespace=team03

cd ../../
cd taximanage_eng/kubernetes
kubectl apply -f deployment.yml --namespace=team03
kubectl apply -f service.yaml --namespace=team03

cd ../../
cd taxiassign_eng/kubernetes
kubectl apply -f deployment.yml --namespace=team03
kubectl apply -f service.yaml --namespace=team03

cd ../../
cd gateway_eng/kubernetes
kubectl apply -f deployment.yml --namespace=team03
kubectl apply -f service.yaml --namespace=team03
```
<Deploy cutomer>
	
![deploy_customer](https://user-images.githubusercontent.com/78134019/109744443-a471a200-7c15-11eb-94c9-a0c0a7999d04.png)

<Deploy gateway>
	
![deploy_gateway](https://user-images.githubusercontent.com/78134019/109744457-acc9dd00-7c15-11eb-8502-ff65e779e9d2.png)

<Deploy taxiassign>
	
![deploy_taxiassign](https://user-images.githubusercontent.com/78134019/109744471-b3585480-7c15-11eb-8d68-bba9c3d8ce01.png)

<Deploy taxicall>
	
![deploy_taxicall](https://user-images.githubusercontent.com/78134019/109744487-bb17f900-7c15-11eb-8bd0-ff0a9fc9b2e3.png)


![deploy_taximanage](https://user-images.githubusercontent.com/78134019/109744591-e69ae380-7c15-11eb-834a-44befae55092.png)



- 서비스확인
```
kubectl get all -n team03
```
![image](https://user-images.githubusercontent.com/78134019/109777026-9be58f80-7c46-11eb-9eac-a55ebcf91989.png)



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현하였습니다.

- Hystrix 를 설정:  

요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true

# To set thread isolation to SEMAPHORE
#hystrix:
#  command:
#    default:
#      execution:
#        isolation:
#          strategy: SEMAPHORE

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```
![hystrix](https://user-images.githubusercontent.com/78134019/109652345-0218d680-7ba3-11eb-847b-708ba071c119.jpg)


부하테스트


* Siege 리소스 생성

```
kubectl run siege --image=apexacme/siege-nginx -n team03
```

* 실행

```
kubectl exec -it pod/siege-5459b87f86-hlfm9 -c siege -n team03 -- /bin/bash
```

*부하 실행

```
siege -c200 -t60S -r10 -v --content-type "application/json" 'http://20.194.36.201:8080/taxicalls POST {"tel": "0101231234"}'
```

- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 택시호출(taxicall) 서비스에서 처리되면서 
다시 taxicall에서 서비스를 받기 시작 합니다

![secs1](https://user-images.githubusercontent.com/78134019/109786899-01d71480-7c51-11eb-9e6c-0a819e85b020.png)


- report

![secs2](https://user-images.githubusercontent.com/78134019/109786922-07345f00-7c51-11eb-900a-315f7d0d6484.png)





### 오토스케일 아웃



```
# autocale out 설정
 deployment.yml 설정
```


![auto1](https://user-images.githubusercontent.com/78134019/109794479-3ea70980-7c59-11eb-8d32-fbc039106c8c.jpg)


```
kubectl autoscale deploy taxicall --min=1 --max=10 --cpu-percent=15 -n team03
```


```
root@labs--279084598:/home/project# kubectl exec -it pod/siege-5459b87f86-hlfm9 -c siege -n team03 -- /bin/bash
root@siege-5459b87f86-hlfm9:/# siege -c100 -t120S -r10 -v --content-type "application/json" 'http://20.194.36.201:8080/taxicalls POST {"tel": "0101231234"}'
```
![auto4](https://user-images.githubusercontent.com/78134019/109794919-b70dca80-7c59-11eb-9710-8ff6b4dd5f54.jpg)



- 오토스케일링에 대한 모니터링:
```
kubectl get deploy taxicall -w -n team03
```
![auto_final](https://user-images.githubusercontent.com/78134019/109796515-98a8ce80-7c5b-11eb-9512-a0a927217a38.jpg)



## 무정지 재배포

- deployment.yml에 readiness 옵션을 추가 


![무정지 배포1](https://user-images.githubusercontent.com/78134019/109809110-45d71300-7c6b-11eb-955c-9b8a3b3db698.png)


- seige 실행
```
siege -c100 -t120S -r10 -v --content-type "application/json" 'http://20.194.36.201:8080/taxicalls POST {"tel": "0101231234"}'
```


- Availability: 100.00 % 확인


![무정지 배포2](https://user-images.githubusercontent.com/78134019/109810318-bd597200-7c6c-11eb-88e4-197386b1e338.png)


![무정지 배포3](https://user-images.githubusercontent.com/78134019/109810688-2fca5200-7c6d-11eb-9c67-d252d703064a.png)



## Config Map

- apllication.yml 설정

* default 프로파일

![configmap1](https://user-images.githubusercontent.com/31096538/109798636-5df46580-7c5e-11eb-982d-16482f98b13f.JPG)

* docker 프로파일

![configmap2](https://user-images.githubusercontent.com/31096538/109798699-6e0c4500-7c5e-11eb-9d0d-47b90d637ae9.JPG)

- Deployment.yml 설정

![configmap3](https://user-images.githubusercontent.com/31096538/109798713-72d0f900-7c5e-11eb-8458-8fb9d6225c49.JPG)

- config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://taxicall:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n team03
```
![configmap4](https://user-images.githubusercontent.com/31096538/109798727-76fd1680-7c5e-11eb-9818-327870ea2e4d.JPG)

- 설정한 url로 주문 호출
```
http 20.194.36.201:8080/taxicalls tel="01012345678" status="call" location="mapo" cost=25000
```

![configmap5](https://user-images.githubusercontent.com/31096538/109798744-7c5a6100-7c5e-11eb-8aaa-03fa8277cee6.JPG)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiurl -n team03
kubectl get pod/taxicall-74f7dbc967-mtbmq -n team03 -o yaml | kubectl replace --force -f-
```
![configmap6](https://user-images.githubusercontent.com/31096538/109798766-811f1500-7c5e-11eb-8008-1b9073cb6722.JPG)

- configmap 삭제된 상태에서 주문 호출   
```
http 20.194.36.201:8080/taxicalls tel="01012345678" status="call" location="mapo" cost=25000
kubectl get all -n team03
```
![configmap7](https://user-images.githubusercontent.com/31096538/109798785-85e3c900-7c5e-11eb-8769-ab416b1e17b2.JPG)


![configmap8](https://user-images.githubusercontent.com/31096538/109798805-8bd9aa00-7c5e-11eb-8d05-1db2457d3611.JPG)


![configmap9](https://user-images.githubusercontent.com/31096538/109798824-9005c780-7c5e-11eb-9d5b-6f14f9b6bba9.JPG)


## Self-healing (Liveness Probe)


- deployment.yml 에 Liveness Probe 옵션 추가
```
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![selfhealing](https://user-images.githubusercontent.com/78134019/109805068-589b1900-7c66-11eb-9565-d44adde4ffc5.jpg)








