# 11. **CH11 스트림 처리(데이터 베이스와 스트림, 스트림 처리)**

# Overview

로그 기반 브로커가 데이터베이스에서 아이디어를 얻어 메시징 적용에 성공한 것처럼, 그 반대로

메시징과 스트림에서 아이디어를 가져와 데이터베이스에 적용이 가능하다.

데이터베이스에 뭔가를 기록한다는 사실은(`사건` ) 캡처해서 저장하고 처리할 수 있는 `이벤트` 다.

복제로그는 데이터베이스 기록 이벤트의 스트림으로, 트랜잭션 처리시 리더는 데이터베이스 기록 이벤트를 생성한다. 팔로워는 해당 기록 스트림을 데이터베이스 복제본에 기록해 완전히 동일한 복사본을 만든다.(복제 로그 이벤트는 데이터에 변경이 발생했음을 나타낸다)

## 시스템 동기화 유지하기

실제로 대부분의 중요 애플리케이션이 요구사항을 만족하기 위해 OLTP 데이터베이스, 캐시, 전문 색인, 데이터 웨어하우스 등의 다른 기술의 조합이 필요하다.

각 시스템들은 각각의 데이터의 복제본이 있으며, 목적에 맞게 최적화된 형태로 각각 저장된다.

관련이 있거나, 동일한 데이터가 여러 장소에 나타나는 경우 서로 동기화가 필수이며, 데이터 웨어하우스에서는 이 동기화 과정을 대개 ETL 과정에서 수행한다.(흔히 데이터베이스 전체를 복사하고 변환하여 벌크로드하는 일괄처리를 한다.)

주기적으로 데이터베이스 전체를 덤프하는 작업이 너무 느리면, 대안으로 `이중기록` 을 사용하는 방법이 있다.

이중 기록을 사용하면 데이터가 변할 때마다 애플리케이션 코드에서 명시적으로 각 시스템에 기록한다.(데이터베이스에 기록한 후 검색 색인을 갱신하고 캐시 엔트리를 무효화 하거나, 해당 쓰기 작업을 동시에 수행한다.)

이러한 이중 기록은 경쟁조건에서 동시성의 문제가 있으며, 이는 버전 벡터와 같은 동시성 감지 메커니즘을 따로 사용하지 않으면 동시에 쓰기가 발생해도 알아차리기 힘들다.

또 다른 문제는 한쪽 쓰기가 성공할 때 다른 쪽 쓰기는 실패할 수 있다는 점이다. 이는 `내결함성` 문제로 두 시스템 간 불일치가 발생하는 현상이 생긴다.

동시성공 혹은 동시 실패를 보장하는 방식은 `원자적 커밋` 문제로 이 문제를 해결하는 데는 비용이 많이 든다.(원자적 커밋과 2단계 커밋(2PC))

## 변경 데이터 캡쳐

수십 년 동안 많은 데이터베이스는 데이터 변경 로그를 얻는 방법에 대해 기술한 문서를 제공하지 않았고ㅡ 이는 데이터 변화를 감지해서 변견됭 내용을 검색 색인, 캐시, 데이터 웨어하우스 같은 다른 저장 기술에 복제하기 어려웠다.

최근 들어 `변경 데이터 캡쳐(Change Data Capture, CDC)` 에 관심이 높아지고 있다. 데이터베이스에 기록하는 모든 데이터의 변화를 관찰해 다른 시스템으로 데이터를 복제할 수 있는 형태로 추출하는 과정으로 CDC는 데이터가 기록되지마자 변경 내용을 스트림으로 제공할 수 있으면 특히 유용하다.

### 변경 데이터 캡처의 구현

검색 색인과 데이터 웨어하우스에 저장된 데이터는 레코드 시스템에 저장된 데이터의 또 다른 뷰일 뿐이므로 로그 소비자를 `파생 데이터 시스템` 이라고 할 수 있다.

CDC는 이러한 파생 데이터 시스템이 레코드 시스템의 정확한 데이터 복제본을 가지게 하기 위해 레코드 시스템의 모든 변경 사항을 바생 데이터 시스템에 반영하는 것을 보장하는 메커니즘이다.

변경 데이터 캡처를 구현하는 데 데이터베이스 트리거를 사용하기도 하지만, 이는 고장 나기 쉽고 성능 오버헤드가 상당하다.

변경 데이터 챕처는 메시지 브로커와 동일하게 비동기 방식으로 동작하며, 레코드 데이터베이스 시스템은 변경 사항을 커밋하기 전에 변경 사항이 소비자에게 적용될 때까지 기다리지 않는다.

운영상 이점이 있는 설계로 느린 소비자가 추가되더라도 레코드 시스템에 미치는 영향이 적지만 복제 지연의 모든 문제가 발생하는 단점이 있다.

