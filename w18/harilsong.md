# 일괄 처리

시스템을 구축하는 방식에는 아래와 같은 3가지 방식이 자주 사용된다.

- 서비스(온라인 시스템)
- **일괄 처리 시스템(오프라인 시스템)**
- 스트림 처리 시스템(준실시간 시스템)

이 중 일괄 처리는 신뢰할 수 있고 확장 가능하며 유지보수하기 쉬운 애플리케이션을 구축하는 데 매우 중요한 구성요소.

일괄 처리 알고리즘인 **맵리듀스(MapReduce)** 는 "구글을 대규모로 확장 가능하게 만든 알고리즘"이라 불렸다. 이후 [[Hadoop|Hadoop]], [[Couch DB]], [[MongoDB]] 등 다양한 오픈소스 시스템에서 구현됐다.

## 유닉스 도구로 일괄 처리하기

### 단순 로그 분석

```bash
cat /var/log/nginx/access.log | # 로그를 읽는다
    awk '{print $7}' | # 줄마다 공백으로 분리해서 7번째 필드만 출력
    sort | # 알파벳 순으로 정렬
    uniq -c | # 중복을 제거하며 중복 횟수를 함께 출력
    sort -r -n | # -n 옵션을 사용하여 첫 번째로 나타나는 숫자를 기준으로 내림차순(-r) 정렬
    head -n 5. # 앞에서부터 5줄만(-n 5) 출력
```

이런 유닉스 명령 방식은 짧지만 상당히 강력하다. 수 기가바이트의 로그 파일을 수 초 내로 처리할 수 있고, 필요에 따라 분석 방법을 수정하기 쉽다.

```bash
# 설치된 formula 중에서 원하는 목록을 리스트업하기, uninstall 의 인자로 전달하면 한 번에 삭제도 가능하다
brew list --formula | fzf -m >> ~/selected
```

#### 연쇄 명령 대 맞춤형 프로그램

```ruby
counts = Hash.new(0)

File.open('/var/log/nginx/access.log') do |file|
    file.each do |line|
        url = line.split[6]
        counts[url] += 1
    end
end

top5 = counts.map{|url, count| [count, url] }.sort.reverse[0...5]
top5.each{|count, url| puts "#{count} #{url}"}
```

이 프로그램은 유닉스 파이프보다 간결하지는 않지만 더 읽기 쉽다. 그러나 두 가지 방법은 실행 흐름이 크게 다르다. 대용량 파일을 분석해보면 차이가 확연히 드러난다.

#### 정렬 대 인메모리 집계

앞선 루비 스크립트는 URL 해시 테이블을 메모리에 유지한다. 해시 테이블에는 각 URL 이 출현한 수를 매핑한다. 유닉스 파이프라인 예제에는 이런 해시 테이블이 없다. 대신 정렬된 목록에서 같은 URL 이 반복해서 나타난다.

- 중소 규모의 웹사이트 대부분은 고유 URL 과 해당 URL 카운트를 대략 1GB 메모리에 담을 수 있음
- 작업 세트(임의 접근이 필요한 메모리 양)이 충분히 작다면 인메모리 해시테이블도 잘 작동

**허용 메모리보다 작업 세트가 크다면 정렬 접근법**을 사용하는 것이 좋다.

1. 데이터 청크를 메모리에서 정렬하고 청크를 세그먼트 파일로 디스크에 저장
2. 각각 정렬된 세그먼트 파일 여러 개를 한 개의 큰 정렬 파일로 병합
3. 병합 정렬은 순차적 접근 패턴을 따르고 이 패턴은 디스크 상에서 좋은 성능을 보여줌

GNU Coreutils(리눅스)에 포함된 sort 유틸리티는 메모리보다 큰 데이터셋을 자동으로 디스크로 보내고 자동으로 여러 CPU 코어에서 병렬로 정렬한다. 따라서 유닉스 연쇄 명령이 메모리 부족 없이 큰 데이터 셋으로 확장 가능하며, 병목이 있다면 디스크에서 입력 파일을 읽는 속도일 것이다.

### 유닉스 철학

