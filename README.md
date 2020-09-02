# 예제 - PT플래폼

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - PT서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오


기능적 요구사항
1. 고객이 PT수강신청을 한다.
2. PT수강신청에 대한 접수가 되면, 고객관리 담당자가 강사를 할당한다.
3. 강사가 할당 되면 해당 강사에게 스케줄 확정 요청을 한다.
4. 강사는 스케줄을 확정한다.
5. 강사가 스케줄을 확정하면 PT수강신청이 완료된다.
6. 강사는 수업을 진행한 후 수업결과를 등록한다.
7. 수업결과가 등록되면 고객은 수업결과 확인 알림을 보낸다.
8. 고객은 PT수강신청을 취소할 수 있다.
9. PT수강신청이 취소되면 배정된 강사와 확정된 스케줄은 취소된다. (강사 배정과 스케줄 취소는 Req/Res 동기 처리)
10. 고객관리 담당자는 강사 배정 및 확정 스케줄을 수시로 확인할 수 있다.
11. 고객은 진행한 수업결과를 수시로 확인할 수 있다.


비기능적 요구사항
1. 트랜잭션
    1. PT수강신청 취소는 강사배정 및 확정스케줄 취소가 동시에 이루어지도록 한다.
1. 장애격리
    1. PT수강신청과 취소는 고객관리 담당자와 강사 스케줄 확정, 수업결과 등록 처리와 관계없이 항상 처리 가능하다.
1. 성능
    1. 고객관리 담당자는 강사 배정 현황과 스케줄 현황을 수시로 확인하여 모니터링 한다. (CQRS)
    1. 고객은 수업결과 현황을 수시로 확인할 수 있다. (CQRS)


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


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)



## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/gDWswgJxUeMj58uUHULzZqcusCA3/mine/b0adb9dc3b6b4d604f7e7b7fdd5e69f7/-MG9u1XUmo4gfSX4HJ8_


### 이벤트 도출
![image](https://user-images.githubusercontent.com/19251601/91854433-b2faf300-ec9e-11ea-9234-58ea159c6920.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/19251601/91854295-7af3b000-ec9e-11ea-9de1-7e195e9c15ae.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
    - 중복/불필요, 처리 프로세스에 해당하는 이벤트 제거
    
### 폴리시 부착
![image](https://user-images.githubusercontent.com/19251601/91854584-e8074580-ec9e-11ea-8626-e2d3fa9b360c.png)

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/19251601/91854676-0bca8b80-ec9f-11ea-892c-5136a4d54f2e.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/19251601/91854880-5cda7f80-ec9f-11ea-8f50-efee8df61055.png)

    - PT수강신청의 신청 및 취소신청, 고객관리의 수강확정 및 취소접수, 트레이너의 스켸쥴 및 수업결과는 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/19251601/91854989-81cef280-ec9f-11ea-8bd5-50602fced150.png)

    - 도메인 서열 분리 : PT수강신청 > 고객관리 = 트레이너 로 정의


### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/19251601/91855088-9f03c100-ec9f-11ea-8070-ac620fc71e2a.png)

### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/19251601/91855733-8647db00-eca0-11ea-8ca2-5de23b7b9c3b.png)

    - View Model 추가

### 시나리오 체크(1) : 예약부터 확정 Noti까지

