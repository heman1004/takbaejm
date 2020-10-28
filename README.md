# 택배시스템
본 프로그램은 택배시스템이다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [택배시스템](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [Configmap](#Configmap)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 고객이 택배물품을 접수한다.
1. 고객이 결제한다
1. 결제가 되면 접수내역이 택배기사에게 전달된다
1. 택배기사가 확인하여 택배배달 출발한다
1. 고객이 접수를 취소할 수 있다
1. 접수가 취소되면 결제가 취소된다
1. 고객이 현재 택배상태를 조회한다
1. 택배배달이 완료되면 포인트가 적립된다
1. 고객으로 부터 반납을 처리한다.
1. 반납 처리가 되면 배송 상태를 변경한다.
1. 반납 처리 시 포인트는 반환된다.
1. 반납 배송을 취소하면 반납도 취소가 된다.
1. 반납 시 입력한 반납 이유를 조회한다.


비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 접수건은 아예 거래가 성립되지 않아야 한다  Sync 호출
    1. 반납이 되지 않는 배송건은 상태가 반납으로 변경되지 않아야 한다. Sync 호출
1. 장애격리
    1. 포인트 기능이 수행되지 않더라도 접수는 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
    1. 배송 기능이 수행되지 않더라도 반납은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
    1. 배송 기능이 과중되면 고객을 잠시동안 받지 않고 반납을 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 배달상태를 시스템에서 확인할 수 있다  CQRS
    1. 배달이 완료되면 자동으로 포인트가 적립된다  Event driven
    1. 고객이 반납상태를 시스템에서 확인할 수 있다  CQRS
    1. 반납 배달이 완료되면 자동으로 포인트가 재 적립된다  Event driven
    



# 체크포인트

- 분석 설계
  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가? 
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://www.msaez.io/#/storming/lDc01C8D74YedeH39AqSkSqPoWI3/share/c84563094cddebd6bea82140e3fce52f/-MKI68IU8OMs49uIWAYw

### 이벤트 도출
![image](https://user-images.githubusercontent.com/69283661/97399113-c38eb900-192f-11eb-9ab7-c7e4f1228bb4.png)

```
- 도메인 서열 분리 
    - Core Domain:  request,  delivery : 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 request의 경우 1주일 1회 미만, delivery의 경우 1개월 1회 미만
    - Supporting Domain: delivery Dash Board, point, return 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
    - General Domain:   Payment(결제) : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)
```

## 헥사고날 아키텍처 다이어그램 도출
![image](https://user-images.githubusercontent.com/69283661/97400098-df935a00-1931-11eb-89d0-3313b7e51d59.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd gateway
mvn spring-boot:run

cd delivery
mvn spring-boot:run 

cd deliveryboard
mvn spring-boot:run  

cd payment
mvn spring-boot:run 

cd point
mvn spring-boot:run 

cd request
mvn spring-boot:run 

cd return
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 return 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하였고, 모든 구현에 있어서 영문으로 사용하여 별다른  오류없이 구현하였다.

```
package takbaejm;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Return_table")
public class Return {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long memberId;
    private Long qty;
    private String reason;
    private Long requestId;

    @PostPersist
    public void onPostPersist(){
        Returned returned = new Returned();
        BeanUtils.copyProperties(this, returned);
        returned.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        takbaejm.external.Delivery delivery = new takbaejm.external.Delivery();

        delivery.setRequestId(this.getRequestId());
        delivery.setStatus("Returned");

        // mappings goes here
        ReturnApplication.applicationContext.getBean(takbaejm.external.DeliveryService.class)
            .delivery(delivery);


    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getMemberId() {
        return memberId;
    }

    public void setMemberId(Long memberId) {
        this.memberId = memberId;
    }
    public Long getQty() {
        return qty;
    }

    public void setQty(Long qty) {
        this.qty = qty;
    }
    public String getReason() {
        return reason;
    }

    public void setReason(String reason) {
        this.reason = reason;
    }
    public Long getRequestId() {
        return requestId;
    }

    public void setRequestId(Long requestId) {
        this.requestId = requestId;
    }




}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package takbaejm;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface DeliveryRepository extends PagingAndSortingRepository<Delivery, Long>{


}
```

- 적용 후 REST API 의 테스트
```
# request 서비스의 접수처리 후 delivery
http localhost:8081/requests memberId=40 qty=40
http http://localhost:8083/deliveries
```
![image](https://user-images.githubusercontent.com/69283661/97407339-e32cde00-193d-11eb-9030-e3732e165eb1.png)
```

# return 서비스의 반납처리
http http://localhost:8086/returns requestId=4 reason="test return"
```
![image](https://user-images.githubusercontent.com/69283661/97408949-33a53b00-1940-11eb-87ec-a9027dc804e3.png)


## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 반납(return)->배송(delivery) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient를 이용하여 호출하도록 한다. 

- 배송서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (delivery) DeliveryService.java

package takbaejm.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="delivery", url="${api.url.delivery}")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void delivery(@RequestBody Delivery delivery);

}
```

- 반납을 받은 직후(@PostPersist) 배송을 요청하도록 처리
```
# Return.java (Entity)
@PostPersist
    public void onPostPersist(){
        Returned returned = new Returned();
        BeanUtils.copyProperties(this, returned);
        returned.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        takbaejm.external.Delivery delivery = new takbaejm.external.Delivery();

        delivery.setRequestId(this.getRequestId());
        delivery.setStatus("Returned");

        // mappings goes here
        ReturnApplication.applicationContext.getBean(takbaejm.external.DeliveryService.class)
            .delivery(delivery);


    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 배송 시스템이 장애가 나면 반납도 못받는다는 것을 확인:


```
# 배송(delivery) 서비스를 잠시 내려놓음 (ctrl+c)

# 반납처리
http http://localhost:8086/returns requestId=3 reason="test sync"
```
![image](https://user-images.githubusercontent.com/69283661/97410707-c2b35280-1942-11eb-84cb-6888bb030d01.png)
```
# 배송 서비스 재기동
cd delivery
mvn spring-boot:run

#반납처리
http http://localhost:8086/returns requestId=3 reason="test sync"
```
![image](https://user-images.githubusercontent.com/69283661/97411146-3c4b4080-1943-11eb-950c-a21a416c2cf6.png)

```
- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)

```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트
반납이 왼료되어진 후에 포인트시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 포인트 시스템의 처리를 위하여 반납이 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 반납에 기록을 남긴 후에 곧바로 반납 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
- 포인트 서비스에서는 배달완료 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.

```
package takbaejm;

import takbaejm.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.Iterator;
import java.util.Optional;

@Service
public class PolicyHandler{

    @Autowired
    PointRepository pointRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }
    .....
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReturned_GetPointPol(@Payload Returned returned){

        if(returned.isMe()){
            int flag=0;
            Iterator<Point> iterator = pointRepository.findAll().iterator();
            while(iterator.hasNext()){
                Point pointTmp = iterator.next();
                if(pointTmp.getMemberId() == returned.getMemberId()){
                    Optional<Point> PointOptional = pointRepository.findById(pointTmp.getId());
                    Point point = PointOptional.get();
                    point.setPoint(point.getPoint()-100);
                    pointRepository.save(point);
                    flag=1;
                }
            }

            if (flag==0 ){
                Point point = new Point();
                point.setMemberId(returned.getMemberId());
                point.setPoint((long)-100);
                pointRepository.save(point);
            }

            System.out.println("##### listener GetPointPol : " + returned.toJson());
        }
    }

}

```

```
포인트 시스템은 반납시스템과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 포인트시스템이 유지보수로 인해 잠시 내려간 상태라도 하는데 문제가 없다
포인트 서비스를 내린 상태에서 반납 처리가 완료된다.
포인트 서비스를 다시 활성화 시키면 이벤트 수신 후 포인트 정보가 생성된다.
```
![image](https://user-images.githubusercontent.com/69283661/97414662-9bab4f80-1947-11eb-8e42-03ab768c5a66.png)

# CQRS 적용
접수된 택배 현황을 view로 구현함.

![image](https://user-images.githubusercontent.com/69283661/97416059-4b34f180-1949-11eb-9c57-0747de9a7a8c.png)


# gateway 적용
-소스적용
![image](https://user-images.githubusercontent.com/69283661/97416345-a7981100-1949-11eb-91ac-80c8e9fc69a0.png)

-호출확인
![image](https://user-images.githubusercontent.com/68535067/97380473-74825d00-190a-11eb-9afd-8def71948ee9.png)

# 운영

## CI 설정
![image](https://user-images.githubusercontent.com/68535067/97244041-f1de9c80-183a-11eb-82c2-a42a1f4c9957.png)

![image](https://user-images.githubusercontent.com/69283675/97380059-96c7ab00-1909-11eb-901f-b5aaf6c51772.png)

## CD 설정
![image](https://user-images.githubusercontent.com/68535067/97244102-163a7900-183b-11eb-8a98-bdfdcce1aafd.png)

![image](https://user-images.githubusercontent.com/69283675/97380141-c4145900-1909-11eb-8bd9-17de9d8081a3.png)


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 deployment.yml, service.yml 에 포함되었다.


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 접수(request)-->결제(payment) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 680 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 680

```

- 피호출 서비스(결제:payment) 의 임의 부하 처리 - 400 밀리에서 증감 300 밀리 정도 왔다갔다 하게
```
# (payment) 결제이력.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();

        try {
            Thread.sleep((long) (400 + Math.random() * 300));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 1명
- 10초 동안 실시
![image](https://user-images.githubusercontent.com/68535067/97244569-66660b00-183c-11eb-9b7e-cda86b0d59ac.png)

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 81.82% 가 성공하였고, 18.18%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
```
![image](https://user-images.githubusercontent.com/68535067/97245477-75e65380-183e-11eb-9557-d247d53be45f.png)

- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://request:8080/requests POST {"memberId": "100", "qty":5}'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![image](https://user-images.githubusercontent.com/68535067/97246490-d1b1dc00-1840-11eb-8ef2-ec4d6610f3e2.png)

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
![image](https://user-images.githubusercontent.com/68535067/97247046-2b66d600-1842-11eb-8648-1715eedf0d58.png)

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://request:8080/requests POST {"memberId": "100", "qty":5}'

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
![image](https://user-images.githubusercontent.com/68535067/97380841-36396d80-190b-11eb-8aa9-0d1e4efbcd11.png)


배포기간중 Availability 가 평소 100%에서 60% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
![image](https://user-images.githubusercontent.com/68535067/97247604-569df500-1843-11eb-9c50-a405a5f6c9c4.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.



## Configmap
- configmap.yaml 파일설정

![image](https://user-images.githubusercontent.com/68535067/97378420-a8a74f00-1905-11eb-94e6-f76f9e9fda40.png)

- deployment.yaml파일 설정
![image](https://user-images.githubusercontent.com/68535067/97378494-c8d70e00-1905-11eb-8e8e-208ade4772fe.png)

- application.yaml 파일 설정
![image](https://user-images.githubusercontent.com/68535067/97378559-edcb8100-1905-11eb-804d-e8c61afed969.png)

- paymentService 파일 설정
![image](https://user-images.githubusercontent.com/68535067/97378637-28351e00-1906-11eb-8774-98d9dd0eee0e.png)


- 80포트로 설정하여 테스트
![image](https://user-images.githubusercontent.com/68535067/97378359-8ad9ea00-1905-11eb-952d-641a380a02c9.png)

## Livness구현
- POINT의 depolyment.yaml 소스설정
- http get방식에서 tcp방식으로 변경, 서비스포트 8080이 아닌 고의로 8081로 포트  변경하여  
![image](https://user-images.githubusercontent.com/68535067/97388731-08105980-191c-11eb-9c50-c7edace5cff5.png)
-restart확인
![image](https://user-images.githubusercontent.com/68535067/97383715-93382200-1911-11eb-8db0-f1ad417ced41.png)

- describe 확인
![image](https://user-images.githubusercontent.com/68535067/97383827-d4c8cd00-1911-11eb-8cc6-03ac56635131.png)

- 원복후 정상 확인
![image](https://user-images.githubusercontent.com/68535067/97384522-3178b780-1913-11eb-8625-a2bb3ae953b7.png)








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