### 초기 스냅숏

데이터베이스에서 발생한 모든 변경 로그가 있다면 로그를 재현하여 데이터베이스의 전체 상태를 재국축할 수 있다. 그러나, 모든 변경 사항을 영구적으로 보관하는 일은 디스크 공간이 너무 많이 필요하고 작업도 오래 걸려 로그를 적당히 잘라야 한다.

전문 색인을 새로 구축할 때를 예로 들면 전체 데이터베이스 복사본이 필요하며, 최근에 갱신하지 않은 항목은 로그에 없기 때문에 최근 변경 사항만 반영하는 것으로는 충분하지 않다.

따라서 전체 로그 히스토리가 없다면 일관성 있는 스냅숏을 사용해야 한다. 데이터베이스 스냅숏은 변경 로그의 위치나 오프셋에 대응돼야 하며 일부 CDC 도구는 이런 스냅숏 기능을 내장하고 있으나 수작업으로 진행해야 하는 CDC 도구도 있다.

### 로그 컴팩션

로그 히스토리의 양을 제한한다면 새로운 파생 데이터 시스템을 추가할 때마다 스냅숏을 만들어야 하지만 주기적으로 같은 키의 로그 레코드를 찾아 중복을 제거하고 각 키에 대해 가장 최근에 갱신된 내용만 유지하는 `로그 컴팩션` 이라는 대안이 있다.

컴팩션한 로그를 저장하는 데 필요한 디스크 공간은 지금까지 데이터 베이스에 발생한 쓰기 수가 아닌 현재 데이터베이스에 있는 내용에 달려있다. 같은 키를 여러 번 덮어 썼다면 이전 값은 결국 가비지 컬렉션되고 최신 값이 유지된다.

로그 기반 메시지 브로커와 CDC 맥락에서도 마찬가지로 모든 변경에 기본키가 포함되게 하고 키의 갱신이 해당 키의 이전 값을 교체한다면 특정 키에 대한 최신 쓰기만 유지하면 충분하다.

검색 색인과 같은 파생 데이터 시스템을 재구축할 때마다 새 소비자는 컴팩션된 로그 토픽의 `오프셋 0` 부터 시작해 순차적으로 모든 키를 스캔하면 된다. 즉 CDC 원본 데이터베이스의 스냅숏을 만들지 않고도 데이터베이스 콘텐츠 전체의 복사본을 얻을 수 있다.

아파치 카프카는 로그 컴팩션 기능(`로그 세그먼트 삭제` , `로그 세그먼트 컴팩션` )을 제공한다.

### 변경 스트림용 API 지원

최근 데이터베이스들은 기능 개선이나 리버스 엔지니어링을 통해 CDC 지원을 하기보다 점진적으로 변경 스트림을 기본 인터페이스로서 지원하기 시작했다.

카프카 커넥트는 카프카를 광범위한 데이터 시스템용 CDC로 활용하기 위한 오력의 일환으로, 변경 이벤트를 스트림 하는데 카프카를 사용하면 검색 색인과 같은 파생 데이터 시스템을 갱신하는 데 사용 가능하고 스트림 처리 시스템에도 이벤트 공급이 가능하다.

## 이벤트 소싱

`이벤트 소싱` 은 도메인 주도설계(DDD) 커퓨니티에서 개발한 기법으로 CDC와 유사하게 애플리케이션 상태 변화를 모두 변경 이벤트 로그로 저장한다. CDC와의 가장 큰 차이점은 이 아이디어를 적용하는 추상화 레벨이 다르다는 점이다.

- CDC에서 애플리케이션은 데이터베이스를 변경 가능한 방식으로 사용해 레코드를 자유롭게 갱신하고 삭제한다. 변경 로그는 데이터베이스에서 추출한 쓰기 순서가 실제로 데이터를 기록한 순서와 일치하며, 경쟁 조건이 나타나지 않게 보장한다.
- 이벤트 소싱에서 애플리케이션 로직은 이벤트 로그에 기록된 불변 이벤트를 기반으로 명시적으로 구축한다. 이때 이벤트 저장은 단지 추가만 가능하고 갱신, 삭제는 권장하지 않거나 금지한다.
  이벤트는 저 수준 상태 변경이 아닌, 애플리케이션 수준에서 발생한 일을 반영하게끔 설계됐다.

이벤트 소싱은 데이터 모델링에 쓸 수 있는 강력한 기법으로 애플리케이션 관점에서 사용자의 행동을 불변 이벤트로 기록하는 방식은 변경 가능한 데이터베이스 상에서 사용자의 행동에 따른 효과를 기록하는 방식보다 훨씬 유의미 하다.

