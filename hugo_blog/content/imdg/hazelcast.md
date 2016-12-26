
date = "2016-12-12T14:04:52+09:00"
title = "20161212: hazelcast 교육 내용 정리"

+++

### HAZELCAST

###[참고 링크](http://estech.biz/2016/11/05/seoul-hazelcast-meetup-ppt-2016-11-03/)
###[PDF 링크](../hazelcast.pdf)

(참고 - 여기서 사용하는 노드는 hazelcast가 올라간 jvm을 의미)

### 장점 
##### 1. 기존 코드를 거의 고치지 않고 캐싱 사용 가능
[기존]
```java
public class DistributedMap {
    public static void main(String[] args) {
        Config config = new Config();
        Map<String, String> map = new HashMap<>();
        map.put("key", "value");
        map.get("key");

        //Concurrent Map methods
        map.putIfAbsent("somekey", "somevalue");
        map.replace("key", "value", "newvalue");
    }
} 
```
[하젤캐스트 사용]
```java
public class DistributedMap {
    public static void main(String[] args) {
        Config config = new Config();
        HazelcastInstance h = Hazelcast.newHazelcastInstance(config);
        ConcurrentMap<String, String> map = h.getMap("my-distributed-map");
        map.put("key", "value");
        map.get("key");

        //Concurrent Map methods
        map.putIfAbsent("somekey", "somevalue");
        map.replace("key", "value", "newvalue");
    }
} 
```
    
    
    기존 코드에서 몇 줄만 고치면 hazelcast 사용 가능
    HazelcastInstance h = Hazelcast.newHazelcastInstance(config);
    ConcurrentMap<String, String> map = h.getMap("my-distributed-map");
    
            
##### 2. 자바 상임의원이 hazelcast에 몸담고 있음
- jcache등 자바 표준의 기술들을 기반으로 개발하고 있고 적극적으로 사용하는 모습을 보임. 자바쪽 진영에 영향력이 있어서 표준을 이끌수 있어 보임

##### 3. 스프링과의 연동의 쉬움
- 적극적으로 스프링과의 integration을 지원함.(스프링 커뮤니티에서 hazelcast를 적극 추천하는 이유를 알 수 있었음.)

### hazelcast HD(유료)
heap을 사용하지 않고 자체적으로 개발한 메모리 모델을 사용 - FULL GC 발생 안됨. 
1. 흔히들 full gc시 Stop the world을 방지하기 위해 64G 메모리에 4G * 16(jvm)를 띄워서 Full GC 대응을 한다. 
2. 하지만, jvm 갯수가 늘어나는 만큼 관리 비용이 크다. 
3. HD를 사용하면 Full GC 걱정 안해도 되고 물리 메모리를 최대한 확보해도 상관 없다.(60G를 hazelcast의 HD에 할당해도 GC로 인한 frozen 걱정은 최소화된다. - 하드웨어 자원 최대한 사용 가능해짐.)

### 분산 연산 지원 - Distribute Computing
하젤 캐스트 서비스 레이어에 Callable/Runable 객체만 던지면 하젤캐스트 노드들간에서 분산 처리후 결과를 던지는 기능(hadoop의 map-reduce와 비슷한듯. spark의 RDD) 
- 노드가 늘어날 수록 고속 연산에 유리해짐.
- 노드들간에 syncronize 가능 - lock 전략 사용 -> 이건 나라면 안쓸듯.

### Partition & Sharding
hash를 사용해서 데이터를 파티셔닝함.(파티션을 전체 데이터의 일부 데이터 셋으로 보면 됨. )
파티셔닝한 데이터를 두가지로 관리 
1. major(원래 노드가 관리하는 파티션) 
2. backup(다른 노드에서 관리하는 파티션의 일부를 저장)
-> 만약 하나의 노드가 깨지면 백업 파티션 및 major 파티션에서 알아서 분산 저장 실행.

#### hash process
KEY -> byte[] -> string-> hashcode -> node의 갯수 기준으로 나머지 연산(%) 후 적절한 노드 선택후 저장

### 노드구성시 주의사항.
LAN으로(WAN X) 노드를 관리해야함. nextwork priority 개념이 없음.
-> network간 거리가 멀게 구성하면 성능 저하의 요인.

### hazelcast heimdall(유료)
기존 : service -> JDBC -> RDB
변경 : service -> (heimdall JDBC) -> JDBC -> RDB

기존 JDBC 사용하는 코드에 라이브러리만 헤임달 JDBC로 변경해주면 알아서 데이터베이스 캐싱 및 데이터 베이스 장애시에 해임달이 캐싱한(테이블 구조를 그대로 캐싱) 데이터를 전달
-> netflix의 hystrix랑 비슷한 구조.


### 장애대응
장애로 인한 노트가 망가지고 네트워크 트래픽이 많은 경우 hazelcast는 자동으로 감지해서 대응한다.
파티션의 갯수가 1000이고 전체가 100G의 데이터인경우 하나의 파티션은 100M를 차지함 분산 저장시 10개의 파티션을 기본으로 설정하면. 1G의 네트워크 전송량이 필요 
=> 파티션을 10000개로 10배 늘리면 100G의 데이터의 하나의 파티션의 용량은 10M로 줄어들고 1회 전송하는 파티셧의 셋의 갯수를 5개로 줄이면 50M의 네트워크 전송량 필요(20배를 줄여서 내부 부하를 줄이는 전략을 취함.)
=> 위의 과정이 자동으로 전환

### Deploy
2가지 Mode지원 
Embedded(같은 jvm내에 hazelcast 사용),  Client-Server(다른 jvm에 띄움 - 같은 머신일수도 다른 머신일 수도~)
=> 간단한 설정 변경으로 둘 다 편하게 사용 가능

### Smart routing
1의 클라이언트는 하나의 노드와 연결을 맺음. => smart routing -_-
 - 모든 노드가 connection을 맺지 않는다
 - 이 컨셉으로 고가용성(high avaiablity)이 가능
 - network 장애, gc 응답없음(stop the world)일 경우 fault tolerance 가능하게 함.
 - hazelcast의 router가 클라이언트 - 노드간 reconnect

### Hazelcast zet (무료!!)
기존의 [cascading](https://en.wikipedia.org/wiki/Cascading_(software))보다 편하다고함. 
스파크보다 분산 처리 성능이 44배 빠르다고함(업체측 자료만을 못믿음.)
하둡처럼 대용량 데이터, 스파크처럼 in memory 분산 처리 뭐 이런 것을 하나의 솔루션으로 처리 가능한 것으로 보임.(람다 아키텍쳐를 hazelcast zet으로 한방에 처리 뭐 이런 느낌)


### 발표자 주소
rahul@haselcast.com
james@estech21.com => 12월말까지 hazelcast korea로 변경 hazelcast.com

















 
