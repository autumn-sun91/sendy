# Sendy

간편 송금 서비스


## 프로젝트 설명
- **목표와 주요 기능**
    - 목표: 사용자가 휴대폰 번호로 간편하게 송금할 수 있는 간편 송금 서비스 **SENDY** 개발 및 AWS 배포
    - 해당 프로젝트를 통해 금융 거래의 안정성과 일관성을 보장하며 분산환경에서 발생할 수 있는 데이터 불일치 문제를 해결, 금융권에서 요구하는 보안기준과 규제사항들의 이해를 통해 개발에 필요한 핵심 기술들을 경험하였습니다.
    - 주요 기능:
        1. 휴대폰 번호 기반 간편 송금 시스템
        2. 실시간 송금 처리 및 알림, 예약 송금 기능
        3. 간편인증 및 보안 강화된 인증 시스템
        4. 거래 내역 및 분석 대시보드
- MVP 사용 기술 스택
  - 모놀리식 시스템 채택

|제목|설명|
|------|---|
|언어|Kotlin(1.9.25)|
|프레임워크|Spring Boot(MVC) - 3.5.3(GA), JPA|
|DB|MySQL - 8.0.42(GA), Prometheus|
|Message Queue(Broker)|X(필요 시 추가)|
|Monitoring|Grafana|
|Container|Docker|
|CI/CD|Github Action|
|Testing|Junit5, Mockk|

## 송금 Sequence Diagram
<img width="600" height="800" alt="image(1)" src="https://github.com/user-attachments/assets/d34ff486-1ec8-48df-9a3d-c9c2501e3298" />
<img width="600" height="800" alt="image(2)" src="https://github.com/user-attachments/assets/6a446815-0a7e-42ff-bcf9-aa0ce5e22c96" />
<img width="600" height="800" alt="image(3)" src="https://github.com/user-attachments/assets/22046a9b-f84e-4ae3-8004-d5512c5b3bd7" />

## 기술적 의사결정
- **기술적 의사결정**
    - **왜 Kotlin 인가?**
        - nullable 안정성
        - 문법의 간결성
        - 개발 효율성
        - 기술적 도전
    - **왜 MySQL 인가?**
        - 전통적으로 금융권에서 Oracle 을 사용했으나 사이드 프로젝트에서는 실제로 구현하기에는 제약 사항 존재, 팀원들의 익숙한 DB 와 MVP 로 빠르게 개발할 수 있는 MySQL 로 결졍
    - **분산 환경에서 금융 트랜잭션을 어떻게 해결할 것 인가?**
        - 계좌, 이체 송금 요청 거래내역, 계좌의 입출금 내역 해당 3개 테이블을 기준으로 확장
        - 해결책 1:  단일 트랜잭션 적용
            - 완전한 데이터 일관성 확보
            - DB I/O 병목 가능성 높음
        - 해결책 2:   물리적인 DB 분리
            1. 송금 이력, 거래 원장 물리적인 DB 를 별도 구성하여 I/O 병목 해소 
            2. 부하 분산 처리
            3. 시스템 복잡도 증가
        - 해결책 3: LB + DB Replication
            - 부하 분산 처리
            - 동기화 지연
## MVP - 송금 관련 고민 사항 및 트러블 슈팅

### Lock 을 어떻게 설정할 것 인가?

- 입금과 출금 이뤄지는 순간에는 다른 쓰레드에 접근을 차단하여 데이터 원자성을 확보해야됨
- 즉 조회 부터 Lock 을 걸어두어서 출금과 입금의 대한 금액 정보라는 공유 자원을 다른 쓰레드에서 점유해서 write 를 시도 하면 안됨.