이벤트 소싱을 사용하면 애플리케이션을 지속해서 개선하기 매우 유리하며, 어떤 상황이 발생한 후에 상황 파악이 쉽기 때문에 디버깅에 도움이 되고 애플리케이션 버그를 방지한다.

이벤트 스토어 같은 특화된 데이터베이스는 이벤트 소싱을 사용하는 애플리케이션을 지원하게끔 개발하고 있지만, 일반적으로 이벤트 소싱 접근법은 특정 도구와 독립적으로 일반 데이터베이스나 로그 기반 메시지 브로커도 이런 방식으로 애플리케이션을 구축하는 데 사용할 수 있다.

### 이벤트 로그에서 현재 상태 파생하기

이벤트 로그는 그 자체로는 그렇게 유용하지 않다. 사용자는 일반적으로 시스템의 현재 상태를 보기를 원하지 수정 히스토리를 원하지 않기 때문이다.

따라서 이벤트 소싱을 사용하는 애플리케이션은 시스템에 기록한 데이터를 표현한 이벤트 로그를 가져와 사용자에게 보여주기 위해 적당한 애플리케이션 상태로 변환해야한다.

변경 데이터 캡처와 마찬가지로 이벤트 로그를 재현하면 현재 시스템 상태를 재구성 할 수 있다. 하지만 로그 컴팩션은 다르게 처리해야 한다.

- 기본키의 가장 최신 이벤트로 결정되며 이전 이벤트는 로그 컴팩션을 통해 버리는 CDC 이벤트와 다르게 발생한 이벤트 소싱은 발생한 이벤트가 앞선 이벤트를 덮어쓰지 않고 마지막 상태를 재구축 하기위해서 이벤트의 전체 히스토리가 필요하다.(이런 방식은 로그 컴팩션이 불가능하다)

이벤트 소싱을 사용하는 애플리케이션은 일반적으로 이벤트 로그에서 파생된 현재 상태의 스냅숏을 저장하는 매커니즘이 있어 로그를 반복해 처리할 필요는 없지만 이 메커니즘은 장애 발생 시 읽고 복구하는 성능을 높여주는 최적화에 불과하다.

### 명령과 이벤트

이벤트 소싱 철학은 `이벤트` 와 `명령` 을 구분하는 데 주의한다. 사용자 요청이 처음 도착했을 때 이 요청은 명령이며, 이 시점에서는 명령이 실패할 수도 있어 애플리케이션은 먼저 명령이 실행 가능한지 확인해야 한다.

이벤트는 생성 시점에 `사실` 이 된다. 사용자가 나중에 예약을 변경하거나 취소하더라도 이전에 특정 좌성을 예약했다는 사실은 여전히 진실이며, 변경이나 취소는 나중에 추가된 독립적인 이벤트다.

소비자가 이벤트를 받은 시점에는 이벤트는 이미 불변 로그의 일부분으로 명령의 유효성은 이벤트가 되기 전에 동기적식으로 검증해야 한다. 이를 테면 직렬성 트랜잭션을 사용해 원자적으로 명령을 검증하고 이벤트를 발행할 수 있다.

혹은 좌석을 예약하는 사용자 요청을 이벤트 두 개로 분할하여 하나는 가예약, 다른 하나는 유효한 예약에 대한 확정 이벤트로 분할하여 비동기 처리로 유효성 검사를 할 수 있다.

## 상태와 스트림 그리고 불변성

데이터베이스는 데이터 삽입 외에도 데이터 갱신과 삭제를 지원한다. 이것이 불변성과 어떻게 어울릴 수 있을까?

상태가 변할 때마다 해당 상태는 시간이 흐름에 따라 변한 이벤트의 마지막 결과다. 상태가 어떻게 바뀌었든 항상 변화를 일으킨 일련의 이벤트가 있다. 사건이 발생했다가 취소되더라도 이벤트가 발생했다는 점은 엄연한 사실이다. 즉 모든 `변경 로그` 는 시간이 지남에 따라 바뀌는 상태를 나타낸다.

데이터베이스의 내용은 로그의 최근 레코드 값을 캐시하고 있는 셈이며, 데이터베이스는 로그의 부분 집합의 캐시로 캐시한 부분 집합은 로그로부터 가져온 각 레코드와 색인의 최신 값이다.

### 불변 이벤트의 장점

데이터베이스에서 불변성을 이용하는 아이디어는 생각보다 오래됐다. 예를 들어 거래가 발생하면 거래 정보를 원장에 추가만 하는 방식으로 기록한다. 실수가 발생해도 원장을 수정하는 것이 아닌 수정에 대한 거래 내역을 추가한다.