1. 각 프로그램이 한 가지 일만 하도록 작성하라. 새 작업을 하려면 새로운 프로그램을 작성하라.
2. 모든 프로그램의 출력은 다른 프로그램의 입력으로 쓰일 수 있다고 생각하라.
3. 소프트웨어를 빠르게 써볼 수 있도록 설계하고 구축하라.
4. 프로그래밍 작업을 줄이려면 미숙한 도움보단 도구를 사용하라. 사용 후 바로 버린다고 할지라도 도구를 써라.

sort 는 이 훌륭한 예다. sort 의 구현은 대부분의 프로그래밍 언어에 포함된 표준 라이브러리보다 확실히 뛰어나다. 단독으로는 그렇게 유용하지 않지만, 다른 도구(uniq 등)와 결합할 때 비로소 강력해진다.

bash 같은 유닉스 셸을 사용하면 작은 프로그램들을 가지고 놀랄 만큼 강력한 데이터 처리 작업을 쉽게 구성할 수 있다. 많은 도구가 서로 다른 그룹, 사용자에 의해 만들어졌음에도 유닉스에 이런 결합성을 부여하는 것은 무엇일까?

#### 동일 인터페이스

어떤 프로그램의 출력을 다른 프로그램의 입력으로 쓰고자 한다면 이들 프로그램은 같은 데이터 형식을 사용해야 한다. 특정 프로그램이 다른 어떤 프로그램과도 연결 가능하려면 프로그램 모두가 같은 입출력 인터페이스를 사용해야 한다.

**유닉스에서의 인터페이스는 파일(좀 더 정확히는 파일 디스크립터)** 이다. 파일은 단지 순서대로 정렬된 바이트의 연속이다. 다음과 같은 것들을 표현할 수 있다.

- 파일시스템의 실제 파일
- 프로세스 간의 통신 채널(유닉스 소켓, 표준 입력, 표준 출력)
- 장치 드라이버
- TCP 연결을 나타내는 소켓

이 정도로 매끄럽게 협동하는 프로그램이 있다는 건 아주 예외적이다. 동일한 데이터 모델인 데이터베이스 간에도 한쪽에서 다른 쪽으로 데이터를 옮기는 게 쉽지 않다.

#### 로직과 연결의 분리

유직스 도구의 다른 특징으로는 표준 입력(stdin)과 표준 출력(stdout)을 사용한다는 점. 파이프는 한 프로세스의 stdout 을 다른 프로세스의 stdin 과 연결한다. 이 때 **중간 데이터를 디스크에 쓰지 않고 작은 인메모리 버퍼를 사용해 프로세스 간 데이터를 전송**한다.

프로그램에서 입출력을 연결하는 부분을 분리하면 작은 도구로부터 큰 시스템을 구성하기가 훨씬 수월하다.

프로그램을 직접 작성해 운영체제에서 지원하는 도구와 같이 사용할 수 있다. 프로그램을 단순히 stdin 으로부터 입력을 읽어 stdout 으로 출력하도록 작성하면 데이터 처리 파이프라인에 바로 끼워 사용할 수 있다.

#### 투명성과 실험

- 유닉스 명령에 들어가는 입력 파일은 일반적으로 불변으로 처리된다.
- 어느 시점이든 파이프라인을 중단하고 출력을 파이프를 통해 less 로 보내 원하는 형태의 출력이 나오는지 확인할 수 있다.
- 특정 파이프라인 단계의 출력을 파일에 쓰고 그 파일을 다음 단계의 입력으로 사용할 수 있다.

유닉스 도구는 관계형 데이터베이스의 질의 최적화 도구와 비교하면 상당히 불친절하고 단순하지만, 놀라울 정도로 유용해서 실험용으로 매우 좋다.

유닉스 도구를 사용하는데 **가장 큰 제약은 단일 장비에서만 실행된다는 점**이다. 바로 이 점이 [[Hadoop]] 같은 도구가 필요한 이유다.

## 맵리듀스와 분산 파일 시스템

- 유닉스 도구와 비슷한 면이 있지만 수천 대의 장비로 분산해서 실행이 가능하다는 점에서 차이.
- 분산 파일 시스템 상의 파일을 입력과 출력으로 사용

하둡 맵리듀스 구현에서 이 파일 시스템은 HDFS(Hadoop Distributed File System)라고 하는데 GFS(Google File System)를 재구현한 오픈소스다.

HDFS 는 **비공유 원칙을 기반**으로 하는데 NAS(Network Attached Storage)와 SAN(Storage Area Network) 아키텍처에서 사용하는 공유 디스크 방식과는 반대다.

