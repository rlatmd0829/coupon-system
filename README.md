# 실습으로 배우는 선착순 이벤트

## 요구사항
선착순 100명에게 할인쿠폰을 제공하는 이벤트를 진행하고자 한다.

이 이벤트는 아래와 같은 조건을 만족하여야 한다.
- 선착순 100명에게만 지급되어야한다.
- 101개 이상이 지급되면 안된다.
- 순간적으로 몰리는 트래픽을 버틸 수 있어야합니다.


### Race condition 해결방법

#### 1. synchronized
* synchronized는 서버 인스턴스가 여러개일 경우 race condition이 발생할수 있다.

#### 2. 데이터베이스 락
* 데이터베이스 락을 걸 경우 쿠폰 조회부터 완료까지 락이 걸리게 되므로 성능 저하가 발생할 수 있다
* 예를들어, 쿠폰 저장까지 2초가 걸린다면 2초만큼 계속 락이 발생

#### 3. Redis
* 쿠폰 갯수만 정확하게 관리하면 되므로 redis incr 명령어를 활용해 쿠폰 갯수 관리 (redis는 싱글스레드로 돌아가므로 race condition을 해결할 수 있다.)
* 그러나 발급하는 쿠폰에 갯수가 많아져 트래픽이 몰릴경우 db에 부담을 주게 된다.
    
#### 4. kafka + Redis
* kafka를 사용하면 처리량을 조절할수 있게되어 데이터베이스에 부하를 줄일수 있는 장점이 있다
* 그러나 쿠폰생성까지 약간의 텀이 발생하는 단점도 있다.

## 추가 요구사항
* 한명당 한개의 쿠폰만 발급받을 수 있어야 한다.

### 한명당 한개의 쿠폰만 발급하도록 하는 방법

#### 1. 유니크 키 
* userId + couponType 으로 유니크 키를 설정하면 한명당 한개의 쿠폰만 발급받게 할수있다
* 그러나 보통 서비스에서 한명의 사용자가 쿠폰타입을 동시에 여러개 가질수 있으므로 실용적인 방법은 아니다

#### 2. 데이터베이스 락

* 범위 lock을 걸어서 쿠폰발급 여부를 확인하고 진행하는 방식
* 현재 방식은 consumer에서 받아서 쿠폰을 생성하는 로직으로 시간차가 있다. 그래서 발급할때 lock을 걸어서 확인하더라도 kafka를 통해 받아서 발급할때까지 시간이 걸리기 때문에 한명이 두개가 발급될수 있다
* api 모듈에서 직접 발급한다고 해도 lock 범위가 너무 넓어져 성능 저하가 발생한다.

![KakaoTalk_Photo_2024-10-20-15-00-49 001](https://github.com/user-attachments/assets/df155045-c986-490e-947e-cda86ef515a5)
    
#### 3. Redis set 자료구조 사용
* Redis 에 있는 set 자료구조를 사용하여 발급전에 set에 userId를 넣어두고 발급할때 set에 userId가 있는지 확인하고 발급한다
* 해당 강의에서 사용하는 방법

### 분산락을 사용하지않고 Redis Set만 사용 했을경우에 발생할 수 있는 동시성문제

Redis의 SET만 사용하는 경우, 요청이 동시에 들어올 때 경쟁 상태가 발생할 수 있고, 그로 인해 중복 쿠폰 발급이 될 수 있습니다.

분산 락을 사용하면 경쟁 상태를 방지할 수 있으며, 동시성 문제로 인해 중복된 쿠폰 발급이 발생하지 않도록 보장할 수 있습니다.

#### 동시성 문제 시나리오 예시
1. 요청 A가 들어와서 Redis의 SET을 조회합니다. 아직 userId가 없으므로, 다음 단계로 넘어갑니다.
2. 요청 B가 들어와서 동시에 Redis의 SET을 조회합니다. 요청 A가 아직 userId를 SET에 추가하기 전이므로, 요청 B도 userId가 없다고 판단합니다.
3. 요청 A가 userId를 SET에 추가합니다.
4. 요청 B가 userId를 또다시 SET에 추가합니다.
5. 결과적으로 같은 유저에게 두 개의 쿠폰이 발급됩니다.

Redis는 싱글 스레드로 동작하며, 단일 명령어는 원자적으로 처리되기 때문에 경쟁 상태가 발생하지 않습니다.

하지만 명령어를 순차적으로 조합해서 사용하는 경우 경쟁 상태가 발생할 수 있습니다.


### consumer에서 쿠폰을 발급하다가 에러가 발생할 경우

* Coupon 을 save하다가 에러가 날 경우 failedEvent 엔티티를 만들어서 거기다가 userId 저장해서 기록한다 -> 그리고 배치를 이용해 다시 생성한다

![KakaoTalk_Photo_2024-10-20-15-00-50 002](https://github.com/user-attachments/assets/56d33856-fd72-4135-ae44-16aa832cfeb7)

### 실습하면서 에러상황

* kafkaProducerConfig에서 Serializer 의존성을 apache.kafka 를 안써서 consume 실패,,
* https://www.inflearn.com/community/questions/1174832/consumer%EC%97%90%EC%84%9C-%EC%88%AB%EC%9E%90%EA%B0%80-%EC%B6%9C%EB%A0%A5%EB%90%98%EC%A7%80-%EC%95%8A%EC%8A%B5%EB%8B%88%EB%8B%A4%E3%85%A0

### Reference

* [실습으로 배우는 선착순 이벤트 시스템](https://www.inflearn.com/course/%EC%84%A0%EC%B0%A9%EC%88%9C-%EC%9D%B4%EB%B2%A4%ED%8A%B8-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%8B%A4%EC%8A%B5?attributionToken=igHwiQoMCLu70rgGEJOpuLkCEAEaJDY3MTYxOWY5LTAwMDAtMmVmZS1hYjMwLWY0ZjVlODA1Y2M0MCoGMzg5Nzk5MjCjgJcit7eMLajlqi2Y1rctwvCeFcXL8xeOvp0V1LKdFZvWty2Q97IwjpHJMJnuxjA6DmRlZmF1bHRfc2VhcmNoQAFIAWgBegJzaQ)