이는 금융 시스템 처럼 엄격한 규제가 필요하지 않은 많은 다른 시스템에도 유용하다. 우연히 버그가 있는 코드를 배포해서 데이터베이스에 잘못된 데이터를 기록했을때 코드가 데이터를 덮어썼다면 복구하기가 매우 어렵다. 추가만 하는 불변 이벤트 로그를 썼다면 문제 상황의 진단과 복구가 훨씬 쉽다.

또한 불변 이벤트는 현재 상태보다 훨씬 많은 정보를 포함한다. 예로 고객이 장바구니에 항목 하나를 넣었다가 제거했다고 가정할 경우 주문 이행 관점에선 단지 취소에 불과하지만 분석가에게는 고객이 특정 항목을 구매하려 했다가 하지 않았다는 것을 알 수 있는 유용한 정보다.

### 동일한 이벤트 로그로 여러 가지 뷰 만들기

불변 이벤트 로그에서 가변 상태를 분리하면 동일한 이벤트 로그로 다른 여러 읽기 전용 뷰를 만들 수 도 있다. 이것은 한 스트림이 여러 소비자를 가질 때와 동일한 방식으로 작동한다.

이벤트 로그에서 데이터베이스로 변환하는 명시적인 단계가 있으면 시간이 흐름에 따라 애플리케이션을 발전시키기 쉽니다. 기존 데이터를 새로운 방식으로 표현하는 새 기능을 추가하려면 이벤트 로그를 사용해 신규 기능용으로 분리한 읽기 최적화된 뷰를 구축할 수 있다. 또한 기존 시스템을 수정할 필요가 없고 기존 시스템과 함께 운용이가능하다.

일반적으로 데이터에 어떻게 질의하고 접근하는지 신경 쓰지 않는다면 데이터 저장은 상당히 직관적인 작업이다. 스키마 설계, 색인, 저장소 엔진이 가진 복잡성은 특정 질의와 특정 접근 형식을 지원하기 위한 결과로 발생한다. 이런 이유로 데이터를 쓰는 형식과 읽는 현식을 분리해 다양한 읽기 뷰를 허용하는 `명령과 질의 책임의 분리 CQRS` 를 통해 상당한 유연성을 얻을 수 있다.

### 동시성 제어

이벤트 소싱과 CDC의 가장 큰 단점은 이벤트 로그의 소비가 대개 비동기로 이뤄진다는 점이며, 이는 상요자가 로그에 이벤트를 기록하고 이어서 로그에서 파생된 뷰를 읽어도 기록한 이벤트가 아직 반영되지 않았을 가능성이 있다.

해결책 하나는 읽기 뷰의 갱신과 로그에 이벤트를 추가하는 작업을 동기식으로 수행하는 방법이다.

이 방법을 쓰려면 트랜잭션에서 여러 쓰기를 원자적 단위로 결합해야 하므로 이벤트 로그와 읽기 뷰를 같은 저장 시스템에 담아야 하며, 다른 시스템에 있다면 분산 트챈잭션이 필요하다.

반면 이벤트 로그로 현재 상태를 만들면 동시성 제어 측면이 단순해진다. 이벤트 소싱을 사용하면 사용자 동작에 대한 설명을 자체적으로 포함하는 이벤트를 설계할 수있다. 그러면 사용자 동작은 한 장소에서 한 번 쓰기만 필요하며 이는 원자적으로 만들기 쉽다.(`이벤트 소싱 방식<-> 상태 기반 방식`)

- 각 인출, 입금, 이체 등의 동작에 대한 이벤트를 기록합니다.
- 계좌의 현재 잔액은 이벤트들을 통해 계산됩니다.
- 각 이벤트는 한 번만 기록되기 때문에 동시성 문제가 줄어들고, 각 이벤트는 원자적으로 처리될 수 있습니다.

에빈트 로그와 애플리케이션 상태를 같은 방식으로 파티셔닝하면 간단한 단일 스레드 로그 소비자는 쓰기용 동시성 제어는 필요하지 않다. 단지 단일 이벤트를 한 번에 하나씩 처리하며, 파티션 내에서 이벤트의 직렬 순서를 정의하면 로그에서 동시성의 비결정성을 제거할 수 있다.

### 불변성의 한계

영구적으로 모든 변화의 불변 히스토리를 유지하는 것은 데이터 셋이 뒤틀리는 양에 따라 다르다. 대부분 데이터를 추가하는 작업이고 갱신이나 삭제처럼 드물게 발생하는 작업부하는 불변으로 만들기 쉽다.

상대적으로 작은 데이터셋에서 매우 빈번히 갱신과 삭제를 하는 작업부하는 불변 히스토리가 감당히기 힘들정도로 커지거나, 파편화 문제가 발생할 수도 있다.