- 각 장비에서 실행되는 데몬 프로세스로 구성
- 네임노드(NameNode)라고 부르는 중앙 서버는 특정 파일 블록이 어떤 장비에 저장됐는지 추적
- 파일 블록은 여러 장비에 복제

### 맵리듀스 작업 실행하기

1. 입력 파일을 읽어 레코드로 쪼갠다.
2. 각 입력 레코드마다 매퍼 함수를 호출해 키와 값을 추출한다.
3. 키를 기준으로 키-값 쌍을 모두 정렬한다.
4. 정렬된 키-값 쌍 전체를 대상으로 리듀스 함수를 호출한다.

맵리듀스 작업 하나는 4가지 단계로 수행

2단계(맵)과 4단계(리듀스)는 사용자가 직접 작성한 데이터 처리 코드

1단계는 입력 형식 파서를 사용하고 3단계는 정렬 단계로 맵리듀스에 내재하는 단계라서 직접 작성할 필요가 없다. 매퍼의 출력은 리듀서로 들어가기 전에 이미 정렬됐다.

맵리듀스 작업을 생성하려면 매퍼와 리듀서라는 두 가지 콜백 함수를 구현해야 한다.

- 매퍼(Mapper): 모든 입력 레코드마다 한 번씩만 호출. 키와 값을 추출하는 작업.
- 리듀서(Reducer): 맵리듀스 프레임워크는 키-값 쌍을 받아 같은 키를 가진 레코드를 모으고 해당 값의 집합을 반복해 리듀서 함수를 호출. 리듀서는 출력 레코드를 생산.

#### 맵리듀스의 분산 실행

유닉스 명령어 파이프라인과의 가장 큰 차이점

- 맵리듀스가 병렬로 수행하는 코드를 직접 작성하지 않고도 여러 장비에서 동시에 처리가 가능하다는 점

매퍼와 리듀서는 한 번에 하나의 레코드만 처리하고 입력이 어디서 오는지, 출력이 어디로 가는지 신경 쓰지 않는다. 데이터가 이동하는 복잡한 부분은 맵리듀스 프레임워크가 처리한다.

- 맵리듀스 작업의 병렬 실행은 파티셔닝을 기반으로 한다.
- 작업 입력으로는 HDFS 상의 디렉터리를 사용하는 것이 일반적.
- 입력 파일은 보통 크기가 수백 메가바이트에 달한다.
- 매퍼 입력 파일의 복제본이 있는 장비에 RAM 과 CPU 에 여유가 충분하다면 맵리듀스 스케쥴러는 입력 파일이 있는 장비에서 작업을 수행하려 한다. -> 이 원리를 **데이터 가까이에서 연산**하기라 하는데, 네트워크를 통해 파일을 복사하는 부담을 줄이고 지역성이 증가한다.

맵리듀스 프레임워크는 같은 키를 가진 모든 키-값 쌍을 같은 리듀서에서 처리하는 것을 보장하는데, 어느 리듀스 태스크에서 수행될지 결정하기 위해 키의 해시값을 사용한다.

키-값 쌍은 반드시 정렬돼야 하지만 대게 데이터셋이 매우 크기 때문에, 단계를 나누어 정렬을 수행한다.

1. 각 맵 태스크는 키의 해시값을 기반으로 출력을 리듀서로 파티셔닝한다.
2. 각 파티션을 매퍼의 로컬 디스크에 정렬된 파일로 기록한다.
3. 매퍼가 입력 파일을 읽어서 정렬된 출력파일을 기록하기를 완료하면 맵리듀스 스케줄러는 그 매퍼에서 출력 파일을 가져올 수 있다고 리듀서에게 알려준다.
4. 리듀서는 각 매퍼와 연결해서 리듀서가 담당하는 파티션에 해당하는 정렬된 키-값 쌍 파일을 다운로드한다.
5. 리듀스 태스크는 매퍼로부터 파일을 가져와 정렬된 순서를 유지하면서 병합한다.
6. 리듀서는 레코드들을 처리하고 출력 레코드는 분산 파일 시스템에 파일로 기록된다.

#### 맵리듀스 워크플로

맵리듀스 작업 하나로 해결할 수 있는 문제의 범위는 제한적이다.

