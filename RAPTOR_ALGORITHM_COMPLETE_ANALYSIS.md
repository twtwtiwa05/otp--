# RAPTOR 알고리즘 완벽 분석서

> **OpenTripPlanner Raptor 모듈 심층 분석**
> **작성 목적**: 한국형 OTP 개발을 위한 Raptor 알고리즘 완벽 이해 및 한국 GTFS 연결 가이드
> **분석 대상**: `OpenTripPlanner-dev-2.x/raptor/` 모듈 (351개 Java 파일)

---

## 목차

1. [개요](#1-개요)
2. [Raptor 알고리즘 이론](#2-raptor-알고리즘-이론)
3. [모듈 아키텍처](#3-모듈-아키텍처)
4. [핵심 패키지 상세 분석](#4-핵심-패키지-상세-분석)
5. [알고리즘 실행 흐름](#5-알고리즘-실행-흐름)
6. [핵심 데이터 구조](#6-핵심-데이터-구조)
7. [OTP의 Raptor SPI 구현 분석](#7-otp의-raptor-spi-구현-분석)
8. [한국 GTFS 연결 가이드](#8-한국-gtfs-연결-가이드)
9. [핵심 참조 코드 위치](#9-핵심-참조-코드-위치)
10. [부록](#10-부록)

---

## 1. 개요

### 1.1 Raptor 알고리즘이란?

**RAPTOR** (Round-bAsed Public Transit Optimized Router)는 Microsoft Research에서 2012년에 발표한 대중교통 경로 탐색 알고리즘이다.

**핵심 특징:**
- **라운드 기반 탐색**: 환승 횟수를 라운드로 표현
- **시간 기반 최적화**: Priority Queue 없이 배열 기반으로 초고속 탐색
- **Pareto 최적화**: 다중 기준(시간, 환승, 비용) 동시 최적화

**논문 참조:**
```
Delling, D., Pajor, T., & Werneck, R. F. (2012).
"Round-Based Public Transit Routing"
Microsoft Research Technical Report MSR-TR-2012-98
```

### 1.2 OTP에서 Raptor를 사용하는 이유

| 기존 A* 알고리즘 | Raptor 알고리즘 |
|-----------------|----------------|
| Priority Queue 사용 | 배열 기반, 큐 없음 |
| 느린 메모리 접근 | 캐시 친화적 |
| 단일 기준 최적화 | 다중 기준 Pareto 최적화 |
| 복잡한 상태 관리 | 단순한 라운드 기반 |

**성능**: Raptor는 A*보다 **10배 이상** 빠름

### 1.3 Raptor 모듈의 독립성

```
┌─────────────────────────────────────────────────────────────┐
│                    OTP Application                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              OTP Domain Model                        │   │
│  │    (Stop, Route, Trip, Pattern, Transfer...)        │   │
│  └──────────────────────┬──────────────────────────────┘   │
│                         │                                   │
│                         │ SPI (Service Provider Interface)  │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              RAPTOR MODULE                           │   │
│  │    (완전히 독립적 - OTP 코드 의존성 ZERO)            │   │
│  │    - 순수 알고리즘만 포함                            │   │
│  │    - utils 클래스만 의존                             │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**핵심 포인트**: Raptor 모듈은 OTP 없이도 독립적으로 사용 가능!

---

## 2. Raptor 알고리즘 이론

### 2.1 기본 Raptor 알고리즘

#### 라운드(Round) 개념

```
Round 0: 출발지에서 걸어서 도달 가능한 정류장 (Access)
Round 1: 첫 번째 대중교통 탑승 후 도달 가능한 정류장
Round 2: 1회 환승 후 도달 가능한 정류장
Round k: (k-1)회 환승 후 도달 가능한 정류장
```

#### 의사 코드 (Pseudocode)

```
Algorithm RAPTOR(source, target, departure_time):
    // 초기화
    for all stops s:
        τ(s) = ∞           // 최단 도착 시간
        τ*(s) = ∞          // 현재 라운드 최단 시간

    // Round 0: Access
    for all access paths from source:
        τ(stop) = departure_time + access_duration
        mark stop as "reached"

    // 라운드 반복
    for round k = 1 to MAX_ROUNDS:
        // Phase 1: Transit (대중교통 탑승)
        for each route r passing through marked stops:
            board earliest possible trip at each stop
            update arrival times at subsequent stops

        // Phase 2: Transfer (환승/도보)
        for each stop s reached by transit:
            for each transfer from s to s':
                if τ(s) + transfer_time < τ(s'):
                    τ(s') = τ(s) + transfer_time
                    mark s' as "reached"

        // 종료 조건
        if no improvement:
            break

    return best path to target
```

### 2.2 Range Raptor (RR)

**Range Raptor**는 시간 범위 내에서 모든 출발 시간에 대해 검색하는 알고리즘이다.

#### 핵심 아이디어

```
검색 범위: 12:00 ~ 13:00 (1시간)
반복 순서: 13:00 → 12:59 → 12:58 → ... → 12:00 (역순)

왜 역순인가?
- 이전 반복의 결과를 재사용 가능
- 12:59에서 찾은 경로는 12:58에서도 유효 (더 일찍 출발)
```

#### Range Raptor 의사 코드

```
Algorithm RANGE_RAPTOR(source, target, earliest_departure, search_window):
    latest_departure = earliest_departure + search_window

    // 뒤에서부터 1분 단위로 반복
    for departure_time = latest_departure down to earliest_departure:
        run RAPTOR(source, target, departure_time)

        // 이전 반복 결과 재사용
        inherit best times from previous iteration
```

### 2.3 Multi-Criteria Range Raptor (McRR)

**Multi-Criteria**는 여러 기준을 동시에 최적화한다.

#### 최적화 기준 (Criteria)

| 기준 | 설명 | 변수명 |
|------|------|--------|
| **도착 시간** | 가장 빨리 도착 | `arrivalTime` |
| **출발 시간** | 가장 늦게 출발 | `departureTime` |
| **환승 횟수** | 최소 환승 | `numberOfTransfers` |
| **총 이동 시간** | 가장 짧은 여정 | `travelDuration` |
| **일반화 비용** | 종합 비용 (C1) | `c1` / `generalizedCost` |
| **추가 기준** | 경유지, 우선순위 등 (C2) | `c2` |

#### Pareto 최적화

```
Pareto 최적해란?
- 어느 하나의 기준도 다른 해보다 나쁘지 않고
- 적어도 하나의 기준에서 더 좋은 해

예시:
해 A: [도착 09:00, 환승 2회]
해 B: [도착 09:10, 환승 1회]
해 C: [도착 09:15, 환승 2회]

→ A, B는 Pareto 최적 (서로 지배하지 않음)
→ C는 탈락 (A가 모든 면에서 더 좋음)
```

### 2.4 Forward vs Reverse 검색

```
Forward Search (순방향):
출발지 → → → → → 도착지
시간 흐름대로 검색
"가장 빨리 도착하는 경로"

Reverse Search (역방향):
출발지 ← ← ← ← ← 도착지
시간 역순으로 검색
"가장 늦게 출발하는 경로"
```

**사용 사례:**
- **Depart After** (출발 후): Forward Search
- **Arrive By** (도착 전): Reverse Search
- **Heuristics 계산**: 양방향 모두 사용

---

## 3. 모듈 아키텍처

### 3.1 전체 패키지 구조

```
raptor/src/main/java/org/opentripplanner/raptor/
│
├── api/                          ← 외부 API (사용자가 사용)
│   ├── model/                    ← 핵심 데이터 모델 인터페이스
│   │   ├── RaptorTripSchedule.java      ← 운행 스케줄
│   │   ├── RaptorTripPattern.java       ← 노선 패턴
│   │   ├── RaptorTransfer.java          ← 환승 정보
│   │   ├── RaptorAccessEgress.java      ← 접근/이탈 경로
│   │   ├── RaptorConstants.java         ← 상수 정의
│   │   ├── RaptorConstrainedTransfer.java
│   │   ├── DominanceFunction.java
│   │   ├── RelaxFunction.java
│   │   ├── SearchDirection.java
│   │   └── TransitArrival.java
│   │
│   ├── request/                  ← 검색 요청
│   │   ├── RaptorRequest.java           ← 메인 요청 객체
│   │   ├── RaptorRequestBuilder.java
│   │   ├── SearchParams.java            ← 검색 파라미터
│   │   ├── SearchParamsBuilder.java
│   │   ├── RaptorProfile.java           ← 검색 프로필 (Standard/MC)
│   │   ├── MultiCriteriaRequest.java
│   │   ├── DebugRequest.java
│   │   └── RaptorTuningParameters.java
│   │
│   ├── response/                 ← 검색 결과
│   │   ├── RaptorResponse.java          ← 응답 객체
│   │   └── StopArrivals.java
│   │
│   ├── path/                     ← 경로 결과
│   │   ├── RaptorPath.java              ← 경로 인터페이스
│   │   ├── PathLeg.java                 ← 경로 다리 (추상)
│   │   ├── AccessPathLeg.java           ← 접근 다리
│   │   ├── TransitPathLeg.java          ← 대중교통 다리
│   │   ├── TransferPathLeg.java         ← 환승 다리
│   │   └── EgressPathLeg.java           ← 이탈 다리
│   │
│   ├── view/                     ← 디버깅용 뷰
│   │   ├── ArrivalView.java
│   │   └── PatternRideView.java
│   │
│   └── debug/                    ← 디버깅 도구
│       ├── DebugLogger.java
│       ├── DebugEvent.java
│       └── DebugTopic.java
│
├── spi/                          ← Service Provider Interface (★ 핵심!)
│   ├── RaptorTransitDataProvider.java   ← 메인 데이터 제공자 인터페이스
│   ├── RaptorRoute.java                 ← 노선 인터페이스
│   ├── RaptorTimeTable.java             ← 시간표 인터페이스
│   ├── RaptorTripScheduleSearch.java    ← 트립 검색 인터페이스
│   ├── RaptorBoardOrAlightEvent.java    ← 승/하차 이벤트
│   ├── RaptorCostCalculator.java        ← 비용 계산기
│   ├── RaptorSlackProvider.java         ← 여유시간 제공자
│   ├── RaptorConstrainedBoardingSearch.java
│   ├── RaptorPathConstrainedTransferSearch.java
│   └── IntIterator.java
│
├── rangeraptor/                  ← Range Raptor 구현
│   ├── RangeRaptor.java                 ← 메인 알고리즘 엔진
│   ├── DefaultRangeRaptorWorker.java    ← 실제 작업자
│   │
│   ├── internalapi/              ← 내부 API
│   │   ├── RaptorRouter.java
│   │   ├── RaptorWorkerState.java
│   │   ├── RoutingStrategy.java         ← 라우팅 전략 인터페이스
│   │   ├── WorkerLifeCycle.java
│   │   └── Heuristics.java
│   │
│   ├── standard/                 ← 표준 Raptor 구현
│   │   ├── ArrivalTimeRoutingStrategy.java
│   │   ├── MinTravelDurationRoutingStrategy.java
│   │   ├── StdRangeRaptorWorkerState.java
│   │   ├── StdWorkerState.java
│   │   ├── besttimes/
│   │   │   └── BestTimes.java           ← 최적 시간 추적
│   │   ├── stoparrivals/
│   │   │   ├── StopArrivalState.java
│   │   │   └── StdStopArrivals.java
│   │   └── configure/
│   │       └── StdRangeRaptorConfig.java
│   │
│   ├── multicriteria/            ← 다중 기준 Raptor 구현
│   │   ├── MultiCriteriaRoutingStrategy.java
│   │   ├── McRangeRaptorWorkerState.java
│   │   ├── McStopArrivals.java
│   │   ├── StopArrivalParetoSet.java
│   │   ├── arrivals/
│   │   │   ├── McStopArrival.java
│   │   │   ├── McStopArrivalFactory.java
│   │   │   ├── c1/                      ← 단일 비용 (C1)
│   │   │   └── c2/                      ← 이중 비용 (C1+C2)
│   │   ├── ride/
│   │   │   ├── PatternRide.java
│   │   │   └── PatternRideFactory.java
│   │   ├── heuristic/
│   │   │   └── HeuristicsProvider.java
│   │   └── configure/
│   │       └── McRangeRaptorConfig.java
│   │
│   ├── context/                  ← 검색 컨텍스트
│   │   ├── SearchContext.java
│   │   └── SearchContextBuilder.java
│   │
│   ├── path/                     ← 경로 빌더
│   │   ├── DestinationArrival.java
│   │   ├── DestinationArrivalPaths.java
│   │   ├── PathMapper.java
│   │   ├── ForwardPathMapper.java
│   │   └── ReversePathMapper.java
│   │
│   ├── transit/                  ← Transit 계산기
│   │   ├── RaptorTransitCalculator.java
│   │   ├── ForwardRaptorTransitCalculator.java
│   │   ├── ReverseRaptorTransitCalculator.java
│   │   ├── AccessPaths.java
│   │   └── EgressPaths.java
│   │
│   ├── lifecycle/                ← 생명주기 관리
│   │   ├── LifeCycleEventPublisher.java
│   │   └── LifeCycleSubscriptions.java
│   │
│   └── debug/                    ← 디버깅
│       ├── DebugHandlerFactory.java
│       └── DebugHandlerStopArrivalAdapter.java
│
├── configure/                    ← 설정 및 DI
│   └── RaptorConfig.java                ← 컴포넌트 조립
│
├── service/                      ← 서비스 레이어
│   ├── RangeRaptorDynamicSearch.java    ← 동적 검색
│   ├── HeuristicSearchTask.java
│   └── DefaultStopArrivals.java
│
├── path/                         ← 경로 구현
│   ├── Path.java
│   ├── PathBuilder.java
│   └── PathBuilderLeg.java
│
├── util/                         ← 유틸리티
│   ├── paretoset/                ← Pareto 집합
│   │   ├── ParetoSet.java               ← 핵심!
│   │   ├── ParetoComparator.java
│   │   └── ParetoSetEventListener.java
│   └── BitSetIterator.java
│
└── RaptorService.java            ← 진입점 (Entry Point)
```

### 3.2 레이어 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           APPLICATION LAYER                              │
│                         (RaptorService.java)                            │
│                                                                         │
│   route(RaptorRequest) → RaptorResponse                                 │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           SERVICE LAYER                                  │
│                    (RangeRaptorDynamicSearch.java)                      │
│                                                                         │
│   - 휴리스틱 검색 (Forward/Reverse)                                      │
│   - 동적 검색 윈도우 계산                                                │
│   - 병렬 처리 관리                                                       │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           ALGORITHM LAYER                                │
│                          (RangeRaptor.java)                             │
│                                                                         │
│   - 메인 알고리즘 루프                                                   │
│   - 반복(iteration) 관리                                                 │
│   - 라운드(round) 제어                                                   │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           WORKER LAYER                                   │
│                    (DefaultRangeRaptorWorker.java)                      │
│                                                                         │
│   - Transit 탐색 (findTransitForRound)                                  │
│   - Transfer 탐색 (findTransfersForRound)                               │
│   - Access 처리 (findAccessOnStreetForRound)                            │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           STRATEGY LAYER                                 │
│                     (RoutingStrategy 구현체들)                           │
│                                                                         │
│   ┌─────────────────────────┐    ┌─────────────────────────┐           │
│   │ ArrivalTimeRouting      │    │ MultiCriteriaRouting    │           │
│   │ Strategy                │    │ Strategy                │           │
│   │ (Standard Raptor)       │    │ (MC Raptor)             │           │
│   └─────────────────────────┘    └─────────────────────────┘           │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           STATE LAYER                                    │
│                     (RaptorWorkerState 구현체들)                         │
│                                                                         │
│   ┌─────────────────────────┐    ┌─────────────────────────┐           │
│   │ StdRangeRaptorWorker    │    │ McRangeRaptorWorker     │           │
│   │ State                   │    │ State                   │           │
│   │ - BestTimes             │    │ - McStopArrivals        │           │
│   │ - StopArrivalsState     │    │ - ParetoSet             │           │
│   └─────────────────────────┘    └─────────────────────────┘           │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │
┌─────────────────────────────────────▼───────────────────────────────────┐
│                           SPI LAYER (★ 한국 GTFS 연결점!)                │
│                    (RaptorTransitDataProvider)                          │
│                                                                         │
│   - getTransfersFromStop(stopIndex)                                     │
│   - routeIndexIterator(stops)                                           │
│   - getRouteForIndex(routeIndex)                                        │
│   - numberOfStops()                                                     │
│   - multiCriteriaCostCalculator()                                       │
│   - slackProvider()                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 핵심 패키지 상세 분석

### 4.1 api/model 패키지

#### 4.1.1 RaptorTripSchedule.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/api/model/RaptorTripSchedule.java`

**역할**: 하나의 트립(운행)의 시간표를 나타내는 인터페이스

```java
public interface RaptorTripSchedule {

    /**
     * 주어진 정류장 위치에서의 도착 시간 (초 단위)
     * @param stopPosInPattern 패턴 내 정류장 위치 (0부터 시작)
     * @return 도착 시간 (자정 이후 초)
     */
    int arrival(int stopPosInPattern);

    /**
     * 주어진 정류장 위치에서의 출발 시간 (초 단위)
     * @param stopPosInPattern 패턴 내 정류장 위치 (0부터 시작)
     * @return 출발 시간 (자정 이후 초)
     */
    int departure(int stopPosInPattern);

    /**
     * 이 스케줄이 속한 패턴
     */
    RaptorTripPattern pattern();
}
```

**한국 GTFS 매핑**:
```
GTFS stop_times.txt:
  trip_id, arrival_time, departure_time, stop_id, stop_sequence

→ RaptorTripSchedule:
  - arrival(stopPos) = stop_times.arrival_time (초로 변환)
  - departure(stopPos) = stop_times.departure_time (초로 변환)
  - pattern() = 같은 stop_sequence를 가진 트립들의 그룹
```

#### 4.1.2 RaptorTripPattern.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/api/model/RaptorTripPattern.java`

**역할**: 노선 패턴(정류장 순서)을 나타내는 인터페이스

```java
public interface RaptorTripPattern {

    /**
     * 슬랙(여유시간) 계산에 사용되는 인덱스
     * 버스, 지하철 등 교통수단별로 다른 슬랙 적용 가능
     */
    int slackIndex();

    /**
     * 패턴 내 정류장 개수
     */
    int numberOfStopsInPattern();

    /**
     * 패턴 내 위치로 실제 정류장 인덱스 조회
     * @param stopPositionInPattern 패턴 내 위치 (0부터 시작)
     * @return 전역 정류장 인덱스
     */
    int stopIndex(int stopPositionInPattern);

    /**
     * 해당 위치에서 승차 가능 여부
     */
    boolean boardingPossibleAt(int stopPositionInPattern);

    /**
     * 해당 위치에서 하차 가능 여부
     */
    boolean alightingPossibleAt(int stopPositionInPattern);
}
```

**한국 GTFS 매핑**:
```
GTFS:
  routes.txt + trips.txt + stop_times.txt

→ RaptorTripPattern:
  - 같은 정류장 순서를 가진 트립들을 그룹화
  - slackIndex: route_type으로 결정 (버스=0, 지하철=1, 등)
  - numberOfStopsInPattern: stop_times에서 계산
  - stopIndex: stops.txt의 stop_id를 정수 인덱스로 매핑
```

#### 4.1.3 RaptorTransfer.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/api/model/RaptorTransfer.java`

**역할**: 정류장 간 환승(도보) 정보

```java
public interface RaptorTransfer {

    /**
     * 환승 도착 정류장 (전역 인덱스)
     */
    int stop();

    /**
     * 환승에 걸리는 시간 (초)
     */
    int durationInSeconds();

    /**
     * 환승 비용 (centi-seconds, 1/100초 단위)
     * Multi-criteria 검색에서 사용
     */
    default int c1() {
        return RaptorCostConverter.toRaptorCost(durationInSeconds());
    }
}
```

**한국 GTFS 매핑**:
```
GTFS transfers.txt:
  from_stop_id, to_stop_id, transfer_type, min_transfer_time

→ RaptorTransfer:
  - stop() = to_stop_id (정수 인덱스로 변환)
  - durationInSeconds() = min_transfer_time
  - transfer_type = 2 (최소 환승 시간 필요)인 경우만 사용
```

#### 4.1.4 RaptorAccessEgress.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/api/model/RaptorAccessEgress.java`

**역할**: 출발지→정류장(Access) 또는 정류장→목적지(Egress) 경로

```java
public interface RaptorAccessEgress {

    /**
     * Access: 도착 정류장 / Egress: 출발 정류장
     */
    int stop();

    /**
     * 일반화 비용 (centi-seconds)
     */
    int c1();

    /**
     * 이동 시간 (초)
     */
    int durationInSeconds();

    /**
     * 시간 페널티 (Flex, Park&Ride 등에서 사용)
     */
    default int timePenalty() {
        return RaptorConstants.TIME_NOT_SET;
    }

    /**
     * 가장 빠른 출발 시간 (Flex 등 시간 제약이 있는 경우)
     */
    int earliestDepartureTime(int requestedDepartureTime);

    /**
     * 가장 늦은 도착 시간
     */
    int latestArrivalTime(int requestedArrivalTime);

    /**
     * 탑승 횟수 (Flex 등에서 사용)
     */
    default int numberOfRides() {
        return 0;
    }

    /**
     * 대중교통으로 정류장에 도착했는지 여부
     */
    default boolean stopReachedOnBoard() {
        return false;
    }
}
```

#### 4.1.5 RaptorConstants.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/api/model/RaptorConstants.java`

**역할**: Raptor에서 사용하는 상수 정의

```java
public class RaptorConstants {

    /** 0 */
    public static final int ZERO = 0;

    /** 1분 = 60초 */
    public static final int ONE_MINUTE = 60;

    /** 값이 설정되지 않음 (-1,999,000,000) */
    public static final int NOT_SET = -1_999_000_000;

    /** 찾지 못함 (-2,111,000,000) */
    public static final int NOT_FOUND = -2_111_000_000;

    /** 도달 불가 (순방향, +2,000,000,000) */
    public static final int UNREACHED_HIGH = 2_000_000_000;

    /** 도달 불가 (역방향, -2,000,000,000) */
    public static final int UNREACHED_LOW = -2_000_000_000;

    // 별칭
    public static final int TIME_NOT_SET = NOT_SET;
    public static final int TIME_UNREACHED_FORWARD = UNREACHED_HIGH;
    public static final int TIME_UNREACHED_REVERSE = UNREACHED_LOW;
    public static final int N_TRANSFERS_UNREACHED = UNREACHED_HIGH;
}
```

**왜 이런 값을 사용하는가?**
- `-1`은 계산 오류로 자주 발생하므로 피함
- `Integer.MIN_VALUE`/`MAX_VALUE`는 오버플로우 위험
- 큰 절대값의 "랜덤" 숫자로 디버깅 용이

### 4.2 spi 패키지 (★ 한국 GTFS 연결 핵심!)

#### 4.2.1 RaptorTransitDataProvider.java (가장 중요!)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorTransitDataProvider.java`

**역할**: Raptor가 필요로 하는 모든 대중교통 데이터를 제공하는 메인 인터페이스

```java
public interface RaptorTransitDataProvider<T extends RaptorTripSchedule> {

    // ========== 필수 구현 메서드 ==========

    /**
     * 전체 정류장 개수
     * 정류장은 0부터 numberOfStops()-1까지의 정수로 표현
     */
    int numberOfStops();

    /**
     * 주어진 정류장에서 출발하는 환승 목록
     * @param fromStop 출발 정류장 인덱스
     * @return 환승 Iterator
     */
    Iterator<? extends RaptorTransfer> getTransfersFromStop(int fromStop);

    /**
     * 주어진 정류장들을 지나는 모든 노선의 인덱스 반환
     * @param stops 정류장 인덱스 Iterator
     * @return 노선 인덱스 Iterator
     */
    IntIterator routeIndexIterator(IntIterator stops);

    /**
     * 노선 인덱스로 노선 정보 조회
     * @param routeIndex 노선 인덱스
     * @return RaptorRoute 객체
     */
    RaptorRoute<T> getRouteForIndex(int routeIndex);

    // ========== Multi-criteria용 메서드 ==========

    /**
     * 일반화 비용 계산기 반환
     */
    RaptorCostCalculator<T> multiCriteriaCostCalculator();

    /**
     * 슬랙(여유시간) 제공자 반환
     */
    RaptorSlackProvider slackProvider();

    // ========== 선택적 메서드 (기본 구현 있음) ==========

    /**
     * 역방향 검색용 환승 목록
     */
    default Iterator<? extends RaptorTransfer> getTransfersToStop(int toStop) {
        return getTransfersFromStop(toStop);
    }

    /**
     * 정류장 이름 변환기 (디버깅용)
     */
    default RaptorStopNameResolver stopNameResolver() {
        return (int stopIndex) -> Integer.toString(stopIndex);
    }

    /**
     * 유효한 데이터 시작 시간
     */
    default int getValidTransitDataStartTime() {
        return 0;
    }

    /**
     * 유효한 데이터 종료 시간
     */
    default int getValidTransitDataEndTime() {
        return Integer.MAX_VALUE;
    }
}
```

#### 4.2.2 RaptorRoute.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorRoute.java`

**역할**: 하나의 노선(Route)을 나타내는 인터페이스

```java
public interface RaptorRoute<T extends RaptorTripSchedule> {

    /**
     * 시간표 반환 (이 노선의 모든 운행 스케줄)
     */
    RaptorTimeTable<T> timetable();

    /**
     * 패턴 반환 (정류장 순서)
     */
    RaptorTripPattern pattern();
}
```

#### 4.2.3 RaptorTimeTable.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorTimeTable.java`

**역할**: 시간표(트립 스케줄 목록)를 나타내는 인터페이스

```java
public interface RaptorTimeTable<T extends RaptorTripSchedule> {

    /**
     * 인덱스로 트립 스케줄 조회
     * ★ 성능 중요: 가장 자주 호출되는 메서드!
     * @param index 트립 인덱스 (0부터 시작)
     */
    T getTripSchedule(int index);

    /**
     * 트립 스케줄 개수
     */
    int numberOfTripSchedules();

    /**
     * 트립 검색 객체 생성
     * @param direction 검색 방향 (FORWARD/REVERSE)
     */
    RaptorTripScheduleSearch<T> tripSearch(SearchDirection direction);
}
```

#### 4.2.4 RaptorTripScheduleSearch.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorTripScheduleSearch.java`

**역할**: 시간표에서 탑승 가능한 트립을 검색하는 인터페이스

```java
public interface RaptorTripScheduleSearch<T extends RaptorTripSchedule> {

    /** 모든 트립을 검색 대상에 포함 */
    int UNBOUNDED_TRIP_INDEX = -1;

    /**
     * 주어진 시간과 위치에서 탑승 가능한 트립 검색
     * @param earliestBoardTime 가장 빠른 탑승 시간
     * @param stopPositionInPattern 패턴 내 정류장 위치
     * @param tripIndexLimit 검색할 트립 인덱스 상한
     * @return 탑승/하차 이벤트 (Flyweight 패턴)
     */
    RaptorBoardOrAlightEvent<T> search(
        int earliestBoardTime,
        int stopPositionInPattern,
        int tripIndexLimit
    );
}
```

#### 4.2.5 RaptorCostCalculator.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorCostCalculator.java`

**역할**: Multi-criteria 검색에서 일반화 비용(Generalized Cost)을 계산

```java
public interface RaptorCostCalculator<T extends RaptorTripSchedule> {

    int ZERO_COST = 0;

    /**
     * 탑승 비용 계산
     * @param firstBoarding 첫 번째 탑승 여부
     * @param prevArrivalTime 이전 도착 시간
     * @param boardStop 탑승 정류장
     * @param boardTime 탑승 시간
     * @param trip 탑승 트립
     * @param transferConstraints 환승 제약
     */
    int boardingCost(
        boolean firstBoarding,
        int prevArrivalTime,
        int boardStop,
        int boardTime,
        T trip,
        RaptorTransferConstraint transferConstraints
    );

    /**
     * 탑승 중 상대 비용 계산
     */
    int onTripRelativeRidingCost(int boardTime, T tripScheduledBoarded);

    /**
     * 대중교통 도착 비용 계산
     */
    int transitArrivalCost(
        int boardCost,
        int alightSlack,
        int transitTime,
        T trip,
        int toStop
    );

    /**
     * 대기 비용 계산
     */
    int waitCost(int waitTimeInSeconds);

    /**
     * 남은 최소 비용 추정 (휴리스틱용)
     */
    int calculateRemainingMinCost(int minTravelTime, int minNumTransfers, int fromStop);

    /**
     * Egress 비용 계산
     */
    int costEgress(RaptorAccessEgress egress);
}
```

#### 4.2.6 RaptorSlackProvider.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/spi/RaptorSlackProvider.java`

**역할**: 승차/하차/환승 시 여유시간(Slack) 제공

```java
public interface RaptorSlackProvider {

    /**
     * 환승 슬랙 (환승 시 추가 시간)
     * 단위: 초
     */
    int transferSlack();

    /**
     * 탑승 슬랙 (탑승 전 대기 시간)
     * @param slackIndex 교통수단별 인덱스
     * 단위: 초
     */
    int boardSlack(int slackIndex);

    /**
     * 하차 슬랙 (하차 후 추가 시간)
     * @param slackIndex 교통수단별 인덱스
     * 단위: 초
     */
    int alightSlack(int slackIndex);

    /**
     * 대중교통 슬랙 (탑승+하차)
     */
    default int transitSlack(int slackIndex) {
        return boardSlack(slackIndex) + alightSlack(slackIndex);
    }

    /**
     * 일반 환승 시간 계산
     */
    default int calcRegularTransferDuration(
        int transferDurationInSeconds,
        int fromTripAlightSlackIndex,
        int toTripBoardSlackIndex
    ) {
        return alightSlack(fromTripAlightSlackIndex) +
               transferDurationInSeconds +
               transferSlack() +
               boardSlack(toTripBoardSlackIndex);
    }
}
```

**슬랙 적용 예시**:
```
환승 시나리오:
  버스 A 하차 (09:00) → 도보 5분 → 지하철 B 탑승 (09:07)

슬랙 적용:
  - 버스 하차 슬랙: 1분 (09:01)
  - 도보 시간: 5분 (09:06)
  - 환승 슬랙: 0분
  - 지하철 탑승 슬랙: 1분 (09:07)

실제 탑승 가능 시간: 09:07
```

### 4.3 rangeraptor 패키지

#### 4.3.1 RangeRaptor.java (메인 알고리즘 엔진)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/RangeRaptor.java`

**역할**: Range Raptor 알고리즘의 메인 루프를 실행

```java
public final class RangeRaptor<T extends RaptorTripSchedule> implements RaptorRouter<T> {

    private final RangeRaptorWorker<T> worker;
    private final IntIterator departureTimeIterator;
    private final LifeCycleEventPublisher lifeCycle;

    @Override
    public RaptorRouterResult<T> route() {
        // 생명주기: 검색 시작
        lifeCycle.notifyRouteSearchStart();

        // 시간 범위를 뒤에서부터 반복 (Range Raptor 핵심!)
        while (departureTimeIterator.hasNext()) {
            int iterationDepartureTime = departureTimeIterator.next();
            runRaptorForMinute(iterationDepartureTime);
        }

        // 생명주기: 검색 종료
        lifeCycle.notifyRouteSearchEnd();

        return worker.result();
    }

    private void runRaptorForMinute(int iterationDepartureTime) {
        // 반복 설정
        lifeCycle.setupIteration(iterationDepartureTime);

        // Round 0: Access (걸어서 접근)
        worker.findAccessOnStreetForRound();

        // 라운드 반복
        while (worker.hasMoreRounds()) {
            lifeCycle.prepareForNextRound(roundCounter);

            // Phase 1: Transit 탐색
            worker.findTransitForRound();
            lifeCycle.transitsForRoundComplete();

            // Phase 2: Transfer 탐색
            worker.findTransfersForRound();
            lifeCycle.transfersForRoundComplete();

            roundCounter++;
        }
    }
}
```

#### 4.3.2 DefaultRangeRaptorWorker.java (실제 작업자)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/DefaultRangeRaptorWorker.java`

**역할**: 실제 탐색 작업을 수행하는 Worker

```java
public final class DefaultRangeRaptorWorker<T extends RaptorTripSchedule>
    implements RangeRaptorWorker<T> {

    private final RoutingStrategy<T> transitWorker;
    private final RaptorTransitDataProvider<T> transitData;
    private final RaptorWorkerState<T> state;
    private final AccessPaths accessPaths;
    private final SlackProvider slackProvider;

    private int iterationDepartureTime;

    /**
     * Round 0: Access 경로로 정류장 도착
     */
    @Override
    public void findAccessOnStreetForRound() {
        accessPaths.forEachPath(this::addAccessPath);
    }

    private void addAccessPath(RaptorAccessEgress access) {
        int departureTime = calculator.calculateAccessDepartureTime(
            access, iterationDepartureTime
        );

        if (departureTime != TIME_NOT_SET) {
            state.setAccessToStop(access, departureTime);
        }
    }

    /**
     * Phase 1: Transit 탐색
     */
    @Override
    public void findTransitForRound() {
        // 이전 라운드에서 도달한 정류장들
        IntIterator stops = state.stopsTouchedPreviousRound();

        // 그 정류장들을 지나는 모든 노선
        IntIterator routeIndexIterator = transitData.routeIndexIterator(stops);

        while (routeIndexIterator.hasNext()) {
            int routeIndex = routeIndexIterator.next();
            var route = transitData.getRouteForIndex(routeIndex);
            var pattern = route.pattern();
            var timetable = route.timetable();

            // RoutingStrategy에게 탐색 위임
            transitWorker.prepareForTransitWith(timetable);

            // 패턴의 각 정류장 순회
            IntIterator stop = calculator.patternStopIterator(
                pattern.numberOfStopsInPattern()
            );

            while (stop.hasNext()) {
                int stopPos = stop.next();
                int stopIndex = pattern.stopIndex(stopPos);

                // 하차 시도 (이미 탑승 중이면)
                transitWorker.alightOnlyRegularTransferExist(
                    stopIndex, stopPos, alightSlack
                );

                // 승차 시도 (이전 라운드에서 도착했으면)
                if (state.isStopReachedInPreviousRound(stopIndex)) {
                    int earliestBoardTime = calculator.calculateEarliestBoardTime(
                        state.bestTimePreviousRound(stopIndex),
                        boardSlack
                    );

                    transitWorker.boardWithRegularTransfer(
                        stopIndex, stopPos, earliestBoardTime
                    );
                }
            }
        }
    }

    /**
     * Phase 2: Transfer 탐색
     */
    @Override
    public void findTransfersForRound() {
        IntIterator stops = state.stopsTouchedByTransitCurrentRound();

        while (stops.hasNext()) {
            int fromStop = stops.next();

            Iterator<? extends RaptorTransfer> transfers =
                transitData.getTransfersFromStop(fromStop);

            state.transferToStops(fromStop, transfers);
        }
    }
}
```

#### 4.3.3 RoutingStrategy 인터페이스

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/internalapi/RoutingStrategy.java`

**역할**: 탐색 전략을 정의하는 인터페이스 (Standard vs Multi-criteria)

```java
public interface RoutingStrategy<T extends RaptorTripSchedule> {

    /**
     * 시간표 준비
     */
    void prepareForTransitWith(RaptorTimeTable<T> timeTable);

    /**
     * 일반 환승으로 탑승
     */
    void boardWithRegularTransfer(int stop, int stopPos, int earliestBoardTime);

    /**
     * 제약 있는 환승으로 탑승
     */
    void boardWithConstrainedTransfer(
        int stop, int stopPos, int earliestBoardTime,
        RaptorConstrainedBoardingSearch<T> transfersSearch
    );

    /**
     * 일반 환승만 있을 때 하차
     */
    void alightOnlyRegularTransferExist(int stop, int stopPos, int alightSlack);

    /**
     * 제약 있는 환승도 있을 때 하차
     */
    void alightConstrainedTransferExist(
        int stop, int stopPos, int alightSlack,
        RaptorConstrainedBoardingSearch<T> transfersSearch
    );
}
```

#### 4.3.4 ArrivalTimeRoutingStrategy.java (Standard Raptor)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/standard/ArrivalTimeRoutingStrategy.java`

**역할**: 도착 시간만 최적화하는 표준 전략

```java
public final class ArrivalTimeRoutingStrategy<T extends RaptorTripSchedule>
    implements RoutingStrategy<T> {

    private final StdWorkerState<T> state;
    private final TransitCalculator<T> calculator;

    private RaptorTripScheduleSearch<T> tripSearch;
    private T onTrip;        // 현재 탑승 중인 트립
    private int onTripIndex; // 현재 트립 인덱스
    private int boardStop;   // 탑승 정류장
    private int boardTime;   // 탑승 시간

    @Override
    public void prepareForTransitWith(RaptorTimeTable<T> timetable) {
        this.tripSearch = timetable.tripSearch(FORWARD);
        this.onTrip = null;
        this.onTripIndex = UNBOUNDED_TRIP_INDEX;
    }

    @Override
    public void boardWithRegularTransfer(int stop, int stopPos, int earliestBoardTime) {
        // 탑승 가능한 트립 검색
        var boarding = tripSearch.search(earliestBoardTime, stopPos, onTripIndex);

        if (boarding.empty()) {
            return;
        }

        // 현재 트립보다 더 빠른 트립이 있으면 갈아탐
        if (boarding.tripIndex() < onTripIndex) {
            onTrip = boarding.trip();
            onTripIndex = boarding.tripIndex();
            boardStop = stop;
            boardTime = boarding.time();
        }
    }

    @Override
    public void alightOnlyRegularTransferExist(int stop, int stopPos, int alightSlack) {
        if (onTrip == null) {
            return;
        }

        // 도착 시간 계산
        int alightTime = cycleCalculator.calculateArrivalTime(
            onTrip, stopPos, alightSlack
        );

        // 상태 업데이트
        state.transitToStop(stop, alightTime, boardStop, boardTime, onTrip);
    }
}
```

#### 4.3.5 MultiCriteriaRoutingStrategy.java (MC Raptor)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/multicriteria/MultiCriteriaRoutingStrategy.java`

**역할**: 다중 기준을 동시에 최적화하는 전략

```java
public final class MultiCriteriaRoutingStrategy<T extends RaptorTripSchedule>
    implements RoutingStrategy<T> {

    private final McRangeRaptorWorkerState<T> state;
    private final TransitCalculator<T> calculator;
    private final RaptorCostCalculator<T> c1Calculator;
    private final PatternRideFactory<T> patternRideFactory;
    private final ParetoSet<PatternRide<T>> patternRides;

    @Override
    public void boardWithRegularTransfer(int stop, int stopPos, int earliestBoardTime) {
        // 이전 라운드의 모든 도착들에 대해
        for (var prevArrival : state.listStopArrivalsPreviousRound(stop)) {

            int actualBoardTime = Math.max(
                earliestBoardTime,
                prevArrival.arrivalTime()
            );

            // 탑승 가능한 트립 검색
            var boarding = tripSearch.search(actualBoardTime, stopPos);

            if (boarding.empty()) {
                continue;
            }

            // 탑승 비용 계산
            int boardCost = c1Calculator.boardingCost(
                prevArrival.arrivedOnBoard(),
                prevArrival.arrivalTime(),
                stop,
                boarding.time(),
                boarding.trip(),
                null
            );

            // 패턴 라이드 생성 및 Pareto 집합에 추가
            var ride = patternRideFactory.createRide(
                prevArrival,
                stop,
                stopPos,
                boarding.time(),
                boardCost,
                boarding.trip()
            );

            patternRides.add(ride);
        }
    }

    @Override
    public void alightOnlyRegularTransferExist(int stop, int stopPos, int alightSlack) {
        // 모든 패턴 라이드에 대해 하차 시도
        for (var ride : patternRides) {
            int alightTime = cycleCalculator.calculateArrivalTime(
                ride.trip(), stopPos, alightSlack
            );

            state.transitToStop(ride, stop, alightTime, alightSlack);
        }
    }
}
```

### 4.4 configure 패키지

#### RaptorConfig.java

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/configure/RaptorConfig.java`

**역할**: 검색 프로필에 따라 컴포넌트를 조립하는 팩토리

```java
public class RaptorConfig<T extends RaptorTripSchedule> {

    private final RaptorTuningParameters tuningParameters;
    private final ExecutorService threadPool;

    /**
     * Standard Worker 생성
     */
    public RaptorRouter<T> createRangeRaptorWithStdWorker(
        RaptorTransitDataProvider<T> transitData,
        RaptorRequest<T> request
    ) {
        return new StdRangeRaptorConfig<>(this, transitData, request)
            .createSearch();
    }

    /**
     * Multi-Criteria Worker 생성
     */
    public RaptorRouter<T> createRangeRaptorWithMcWorker(
        RaptorTransitDataProvider<T> transitData,
        RaptorRequest<T> request,
        Heuristics heuristics,
        ExtraMcRouterSearch<T> extraMcSearch
    ) {
        return new McRangeRaptorConfig<>(this, transitData, request)
            .withHeuristics(heuristics)
            .createSearch();
    }
}
```

### 4.5 util/paretoset 패키지

#### ParetoSet.java (핵심!)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/util/paretoset/ParetoSet.java`

**역할**: Pareto 최적해만 유지하는 집합

```java
public class ParetoSet<T> extends AbstractCollection<T> {

    private final ParetoComparator<T> comparator;
    private T[] elements;
    private int size = 0;
    private T goodElement = null; // 빠른 거부용 캐시

    @Override
    public boolean add(T newValue) {
        if (size == 0) {
            acceptAndAppendValue(newValue);
            return true;
        }

        // 빠른 거부: 캐시된 좋은 원소가 지배하면 바로 거부
        if (goodElement != null && leftDominates(goodElement, newValue)) {
            notifyElementRejected(newValue, goodElement);
            return false;
        }

        boolean mutualDominanceExist = false;
        boolean equivalentVectorExist = false;

        for (int i = 0; i < size; ++i) {
            T existing = elements[i];

            boolean leftDominance = leftDominanceExist(newValue, existing);
            boolean rightDominance = rightDominanceExist(newValue, existing);

            if (leftDominance && rightDominance) {
                // 상호 지배: 일부 기준에서 각각 더 좋음
                mutualDominanceExist = true;
            } else if (leftDominance) {
                // 새 값이 기존 값을 완전 지배
                removeDominatedAndAdd(newValue, i);
                return true;
            } else if (rightDominance) {
                // 기존 값이 새 값을 완전 지배
                goodElement = existing;
                notifyElementRejected(newValue, existing);
                return false;
            } else {
                // 동등 (비교 불가)
                equivalentVectorExist = true;
            }
        }

        if (mutualDominanceExist && !equivalentVectorExist) {
            acceptAndAppendValue(newValue);
            return true;
        }

        // 모든 기존 값과 동등 → 거부
        notifyElementRejected(newValue, elements[0]);
        return false;
    }

    private boolean leftDominates(T left, T right) {
        return leftDominanceExist(left, right) && !rightDominanceExist(left, right);
    }
}
```

**Pareto 비교 예시**:
```
기존: { [도착 09:00, 환승 2회, 비용 100],
        [도착 09:15, 환승 1회, 비용 80] }

새로운 경로 [도착 09:05, 환승 2회, 비용 90]:
  vs [09:00, 2회, 100]: 도착 늦음, 환승 같음, 비용 낮음 → 상호 지배
  vs [09:15, 1회, 80]: 도착 빠름, 환승 많음, 비용 높음 → 상호 지배

결과: 추가됨! (Pareto 최적)

새로운 경로 [도착 09:10, 환승 2회, 비용 110]:
  vs [09:00, 2회, 100]: 도착 늦음, 환승 같음, 비용 높음 → 지배당함!

결과: 거부됨
```

---

## 5. 알고리즘 실행 흐름

### 5.1 전체 실행 흐름 다이어그램

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         RaptorService.route()                            │
│                              [진입점]                                    │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    RangeRaptorDynamicSearch.route()                      │
│                                                                          │
│  1. 휴리스틱 검색 활성화 결정                                             │
│  2. Forward/Reverse 휴리스틱 실행 (병렬 가능)                             │
│  3. 동적 검색 파라미터 계산 (EDT, LAT, SearchWindow)                      │
│  4. 메인 RangeRaptor 검색 실행                                            │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
           ┌─────────────────────────┴─────────────────────────┐
           │                                                   │
           ▼                                                   ▼
┌──────────────────────┐                         ┌──────────────────────┐
│  Forward Heuristic   │                         │  Reverse Heuristic   │
│  (목적지까지 최소시간)│                         │  (출발지까지 최소시간)│
└──────────────────────┘                         └──────────────────────┘
           │                                                   │
           └─────────────────────────┬─────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         RangeRaptor.route()                              │
│                          [메인 알고리즘]                                  │
│                                                                          │
│  for departure_time = latest downto earliest:                            │
│      runRaptorForMinute(departure_time)                                  │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     runRaptorForMinute(departure_time)                   │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Round 0: Access                                                     │ │
│  │ worker.findAccessOnStreetForRound()                                │ │
│  │   → 출발지에서 걸어서 갈 수 있는 정류장들 마킹                        │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                     │                                    │
│                                     ▼                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ while hasMoreRounds():                                              │ │
│  │   ┌──────────────────────────────────────────────────────────────┐ │ │
│  │   │ Round k Phase 1: Transit                                      │ │ │
│  │   │ worker.findTransitForRound()                                 │ │ │
│  │   │   → 마킹된 정류장을 지나는 노선 탐색                           │ │ │
│  │   │   → 탑승 가능한 트립 찾기                                      │ │ │
│  │   │   → 하차 가능한 정류장 업데이트                                │ │ │
│  │   └──────────────────────────────────────────────────────────────┘ │ │
│  │                              │                                      │ │
│  │                              ▼                                      │ │
│  │   ┌──────────────────────────────────────────────────────────────┐ │ │
│  │   │ Round k Phase 2: Transfer                                     │ │ │
│  │   │ worker.findTransfersForRound()                               │ │ │
│  │   │   → 대중교통으로 도착한 정류장에서 환승                         │ │ │
│  │   │   → 걸어서 갈 수 있는 정류장 업데이트                          │ │ │
│  │   └──────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.2 findTransitForRound() 상세

```
findTransitForRound():
│
├── 1. 이전 라운드에서 도착한 정류장 목록 가져오기
│       stops = state.stopsTouchedPreviousRound()
│
├── 2. 그 정류장들을 지나는 모든 노선 가져오기
│       routes = transitData.routeIndexIterator(stops)
│
└── 3. 각 노선에 대해:
        │
        ├── route = transitData.getRouteForIndex(routeIndex)
        ├── pattern = route.pattern()
        ├── timetable = route.timetable()
        │
        ├── transitWorker.prepareForTransitWith(timetable)
        │
        └── 패턴의 각 정류장에 대해 (순서대로):
            │
            ├── stopPos = 0, 1, 2, ...
            ├── stopIndex = pattern.stopIndex(stopPos)
            │
            ├── (A) 하차 시도:
            │   └── if 탑승 중:
            │       └── 이 정류장에서 하차 → 상태 업데이트
            │
            └── (B) 승차 시도:
                └── if 이전 라운드에서 이 정류장에 도착:
                    ├── earliestBoardTime 계산
                    └── 탑승 가능한 트립 검색 → 탑승
```

### 5.3 라운드별 상태 변화 예시

```
시나리오: 서울역 → 강남역 (09:00 출발)

Round 0 (Access):
  출발지(서울역) → 걸어서 5분 → 서울역 정류장

  상태: { 서울역정류장: 09:05 }

Round 1 (첫 번째 대중교통):
  Transit: 지하철 1호선 09:10 탑승 → 시청역 09:15 도착
  Transit: 지하철 4호선 09:08 탑승 → 충무로역 09:12 도착
  Transfer: 시청역에서 걸어서 2분 → 시청역(2호선)

  상태: { 서울역정류장: 09:05, 시청역: 09:15, 시청역(2호선): 09:17, 충무로역: 09:12 }

Round 2 (1회 환승):
  Transit: 2호선 시청역 09:20 탑승 → 강남역 09:45 도착
  Transit: 4호선 충무로역 09:15 탑승 → 동대문역 09:18 도착

  상태: { ..., 강남역: 09:45, 동대문역: 09:18 }

Round 3 (2회 환승):
  (더 나은 경로 없음 → 종료)

결과: 서울역 → 지하철 → 시청역 → 환승 → 2호선 → 강남역 (도착 09:45)
```

---

## 6. 핵심 데이터 구조

### 6.1 BestTimes (표준 Raptor)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/standard/besttimes/BestTimes.java`

```java
public final class BestTimes {

    /** 각 정류장의 최적 도착 시간 (전체) */
    private final int[] times;                    // [stopIndex] = 도착시간

    /** 각 정류장의 최적 대중교통 도착 시간 */
    private final int[] transitArrivalTimes;      // [stopIndex] = 대중교통 도착시간

    /** 현재 라운드에서 대중교통으로 도착한 정류장 */
    private final BitSet reachedByTransitCurrentRound;

    /** 현재 라운드에서 도착한 정류장 */
    private BitSet reachedCurrentRound;

    /** 이전 라운드에서 도착한 정류장 */
    private BitSet reachedLastRound;

    /**
     * 새로운 최적 시간 업데이트
     */
    public boolean updateNewBestTime(int stop, int time) {
        if (calculator.isBefore(time, times[stop])) {
            times[stop] = time;
            reachedCurrentRound.set(stop);
            return true;
        }
        return false;
    }

    /**
     * 라운드 전환 시 BitSet 교체
     */
    private void prepareForNextRound() {
        // 현재 → 이전으로 교체 (객체 재사용)
        BitSet tmp = reachedLastRound;
        reachedLastRound = reachedCurrentRound;
        reachedCurrentRound = tmp;
        reachedCurrentRound.clear();
    }
}
```

### 6.2 McStopArrivals (Multi-criteria)

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/multicriteria/McStopArrivals.java`

```java
public class McStopArrivals<T extends RaptorTripSchedule> {

    /** 각 정류장별 Pareto 최적 도착 집합 */
    private final StopArrivalParetoSet<T>[] arrivals;

    /**
     * 새로운 도착 추가
     */
    public void addStopArrival(McStopArrival<T> arrival) {
        int stop = arrival.stop();
        arrivals[stop].add(arrival);
        touchedStops.set(stop);
    }

    /**
     * 이전 마커 이후의 도착들 조회
     */
    public Iterable<McStopArrival<T>> listArrivalsAfterMarker(int stop) {
        return arrivals[stop].elementsAfterMarker();
    }
}
```

### 6.3 PatternRide

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/multicriteria/ride/PatternRide.java`

**역할**: 현재 탑승 중인 트립 정보를 담는 객체

```java
public record PatternRide<T extends RaptorTripSchedule>(
    McStopArrival<T> prevArrival,  // 이전 도착
    int boardStop,                  // 탑승 정류장
    int boardPos,                   // 패턴 내 탑승 위치
    int boardTime,                  // 탑승 시간
    int boardC1,                    // 탑승 비용
    T trip,                         // 탑승 트립
    int c2                          // 추가 비용 (C2)
) {
    public int relativeC1(int alightC1, int alightTime) {
        // 하차 비용 = 탑승 비용 + (하차시간 - 탑승시간) * 비용계수
        return boardC1 + alightC1;
    }
}
```

### 6.4 DestinationArrivalPaths

**위치**: `raptor/src/main/java/org/opentripplanner/raptor/rangeraptor/path/DestinationArrivalPaths.java`

**역할**: 목적지에 도착한 경로들을 관리

```java
public class DestinationArrivalPaths<T extends RaptorTripSchedule> {

    /** 목적지 도착 경로들의 Pareto 집합 */
    private final ParetoSet<RaptorPath<T>> paths;

    /** 경로 생성기 */
    private final PathMapper<T> pathMapper;

    /**
     * 새로운 목적지 도착 추가
     */
    public void add(DestinationArrival<T> arrival) {
        RaptorPath<T> path = pathMapper.mapToPath(arrival);
        paths.add(path);
    }

    /**
     * 모든 Pareto 최적 경로 반환
     */
    public Collection<RaptorPath<T>> listPaths() {
        return paths;
    }
}
```

---

## 7. OTP의 Raptor SPI 구현 분석

### 7.1 RaptorRoutingRequestTransitData 상세

**위치**: `application/src/main/java/org/opentripplanner/routing/algorithm/raptoradapter/transit/request/RaptorRoutingRequestTransitData.java`

OTP가 SPI를 어떻게 구현했는지 분석:

```java
public class RaptorRoutingRequestTransitData
    implements RaptorTransitDataProvider<TripSchedule> {

    // 핵심 데이터 소스
    private final RaptorTransitData raptorTransitData;

    // 정류장별 활성 노선 패턴 (캐시)
    private final List<int[]> activeTripPatternsPerStop;

    // 노선 인덱스 → TripPatternForDates 매핑
    private final List<TripPatternForDates> patternIndex;

    // 환승 인덱스
    private final RaptorTransferIndex transferIndex;

    // 비용 계산기
    private final RaptorCostCalculator<TripSchedule> generalizedCostCalculator;

    // 슬랙 제공자
    private final RaptorSlackProvider slackProvider;

    @Override
    public Iterator<? extends RaptorTransfer> getTransfersFromStop(int stopIndex) {
        return transferIndex.getForwardTransfers(stopIndex).iterator();
    }

    @Override
    public IntIterator routeIndexIterator(IntIterator stops) {
        BitSet activeTripPatterns = new BitSet();

        while (stops.hasNext()) {
            int[] patterns = activeTripPatternsPerStop.get(stops.next());
            for (int i : patterns) {
                activeTripPatterns.set(i);
            }
        }

        return new BitSetIterator(activeTripPatterns);
    }

    @Override
    public RaptorRoute<TripSchedule> getRouteForIndex(int routeIndex) {
        return patternIndex.get(routeIndex);
    }

    @Override
    public int numberOfStops() {
        return raptorTransitData.getStopCount();
    }

    @Override
    public RaptorCostCalculator<TripSchedule> multiCriteriaCostCalculator() {
        return generalizedCostCalculator;
    }

    @Override
    public RaptorSlackProvider slackProvider() {
        return slackProvider;
    }
}
```

### 7.2 TripSchedule 구현

**위치**: `application/src/main/java/org/opentripplanner/routing/algorithm/raptoradapter/transit/TripSchedule.java`

```java
public interface TripSchedule extends DefaultTripSchedule {

    /** 서비스 날짜 */
    LocalDate getServiceDate();

    /** 원본 TripTimes (OTP 도메인 객체) */
    TripTimes getOriginalTripTimes();

    /** 원본 TripPattern (OTP 도메인 객체) */
    TripPattern getOriginalTripPattern();

    /** Frequency 기반 트립 여부 */
    default boolean isFrequencyBasedTrip() {
        return false;
    }

    /** Frequency 배차 간격 */
    default int frequencyHeadwayInSeconds() {
        return -999;
    }
}
```

### 7.3 데이터 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         GTFS 데이터 로드                                 │
│                                                                         │
│   stops.txt → Stop 객체들                                               │
│   routes.txt → Route 객체들                                             │
│   trips.txt → Trip 객체들                                               │
│   stop_times.txt → TripTimes 객체들                                     │
│   transfers.txt → Transfer 객체들                                       │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Graph Building 단계                                 │
│                                                                         │
│   - Stop → 정수 인덱스 매핑                                              │
│   - TripPattern 생성 (같은 정류장 순서를 가진 트립들 그룹화)              │
│   - Transfer 인덱스 구축                                                 │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      RaptorTransitData 생성                              │
│                                                                         │
│   - stopIndex: Map<Stop, Integer>                                       │
│   - patternIndex: List<TripPatternForDates>                             │
│   - transferIndex: RaptorTransferIndex                                  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│               RaptorRoutingRequestTransitData 생성                       │
│                       (요청별로 필터링)                                   │
│                                                                         │
│   - 날짜별 활성 트립 필터링                                              │
│   - 교통수단별 필터링                                                    │
│   - 비용 계산기 설정                                                     │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Raptor 검색 실행                                 │
│                                                                         │
│   RaptorService.route(request, transitDataProvider)                     │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 한국 GTFS 연결 가이드

### 8.1 구현해야 할 인터페이스 목록

#### 필수 구현 (반드시 필요)

| 인터페이스 | 역할 | 구현 난이도 |
|-----------|------|------------|
| `RaptorTransitDataProvider<T>` | 메인 데이터 제공자 | ★★★ |
| `RaptorTripSchedule` | 트립 시간표 | ★★ |
| `RaptorTripPattern` | 노선 패턴 | ★★ |
| `RaptorRoute<T>` | 노선 정보 | ★ |
| `RaptorTimeTable<T>` | 시간표 | ★★ |
| `RaptorTripScheduleSearch<T>` | 트립 검색 | ★★★ |
| `RaptorTransfer` | 환승 정보 | ★ |
| `RaptorAccessEgress` | 접근/이탈 경로 | ★★ |

#### Multi-criteria 사용 시 추가 구현

| 인터페이스 | 역할 | 구현 난이도 |
|-----------|------|------------|
| `RaptorCostCalculator<T>` | 비용 계산기 | ★★★ |
| `RaptorSlackProvider` | 슬랙 제공자 | ★ |

### 8.2 GTFS 데이터 → Raptor 모델 매핑

#### stops.txt → Stop Index

```java
// 정류장 ID를 정수 인덱스로 매핑
Map<String, Integer> stopIdToIndex = new HashMap<>();
List<GtfsStop> stopsByIndex = new ArrayList<>();

int index = 0;
for (GtfsStop stop : gtfsStops) {
    stopIdToIndex.put(stop.getStopId(), index);
    stopsByIndex.add(stop);
    index++;
}
```

#### stop_times.txt → TripSchedule 구현

```java
public class KoreanTripSchedule implements RaptorTripSchedule {

    private final int[] arrivals;    // 정류장별 도착 시간 (초)
    private final int[] departures;  // 정류장별 출발 시간 (초)
    private final RaptorTripPattern pattern;

    public KoreanTripSchedule(
        List<GtfsStopTime> stopTimes,
        RaptorTripPattern pattern
    ) {
        this.pattern = pattern;
        this.arrivals = new int[stopTimes.size()];
        this.departures = new int[stopTimes.size()];

        for (int i = 0; i < stopTimes.size(); i++) {
            GtfsStopTime st = stopTimes.get(i);
            arrivals[i] = timeToSeconds(st.getArrivalTime());
            departures[i] = timeToSeconds(st.getDepartureTime());
        }
    }

    @Override
    public int arrival(int stopPosInPattern) {
        return arrivals[stopPosInPattern];
    }

    @Override
    public int departure(int stopPosInPattern) {
        return departures[stopPosInPattern];
    }

    @Override
    public RaptorTripPattern pattern() {
        return pattern;
    }

    private int timeToSeconds(String gtfsTime) {
        // "HH:MM:SS" → 초 변환
        String[] parts = gtfsTime.split(":");
        int hours = Integer.parseInt(parts[0]);
        int minutes = Integer.parseInt(parts[1]);
        int seconds = Integer.parseInt(parts[2]);
        return hours * 3600 + minutes * 60 + seconds;
    }
}
```

#### routes.txt + trips.txt → TripPattern 구현

```java
public class KoreanTripPattern implements RaptorTripPattern {

    private final int slackIndex;      // 교통수단별 슬랙 인덱스
    private final int[] stopIndices;   // 패턴 내 정류장들
    private final boolean[] canBoard;  // 승차 가능 여부
    private final boolean[] canAlight; // 하차 가능 여부

    public KoreanTripPattern(
        int routeType,
        List<Integer> stops,
        List<GtfsStopTime> stopTimes
    ) {
        this.slackIndex = routeTypeToSlackIndex(routeType);
        this.stopIndices = stops.stream().mapToInt(i -> i).toArray();

        this.canBoard = new boolean[stops.size()];
        this.canAlight = new boolean[stops.size()];

        for (int i = 0; i < stopTimes.size(); i++) {
            GtfsStopTime st = stopTimes.get(i);
            canBoard[i] = st.getPickupType() != 1;
            canAlight[i] = st.getDropOffType() != 1;
        }
    }

    private int routeTypeToSlackIndex(int routeType) {
        return switch (routeType) {
            case 1 -> 0;  // 지하철
            case 3 -> 1;  // 버스
            case 2 -> 2;  // 철도
            default -> 1;
        };
    }

    @Override
    public int slackIndex() {
        return slackIndex;
    }

    @Override
    public int numberOfStopsInPattern() {
        return stopIndices.length;
    }

    @Override
    public int stopIndex(int stopPositionInPattern) {
        return stopIndices[stopPositionInPattern];
    }

    @Override
    public boolean boardingPossibleAt(int stopPositionInPattern) {
        return canBoard[stopPositionInPattern];
    }

    @Override
    public boolean alightingPossibleAt(int stopPositionInPattern) {
        return canAlight[stopPositionInPattern];
    }
}
```

#### transfers.txt → RaptorTransfer 구현

```java
public class KoreanTransfer implements RaptorTransfer {

    private final int toStopIndex;
    private final int durationSeconds;

    public KoreanTransfer(int toStopIndex, int durationSeconds) {
        this.toStopIndex = toStopIndex;
        this.durationSeconds = durationSeconds;
    }

    @Override
    public int stop() {
        return toStopIndex;
    }

    @Override
    public int durationInSeconds() {
        return durationSeconds;
    }
}
```

### 8.3 단계별 구현 가이드

#### Step 1: 데이터 모델 생성

```
1. GTFS 파일 파싱
   - stops.txt 파싱 → GtfsStop 리스트
   - routes.txt 파싱 → GtfsRoute 리스트
   - trips.txt 파싱 → GtfsTrip 리스트
   - stop_times.txt 파싱 → GtfsStopTime 리스트
   - transfers.txt 파싱 → GtfsTransfer 리스트

2. 정류장 인덱스 생성
   - stopId → Integer 매핑

3. 패턴 그룹화
   - 같은 정류장 순서를 가진 트립들을 그룹화
   - Map<List<Integer>, List<GtfsTrip>>

4. 시간표 정렬
   - 각 패턴 내에서 트립을 첫 정류장 출발 시간 순으로 정렬
```

#### Step 2: SPI 인터페이스 구현

```java
public class KoreanTransitDataProvider
    implements RaptorTransitDataProvider<KoreanTripSchedule> {

    private final int numberOfStops;
    private final List<List<KoreanTransfer>> transfersByStop;
    private final List<int[]> routesByStop;
    private final List<KoreanRoute> routes;

    public KoreanTransitDataProvider(GtfsData gtfs) {
        // 데이터 초기화
        this.numberOfStops = gtfs.getStops().size();
        this.transfersByStop = buildTransferIndex(gtfs);
        this.routesByStop = buildRouteIndex(gtfs);
        this.routes = buildRoutes(gtfs);
    }

    @Override
    public int numberOfStops() {
        return numberOfStops;
    }

    @Override
    public Iterator<? extends RaptorTransfer> getTransfersFromStop(int fromStop) {
        return transfersByStop.get(fromStop).iterator();
    }

    @Override
    public IntIterator routeIndexIterator(IntIterator stops) {
        BitSet activeRoutes = new BitSet();
        while (stops.hasNext()) {
            int[] routeIndices = routesByStop.get(stops.next());
            for (int routeIndex : routeIndices) {
                activeRoutes.set(routeIndex);
            }
        }
        return new BitSetIterator(activeRoutes);
    }

    @Override
    public RaptorRoute<KoreanTripSchedule> getRouteForIndex(int routeIndex) {
        return routes.get(routeIndex);
    }

    @Override
    public RaptorCostCalculator<KoreanTripSchedule> multiCriteriaCostCalculator() {
        return new KoreanCostCalculator();
    }

    @Override
    public RaptorSlackProvider slackProvider() {
        return new KoreanSlackProvider();
    }
}
```

#### Step 3: 검색 실행

```java
// 1. Raptor 설정
RaptorTuningParameters tuning = RaptorTuningParameters.defaultParameters();
RaptorConfig<KoreanTripSchedule> config = new RaptorConfig<>(tuning);

// 2. 데이터 제공자 생성
KoreanTransitDataProvider transitData = new KoreanTransitDataProvider(gtfs);

// 3. 요청 생성
RaptorRequest<KoreanTripSchedule> request = RaptorRequest
    .<KoreanTripSchedule>of()
    .searchParams()
        .earliestDepartureTime(9 * 3600)  // 09:00
        .searchWindowInSeconds(3600)       // 1시간
        .addAccessPath(new WalkAccess(startStopIndex, 300))  // 5분 도보
        .addEgressPath(new WalkEgress(endStopIndex, 180))    // 3분 도보
    .build()
    .profile(RaptorProfile.MULTI_CRITERIA)
    .build();

// 4. 검색 실행
RaptorService<KoreanTripSchedule> service = new RaptorService<>(config);
RaptorResponse<KoreanTripSchedule> response = service.route(request, transitData);

// 5. 결과 처리
for (RaptorPath<KoreanTripSchedule> path : response.paths()) {
    System.out.println("경로: " + path);
    System.out.println("도착 시간: " + path.endTime() / 3600 + ":" + (path.endTime() % 3600) / 60);
    System.out.println("환승 횟수: " + path.numberOfTransfers());
}
```

### 8.4 주의사항 및 팁

#### 성능 최적화

```
1. 정류장 인덱스는 연속적인 정수로 (0, 1, 2, ...)
   - 배열 접근이 가장 빠름

2. 시간은 모두 "초"로 통일
   - 자정 기준 초 (09:00 = 32400)
   - 자정 넘어가면 25:00:00 = 90000초

3. 트립은 첫 정류장 출발 시간 순으로 정렬
   - 이진 검색 가능

4. 환승은 정류장별로 인덱싱
   - List<List<Transfer>> 구조

5. BitSet 활용
   - 도착 정류장 추적에 효율적
```

#### 한국 GTFS 특이사항

```
1. 정류장 ID
   - 서울: 숫자 형태 (예: "102000001")
   - 전국표준: 복합 형태

2. 노선 타입
   - route_type: 3 (버스), 1 (지하철), 2 (철도)

3. 환승
   - 무료환승 시간 (30분~1시간)
   - 환승할인 정보 (fare_rules.txt)

4. 평일/주말 운행
   - calendar.txt, calendar_dates.txt 확인

5. 심야버스
   - 25:00, 26:00 형태의 시간 처리
```

---

## 9. 핵심 참조 코드 위치

### 9.1 반드시 참고해야 할 파일 목록

#### Raptor 모듈 (raptor/)

| 파일 | 경로 | 중요도 | 역할 |
|------|------|--------|------|
| RaptorService.java | raptor/ | ★★★ | 진입점 |
| RaptorTransitDataProvider.java | raptor/spi/ | ★★★ | 메인 SPI |
| RangeRaptor.java | raptor/rangeraptor/ | ★★★ | 메인 알고리즘 |
| DefaultRangeRaptorWorker.java | raptor/rangeraptor/ | ★★★ | 작업자 |
| RaptorTripSchedule.java | raptor/api/model/ | ★★★ | 트립 인터페이스 |
| RaptorTripPattern.java | raptor/api/model/ | ★★★ | 패턴 인터페이스 |
| RaptorRoute.java | raptor/spi/ | ★★ | 노선 인터페이스 |
| RaptorTimeTable.java | raptor/spi/ | ★★ | 시간표 인터페이스 |
| RaptorRequest.java | raptor/api/request/ | ★★ | 요청 객체 |
| RaptorPath.java | raptor/api/path/ | ★★ | 결과 경로 |
| BestTimes.java | raptor/rangeraptor/standard/besttimes/ | ★★ | 최적 시간 추적 |
| ParetoSet.java | raptor/util/paretoset/ | ★★ | Pareto 집합 |
| ArrivalTimeRoutingStrategy.java | raptor/rangeraptor/standard/ | ★★ | 표준 전략 |
| MultiCriteriaRoutingStrategy.java | raptor/rangeraptor/multicriteria/ | ★★ | MC 전략 |
| RaptorCostCalculator.java | raptor/spi/ | ★ | 비용 계산기 |
| RaptorSlackProvider.java | raptor/spi/ | ★ | 슬랙 제공자 |

#### OTP Application 모듈 (application/)

| 파일 | 경로 | 중요도 | 역할 |
|------|------|--------|------|
| RaptorRoutingRequestTransitData.java | routing/algorithm/raptoradapter/transit/request/ | ★★★ | SPI 구현체 |
| TripSchedule.java | routing/algorithm/raptoradapter/transit/ | ★★ | 트립 구현 |
| TripPatternForDates.java | routing/algorithm/raptoradapter/transit/ | ★★ | 패턴 구현 |
| RaptorTransferIndex.java | routing/algorithm/raptoradapter/transit/ | ★ | 환승 인덱스 |
| DefaultSlackProvider.java | routing/algorithm/raptoradapter/transit/ | ★ | 슬랙 구현 |

### 9.2 테스트 코드 위치

```
raptor/src/test/java/org/opentripplanner/raptor/
├── _data/                          ← 테스트용 가짜 데이터
│   ├── transit/
│   │   ├── TestTransitData.java    ← 테스트용 TransitDataProvider
│   │   └── TestTripSchedule.java   ← 테스트용 TripSchedule
│   └── api/
│       └── TestPathBuilder.java
│
├── rangeraptor/
│   ├── RangeRaptorTest.java        ← 메인 알고리즘 테스트
│   └── multicriteria/
│       └── McRangeRaptorTest.java
│
└── service/
    └── RaptorServiceTest.java
```

### 9.3 예제 코드 위치

```
application/src/test/java/org/opentripplanner/
├── raptorlegacy/
│   └── _data/
│       └── transit/
│           └── TestTransitData.java   ← SPI 구현 예제
│
└── routing/algorithm/raptoradapter/
    └── transit/
        └── request/
            └── RaptorRoutingRequestTransitDataTest.java
```

---

## 10. 부록

### 10.1 용어 정리

| 용어 | 영문 | 설명 |
|------|------|------|
| 라운드 | Round | 탐색 반복 횟수 (≈ 환승 횟수) |
| 반복 | Iteration | 출발 시간별 탐색 (Range Raptor) |
| 패턴 | Pattern | 같은 정류장 순서를 가진 트립들의 그룹 |
| 트립 | Trip | 하나의 운행 |
| 스케줄 | Schedule | 트립의 시간표 |
| 슬랙 | Slack | 여유 시간 (승차/하차/환승) |
| Access | Access | 출발지 → 첫 정류장 |
| Egress | Egress | 마지막 정류장 → 목적지 |
| Pareto | Pareto | 다중 기준 최적화 |
| C1 | C1/Generalized Cost | 일반화 비용 (centi-seconds) |
| C2 | C2 | 추가 비용 기준 |

### 10.2 시간 단위

| 단위 | 설명 | 예시 |
|------|------|------|
| 초 (seconds) | 기본 시간 단위 | 09:00 = 32400 |
| centi-seconds | 비용 단위 (1/100초) | 1분 = 6000 |
| 분 (minutes) | 반복 간격 | searchWindow / 60 |

### 10.3 성능 최적화 팁

```
1. 배열 > 리스트 > 맵
   - int[] 배열이 가장 빠름

2. 객체 생성 최소화
   - Flyweight 패턴 활용 (RaptorTripScheduleSearch)

3. BitSet 활용
   - 정류장 방문 여부 추적

4. 이진 검색
   - 정렬된 시간표에서 트립 검색

5. 캐시 친화적 접근
   - 연속 메모리 접근이 빠름

6. 조기 종료
   - 목적지 도달 후 추가 라운드 제한
```

### 10.4 디버깅 방법

#### 디버그 로그 활성화

```java
RaptorRequest<T> request = RaptorRequest
    .<T>of()
    // ... 검색 파라미터
    .debug()
        .stopArrivalListener(...)  // 정류장 도착 이벤트
        .pathListener(...)         // 경로 발견 이벤트
        .addStops(List.of(stopIndex1, stopIndex2))  // 추적할 정류장
    .build()
    .build();
```

#### 디버그 출력 예시

```
[Round 1] Transit arrival at stop 42 (시청역)
  - Time: 09:15:00
  - From: stop 15 (서울역)
  - Trip: 지하철 1호선 (trip_id: 1001)
  - Board time: 09:10:00

[Round 2] Transfer from stop 42 to stop 43
  - Duration: 120 seconds
  - New arrival time: 09:17:00
```

---

## 마무리

이 문서는 OpenTripPlanner의 Raptor 알고리즘을 완벽하게 분석한 것입니다.

**한국형 OTP 개발 시 핵심 포인트:**

1. **SPI 인터페이스 구현**이 가장 중요
2. **RaptorTransitDataProvider**를 구현하면 Raptor 엔진 사용 가능
3. 한국 GTFS 데이터의 특수성 (정류장 ID, 환승 규칙 등) 고려
4. 성능 최적화를 위해 배열 기반 자료구조 사용

**다음 단계:**
1. 한국 GTFS 파서 구현
2. SPI 인터페이스 구현
3. 테스트 및 검증
4. 성능 튜닝

---

*이 문서는 OpenTripPlanner dev-2.x 브랜치 (2024년 기준)를 분석하여 작성되었습니다.*