성능적인 이유 외에도 데이터가 모두 불변성임에도 관리상의 이유로 데이터를 삭제할 필요가 있는 상황일 수 있으며, 이런 상황에서는 이전 데이터를 삭제해야 한다는 또 다른 이벤트를 로그에 추가한다고 해결되지 않는다.

실제로 원하는 바는 히트로리를 새로 쓰고 문제가 되는 데이터를 처음부터 기록하지 않았던 것처럼 하는 것으로 데이토믹은 이 기능을 `적출` 이라 부르고 포씰 버전 관리 시스템에서는 `셔닝` 이라는 개념이 있다.

# 스트림 처리

스트림을 처리하는 방법 중 하나 이상의 입력 스트림을 처리해 하나 이상의 출력 스트림을 생산하며, 스트림은 최종 출력에 이르기까지 여러 처리 단계로 구성된 파이프라인을 통과할 수 도 있다.

이처럼 스트림을 처리하는 코드 조각을 `연산자` 나 `작업` 이라 부른다. 이는 유닉스 프로세스, 맵리듀스 작업과 밀접한 관련이 있다.

일괄처리 작업과 가장 크게 다른 점은 스트림은 끝나지 않는다는 점이며, 이 차이에는 여러 함축된 의미가 있다.

## 스트림 처리의 사용

스트림 처리는 특정 상황이 발생하면 조직에 경고를 주는 모니터링 목적으로 오랜 기간 사용돼 왔다.

- 사기 감시 시스템의 사용 패턴의 감지
- 거래 시스템의 금융 시장의 각겨변화 감지
- 제조 시스템의 공장의 기계상태 모니터링
- 군사 첩보 시스템의 잠재적 침략자의 활동 추적

이러한 종류의 애플리케이션은 상당히 복잡한 패턴 매칭과 상관 관계 규명이 필요했으나 시간이 흘러 다른 용도로 스트림 처리를 하는 사용자들이 나타나기 시작했다.

### 복잡한 이벤트 처리

복잡한 이벤트 처리(`complex event processing, CEP`) 는 특정 이벤트 패턴을 검색해야 하는 애플리케이션에 특히 적합하다. 정규 표현식으로 문자열에서 특정 문자 패턴을 찾는 방식과 유사하게 스트림에서 특정 이벤트 패턴을 찾는 규칙을 규정할 수 있다.

CEP 시스템에서의 질의와 데이터의 관계는 일반적 데이터베이스와 비교했을 때 반대다.

데이터베이스는 일반적으로 데이터를 영구적으로 저장하고 질의를 일시적으로 다룬다. CEP 엔진은 이역할을 서로 바꾼다. 질의는 오랜 기간 저장되고 입력 스트림으로부터 들어오는 이벤트는 지속적으로 질의를 지나 흘러 가면서 이벤트 패턴에 매칭되는 질의를 찾는다.

### 스트림 분석

스트림 처리를 사용하는 다른 영역으로 스트림 `분석(analytics)` 이 있다. CEP와 스트림 분석 사이의 경계는 불분명하지만 일반적으로 분석은 연속한 특정 이벤트 패턴을 찾는 것보다 대량의 이벤트를 집계하고 통계적 지표를 뽑는 것을 더 우선한다.

일반적으로 이런 통계는 고정된 시간 간격 기준으로 계산하며, 이러한 집계 시간 간격을 `윈도우` 라고 한다.

스트림 분석 시스템은 확률적 알고리즘을 사용하기도 하며, 집합 구성원 확인 용도의 블룸필터, 우너소 개수 추정 용도의 하이퍼로그로그, 다양한 백분위 추정 알고리즘 등이 있다.

확률적 알고리즘은 근사 결과를 제공하는 대신 스트림 처리자 내에서 차지하는 메모리가 정확한 알고리즘을 쓸 때보다 상당히 적다. 근사 알고리즘을 사용하다 보면 가끔 스트림 처리 시스템이 항상 데이터를 누락하거나 부정확하다고 생각하기 쉬운데 스트림 처리가 본질적으로 근사적인 것은 아니며, 확률적 알고리즘은 일종의 최적화 기법일 뿐이다.

### 구체화 뷰 유지하기

데이터베이스 변경에 대한 스트림은 캐시, 검색 색인, 데이터 웨어하우스 같은 파생 시스템이 원본 데이터베이스의 최신 내용을 따라 잡게 하는 데 쓸 수 있으며 이런 예들은 `구체화 뷰` 를 유지하는 특별한 사례로 볼 수 있다.