![image](https://user-images.githubusercontent.com/19251601/91857981-990fdf00-eca3-11ea-8e4d-13e67a9fd0e3.png)

    - 고객이 PT수강신청을 한다. (ok)
    - PT수강신청에 대한 접수가 되면, 고객관리 담당자가 강사를 할당한다. (ok)
    - 강사가 할당 되면 해당 강사에게 스케줄 확정 요청을 한다. (ok)
    - 강사는 스케줄을 확정한다. (ok)
    - 강사가 스케줄을 확정하면 PT수강신청이 완료된다.(ok)

### 시나리오 체크(2) : 수업결과 등록과 결과 Noti

![image](https://user-images.githubusercontent.com/19251601/91858085-b04ecc80-eca3-11ea-8f84-740a5cee7842.png)

    - 강사는 수업을 진행한 후 수업결과를 등록한다. (ok)
    - 수업결과가 등록되면 고객은 수업결과 확인 알림을 받는다.(ok)

### 시나리오 체크(3)

![image](https://user-images.githubusercontent.com/19251601/91858150-c3fa3300-eca3-11ea-9b60-68b6ba083497.png)

    - 고객은 PT수강신청을 취소할 수 있다.
    - PT수강신청이 취소되면 배정된 강사와 확정된 스케줄은 취소된다.

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/19251601/91858797-a4173f00-eca4-11ea-8e65-24487c2a24d2.png)

    - 1. PT수강신청 취소는 강사배정 및 확정스케줄 취소가 동시에 이루어지도록 한다.
    - 2. PT수강신청과 취소는 고객관리 담당자와 강사 스케줄 확정, 수업결과 등록 처리와 관계없이 항상 처리 가능하다.
    - 3. 고객관리 담당자는 강사 배정 현황과 스케줄 현황을 수시로 확인하여 모니터링 한다.
         고객은 수업결과 현황을 수시로 확인할 수 있다.



## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/19251601/91858260-e55b1f00-eca3-11ea-8875-0424a8bc7758.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    
## 신규 서비스 추가 시 기존 서비스에 영향이 없도록 열린 아키텍쳐 설계
    
![image](https://user-images.githubusercontent.com/19251601/91858323-fd32a300-eca3-11ea-9ed3-02c6918c5c54.png)

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd pttrainer
mvn spring-boot:run

cd ptstatus
mvn spring-boot:run 

cd ptorder
mvn spring-boot:run  

cd ptmanager
mvn spring-boot:run  

cd ptgateway
mvn spring-boot:run  
```

## DDD 의 적용

각 서비스에서 도출된 핵심 Aggregate인 객체 3개를 Entity로 선언하였다.

```
# Ptorder.java

package ptplatform3;

@Entity
@Table(name="Ptorder_table")
public class Ptorder {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long ptManagerId;
    private Long ptTrainerId;
    private String status;
    private Long ptClassId;
    private String ptClassName;
    private Long customerId;
    private String customerName;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getPtManagerId() {
        return ptManagerId;
    }

    public void setPtManagerId(Long ptManagerId) {
        this.ptManagerId = ptManagerId;
    }
    public Long getPtTrainerId() {
        return ptTrainerId;
    }

    public void setPtTrainerId(Long ptTrainerId) {
        this.ptTrainerId = ptTrainerId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public Long getPtClassId() {
        return ptClassId;
    }

    public void setPtClassId(Long ptClassId) {
        this.ptClassId = ptClassId;
    }
    public String getPtClassName() {
        return ptClassName;
    }

    public void setPtClassName(String ptClassName) {
        this.ptClassName = ptClassName;
    }
    public Long getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }
    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }



# Ptmanager.java
package ptplatform3;

@Entity
@Table(name="Ptmanager_table")
public class Ptmanager {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long ptOrderId;
    private Long ptTrainerId;
    private String status;
    private Long trainerId;
    private String trainerName;



    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getPtOrderId() {
        return ptOrderId;
    }

    public void setPtOrderId(Long ptOrderId) {
        this.ptOrderId = ptOrderId;
    }
    public Long getPtTrainerId() {
        return ptTrainerId;
    }

    public void setPtTrainerId(Long ptTrainerId) {
        this.ptTrainerId = ptTrainerId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public Long getTrainerId() {
        return trainerId;
    }

    public void setTrainerId(Long trainerId) {
        this.trainerId = trainerId;
    }
    public String getTrainerName() {
        return trainerName;
    }

    public void setTrainerName(String trainerName) {
        this.trainerName = trainerName;
    }
}


# Pttrainer.java
package ptplatform3;

@Entity
@Table(name="Pttrainer_table")
public class Pttrainer {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long ptOrderId;
    private Long ptManagerId;
    private String status;
    private String ptResult;
    private String ptClassDate;
    // private Long ptTrainerId4Cancel;

    public Long getId() {
        return id;
    }

    public void setId(final Long id) {
        this.id = id;
    }
    public Long getPtOrderId() {
        return ptOrderId;
    }

    public void setPtOrderId(final Long ptOrderId) {
        this.ptOrderId = ptOrderId;
    }
    public Long getPtManagerId() {
        return ptManagerId;
    }

    public void setPtManagerId(final Long ptManagerId) {
        this.ptManagerId = ptManagerId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(final String status) {
        this.status = status;
    }
    public String getPtResult() {
        return ptResult;
    }

    public void setPtResult(final String ptResult) {
        this.ptResult = ptResult;
    }
    public String getPtClassDate() {
        return ptClassDate;
    }

    public void setPtClassDate(final String ptClassDate) {
        this.ptClassDate = ptClassDate;
    }

    // public Long getPtTrainerId4Cancel() {
    //     return ptTrainerId4Cancel;
    // }

    // public void setPtTrainerId4Cancel(Long ptTrainerId4Cancel) {
    //     this.ptTrainerId4Cancel = ptTrainerId4Cancel;
    // }
}

```


## 동기식 호출과 Fallback 처리

고객관리(ptmanager)의 이벤트 '수강취소 접수됨'과 트레이너(pttrailner)의 커맨드 '수업 스케쥴 취소 됨' 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
- '수업 스케쥴 취소 됨'를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현

```
# Ptmanager.java

???????????????
```

- '수강취소접수됨' 직후(@PostUpdate) '수업스케쥴취소'를 요청하도록 처리
```
# Ptmanager.java

    @PostUpdate
    public void onPostUpdate(){
    try {
    ...
           // REQ-RES 강사스케줄 취소
          System.out.println("[HNR_DEBUG] ###################################################");
          System.out.println("[HNR_DEBUG] ######### ORDER_CANCEL_ACCEPTED (REQ/RES) #########");
          System.out.println("[HNR_DEBUG] ###################################################");

          System.out.println("[HNR_DEBUG] getPtOrderId() : " + getPtOrderId());
          PttrainerService pttrainerService = PtmanagerApplication.applicationContext.getBean(PttrainerService.class);
          pttrainerService.ptScheduleCancellation(getPtOrderId(), "SCHEDULE_CANCELED");
       }  catch (Exception e) {
            e.printStackTrace();
       }
    }
```




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


수강신청(Ptorder) 이 이루어진 후에 고객관리(Ptmanager) 서비스로 이를 알려주는 행위는 비동기식으로 처리하여, 고객관리(Ptmanager) 서비스의 처리를 위하여 수강신청(Ptorder)이 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 PT수강신청에 기록을 남긴 후에 곧바로 PT수강신청이 되었다는 도메인 이벤트를 카프카로 송출한다.(Publish)
 
```
package ptplatform3;

@Entity
@Table(name="Ptorder_table")
public class Ptorder {

 ...
    @PostPersist
    public void onPostPersist(){
            System.out.println("[HNR_DEBUG] ###########################");
            System.out.println("[HNR_DEBUG] ######### ORDERED #########");
            System.out.println("[HNR_DEBUG] ###########################");
            PtOrdered ptOrdered = new PtOrdered();
            BeanUtils.copyProperties(this, ptOrdered);
            ptOrdered.publishAfterCommit();
    }
}
```
- 고객관리(Ptmanager) 서비스에서는 PT수강신청이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.

```
package ptplatform3;

...

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    //HNR-START
    @Autowired
    PtmanagerRepository ptmanagerRepository;
    //HNR--END-

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPtOrdered_PtOrderRequest(@Payload PtOrdered ptOrdered){

        // 고객이 수강신청시 강사 배정 및 수강신청 허용 상태(ORDER_ACCEPTED) 저장
        if(ptOrdered.isMe()){
            System.out.println("##### listener PtOrderRequest : " + ptOrdered.toJson());
            //HNR-START
            Ptmanager ptmanager = new Ptmanager();
            ptmanager.setPtOrderId(ptOrdered.getId());
            ptmanager.setStatus("ORDER_ACCEPTED");
            ptmanager.setTrainerId(ptOrdered.getId() + 2000);
            ptmanager.setTrainerName("Good_Trainer_" + ptOrdered.getId());
            ptmanagerRepository.save(ptmanager);
            //HNR--END-
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPtScheduleConfirmed_PtScheduleConfirmationNotify(@Payload PtScheduleConfirmed ptScheduleConfirmed){

        // PT수업 스케줄 확정시 해당 수강신청 확정 상태(ORDER_CONFIRMED) 저장
        if(ptScheduleConfirmed.isMe()){
            //HNR-START
            try {
                System.out.println("##### listener PtScheduleConfirmationNotify : " + ptScheduleConfirmed.toJson());
                ptmanagerRepository.findById(ptScheduleConfirmed.getPtOrderId())
                        .ifPresent(
                                ptmanager -> {
                                    ptmanager.setStatus("ORDER_CONFIRMED");
                                    ptmanager.setPtTrainerId(ptScheduleConfirmed.getId());
                                    ptmanagerRepository.save(ptmanager);
                                }
                        );
            } catch (Exception e) {
                e.printStackTrace();
            }
            //Ptmanager ptmanager = new Ptmanager();
            //ptmanager.setStatus("ORDER_CONFIRMED");
            //ptmanager.setPtTrainerId(ptScheduleConfirmed.getId());

            //HNR--END-
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPtCancelOrdered_PtCancelOrderRequest(@Payload PtCancelOrdered ptCancelOrdered){

        //PT수강신청 취소 요청 허용(ORDER_CANCEL_ACCEPTED) 상태 저장
        if(ptCancelOrdered.isMe()){
            try {
                System.out.println("##### listener PtCancelOrderRequest : " + ptCancelOrdered.toJson());

                Optional<Ptmanager> pm = ptmanagerRepository.findById(ptCancelOrdered.getId());
                pm.get().setStatus("ORDER_CANCEL_ACCEPTED");
                ptmanagerRepository.save(pm.get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}
```

PT수강신청은 고객관리/트레이너 서비스와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 고객관리 서비스가 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:


## CQRS
PT 수강신청의 상태 조회를 위한 서비스를 CQRS패턴으로 구현하였다.
  - ptorder, ptmanger, pttrainer별 aggregate 통합 조회로 인한 성능 저하를 막을 수 있다.
  - 모든 정보는 비동기 방식으로 발행된 이벤트를 수신하여 처리된다.
  - 별도 서비스(orderStatus), 저장소(#######)로 구현하였다.
  - 설계 : MSAEz 설계의 view 매핑 설정 참조

## API Gateway
API Gateway를 통하여, 마이크로 서비스들의 진입점을 통일한다.
```
# application.yml 파일에 라우팅 경로 설정

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: Order
          uri: http://Order:8080
          predicates:
            - Path=/orders/** 
        - id: ManagementCenter
          uri: http://ManagementCenter:8080
          predicates:
            - Path=/managementCenters/** 
        - id: Installation
          uri: http://Installation:8080
          predicates:
            - Path=/installations/** 
        - id: orderstatus
          uri: http://orderstatus:8080
          predicates:
            - Path=/orderStatuses/** 
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080

```

API Gateway는 buildspec.yml파일에 type을 LoadBalancer로 명시하여 외부 호출에 대한 라우팅을 전담한다.



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다. 아래 Github 소스 코드 변경 시, CodeBuild 빌드/배포가 자동 시작되도록 구성하였다.

git 주소 : 
https://github.com/hannoory/ptgateway.git
https://github.com/hannoory/ptorder.git
https://github.com/hannoory/ptmanager.git
https://github.com/hannoory/pttrainer.git
https://github.com/hannoory/ptstatus.git

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

- istio에서의 서킷 브레이커 설정(DestinationRule)

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) 결제이력.java (Entity)

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
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/orders

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/orders   
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- metric 서버 설치 후, 명령 $ kubectl autoscale deployment ptorder --cpu-percent=1 --min=1 --max=10 적용하여 스케일 아웃 되는 장면

![image](https://user-images.githubusercontent.com/19251601/91865177-163f5200-ecac-11ea-8887-f39d71340179.png)



## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- siege를 돌리는 사이에 배포를 진행한다. 바로 적용하여... 죽는다... avaliability 65.74%...
![image](https://user-images.githubusercontent.com/19251601/91865302-35d67a80-ecac-11ea-9fed-22b60c041570.png)


- liveness readiness probe 적용후 siege 테스트... 무정지 배포가 된다
![image](https://user-images.githubusercontent.com/19251601/91865382-4e469500-ecac-11ea-8f14-3ad6f75ac7e7.png)


## ConfigMAp 적용
  - 설정의 외부주입을 통한 유연성을 제공하기 위해 ConfigMap 사용
  
  
# 운영 모니터링

## 마스터 노드 모니터링

Amazon EKS 제어 플레인 모니터링/로깅은 Amazon EKS 제어 플레인에서 계정의 CloudWatch Logs로 감사 및 진단 로그를 직접 제공한다.

  - 사용할 수 있는 클러스터 제어 플레인 로그 유형은 다음과 같다.
'''
  - Kubernetes API 서버 컴포넌트 로그(api)
  - 감사(audit) 
  - 인증자(authenticator) 
  - 컨트롤러 관리자(controllerManager)
  - 스케줄러(scheduler)

출처 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/logging-monitoring.html
'''

##  워커 노드 모니터링
  - 쿠버네티스 모니터링 솔루션 중에 가장 인기 많은 것은 Heapster와 Prometheus 이다.

  - Heapster는 쿠버네티스에서 기본적으로 제공이 되며, 클러스터 내의 모니터링과 이벤트 데이터를 수집한다.

  - Prometheus는 CNCF에 의해 제공이 되며, 쿠버네티스의 각 다른 객체와 구성으로부터 리소스 사용을 수집할 수 있다.

  - 쿠버네티스에서 로그를 수집하는 가장 흔한 방법은 fluentd를 사용하는 Elasticsearch 이며, fluentd는 node에서 에이전트로 작동하며 커스텀 설정이 가능하다.

  - 그 외 오픈소스를 활용하여 Worker Node 모니터링이 가능하다.
 
 
## 운영 모니터링 istio, mixer, grafana, kiali를 사용한 예이다.





# 시연 시나리오

1. 수강신청 이후 수강신청건 상태(스케쥴 확정)
2. 수업결과 등록 이후 수업결과 완료 상태
3. 수강신청 취소
4.1 고객관리 서비스 장애상황에서의 수강신청
4.2 고객관리 서비스 장애상황 복구 후 수강신청 상처리
4.3 고객관리 서비스 장애상황에서의 수강취소 신청
5. 무정지 배포
6. 오토 스케일링


##  캡쳐
- scale_out_HPA
![scale_out_HPA](https://user-images.githubusercontent.com/19251601/91919704-c4c1b200-ed01-11ea-8ab0-f9e1ba451c78.PNG)

- persistentvolume_1
![1](https://user-images.githubusercontent.com/19251601/91919739-ee7ad900-ed01-11ea-8883-d9b6f65451cc.PNG)

- persistentvolume_2
![2](https://user-images.githubusercontent.com/19251601/91919760-fe92b880-ed01-11ea-8128-4706df992eb5.PNG)

- config_secret_deployment
![config_secret_deployment](https://user-images.githubusercontent.com/19251601/91919822-2b46d000-ed02-11ea-839f-7c1b55c31e24.PNG)

- configmap
![configmap](https://user-images.githubusercontent.com/19251601/91919835-37cb2880-ed02-11ea-80ee-27819089ac5e.PNG)

- configmap_secret_deployment.yaml
![configmap_secret_deployment yaml](https://user-images.githubusercontent.com/19251601/91919856-46194480-ed02-11ea-985e-209e3c72da64.PNG)

- ptstatus_application_yaml
![ptstatus_application_yaml](https://user-images.githubusercontent.com/19251601/91919876-52050680-ed02-11ea-889f-000a064dd42b.PNG)

- secret
![secret](https://user-images.githubusercontent.com/19251601/91919886-5d583200-ed02-11ea-92cf-107aaf6fd4c1.PNG)

- live_deploy_update
![live_deploy_update](https://user-images.githubusercontent.com/19251601/91919962-a14b3700-ed02-11ea-84a9-3075bb5ab2d6.PNG)

- liveness_readiness_after
![liveness_readiness_after](https://user-images.githubusercontent.com/19251601/91919973-ae682600-ed02-11ea-9f03-b1caf994849a.PNG)

- liveness_readiness_before
![liveness_readiness_before](https://user-images.githubusercontent.com/19251601/91920008-baec7e80-ed02-11ea-8ada-95cd2e6810e8.PNG)


- 0_kubectl_get_all_status
![0_kubectl_get_all_status](https://user-images.githubusercontent.com/19251601/91920044-d35c9900-ed02-11ea-83fb-f7a0f4fd2283.PNG)


- 1_POST_Consumer_Status
![1_POST_Consumer_Status](https://user-images.githubusercontent.com/19251601/91920060-e66f6900-ed02-11ea-83a4-47c4471b3c5d.PNG)


- 1_POST_http_cmd
![1_POST_http_cmd](https://user-images.githubusercontent.com/19251601/91920136-1fa7d900-ed03-11ea-80ca-81bacf123509.PNG)

- 2_consumer
![2_consumer](https://user-images.githubusercontent.com/19251601/91920159-2d5d5e80-ed03-11ea-8166-a2c616ca3bbb.PNG)

- 2_PATCH_RESULT_CREATED
![2_PATCH_RESULT_CREATED](https://user-images.githubusercontent.com/19251601/91920168-3c441100-ed03-11ea-91b6-2536450b4380.PNG)

- 3_cancel_consumer
![3_cancel_consumer](https://user-images.githubusercontent.com/19251601/91920192-49610000-ed03-11ea-86e3-fd025f5a5683.PNG)

- 3_CANCEL_ORDER
![3_CANCEL_ORDER](https://user-images.githubusercontent.com/19251601/91920210-57af1c00-ed03-11ea-8808-3f4ba2f75dba.PNG)

- 4_statuses캡처
![4_statuses캡처](https://user-images.githubusercontent.com/19251601/91920227-64337480-ed03-11ea-98d4-e378152aa93a.PNG)

- rolling_out
![rolling_out](https://user-images.githubusercontent.com/19251601/91920257-76151780-ed03-11ea-8e18-52efb2530181.PNG)

-기본2_1
![1](https://user-images.githubusercontent.com/19251601/91920967-5252d100-ed05-11ea-8c27-10d80a52ca82.PNG)

-기본2_2
![2](https://user-images.githubusercontent.com/19251601/91921010-6b5b8200-ed05-11ea-9306-bf2b0d8187c6.PNG)

-기본2_3
![3](https://user-images.githubusercontent.com/19251601/91921032-7adacb00-ed05-11ea-8dde-7fdc55992811.PNG)