따라서 맵리듀스 작업을 연결해 **워크플로(workflow)** 로 구성하는 방식이 일반적이다. 하둡 맵리듀스 프레임워크는 워크플로를 직접 제공하지 않기 때문에 맵리듀스 작업은 디렉터리 이름을 통해 암묵적으로 연결된다.

```mermaid
flowchart LR
    A -- out --> dir -- in --> B
```

맵리듀스 관점에서 보면 두 작업은 완전히 독립적이다.

연결된 맵리듀스 작업은 유닉스 명령 파이프라인(소량의 메모리 버퍼를 사용해 한 프로세스의 출력이 다른 프로세스의 입력으로 직접 전달되는 방식)과 유사하다기보다는 각 명령의 출력을 임시 파일에 쓰고 다음 명령이 그 임시 파일로부터 입력을 읽는 방식에 더 가깝다.

일괄 처리 작업의 출력은 작업이 성공적으로 끝났을 때만 유효하다(작업이 실패하고 남은 부분 출력은 제거한다). 그렇기 때문에 워크플로 상에서 해당 작업의 입력이 완전히 끝나야만 다음 작업을 실행할 수 있다. 하둡 맵리듀스 작업 간 수행 의존성을 관리하기 위해 다양한 스케쥴러가 개발됐다. 스케쥴러의 예로는

- [[Oozie]]
- [[Azkaban]]
- [[Luigi]]
- [[Airflow]]
- [[Pinball]]

등이 있다.

### 리듀스 사이드 조인과 그룹화

연관된 레코드 양쪽 모두에 접근해야하는 코드가 있다면 조인은 필수. 비정규화 작업으로 조인을 줄일 수 있지만 일반적으로 완전히 제거하기는 어렵다.

데이터베이스에서 적은 수의 레코드만 관련된 질의를 실행한다면 일반적으로 색인([[Index]])을 사용해 빠르게 찾을 수 있다. 질의가 조인을 포함하면 여러 색인을 확인해야 할지도 모른다. 하지만 맵리듀스에는 일반적으로 이야기하는 색인 개념이 없다.

파일 집합이 입력으로 주어진 맵리듀스 작업은 입력 파일 전체 내용을 읽는데 데이터베이스에서 이 연산을 **전체 테이블 스캔(full table scan)** 이라 부른다. 적은 수의 레코드를 읽고 싶을 때 전체 테이블 스캔은 굉장히 비효율적이지만, 집계 연산을 위해 분석 질의를 해야하는 경우는 입력 전체를 스캔하는 것이 합리적이다. 여러 장비에 걸쳐 병렬 처리가 가능한 경우 특히 그렇다.

일괄 처리 맥락에서 조인은 데이터셋 내 모든 연관 관계를 다룬다는 뜻이다. 특정 사용자의 데이터만 찾는다면 색인을 사용하는 편이 훨씬 효율적이다.

#### 사용자 활동 이벤트 분석 예제

**각각의 활동 이벤트를 훑으면서 나오는 모든 사용자 ID 마다 데이터베이스에 질의 보내기**

- 데이터베이스와 통신할 때 발생하는 왕복 시간으로 처리량이 제한
- 지역캐시의 효율성이 데이터 분포에 크게 좌우된다
- 많은 질의가 병렬로 실행되면 데이터베이스에 과부하가 걸리기 쉽다

일괄 처리에서 처리량을 높이기 위해서는 가능한 한 장비 내에서 연산을 수행해야 한다. 네트워크로 발생할 수 있는 모든 문제로부터 최대한 자유로워져야 한다.

- 네트워크는 느리다
- 네트워크는 비결정적이다(어떤 연산이 정상적으로 종료될거라고 확신할 수 없다)
- 원격 데이터베이스의 데이터가 변경될 수 있다

따라서 데이터베이스의 사본을 가져와 사용자 활동 이벤트 로그가 저장된 분산 파일 시스템에 넣는 방법이 더 좋다. 그러면 사용자 데이터베이스와 사용자 활동 레코드가 같은 HDFS 상에 존재하고 맵리듀스를 사용해 연관된 레코드끼리 모두 같은 장소로 모아 효율적으로 처리가 가능하다. (=데이터 웨어하우징을 하는 이유)

#### 정렬 병합 조인

설명이 제대로 이해 안됨...

#### 같은 곳으로 연관된 데이터 가져오기