마찬가지로 이벤트 소싱에서 애플리케이션 상태는 이벤트 로그를 적용함으로써 유지된다. 여기서 애플리케이션 상태는 일종의 구체화 뷰다. 스트림 분석 시나리오와는 달리 어떤 시간 윈도우 내의 이벤트만 고려하면 보통 충분하지 않다. 구체화 뷰를 만들려면 잠재적으로 임의의 시간 범위에 발생한 모든 이벤트가 필요하다.

쌈자와 카프카 스트림은 카프카의 로그 컴팩션 지원을 기반으로 구체화 뷰 유지 용도로 사용할 수 있다.

### 스트림 상에서 검색하기

복수 이벤트로 구성된 패턴을 찾는 CEP 외에도 전문 검색 질의와 같은 복잡한 기준을 기반으로 개별 이벤트를 검색해야 하는 경우도 있다.

전통적인 검색 엔진은 먼저 문서를 색인하고 색인을 통해 질의를 신청한다. 반대로 스트림 검색은 처리 순서가 뒤집혀 질의를 먼저 저장하고 CEP와 같이 문서는 질의를 지나가면서 실행된다.

### 메시지 전달과 RPC

액터 모델 등에서 쓰이는 서비스 간 통신 메커니즘으로 메시지 전달 시스템을 사용할 수 있다. 이런 시스템은 메시지 와 이벤트에 기반을 두지만 일반적으로 이것들을 스트림 처리자로 생각하지 않는다.

유사 RPC 시스템과 스트림 처리 사이에 겹치는 영역이 있으며, 예를 들어 아파치 스톰에는 `분산 RPC`라 부르는 기능이 있다. 이 기능을 사용하면 이벤트 스트림을 처리하는 노드 집합에 질의를 맡길 수 있다. 이질의는 입력 스트림 이벤트가 끼워지고 그 결과들을 취합해 사용자에게 돌려준다.

### 시간에 관한 추론

스트림 처리자는 종종 시간을 다뤄야 할 때가 있다. 특히 분석 목적으로 사용하는 경우에 그러며, 이때 주로 `시간 윈도우를` 자주 사용하게 된다.

일괄처리와 다르게 스트림 처리 프레임 워크는 윈도우 시간을 결정할 때 처리하는 장비의시스템 시계를 이용한다.이 접근법은 간단하다는 장범이 있으나, 눈에 띌 정도로 처리가 지연되면, 즉 이벤트가 실제로 발생한 시간보다 처리 시간이 많이 늦어지면 문제가 생긴다.

### 이벤트 시간 대 처리 시간

처리가 지연되는 데는 많은 이유가 있다. 큐대기, 네트워크 결함, 성능 문제, 스트림 소비자의 재시작 등의 이유가 있다.

게다가 메시지가 지연되면 메시지 순서를 예측하지 못할 수도있다. 사람은 불연속에 대해 대처하는 능력이 있지만 스트림 처리 알고리즘은 그런 타이밍과 순서 문제를 처리하게끔 명확히 작성할 필요가 있다.

이벤트 시간과 처리 시간을 혼동하면 좋지 않은 데이터가 만들어 진다. 예를 들어 요청률을 측정하는 스트림에서 스트림 처리자가 재배포 되면 실제 요청률은 안정적일지 모르나, 백로그를 처리하는 동안 요청이 비정상적으로 튀는 것처럼 보이게 된다.

### 준비 여부 인식

이벤트 시간 기준으로 윈도우를 정의할 때 발생하는 까다로운 문제는 특정 윈도우에서 모든 이벤트가 도착했다거나 아직도 이벤트가 계속 들어오고 있는지를 확신할 수 없다는 점이다.

윈도우를 이미 종료한 후에 도착한 `낙오자` 이벤트를 처리할 방법이 필요하며, 크게 두 가지 방법이 있다.

- 낙오자 이벤트는 무시한다.(적은 비율을 차지하기 때문) 놓친 이벤트가 많아질 경우 경고를 보낼 수 있다.
- 수정 값을 발행한다. 수정 값은 낙오자 이벤트가 포함된 윈도우 기준으로 갱신된 값이다. 또한 이전 출력을 취소해야 할 수도 있다.

### 어쨌든 어떤 시계를 사용할 것인가?

이벤트가 시스템의 여러 지점에 버퍼링 됐을 때 이번트에 타임스탬프를 할당하는 것은 더 어렵다.

이런 맥락에서 이벤트의 타임스탬프는 실제 사용자와 상호작용이 발생했던 실제 시각이어야 한다. 그러나 우연히 또는 고의로 잘못된 시간이 설정됐을 가능성이 있기 때문에 사용자가 제어하는 장비의 시계를 항상 신뢰하기는 어렵다.

잘못된 장치 시계를 조정하는 한 가지 방법은 세 가지 타임스탬프를 로그로 남기는 것이다.

