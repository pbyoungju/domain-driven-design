# 11.1 단일 모델의 단점
 - 조회화면은 속도가 중요한데 엔티티의 식별자를 이용한 애그리거트 참조 방식을 사용하면 JPA EAGER LOADING 방식을 이용할 수 없다.
 - 한번의 SELECT로 조회화면에 필요한 데이터를 읽을 수 없다.
 - 애그리거트 간 연관을 식별자가 아니라 직접 참조하는 방식으로 연결해도 EAGER/LAZY 할지 정해야 한다.
 - 필요하면 네이티브 쿼리를 사용해야 한다.
 - 위 내용 모두 단일 도메인 모델을 사용해서 발생되는 문제이다

# 11.2 CQRS
 - Command Query Responsibility Segregation
 - 상태를 변경하는 명령을 위한 모델과 상태를 제공하는 조회를 위한 모델을 분리하는 패턴
 - 명령 모델은 JPA / 조회 모델은 MYBATIS로 구현할 수 있음 

### 명령 모델과 조회 모델은 서로 다른 기술을 이용해서 구현할 수 있다.
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-11-1.png" alt="drawing" width="400"/>

<br />

### 명령 모델과 조회 모델 예시 
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-11-2.png" alt="drawing" width="400"/>

<br />

### 명령 모델과 조회 모델이 서로 다른 DB를 사용할 수 있음 (Eventual Consistency)
<img src="https://image.pbyoungju.com/study/ddd-start/chapter-11-3.png" alt="drawing" width="400"/>

<br />

### Mediator Pattern
https://medium.com/@darshana-edirisinghe/cqrs-and-mediator-design-patterns-f11d2e9e9c2e