매퍼가 키-값 쌍을 내보낼 때 키는 목적지의 주소 역할을 한다. 같은 키를 가진 키-값 쌍은 모두 같은 목적지로 배달된다(같은 리듀서를 호출한다).

- 사용자의 ID 등으로 키를 만든다면 같은 사용자는 항상 같은 리듀서로 모이기 때문에 빠른걸까?
- 조인을 위해 필요한 데이터가 파편화되지 않고 같은 곳에 있게 된다면 확실히 편할 것 같긴 하다.

#### 그룹화

맵리듀스로 그룹화 연산을 구현하는 가장 간단한 방법은 매퍼가 **키-값 쌍을 생성할 때 그룹화할 대상을 키로 하는 것**이다.

- 각 그룹의 레코드 수를 카운트
- 특정 필드 내 모든 값을 더하기
- 어떤 랭킹 함수를 실행했을 때 상위 k개의 레코드 고르기

사용자 세션별 활동 이벤트를 수집 분석할 때도 일반적으로 그룹화를 사용하며 이 과정을 세션화(sessionization)라고 한다.

사용자 요청을 받는 웹서버가 여러 개라면 특정 사용자의 활동 이벤트는 여러 서버로 분산돼 각각 다른 로그 파일에 저장된다. 세션 쿠기, 사용자 ID 나 유사한 식별자를 그룹화 키로 사용해 특정 사용자 활동 이벤트를 몯 후나 곳으로 모으면 세션화를 구현할 수 있다. 서로 다른 사용자의 이벤트는 다른 파티션으로 분산된다.

#### 쏠림 다루기

키 하나에 너무 많은 데이터가 연관된다면 "같은 키를 가지는 모든 레코드를 같은 장소로 모으는" 패턴은 제대로 작동하지 않는다.

- 린치핀 객체 또는 핫 키: 불균형한 활성 데이터베이스 레코드

한 리듀서가 다른 리듀서보다 엄청나게 많은 레코드를 처리해야 한다면, 맵리듀스 작업에 상당한 지연이 발생할 수 있다. 맵리듀스 작업은 모든 매퍼와 리듀서가 완전히 끝나야지만 끝나기 때문에 가장 느린 리듀서가 작업을 완료할 때까지 후속 작업들은 시작하지 못한채 기다려야 한다.

조인 입력에 핫 키가 존재하는 경우에 핫스팟을 완화할 몇 가지 알고리즘이 있다.

- 쏠린 조인(skewed join): 핫 키를 여러 리듀서에 퍼뜨려서 처리하게 하는 방법

키의 해시값을 기반으로 리듀서를 결정하는 관습적인 맵리듀스와는 반대로, 샘플링을 통해 어떤 키가 핫 키인지 결정하고 여러 리듀서 중 임의로 선택한 하나로 레코드를 보낸다.

- 공유 조인(shared join): 쏠린 조인과 비슷하지만 샘플링 작업 대신 핫 키를 명시적으로 지정한다.

핫 키로 레코드를 그룹화하고 집계하는 작업은 두 단계로 수행된다.

1. 레코드를 임의의 리듀서로 보낸다. 각 리듀서는 핫 키 레코드의 일부를 그룹화하고 키별로 집계해 간소화한 값을 출력한다.
2. 첫 단계 모든 리듀서에서 나온 값을 키별로 모두 결합해 하나의 값으로 만든다.

### 맵 사이드 조인

리듀스 사이드 조인의 장단점

- 입력 데이터에 대한 특정 가정이 필요 없다
- 정렬 후 리듀서로 복사한 뒤 입력을 병합하는 과정에 드는 비용이 매우 크다
- 맵리듀스 단계를 거칠 때 허용된 메모리 버퍼에 따라 데이터를 여러 번 디스크에 저장해야 할 수도 있다

**입력 데이터에 대해 특정 가정이 가능하다면 맵사이드 조인(map-side join)** 으로 불리는 기법을 사용해 조인을 더 빠르게 수행할 수 있다.

- 축소된 맵리듀스 작업
- 정렬 작업이 필요없다
- 리듀서도 필요없다
- 매퍼는 분산 파일 시스템에서 입력 파일 블록 하나를 읽어 다시 출력하는 것이 전부다
- 작은 데이터셋과 매우 큰 데이터셋을 조인하는 경우에 간단하게 적용할 수 있다(브로드캐스트 해시 조인)