- 이벤트가 발생한 시간 → 장치 시계를 따른다.
- 이벤트를 서버로 보낸 시간 → 장치시계를 따른다.
- 서버에서이벤트를 받은 시간 → 서버 시계를 따른다.

두 번째와 세번째의 타임스탬프 차이를 구하면 장치 시계와 서버 시계간의 오프셋을 추정할 수 있다. 그러면 계산한 오프셋을 이벤트 타임스탬프에 적용해 이벤트가 실제로 발생한 시간을 추정할 수 있다. 이 문제는 일괄 처리에서도 동일한 문제가 발생하지만 스트림 처리를 할 때는 일괄 처리 때보다 시간의 흐름을 더 잘 알 수 있기 때문에 두드러질 뿐이다.

### 윈도우 유형

윈도우는 이벤트 수를 세거나 윈도우 내 평균값을 구하는 등 집계를 할 때 사용한다.

- 텀블링 윈도우
- 홉핑 윈도우
- 슬라이딩 윈도우
- 세션 윈도우

## 스트림 조인

스트림 처리는 데이터 파이프라인을 끝이 없는 데이터셋의 증분 처리로 일반화하기 때문에 스트림에서도 조인에 대한 필요성은 정확히 동일하다.

스트림 상에서 새로운 이벤트가 언제든 나타날 수 있다는 사실은 스트림상에서 수행하는 조인을 일괄 처리 작업에서 수행하는 조인보다 훨씬 어렵게 만든다.

### 스트림 스트림 조인(윈도우 조인)

웹사이트에 검색된 URL의 최신 경향을 파악한다고 할때, 사용자가 검색 결과를 쓰지 않고 버리거나, 클릭이 발생했더라도 검색과 클릭 사이의 시간은 매우 가변적일 수 있다. 네트워크 지연도 가변적이기 때문에 클릭 이벤트가 검색 이벤트보다 먼저 도착할 수 있다. 그래서 조인을 위한 적절한 윈도우 선택이 필요하다.

검색 품질을 측정하기 위해서는 정확한 클릭율이 필요하며 클릭율을 구하려면 검색 이벤트와 클릭 이벤트 모두가 필요하다.

이런 유형의 조인을 구현하려면 스트림 처리자가 상태를 유지해야한다. 예를 들어 지난 시간에 발생한 모든 이벤트를 세션 ID로 색인하고, 검색 이벤트나 클릭 이벤트가 발생할 때마다 해당 색엔에 추가하고 스트림 처리자는 같은 세션 ID로 이미 도착한 다른 이벤트가 있는지 다른 색인을 확인해야 한다.

### 스트림 테이블 조인(스트림 강화)

사용자 활동 이벤트 집합과 사용자 프로피 ㄹ데이터베이스를 조인하는 일괄 처리와 같은 상황에서

사용자 활동 이벤트를 스트림을 ㅗ간주하고 스트림 처리자에서 동일한 조인을 지속적으로 수행하는 게 자연스럽다.

이때 입력은 사용자 ID를 포함한 활동 이벤트 스트림이고 출력은 해당 ID를 가진 사용자 프로필 정보가 추가된 활동 이벤트다. 이과정을 데이터베이스의 정보로 활동 이벤트를 `강화` 한다고 한다.

이 조인을 수행하기 위해 스트림 처리는 한 번에 하나의 활동 이벤트를 대상으로 데이터베이스에서 이벤트의 사용자 ID를 찾아 활동 이벤트에 프로필 정보를 추가해야한다. 데이터베이스 탐색은 원격 데이터베이스를 질의 하게끔 구현할 수 있으나, 이러한 원격 질의는 느리고 과부화를 줄 위험이 있다.

또 다른 방법은 네트워크 왕복 없이 로컬에서 질의가 가능하도록 스트림 처리자 내부에 데이터베이스 사본을 적재하는 것이다. 이방법은 해시 조인과 상당히 비슷하며, 데이터베이스 로컬 사본 용량이 충분히 작으면 메모리 내 해시 테이블에 넣는 것이 가능하다.

### 테이블 테이블 조인(구체화 뷰 유지)

사용자가 팔로우한 사람 모두를 순회하며 최근 트윗을 찾아 그것을 병합하는 데는 너무 많은 비용이 든다. 이를 위해 일종의 상용자별 `받은 편지함` 과 같은 타임라인 캐시를 사용한다.

스트림 처리자에서 이 캐시 유지를 구현하려면 트윗 이벤트 스트림(전송과 삭제)과 팔로우 관계 이벤트 스트림이 필요하다. 스트림 처리는 새로운 트윗이 도착했을 때 어떤 타임 라인을 갱신해야 하는지 알기 위해 각 사용자의 팔로우 집합이 포함된 데이터베이스를 유지해야 한다.