```sql
CREATE TABLE account
(
    id                 BIGINT      NOT NULL PRIMARY KEY,
    account_number     VARCHAR(13) NOT NULL,
    user_id            BIGINT      NOT NULL,
    password           VARCHAR(64) NOT NULL,
    status             VARCHAR(20) NOT NULL,
    is_primary         BOOLEAN     NOT NULL,
    is_limited_account BOOLEAN     NOT NULL,
    created_at         DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at         DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
    balance            BIGINT      NOT NULL,

    UNIQUE KEY `account_account_number_uk` (account_number)
) engine = InnoDB;

CREATE INDEX `account_user_id_account_number_index` on account (user_id, account_number);
```

- 위 계좌 테이블에는 unique key 가 계좌번호이고, 1명의 유저가 여러개의 계좌를 사용할 수 있는 구조
- 유저 id 정보로만 입력을 받아서 계좌 조회 후 입출금 처리한다면?

```sql
select * from account where user_id=38352658567418872 and is_primary=1 for update
```
- 아래와 같이 불필요한 레코드가 lock wait 걸림
<img width="1280" height="178" alt="image(4)" src="https://github.com/user-attachments/assets/01e3a185-5c67-43ad-b061-775186d4ef58" />

- 만약 user_id, 단일 인덱스로 설정 시에는? 여전히 위와 동일
- 만약 user_id, is_primary 를 복합 인덱스로 설정한다면?
```sql
select * from account where user_id=38352658567418872 and is_primary=1 for update user_id
```
- user_id 가 unique 하지 않아 조회는 한건으로 나오지만, 예상과 다르게 다른 유저id 의 계좌까지 lock이 걸림
<img width="1280" height="178" alt="image(5)" src="https://github.com/user-attachments/assets/d6b6e5aa-fc98-4012-8715-149e8af0778f" />
- 한건에 대한 user_id, account_number 를 lock 을 어떻게 잡아야 될까?

1. 이미 account_number 가 unique 로 잡혀있으니 휴대전화 번호를 계좌 번호로 이용할 수 있게끔
처리
2. 기획 재정의하여 한사람당 계좌 하나만을 만들 수 있도록 처리(계좌 종류를 분류), 즉 account_number 가 아니라 user_id로 unique 처리
--> 2번으로 채택하여 진행, 휴대전화번호를 계좌번호와 동등한 개념으로 사용할 수 있는 부분은 금융 정책 및 내부적으로 입출금 프로세스가 복잡하다고 판단
<img width="1280" height="178" alt="image(6)" src="https://github.com/user-attachments/assets/425cdd61-3291-4b17-a99f-d59ff3dce529" />

MySQL NamedLock
- 세션관리 → 즉 커넥션 풀을 사용해서 락을 거는데 이는 애플케이션에서 커넥션 객체를 재활용하면 문제 생기고 코드의 복잡성이 올라감. → 탈락

## MSA 도입
알림, 송금, 계좌, 회원등 도메인을 멀티모듈을 도입하여 각 개발자 1명씩 ownership 을 가지고 진행, 또한 각 마이크로서비스 간 통신은 http, EDA 를 활용하여 유연성 및 확장성 확보를 목표로 진행 

- 컨벤션
  - 멀티 모듈 전환 시 인텔리제이의 패키지 정렬이 내림차순이기때문에 아래와 같은 패키지명 제안

`sendy-{도메인명}-{용도}` 

  - 도메인명은 송금(transfer), 회원(user), 알림(notification), 인증/인가(auth), 공통(shared) ..
    - 단, admin 경우는 예외적으로 포함
    - 단, shared 모듈 경우 공통으로 사용하는 라이브러리가 포함 될 수 있으므로 기술 이름 허용
        - 해당 모듈은 shared 모듈의 하위로 귀속
        - `kafka`
        - `feign`
        - …
    - 단, 공통으로 사용해야되는 엔티티 경우에는 용도를 `domain` 으로 명명 가능
        - `sendy-transfer-domain`
  - 용도는 아래와 같은 의미를 지님
    - `api` : 알림 api, 송금 api 등 사용자의 요청을 처리
    - `scheduler` : 스케줄러
    - `batch` : 배치
    - `consumer`: 메세지 컨슈머
    - `producer` : 메세지 프로듀서
    - `domain` : 도메인(순수, jpa 엔티티)
   