#### 브로드캐스트 해시 조인

> 작은 데이터셋과 매우 큰 데이터셋을 조인하는 경우

작은 데이터셋은 존체를 각 매퍼 메모리에 적재 가능할 정도로 충분히 작아야 한다.

작은 데이터셋을 메모리에 올린 후, 큰 데이터셋을 읽으며 ID 를 메모리에서 조회하며 조인한다.

작은 입력을 큰 입력의 파티션 하나를 담당하는 매퍼에 "브로드캐스트"한다. 결과적으로 큰 입력의 모든 파티션에 작은 입력이 전달된다.

#### 파티션 해시 조인

맵 사이드 조인의 입력을 파티셔닝한다면 해시 조인 접근법을 각 파티션에 독립적으로 적용할 수 있다...?

여기서부터 잘 이해 안됨...

#### 맵 사이드 병합 조인

#### 맵 사이즈 조인을 사용하는 맵리듀스 워크플로

맵 사이드 조인을 수행하기 위해서는 쿠기, 정렬, 입력 데이터의 파티셔닝 같은 제약사항이 따른다.

### 일괄 처리 워크플로의 출력

일괄 처리의 출력은 분석에 가깝지만 분석 목적으로 사용하는 SQL 질의와는 다르다. 일괄 처리의 출력은 흔히 보고서가 아닌 다른 형태의 구조다.

#### 검색 색인 구축

맵리듀스는 구글에서 검색 엔진에 사용할 색인을 구축하기 위해 처음 사용

정해진 문서 집합을 대상으로 전문 검색(full-text search)이 필요하다면 일괄 처리가 색인을 구축하는 데 매우 효율적이다. 매퍼는 필요에 따라 문서 집합을 파티셔닝하고 각 리듀서가 해당 파티션에 대한 색인을 구축한다. 문서 기준으로 파티셔닝해 색인을 구축하는 과정은 병렬화가 매우 잘된다. 키워드로 검색 색인에 질의하는 연산은 읽기 전용이라서 색인 파일을 한 번 생성하면 불변이다.

#### 일괄 처리의 출력으로 키-값을 저장

일괄 처리는 다음과 같은 시스템을 구축할 수도 있다.

- 스팸 필터
- 이상 검출
- 이미지 인식
- 추천 시스템

이런 일괄 처리 작업의 출력은 흔히 일종의 데이터베이스가 된다. 이 같은 데이터베이스는 하둡 인프라와 별도로 사용자 요청을 받는 웹 애플리케이션에서 질의해야 한다. 배치 프로세스의 출력을 웹 애플리케이션이 질의하는 데이터베이스로 보내는 방법이 있을까?

가장 확실한 방법은 일괄 처리 작업이 한번에 레코드 하나씩 데이터베이스 서버로 직접 요청을 보내는 것이다. 이 방법은 유효하지만 좋은 아이디어는 아니다. 이유는 아래와 같다.

- 네트워크는 느리다.
- 맵리듀스 작업은 대개 많은 태스크를 동시에 실행한다. 맵리듀스에 기대하는 속도로 데이터베이스에 기록한다면 데이터베이스가 과부하에 빠지기 쉽다.
- 외부에 의존하게 됨으로서 사이드 이펙트가 발생한다.

훨씬 좋은 해결책은 **일괄 처리 작업 내부에 완전히 새로운 데이터베이스를 구축해 분산 파일 시스템의 작업 출력 디렉토리에 저장하는 방법**으로 이때 검색 색인과 유사한 구조로 저장한다.

#### 일괄 처리 출력에 관한 철학

- 입력은 언제나 **불변**이며 새출력이 이전 출력을 완벽하게 교체한다. 이는 부수효과를 방지하고 얼마든지 재실행이 가능하게 한다.
- 실패한 출력은 폐기되고, 재시도 관련 스케쥴링은 맵리듀스 프레임워크에 의해 처리된다.
- 동일한 파일 집합이 사용되곤 한다.
- 입출력 디렉토리를 설정하는 등의 연결 작업과 비즈니스 로직을 분리한다.

### 하둡과 분산 데이터베이스 비교

#### 저장소의 다양성

데이터베이스는 특정 모델(예. 관계형 또는 문서형)을 따라 데이터를 구조화해야 한다. 반면 분산 파일시스템의 파일은 어떤 데이터 모델과 인코딩을 사용해서도 기록할 수 있는 연속된 바이트일 뿐이다.