### 조인의 시간 의존성

위 세 가지 조인 유형은 스트림 처리자가 하나의 조인 입력을 기반으로 한 특정 상태를 유지해야 하고 다른 조인 입력에서 온 메시지에 그 상태를 질의하는 공통점이 있다.

상태를 유지하는 이벤트의 순서는 매우 중요하다. 파티셔닝된 로그에서 단일 파티션 내 이벤트 순서는 보존되지만 다른 스트림이나 다른 파티션 사이에서 보장하는 일반적인 방법은 없다.

복수 개의 스트림에 걸친 이벤트 순서가 결정되지 않으면 조인도 비결정적이다. 이것은 동일한 입력으로 같은 작업을 재수행하더라도 반드시 같은 결과를 얻지는 못한다는 의미다.

이 문제를 데이터 웨어하우스에서는 `천천히 변하는 차원(SCD)` 이라 한다. 이문 제는 흔히 조인되는 레코드의 특정 버전을 가리키는 데 유일한 식별자를 사용해 해결한다.

## 내결함성

스트림 처리에서도 내결함성 문제가 발생한다. 그러나 이 문제를 다루는 방법은 덜 직관적이다. 출력을 노출하기 전에 태스크가 완료될  때까지 기다리는 것은 해결책으로 사용할 수없다.  스트림은 무한하고 그래서 처리를 절대 완료할 수 없기 때문이다.

### 마이크로 일괄 처리와 체크포인트

한 가지 해결책은 스트림을 작은 블록으로 나누고 각 블록을 소형 일괄 처리와 같이 다루는 방법이다. 이를 `마이크로 일괄 처리` 라고 한다.

스트림 처리 프레임워크 내에서 마이크로 일괄 처리와 체크포인트 접근법은 일괄 처리와 같이 정확히 한 번 시맨틱을 지원한다. 그러나 출력이 스트림 처리자를 떠나자 마자 스트림 처리 프레임 워크는 실패한 일괄 출력을 더 이상 지울 수 없다. 이경우 실패한 태스크를 재시작하면 외부 부 수 효과가 두 번 발생한다. 마이크로 일괄 처리와 체크포인트 접근법만으로든 이 문제를 방지하기에 충분하지 않다.

### 원자적 커밋 재검토

장애가 발생했을 때 정확히 한 번 처리되는 것처럼 보이려면 처리가 성공 했을 때만 모든 출력과 이벤트 처리의 부수 효과가 발생하게 해야한다.

이런 효과는 원자적으로 모두 일어나거나 또는 모두 일어나지 않아야 하지만 서로 동기화가 깨지면 안된다.

### 멱등성

결국 목표는 처리 효과가 두 번 나타나는 일 없이 안전하게 재처리하기 위해 실패한 태스크의 부분 출력을 버리는 것이다. 분산 트랜잭션이 이 목표를 달성하는 한 가지 방법이지만 그 밖의 다른 방법으로 `멱등성`에 의존하는 방법이 있다.

멱등 연산은 여러 번 수행하더라도 `오직 한 번 수행한 것과 같은 효과` 를 내는 연산이다. 연산 자체가 멱등적이지 않아도 약간의 여분 메타데이터로 연산을 멱등적으로 만들 수 있다.  예를 들어 카프카로부터 메시지를 소비할 때 모든 메시지에는 영속적이고 단조 증가하는 오프셋이 있다.

외부 데이터베이스에 값을 기록할 때 마지막으로 그 값을 기록하라고 트리거한 메시지의 오프셋을 함께 포함한다면 이미 갱신이 적용됐는지 확인할 수 있기 때문에 반복해서 같이 갱신이 수행되는 것을 막을 수 있다.

### 실패 후에 상태 재구축 하기

윈도우 집계나 조인용 테이블과 색인처럼 상태가 필요한 스트림 처리는 실패 후에도 해당 상태가 복구됨을 보장해야 한다.

한 가지 방법은 원격 데이터 저장소에 상태를 유지하고 복제하는 것이다. 다만 개별 메시지를 원격 데이터베이스에 질의하는 것은 느리다.

다른 방법으로는 스트림 처리자의 로컬에 상태를 유지하고 주기적으로 복제하는 것이다. 그러면 스트림 처리자가 실패한 작업을 복구할 때 새 태스크는 복제된 상태를 읽어 데이터 손실 없이 처리를 재개할 수 있다.

그러나 이 모든 트레이드 오프는 기반 인프라스트럭처의 성능 특성에 따라 달려있다. 모든 상황을 만족하는 이상적인 트레이드오프는 없다. 로컬 상태 대 원격 상태의 가치는 저장소와 네트워크 기술의 발전에 따라 얼마든지 바뀔 수 있다.