- 시스템 설계도
<img width="1331" height="613" alt="image(8)" src="https://github.com/user-attachments/assets/89f4a928-a438-4940-bcc9-7136f1989821" />

## MSA - 예약 송금 배치 고민 사항 및 트러블 슈팅
요구사항: 멱등성만 잘지켜서 예약 송금이 이뤄져야됨.

- 멱등성만 잘 지켜지는건 무슨 의미?<br>
→ 중복 송금X<br>
→ 유일 식별자로 요청 시 N번 요청하더라도 동일한 결과값이 나와야됨.

- 10초마다 실행되는 스케줄러가 있는데 예약 송금 처리 되는 시간을 11초로 설정 시 어떻게 될까?<br>
→ 기본적으로 동기식으로 쓰레드가 동작하기 때문에 10초마다 실행 될지라도 작업 사항이 모두 완료가 안되면 blocking
<img width="1699" height="203" alt="image(9)" src="https://github.com/user-attachments/assets/cbeac4ee-5ec7-483b-ad5a-fd1b88ab9095" />

- 그럼 만약 1시간보다 배치가 동작을 하는데 예약 송금이 1시간안에 처리를 못하면 어떻게 될 것 같인가? 해결 할 수 있는 방법은?
1. case 1) JPA 가 아닌 JDBC Template 을 이용한 청크 처리
- 조회 시 페이징 이용
- JPA 가 아닌 JDBC Template 으로 페이징 처리(로우 쿼리 이용) - 성능 우선

2. 병렬 비동기 처리(courutine)
- 대략적으로 50만건 데이터를 조회하는데 3000밀리초가 걸린다고 가정해보자
한번에 조회 시 동기식으로 처리 된다면 아마 아래와 같은 상황이 발생할 것 이다.

2-1). 전체 50만건 조회 → 3000ms 걸림<br>
2-2). 예약 송금도 마찬가지고 3초 안에 송금이 완료되어야됨<br>
2-3). 1건당 max 3초로 잡아보면 50만건 모두 11:00:00 에 예약했다고 가정해보자<br>
```plain
index : 1 → 11:00:06 → 예약 송금 완료
index : 2 → 11:00:09 → 예약 송금 완료
index : 3 → 11:00:12 → 예약 송금 완료
index: 4 → 11:00:15 → 예약 송금 완료
…
…
…
index: 50만번째 → ??? → 예약 송금 완료??
→ 50만 X 3초 = 150만초 = 25000분 = 17일 8시간 40분 뒤에 보내짐
```

→ case 1 번대로 chunk 사이즈 별로 읽어와서 후 처리<br>
→ 1,000 X 500, 즉 500개 그룹이 나눠지는데 이때 비동기로 처리<br>
→ 비동기 처리를 kafka 로 할것인지, 코루틴으로 할것인지 아니면 섞어서 할 것 인지<br>
→ kafka  사용 시 producer 에서 500개 이벤트를 발행할것인지?<br>
→ 이벤트 발행 시 파티션 갯수와 컨슈머 갯수는 어떻게 조정 할 것 인지<br>
→ 코루틴 사용 시 그룹별로 코루틴을 할당할 것 인지 이 과정에서 동시성 이슈가 발생 할 수 있는데 DB Lock 만으로 충분한지? 데드락이 걸릴 가능성은?<br>
→ 분산락을 통해서 진행?<br>
→ https://waterfogsw.tistory.com/60<br>

결정: kafka 에서 청크 사이즈별 파티션 할당하여 이벤트 발행, 컨슈머쪽에서 코루틴으로 예약 송금 진행
<img width="800" height="600" alt="image(7)" src="https://github.com/user-attachments/assets/ecd63755-9208-4852-8680-b098c5cd706c" />