어떤 형태라도 상관없이 HDFS 로 덤프할 수 있는 이 기능은, 데이터를 최대한 빨리 한 곳으로 모으는 걸 가능하게 한다. 양질의 데이터는 데이터 사용자 관점에서 가치있는 일이지만, 현실에서는 데이터를 최대한 빨리 사용 가능하게 만드는 것이 더 가치 있다.

이 아이디어는 데이터 웨어하우스의 개념과 유사하다.

#### 처리 모델의 다양성

대규모 병렬처리, MPP(Massively parallel processing) 데이터베이스는 일체식 구조로서 디스크 저장소 레이아웃과 질의 계획, 스케쥴링과 실행을 다루는 소프트웨어 조각들이 긴밀하게 통합된다. 덕분에 설계된 질의 유형에 대해서는 매우 좋은 성능을 얻을 수 있다.

반면 SQL 질의로 모든 종류의 처리를 표현하지는 못한다. 머신러닝이나 추천 시스템 또는 관련성 랭킹 모델을 탑재한 전문 검색 색인을 구축하거나 이미지 분석을 한다면 더 범용적인 데이터 처리 모델이 필요하다. 이런 처리는 단순한 질의 작성이 아닌 코드 작성이 반드시 필요하다.

맵리듀스를 이용하면 엔지니어는 자신이 작성한 코드를 대용량 데이터셋 상에서 쉽게 실행할 수 있다.

하둡 접근법에서는 다른 종류의 처리를 하기 위해 여러 다른 특정 시스템으로 데이터를 보낼 필요가 없다. 데이터를 옮길 필요가 없기 때문에 데이터로부터 가치를 끌어내기 쉽고 새로운 처리 모델을 사용한 실험도 훨씬 쉽다.

#### 빈번하게 발생하는 결함을 줄이는 설계

MPP 데이터베이스 대부분은 질의 실행 중에 한 장비만 죽어도 전체 질의가 중단된다. 일반적인 질의는 수 초에서 수 분 내로 대부분 수행이 끝나기 때문에 재실행 방식으로 오류를 다루는 방식 또한 비용이 크지 않아 수용 가능하다. MPP 데이터베이스 또한 디스크에서 데이터를 읽는 비용을 피하기 위해 해시 조인 같은 방식을 사용해 가능하면 메모리에 많은 데이터를 유지하는 것을 선호한다.

하지만 맵리듀스는 맵 또는 리듀스 태스크의 실패를 견딜 수 있다. 개별 태스크 수준에서 작업을 재수행하기 때문에 전체 작업으로 보면 영향을 받지 않는다. 또한 맵리듀스는 데이터를 되도록 디스크에 기록하려 한다. 한편으로는 내결함성을 확보하기 위해서이고, 다른 한편으로는 메모리에 올리기에는 데이터셋이 너무 크다는 가정 때문이다.

하지만 이런 가정이 얼마나 현실적일까? 대부분의 클러스터에서 장비 장애는 항상 발생하지만 그렇다고 그렇게 자주 발생하지는 않는다. 내결함성을 확보하기 위해 상당한 오버헤드를 감당하는 게 실제로 가치가 있을까?

이런 설계는 구글의 환경과 관련이 있으며 맵리듀스는 태스크 종료가 예상치 못하게 자주 발생하더라도 견딜 수 있게 설계됐다. 하드웨어를 신뢰할 수 없기 때문이 아니라 프로세스를 임의로 종료할 수 있으면 연산 클러스터에서 자원 활용도를 높일 수 있기 때문이다.

태스크가 그렇게 자주 종료되지 않는 환경이라면 맵리듀스를 선점 방식으로 설계한 점은 선뜻 이해하기 어렵다. 다음 절에서는 다른 방식으로 설계된 맵리듀스의 여러 대안을 살펴본다.

## 맵리듀스를 넘어

### 중간 상태 구체화

#### 데이터플로 엔진

#### 내결함성

#### 구체화에 대한 논의

### 그래프와 반복 처리

#### 프리글 처리 모델

#### 내결함성

#### 병렬 실행

### 고수준 API 와 언어

#### 선언형 질의 언어로 전환

#### 다양한 분야를 지원하기 위한 전문화

## 정리
