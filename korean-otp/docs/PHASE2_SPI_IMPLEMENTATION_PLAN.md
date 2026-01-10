# Phase 2: Raptor SPI 완벽 구현 계획

> **목표**: OTP Raptor 모듈을 100% 활용하는 한국형 경로탐색 엔진 개발
> **형태**: CLI 기반 로컬 엔진 (웹 서비스 X, REST API X)
> **핵심**: 출발지/목적지 입력 → 즉시 경로 출력 (< 1초)

---

## 1. 목표 시스템

### 1.1 사용 방식

```
┌──────────────────────────────────────────────────────────────────┐
│  $ java -jar korean-raptor.jar                                   │
│                                                                  │
│  [데이터 로드 중...]                                              │
│  GTFS 로드 완료: 212,105 정류장, 27,138 노선                      │
│  OSM 로드 완료: 1,234,567 노드                                    │
│  환승 생성 완료: 456,789 환승                                     │
│  Raptor 준비 완료! (소요시간: 45초)                               │
│                                                                  │
│  ─────────────────────────────────────────────────────────────── │
│  출발 좌표: 37.5547, 126.9707                                     │
│  도착 좌표: 37.4979, 127.0276                                     │
│  출발 시간: 09:00                                                 │
│  ─────────────────────────────────────────────────────────────── │
│                                                                  │
│  [경로 탐색 완료] 0.23초                                          │
│                                                                  │
│  ■ 경로 1: 09:00 출발 → 09:32 도착 (32분, 환승 1회)               │
│    1. 도보 4분 (320m) → 서울역 1호선                              │
│    2. 1호선 승차 09:04 → 시청역 하차 09:08                        │
│    3. 환승 도보 3분 → 2호선                                       │
│    4. 2호선 승차 09:11 → 강남역 하차 09:30                        │
│    5. 도보 2분 (150m) → 목적지                                    │
│                                                                  │
│  ■ 경로 2: 09:05 출발 → 09:38 도착 (33분, 환승 0회)               │
│    ...                                                           │
│                                                                  │
│  다음 검색? (q: 종료)                                             │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 성능 목표

| 항목 | 목표 |
|------|------|
| 초기 로드 시간 | < 2분 |
| 경로 탐색 시간 | **< 1초** |
| 메모리 사용량 | < 8GB |

---

## 2. 아키텍처

### 2.1 전체 구조

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          한국형 Raptor 엔진                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   [한국 GTFS]         [한국 OSM]                                             │
│        │                   │                                                │
│        ▼                   ▼                                                │
│   ┌─────────┐        ┌─────────────┐                                       │
│   │ GTFS    │        │ OSM Loader  │                                       │
│   │ Loader  │        │ + A* 엔진   │                                       │
│   └────┬────┘        └──────┬──────┘                                       │
│        │                    │                                               │
│        └────────┬───────────┘                                               │
│                 ▼                                                           │
│        ┌───────────────────┐                                               │
│        │ Transfer Generator │  ← OSM 기반 도보 환승 생성                    │
│        └─────────┬─────────┘                                               │
│                  ▼                                                          │
│        ┌───────────────────┐                                               │
│        │   TransitData     │  ← Raptor용 데이터 구조                        │
│        └─────────┬─────────┘                                               │
│                  │                                                          │
│   ═══════════════╪══════════════════════════════════════════════════════   │
│                  │  SPI (Service Provider Interface)                        │
│   ═══════════════╪══════════════════════════════════════════════════════   │
│                  ▼                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                     OTP RAPTOR MODULE (JAR)                          │  │
│   │                                                                      │  │
│   │   RaptorService ──→ RangeRaptor ──→ MultiCriteria/Standard          │  │
│   │                                                                      │  │
│   │   지원 기능:                                                         │  │
│   │   ✓ Range Raptor (시간 범위 탐색)                                    │  │
│   │   ✓ Multi-Criteria (파레토 최적)                                     │  │
│   │   ✓ Standard Raptor (단일 기준)                                      │  │
│   │   ✓ Reverse Search (역방향 탐색)                                     │  │
│   │   ✓ Debug/Profiling                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                  │                                                          │
│                  ▼                                                          │
│        ┌───────────────────┐                                               │
│        │  RaptorResponse   │  ← 경로 결과                                   │
│        └─────────┬─────────┘                                               │
│                  ▼                                                          │
│        ┌───────────────────┐                                               │
│        │   CLI 출력        │                                               │
│        └───────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 프로젝트 구조

```
korean-otp/
├── build.gradle
├── settings.gradle
│
├── src/main/java/kr/otp/
│   │
│   ├── core/                              ← 핵심 엔진
│   │   ├── KoreanRaptor.java              ← ★ 메인 엔진 클래스
│   │   └── KoreanRaptorBuilder.java       ← 빌더
│   │
│   ├── gtfs/                              ← GTFS 파싱
│   │   ├── model/
│   │   │   ├── GtfsStop.java
│   │   │   ├── GtfsRoute.java
│   │   │   ├── GtfsTrip.java
│   │   │   ├── GtfsStopTime.java
│   │   │   └── GtfsTransfer.java
│   │   ├── loader/
│   │   │   └── GtfsLoader.java
│   │   └── GtfsBundle.java
│   │
│   ├── osm/                               ← OSM 파싱 & 도로망
│   │   ├── model/
│   │   │   ├── OsmNode.java
│   │   │   └── OsmWay.java
│   │   ├── graph/
│   │   │   ├── StreetGraph.java
│   │   │   ├── StreetVertex.java
│   │   │   └── StreetEdge.java
│   │   ├── routing/
│   │   │   └── AStar.java                 ← A* 도보 경로탐색
│   │   └── OsmLoader.java
│   │
│   ├── transfer/                          ← 환승 생성
│   │   ├── TransferGenerator.java
│   │   └── TransferIndex.java
│   │
│   ├── raptor/                            ← ★★★ SPI 구현 (핵심!)
│   │   │
│   │   ├── data/                          ← 데이터 구조
│   │   │   ├── TransitData.java           ← 전체 데이터 컨테이너
│   │   │   ├── TransitDataBuilder.java    ← 빌더
│   │   │   ├── PatternIndex.java
│   │   │   ├── StopIndex.java
│   │   │   └── RoutesByStop.java
│   │   │
│   │   ├── spi/                           ← SPI 구현체
│   │   │   ├── KoreanTransitDataProvider.java   ← ★★★ 메인 SPI
│   │   │   ├── KoreanTripSchedule.java          ← ★★★
│   │   │   ├── KoreanTripPattern.java           ← ★★★
│   │   │   ├── KoreanRoute.java                 ← ★★
│   │   │   ├── KoreanTimeTable.java             ← ★★
│   │   │   ├── KoreanTripScheduleSearch.java    ← ★★ (성능 핵심!)
│   │   │   ├── KoreanTransfer.java              ← ★★
│   │   │   ├── KoreanAccessEgress.java          ← ★★
│   │   │   ├── KoreanCostCalculator.java        ← ★
│   │   │   ├── KoreanSlackProvider.java         ← ★
│   │   │   └── KoreanStopNameResolver.java      ← 디버깅용
│   │   │
│   │   └── search/                        ← 검색 헬퍼
│   │       ├── AccessEgressFinder.java    ← 출발/도착 정류장 검색
│   │       └── RequestBuilder.java        ← RaptorRequest 생성
│   │
│   ├── result/                            ← 결과 처리
│   │   ├── Journey.java                   ← 경로 결과 모델
│   │   ├── Leg.java                       ← 경로 구간
│   │   └── ResultMapper.java              ← RaptorPath → Journey 변환
│   │
│   ├── util/                              ← 유틸리티
│   │   ├── GeoUtil.java                   ← 좌표 계산 (Haversine)
│   │   ├── TimeUtil.java                  ← 시간 변환
│   │   └── BitSetIntIterator.java
│   │
│   └── Main.java                          ← CLI 진입점
│
├── src/test/java/kr/otp/
│   ├── gtfs/
│   ├── osm/
│   ├── raptor/
│   └── integration/
│
└── data/
    ├── gtfs/                              ← Phase 1에서 생성
    └── osm/                               ← 한국 OSM (pbf)
```

---

## 3. OTP Raptor 모듈 완벽 활용

### 3.1 사용할 Raptor 기능

| 기능 | 클래스 | 설명 | 구현 |
|------|--------|------|------|
| **Range Raptor** | `RangeRaptor` | 시간 범위 내 모든 출발 시간 탐색 | ✅ |
| **Multi-Criteria** | `McRangeRaptor` | 파레토 최적 (시간, 환승, 비용) | ✅ |
| **Standard Raptor** | `StdRangeRaptor` | 단일 기준 (최단 시간) | ✅ |
| **Forward Search** | - | 정방향 탐색 | ✅ |
| **Reverse Search** | - | 역방향 탐색 (도착 시간 기준) | ✅ |
| **Heuristics** | `Heuristics` | 탐색 공간 축소 | ✅ |
| **Debug Mode** | `DebugHandlerFactory` | 탐색 과정 추적 | ✅ |

### 3.2 구현해야 할 SPI 인터페이스 (전체)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SPI 인터페이스 계층 구조                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  RaptorTransitDataProvider<T>              ← 최상위 (★★★ 필수)              │
│      │                                                                      │
│      ├── numberOfStops() → int                                             │
│      ├── getTransfersFromStop(int) → Iterator<RaptorTransfer>              │
│      ├── getTransfersToStop(int) → Iterator<RaptorTransfer>                │
│      ├── routeIndexIterator(IntIterator) → IntIterator                     │
│      ├── getRouteForIndex(int) → RaptorRoute<T>                            │
│      ├── multiCriteriaCostCalculator() → RaptorCostCalculator<T>           │
│      ├── slackProvider() → RaptorSlackProvider                             │
│      ├── stopNameResolver() → RaptorStopNameResolver                       │
│      ├── getValidTransitDataStartTime() → int                              │
│      ├── getValidTransitDataEndTime() → int                                │
│      ├── transferConstraintsSearch() → RaptorPathConstrainedTransferSearch │
│      ├── transferConstraintsForwardSearch(int) → RaptorConstrainedBoarding │
│      └── transferConstraintsReverseSearch(int) → RaptorConstrainedBoarding │
│                                                                             │
│  RaptorRoute<T>                            ← 노선 (★★ 필수)                 │
│      ├── pattern() → RaptorTripPattern                                     │
│      └── timetable() → RaptorTimeTable<T>                                  │
│                                                                             │
│  RaptorTripPattern                         ← 패턴 (★★★ 필수)                │
│      ├── patternIndex() → int                                              │
│      ├── numberOfStopsInPattern() → int                                    │
│      ├── stopIndex(int) → int                                              │
│      ├── boardingPossibleAt(int) → boolean                                 │
│      ├── alightingPossibleAt(int) → boolean                                │
│      ├── slackIndex() → int                                                │
│      ├── priorityGroupId() → int                                           │
│      └── debugInfo() → String                                              │
│                                                                             │
│  RaptorTimeTable<T>                        ← 시간표 (★★ 필수)               │
│      ├── getTripSchedule(int) → T                                          │
│      ├── numberOfTripSchedules() → int                                     │
│      └── tripSearch(SearchDirection) → RaptorTripScheduleSearch<T>         │
│                                                                             │
│  RaptorTripSchedule                        ← 트립 (★★★ 필수)                │
│      ├── tripSortIndex() → int                                             │
│      ├── arrival(int) → int                                                │
│      ├── departure(int) → int                                              │
│      └── pattern() → RaptorTripPattern                                     │
│                                                                             │
│  RaptorTripScheduleSearch<T>               ← 트립 검색 (★★ 성능 핵심!)       │
│      └── search(int, int, int) → RaptorBoardOrAlightEvent<T>               │
│                                                                             │
│  RaptorTransfer                            ← 환승 (★★ 필수)                 │
│      ├── stop() → int                                                      │
│      ├── durationInSeconds() → int                                         │
│      └── c1() → int                                                        │
│                                                                             │
│  RaptorAccessEgress                        ← Access/Egress (★★ 필수)        │
│      ├── stop() → int                                                      │
│      ├── c1() → int                                                        │
│      ├── durationInSeconds() → int                                         │
│      ├── earliestDepartureTime(int) → int                                  │
│      ├── latestArrivalTime(int) → int                                      │
│      ├── hasOpeningHours() → boolean                                       │
│      └── openingHoursToString() → String                                   │
│                                                                             │
│  RaptorCostCalculator<T>                   ← 비용 계산 (★ 필수)             │
│      ├── boardingCost(...) → int                                           │
│      ├── onTripRelativeRidingCost(...) → int                               │
│      ├── transitArrivalCost(...) → int                                     │
│      ├── waitCost(int) → int                                               │
│      ├── calculateRemainingMinCost(...) → int                              │
│      └── costEgress(RaptorAccessEgress) → int                              │
│                                                                             │
│  RaptorSlackProvider                       ← 슬랙 (★ 필수)                  │
│      ├── transferSlack() → int                                             │
│      ├── boardSlack(int) → int                                             │
│      └── alightSlack(int) → int                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 OTP 참조 파일 전체 목록

#### Raptor 모듈 (`raptor/`)

| 파일 | 경로 | 역할 |
|------|------|------|
| **RaptorTransitDataProvider.java** | `raptor/spi/` | 메인 SPI |
| **RaptorTripSchedule.java** | `raptor/api/model/` | 트립 인터페이스 |
| **RaptorTripPattern.java** | `raptor/api/model/` | 패턴 인터페이스 |
| **RaptorTransfer.java** | `raptor/api/model/` | 환승 인터페이스 |
| **RaptorAccessEgress.java** | `raptor/api/model/` | Access/Egress |
| RaptorRoute.java | `raptor/spi/` | 노선 |
| RaptorTimeTable.java | `raptor/spi/` | 시간표 |
| RaptorTripScheduleSearch.java | `raptor/spi/` | 트립 검색 |
| RaptorCostCalculator.java | `raptor/spi/` | 비용 계산 |
| RaptorSlackProvider.java | `raptor/spi/` | 슬랙 |
| RaptorStopNameResolver.java | `raptor/spi/` | 정류장 이름 |
| RaptorService.java | `raptor/service/` | 서비스 진입점 |
| RaptorRequest.java | `raptor/api/request/` | 요청 객체 |
| RaptorResponse.java | `raptor/api/response/` | 응답 객체 |
| RaptorPath.java | `raptor/api/path/` | 경로 결과 |
| RangeRaptor.java | `raptor/rangeraptor/` | 메인 알고리즘 |
| SearchDirection.java | `raptor/api/model/` | 탐색 방향 |

#### 테스트 구현체 (참조용)

| 파일 | 경로 | 역할 |
|------|------|------|
| **TestTransitData.java** | `raptor/src/test/.../transit/` | SPI 구현 예제 |
| **TestTripSchedule.java** | `raptor/src/test/.../transit/` | 트립 구현 |
| **TestTripPattern.java** | `raptor/src/test/.../transit/` | 패턴 구현 |
| **TestRoute.java** | `raptor/src/test/.../transit/` | 노선 구현 |
| TestTripScheduleSearch.java | `raptor/src/test/.../transit/` | 검색 구현 |

#### OTP Application 모듈 (참조용)

| 파일 | 경로 | 역할 |
|------|------|------|
| RaptorRoutingRequestTransitData.java | `routing/.../transit/request/` | 실제 SPI 구현 |
| TripScheduleBoardSearch.java | `routing/.../transit/request/` | 트립 검색 |
| DirectTransferGenerator.java | `graph_builder/.../transfer/` | 환승 생성 |
| StreetNearbyStopFinder.java | `graph_builder/.../nearbystops/` | OSM 환승 |
| StraightLineNearbyStopFinder.java | `graph_builder/.../nearbystops/` | 직선 환승 |
| AStar.java | `astar/` | A* 알고리즘 |

---

## 4. SPI 구현 상세

### 4.1 KoreanTransitDataProvider (메인)

```java
public class KoreanTransitDataProvider
    implements RaptorTransitDataProvider<KoreanTripSchedule> {

    private final TransitData data;
    private final KoreanCostCalculator costCalculator;
    private final KoreanSlackProvider slackProvider;

    // ═══════════════════════════════════════════════════════════════
    // 필수 구현
    // ═══════════════════════════════════════════════════════════════

    @Override
    public int numberOfStops() {
        return data.getStopCount();
    }

    @Override
    public Iterator<? extends RaptorTransfer> getTransfersFromStop(int fromStop) {
        return data.getTransfersFrom(fromStop);
    }

    @Override
    public Iterator<? extends RaptorTransfer> getTransfersToStop(int toStop) {
        return data.getTransfersTo(toStop);
    }

    @Override
    public IntIterator routeIndexIterator(IntIterator stops) {
        // 정류장들을 지나는 모든 노선 인덱스 반환
        BitSet activeRoutes = new BitSet();
        while (stops.hasNext()) {
            int stopIndex = stops.next();
            for (int routeIdx : data.getRoutesByStop(stopIndex)) {
                activeRoutes.set(routeIdx);
            }
        }
        return new BitSetIntIterator(activeRoutes);
    }

    @Override
    public RaptorRoute<KoreanTripSchedule> getRouteForIndex(int routeIndex) {
        return data.getRoute(routeIndex);
    }

    @Override
    public RaptorCostCalculator<KoreanTripSchedule> multiCriteriaCostCalculator() {
        return costCalculator;
    }

    @Override
    public RaptorSlackProvider slackProvider() {
        return slackProvider;
    }

    // ═══════════════════════════════════════════════════════════════
    // 디버깅/옵션
    // ═══════════════════════════════════════════════════════════════

    @Override
    public RaptorStopNameResolver stopNameResolver() {
        return stopIndex -> data.getStopName(stopIndex);
    }

    @Override
    public int getValidTransitDataStartTime() {
        return data.getServiceStartTime();  // 예: 04:00 = 14400
    }

    @Override
    public int getValidTransitDataEndTime() {
        return data.getServiceEndTime();    // 예: 26:00 = 93600
    }

    // ═══════════════════════════════════════════════════════════════
    // 제약 환승 (선택적 - 기본 구현)
    // ═══════════════════════════════════════════════════════════════

    @Override
    public RaptorPathConstrainedTransferSearch<KoreanTripSchedule> transferConstraintsSearch() {
        return null;  // 제약 환승 미사용
    }

    @Override
    public RaptorConstrainedBoardingSearch<KoreanTripSchedule> transferConstraintsForwardSearch(int routeIndex) {
        return NoopConstrainedBoardingSearch.INSTANCE;
    }

    @Override
    public RaptorConstrainedBoardingSearch<KoreanTripSchedule> transferConstraintsReverseSearch(int routeIndex) {
        return NoopConstrainedBoardingSearch.INSTANCE;
    }
}
```

### 4.2 KoreanTripSchedule

```java
public class KoreanTripSchedule implements RaptorTripSchedule {

    private final int tripSortIndex;           // 정렬용 (첫 정류장 출발 시간)
    private final int[] arrivalTimes;          // 각 정류장 도착 시간 (초)
    private final int[] departureTimes;        // 각 정류장 출발 시간 (초)
    private final KoreanTripPattern pattern;

    // 원본 정보 (결과 출력용)
    private final String tripId;
    private final String routeShortName;       // "2호선", "402번"

    @Override
    public int tripSortIndex() {
        return tripSortIndex;
    }

    @Override
    public int arrival(int stopPosInPattern) {
        return arrivalTimes[stopPosInPattern];
    }

    @Override
    public int departure(int stopPosInPattern) {
        return departureTimes[stopPosInPattern];
    }

    @Override
    public RaptorTripPattern pattern() {
        return pattern;
    }

    // 결과 출력용 getter
    public String getTripId() { return tripId; }
    public String getRouteShortName() { return routeShortName; }
}
```

### 4.3 KoreanTripPattern

```java
public class KoreanTripPattern implements RaptorTripPattern {

    private final int patternIndex;
    private final int[] stopIndexes;           // 패턴 내 정류장 순서
    private final boolean[] canBoard;          // 승차 가능 여부
    private final boolean[] canAlight;         // 하차 가능 여부
    private final int slackIndex;              // 교통수단별 슬랙 (0=지하철, 1=버스, 2=철도)
    private final int priorityGroupId;         // 우선순위 그룹
    private final String debugInfo;            // "BUS_402" 등

    @Override
    public int patternIndex() {
        return patternIndex;
    }

    @Override
    public int numberOfStopsInPattern() {
        return stopIndexes.length;
    }

    @Override
    public int stopIndex(int stopPositionInPattern) {
        return stopIndexes[stopPositionInPattern];
    }

    @Override
    public boolean boardingPossibleAt(int stopPositionInPattern) {
        // 마지막 정류장에서는 승차 불가
        if (stopPositionInPattern == stopIndexes.length - 1) return false;
        return canBoard[stopPositionInPattern];
    }

    @Override
    public boolean alightingPossibleAt(int stopPositionInPattern) {
        // 첫 정류장에서는 하차 불가
        if (stopPositionInPattern == 0) return false;
        return canAlight[stopPositionInPattern];
    }

    @Override
    public int slackIndex() {
        return slackIndex;
    }

    @Override
    public int priorityGroupId() {
        return priorityGroupId;
    }

    @Override
    public String debugInfo() {
        return debugInfo;
    }
}
```

### 4.4 KoreanTripScheduleSearch (⚡ 성능 핵심!)

```java
public class KoreanTripScheduleSearch
    implements RaptorTripScheduleSearch<KoreanTripSchedule> {

    private final KoreanTripSchedule[] schedules;  // 시간순 정렬됨
    private final SearchDirection direction;

    // Flyweight: 결과 객체 재사용 (GC 최소화)
    private final MutableBoardAlightEvent<KoreanTripSchedule> event;

    public KoreanTripScheduleSearch(
        KoreanTripSchedule[] schedules,
        SearchDirection direction
    ) {
        this.schedules = schedules;
        this.direction = direction;
        this.event = new MutableBoardAlightEvent<>();
    }

    @Override
    public RaptorBoardOrAlightEvent<KoreanTripSchedule> search(
        int earliestBoardTime,
        int stopPositionInPattern,
        int tripIndexLimit
    ) {
        int limit = (tripIndexLimit == UNBOUNDED_TRIP_INDEX)
            ? schedules.length
            : Math.min(tripIndexLimit + 1, schedules.length);

        // 이진 검색으로 탑승 가능한 첫 트립 찾기
        int index = binarySearch(earliestBoardTime, stopPositionInPattern, limit);

        if (index < 0 || index >= limit) {
            return RaptorBoardOrAlightEvent.empty();
        }

        KoreanTripSchedule trip = schedules[index];
        int time = (direction == SearchDirection.FORWARD)
            ? trip.departure(stopPositionInPattern)
            : trip.arrival(stopPositionInPattern);

        // Flyweight: 새 객체 생성 없이 재사용
        event.set(trip, index, stopPositionInPattern, time);
        return event;
    }

    /**
     * 이진 검색: O(log n) 복잡도
     */
    private int binarySearch(int targetTime, int stopPos, int limit) {
        int low = 0;
        int high = limit - 1;
        int result = -1;

        while (low <= high) {
            int mid = (low + high) >>> 1;
            int tripTime = (direction == SearchDirection.FORWARD)
                ? schedules[mid].departure(stopPos)
                : schedules[mid].arrival(stopPos);

            if (direction == SearchDirection.FORWARD) {
                // 정방향: targetTime 이상인 첫 트립
                if (tripTime >= targetTime) {
                    result = mid;
                    high = mid - 1;
                } else {
                    low = mid + 1;
                }
            } else {
                // 역방향: targetTime 이하인 마지막 트립
                if (tripTime <= targetTime) {
                    result = mid;
                    low = mid + 1;
                } else {
                    high = mid - 1;
                }
            }
        }

        return result;
    }
}
```

### 4.5 KoreanRoute & KoreanTimeTable

```java
public class KoreanRoute implements RaptorRoute<KoreanTripSchedule> {

    private final KoreanTripPattern pattern;
    private final KoreanTimeTable timetable;

    @Override
    public RaptorTripPattern pattern() {
        return pattern;
    }

    @Override
    public RaptorTimeTable<KoreanTripSchedule> timetable() {
        return timetable;
    }
}

public class KoreanTimeTable implements RaptorTimeTable<KoreanTripSchedule> {

    private final KoreanTripSchedule[] schedules;  // 시간순 정렬

    // 캐싱: 검색 객체 재사용
    private final KoreanTripScheduleSearch forwardSearch;
    private final KoreanTripScheduleSearch reverseSearch;

    public KoreanTimeTable(KoreanTripSchedule[] schedules) {
        this.schedules = schedules;
        this.forwardSearch = new KoreanTripScheduleSearch(schedules, SearchDirection.FORWARD);
        this.reverseSearch = new KoreanTripScheduleSearch(schedules, SearchDirection.REVERSE);
    }

    @Override
    public KoreanTripSchedule getTripSchedule(int index) {
        return schedules[index];
    }

    @Override
    public int numberOfTripSchedules() {
        return schedules.length;
    }

    @Override
    public RaptorTripScheduleSearch<KoreanTripSchedule> tripSearch(SearchDirection direction) {
        return (direction == SearchDirection.FORWARD) ? forwardSearch : reverseSearch;
    }
}
```

### 4.6 KoreanTransfer & KoreanAccessEgress

```java
public class KoreanTransfer implements RaptorTransfer {

    private final int toStopIndex;
    private final int durationSeconds;
    private final int cost;  // centi-seconds

    @Override
    public int stop() {
        return toStopIndex;
    }

    @Override
    public int durationInSeconds() {
        return durationSeconds;
    }

    @Override
    public int c1() {
        return cost;
    }
}

public class KoreanAccessEgress implements RaptorAccessEgress {

    private final int stopIndex;
    private final int durationSeconds;
    private final int cost;  // centi-seconds
    private final double distanceMeters;

    @Override
    public int stop() {
        return stopIndex;
    }

    @Override
    public int c1() {
        return cost;
    }

    @Override
    public int durationInSeconds() {
        return durationSeconds;
    }

    @Override
    public int earliestDepartureTime(int requestedDepartureTime) {
        return requestedDepartureTime;
    }

    @Override
    public int latestArrivalTime(int requestedArrivalTime) {
        return requestedArrivalTime;
    }

    @Override
    public boolean hasOpeningHours() {
        return false;  // 24시간 이용 가능
    }

    @Override
    public String openingHoursToString() {
        return "";
    }

    // 결과 출력용
    public double getDistanceMeters() {
        return distanceMeters;
    }
}
```

### 4.7 KoreanCostCalculator

```java
public class KoreanCostCalculator
    implements RaptorCostCalculator<KoreanTripSchedule> {

    // 비용 계수 (centi-seconds, 1초 = 100)
    private static final int BOARD_COST = 60 * 100;        // 첫 승차: 1분
    private static final int TRANSFER_COST = 120 * 100;    // 환승: 2분
    private static final double WAIT_RELUCTANCE = 1.0;     // 대기 시간 가중치

    @Override
    public int boardingCost(
        boolean firstBoarding,
        int prevArrivalTime,
        int boardStop,
        int boardTime,
        KoreanTripSchedule trip,
        RaptorTransferConstraint transferConstraints
    ) {
        int waitTime = boardTime - prevArrivalTime;
        int waitCost = (int) (waitTime * 100 * WAIT_RELUCTANCE);

        return firstBoarding
            ? BOARD_COST + waitCost
            : TRANSFER_COST + waitCost;
    }

    @Override
    public int onTripRelativeRidingCost(int boardTime, KoreanTripSchedule trip) {
        return 0;  // 탑승 중 추가 비용 없음
    }

    @Override
    public int transitArrivalCost(
        int boardCost,
        int alightSlack,
        int transitTime,
        KoreanTripSchedule trip,
        int toStop
    ) {
        return boardCost + (transitTime * 100);
    }

    @Override
    public int waitCost(int waitTimeInSeconds) {
        return (int) (waitTimeInSeconds * 100 * WAIT_RELUCTANCE);
    }

    @Override
    public int calculateRemainingMinCost(int minTravelTime, int minNumTransfers, int fromStop) {
        return (minTravelTime * 100) + (minNumTransfers * TRANSFER_COST);
    }

    @Override
    public int costEgress(RaptorAccessEgress egress) {
        return egress.c1();
    }
}
```

### 4.8 KoreanSlackProvider

```java
public class KoreanSlackProvider implements RaptorSlackProvider {

    // 슬랙 인덱스 (route_type 기반)
    public static final int SUBWAY = 0;
    public static final int BUS = 1;
    public static final int RAIL = 2;

    // 환승 슬랙 (모든 환승에 적용)
    private static final int TRANSFER_SLACK = 60;  // 1분

    // 승차 슬랙 (교통수단별)
    private static final int[] BOARD_SLACK = {
        60,   // 지하철: 1분 (문 닫힘 대기)
        30,   // 버스: 30초
        120   // 철도: 2분 (플랫폼 이동)
    };

    // 하차 슬랙 (교통수단별)
    private static final int[] ALIGHT_SLACK = {
        30,   // 지하철: 30초
        10,   // 버스: 10초
        60    // 철도: 1분
    };

    @Override
    public int transferSlack() {
        return TRANSFER_SLACK;
    }

    @Override
    public int boardSlack(int slackIndex) {
        return BOARD_SLACK[slackIndex];
    }

    @Override
    public int alightSlack(int slackIndex) {
        return ALIGHT_SLACK[slackIndex];
    }
}
```

---

## 5. 메인 엔진 클래스

### 5.1 KoreanRaptor.java

```java
public class KoreanRaptor {

    private final TransitData transitData;
    private final KoreanTransitDataProvider dataProvider;
    private final StreetGraph streetGraph;  // OSM 도보 경로용 (nullable)
    private final AStar astar;

    private KoreanRaptor(TransitData transitData, StreetGraph streetGraph) {
        this.transitData = transitData;
        this.dataProvider = new KoreanTransitDataProvider(transitData);
        this.streetGraph = streetGraph;
        this.astar = (streetGraph != null) ? new AStar(streetGraph) : null;
    }

    /**
     * 경로 탐색
     */
    public List<Journey> route(
        double fromLat, double fromLon,
        double toLat, double toLon,
        int departureTime,           // 초 (09:00 = 32400)
        RaptorProfile profile        // MULTI_CRITERIA, STANDARD, etc.
    ) {
        // 1. Access 경로 생성 (출발지 → 가까운 정류장들)
        List<KoreanAccessEgress> accessPaths = findAccessPaths(fromLat, fromLon);

        // 2. Egress 경로 생성 (가까운 정류장들 → 목적지)
        List<KoreanAccessEgress> egressPaths = findEgressPaths(toLat, toLon);

        // 3. Raptor 요청 생성
        RaptorRequest<KoreanTripSchedule> request = RaptorRequest
            .<KoreanTripSchedule>of()
            .profile(profile)
            .searchParams()
                .earliestDepartureTime(departureTime)
                .searchWindow(Duration.ofHours(2))
                .addAccessPaths(accessPaths)
                .addEgressPaths(egressPaths)
                .build()
            .build();

        // 4. Raptor 실행
        RaptorService<KoreanTripSchedule> service = new RaptorService<>(dataProvider);
        RaptorResponse<KoreanTripSchedule> response = service.route(request);

        // 5. 결과 변환
        return ResultMapper.toJourneys(response, transitData, accessPaths, egressPaths);
    }

    /**
     * 역방향 탐색 (도착 시간 기준)
     */
    public List<Journey> routeArriveBy(
        double fromLat, double fromLon,
        double toLat, double toLon,
        int arrivalTime,
        RaptorProfile profile
    ) {
        List<KoreanAccessEgress> accessPaths = findAccessPaths(fromLat, fromLon);
        List<KoreanAccessEgress> egressPaths = findEgressPaths(toLat, toLon);

        RaptorRequest<KoreanTripSchedule> request = RaptorRequest
            .<KoreanTripSchedule>of()
            .profile(profile)
            .searchParams()
                .latestArrivalTime(arrivalTime)
                .searchWindow(Duration.ofHours(2))
                .addAccessPaths(accessPaths)
                .addEgressPaths(egressPaths)
                .build()
            .searchDirection(SearchDirection.REVERSE)
            .build();

        RaptorService<KoreanTripSchedule> service = new RaptorService<>(dataProvider);
        RaptorResponse<KoreanTripSchedule> response = service.route(request);

        return ResultMapper.toJourneys(response, transitData, accessPaths, egressPaths);
    }

    /**
     * 출발지에서 가까운 정류장 찾기
     */
    private List<KoreanAccessEgress> findAccessPaths(double lat, double lon) {
        List<KoreanAccessEgress> paths = new ArrayList<>();
        double maxWalkMeters = 500.0;

        // 후보 정류장 찾기 (직선 거리)
        List<StopDistance> candidates = transitData.findNearbyStops(lat, lon, maxWalkMeters * 1.5);

        for (StopDistance sd : candidates) {
            double walkDuration;
            double actualDistance;

            if (astar != null) {
                // OSM 기반 실제 도보 경로
                WalkPath path = astar.findPath(lat, lon, sd.stop.lat, sd.stop.lon);
                if (path == null || path.distance > maxWalkMeters) continue;
                walkDuration = path.duration;
                actualDistance = path.distance;
            } else {
                // 직선 거리 기반
                if (sd.distance > maxWalkMeters) continue;
                actualDistance = sd.distance;
                walkDuration = sd.distance / 1.2;  // 1.2 m/s
            }

            paths.add(new KoreanAccessEgress(
                sd.stopIndex,
                (int) walkDuration,
                (int) (walkDuration * 100),  // cost
                actualDistance
            ));
        }

        return paths;
    }

    private List<KoreanAccessEgress> findEgressPaths(double lat, double lon) {
        // Access와 동일한 로직
        return findAccessPaths(lat, lon);
    }

    // ═══════════════════════════════════════════════════════════════
    // 빌더
    // ═══════════════════════════════════════════════════════════════

    public static KoreanRaptorBuilder builder() {
        return new KoreanRaptorBuilder();
    }
}
```

### 5.2 KoreanRaptorBuilder.java

```java
public class KoreanRaptorBuilder {

    private Path gtfsDir;
    private Path osmFile;
    private boolean useStreetNetwork = true;

    public KoreanRaptorBuilder gtfs(Path gtfsDir) {
        this.gtfsDir = gtfsDir;
        return this;
    }

    public KoreanRaptorBuilder osm(Path osmFile) {
        this.osmFile = osmFile;
        return this;
    }

    public KoreanRaptorBuilder useStreetNetwork(boolean use) {
        this.useStreetNetwork = use;
        return this;
    }

    public KoreanRaptor build() {
        System.out.println("═══════════════════════════════════════════════════════");
        System.out.println("       한국형 Raptor 엔진 빌드");
        System.out.println("═══════════════════════════════════════════════════════");

        long startTime = System.currentTimeMillis();

        // 1. GTFS 로드
        System.out.println("\n[1/4] GTFS 데이터 로드 중...");
        GtfsLoader gtfsLoader = new GtfsLoader(gtfsDir);
        GtfsBundle gtfs = gtfsLoader.load();
        System.out.printf("  정류장: %,d개%n", gtfs.getStopCount());
        System.out.printf("  노선: %,d개%n", gtfs.getRouteCount());
        System.out.printf("  트립: %,d개%n", gtfs.getTripCount());
        System.out.printf("  시간표: %,d개%n", gtfs.getStopTimeCount());

        // 2. OSM 로드 (선택)
        StreetGraph streetGraph = null;
        if (useStreetNetwork && osmFile != null && Files.exists(osmFile)) {
            System.out.println("\n[2/4] OSM 데이터 로드 중...");
            OsmLoader osmLoader = new OsmLoader();
            OsmData osm = osmLoader.load(osmFile);
            System.out.printf("  노드: %,d개%n", osm.getNodeCount());
            System.out.printf("  웨이: %,d개%n", osm.getWayCount());

            System.out.println("\n[3/4] 도로 그래프 구축 중...");
            StreetGraphBuilder graphBuilder = new StreetGraphBuilder();
            streetGraph = graphBuilder.build(osm, gtfs);
            System.out.printf("  정점: %,d개%n", streetGraph.getVertexCount());
            System.out.printf("  간선: %,d개%n", streetGraph.getEdgeCount());
        } else {
            System.out.println("\n[2/4] OSM 없음 - 직선 거리 환승 사용");
            System.out.println("[3/4] 건너뜀");
        }

        // 3. 환승 생성
        System.out.println("\n[4/4] 환승 데이터 생성 중...");
        TransferGenerator transferGen = new TransferGenerator(gtfs, streetGraph);
        TransferIndex transfers = transferGen.generate();
        System.out.printf("  환승: %,d개%n", transfers.getTotalCount());

        // 4. Raptor 데이터 구조 생성
        System.out.println("\n[5/5] Raptor 데이터 구조 생성 중...");
        TransitDataBuilder dataBuilder = new TransitDataBuilder(gtfs, transfers);
        TransitData transitData = dataBuilder.build();
        System.out.printf("  패턴: %,d개%n", transitData.getPatternCount());

        long elapsed = System.currentTimeMillis() - startTime;
        System.out.println("\n═══════════════════════════════════════════════════════");
        System.out.printf("  빌드 완료! (소요시간: %.1f초)%n", elapsed / 1000.0);
        System.out.println("═══════════════════════════════════════════════════════");

        return new KoreanRaptor(transitData, streetGraph);
    }
}
```

### 5.3 Main.java (CLI)

```java
public class Main {

    private static KoreanRaptor raptor;
    private static final Scanner scanner = new Scanner(System.in);

    public static void main(String[] args) {
        // 데이터 경로
        Path gtfsDir = Path.of("data/gtfs");
        Path osmFile = Path.of("data/osm/south-korea.osm.pbf");

        // Raptor 엔진 빌드
        raptor = KoreanRaptor.builder()
            .gtfs(gtfsDir)
            .osm(osmFile)
            .build();

        // 대화형 CLI
        System.out.println("\n한국형 Raptor 경로탐색 엔진");
        System.out.println("─────────────────────────────────────────────────");

        while (true) {
            try {
                // 입력 받기
                System.out.print("\n출발 좌표 (위도,경도): ");
                String fromInput = scanner.nextLine().trim();
                if (fromInput.equalsIgnoreCase("q")) break;

                System.out.print("도착 좌표 (위도,경도): ");
                String toInput = scanner.nextLine().trim();

                System.out.print("출발 시간 (HH:MM): ");
                String timeInput = scanner.nextLine().trim();

                // 파싱
                double[] from = parseCoord(fromInput);
                double[] to = parseCoord(toInput);
                int departureTime = parseTime(timeInput);

                // 경로 탐색
                long start = System.currentTimeMillis();
                List<Journey> journeys = raptor.route(
                    from[0], from[1],
                    to[0], to[1],
                    departureTime,
                    RaptorProfile.MULTI_CRITERIA
                );
                long elapsed = System.currentTimeMillis() - start;

                // 결과 출력
                printResults(journeys, elapsed);

            } catch (Exception e) {
                System.out.println("오류: " + e.getMessage());
            }
        }

        System.out.println("\n종료합니다.");
    }

    private static void printResults(List<Journey> journeys, long elapsedMs) {
        System.out.println("\n─────────────────────────────────────────────────");
        System.out.printf("[경로 탐색 완료] %.2f초, %d개 경로 발견%n",
            elapsedMs / 1000.0, journeys.size());
        System.out.println("─────────────────────────────────────────────────");

        int idx = 1;
        for (Journey journey : journeys) {
            System.out.printf("%n■ 경로 %d: %s 출발 → %s 도착 (%d분, 환승 %d회)%n",
                idx++,
                formatTime(journey.getDepartureTime()),
                formatTime(journey.getArrivalTime()),
                journey.getDurationMinutes(),
                journey.getTransferCount()
            );

            int legIdx = 1;
            for (Leg leg : journey.getLegs()) {
                if (leg.isWalk()) {
                    System.out.printf("   %d. 도보 %d분 (%.0fm)%n",
                        legIdx++, leg.getDurationMinutes(), leg.getDistanceMeters());
                } else {
                    System.out.printf("   %d. %s 승차 %s → %s 하차 %s%n",
                        legIdx++,
                        leg.getRouteName(),
                        formatTime(leg.getBoardTime()),
                        leg.getAlightStopName(),
                        formatTime(leg.getAlightTime())
                    );
                }
            }
        }
    }

    private static double[] parseCoord(String input) {
        String[] parts = input.split(",");
        return new double[] {
            Double.parseDouble(parts[0].trim()),
            Double.parseDouble(parts[1].trim())
        };
    }

    private static int parseTime(String input) {
        String[] parts = input.split(":");
        int hours = Integer.parseInt(parts[0]);
        int minutes = Integer.parseInt(parts[1]);
        return hours * 3600 + minutes * 60;
    }

    private static String formatTime(int seconds) {
        int h = seconds / 3600;
        int m = (seconds % 3600) / 60;
        return String.format("%02d:%02d", h, m);
    }
}
```

---

## 6. 데이터 변환 흐름

### 6.1 GTFS → Raptor 변환

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  stop_times.txt (2000만 레코드)                                              │
│       │                                                                     │
│       ▼                                                                     │
│  ┌───────────────────────────────────────────┐                             │
│  │ 트립별 그룹화                              │                             │
│  │ tripId → List<StopTime>                   │                             │
│  └─────────────────┬─────────────────────────┘                             │
│                    │                                                        │
│                    ▼                                                        │
│  ┌───────────────────────────────────────────┐                             │
│  │ 패턴 생성 (정류장 순서로 그룹화)            │                             │
│  │                                           │                             │
│  │ [S1→S2→S3→S4] : Trip1, Trip2, Trip3...   │                             │
│  │ [S1→S2→S5→S6] : Trip4, Trip5...          │                             │
│  │ ...                                       │                             │
│  └─────────────────┬─────────────────────────┘                             │
│                    │                                                        │
│                    ▼                                                        │
│  ┌───────────────────────────────────────────┐                             │
│  │ 시간순 정렬 (각 패턴 내)                    │                             │
│  │                                           │                             │
│  │ Pattern1: [Trip1(06:00), Trip2(06:30)...] │                             │
│  │ Pattern2: [Trip4(05:30), Trip5(06:00)...] │                             │
│  └─────────────────┬─────────────────────────┘                             │
│                    │                                                        │
│                    ▼                                                        │
│  ┌───────────────────────────────────────────┐                             │
│  │ Raptor 구조 생성                           │                             │
│  │                                           │                             │
│  │ KoreanTripPattern[]                       │                             │
│  │ KoreanTripSchedule[]                      │                             │
│  │ KoreanRoute[]                             │                             │
│  │ KoreanTimeTable[]                         │                             │
│  └───────────────────────────────────────────┘                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 정류장-노선 인덱스

```java
/**
 * 각 정류장에서 출발하는 노선들의 인덱스
 * (routeIndexIterator에서 사용)
 */
public class RoutesByStop {
    // stopIndex → routeIndex[]
    private final int[][] routesByStop;

    public int[] getRoutes(int stopIndex) {
        return routesByStop[stopIndex];
    }
}
```

---

## 7. 구현 순서

### Step 1: 프로젝트 설정 (0.5일)

```
□ Gradle 프로젝트 생성
□ 의존성 설정 (OTP raptor JAR)
□ 패키지 구조 생성
```

### Step 2: GTFS 로더 (2일)

```
□ GtfsStop, GtfsRoute, GtfsTrip, GtfsStopTime 모델
□ GtfsLoader 구현 (대용량 스트리밍)
□ GtfsBundle 구현

테스트:
□ 파일 로드
□ 레코드 수 검증
```

### Step 3: OSM 로더 & 도로 그래프 (2-3일)

```
□ OsmNode, OsmWay 모델
□ OsmLoader (PBF 파싱)
□ StreetVertex, StreetEdge
□ StreetGraph
□ AStar 알고리즘

테스트:
□ PBF 파싱
□ A* 경로탐색
```

### Step 4: 환승 생성 (1일)

```
□ TransferGenerator
□ TransferIndex

테스트:
□ 환승 거리/시간 검증
```

### Step 5: Raptor SPI 구현 (3-4일)

```
□ KoreanTripSchedule
□ KoreanTripPattern
□ KoreanRoute, KoreanTimeTable
□ KoreanTripScheduleSearch (이진 검색)
□ KoreanTransfer, KoreanAccessEgress
□ KoreanCostCalculator, KoreanSlackProvider
□ KoreanTransitDataProvider

테스트:
□ 각 인터페이스 단위 테스트
□ 트립 검색 성능 테스트
```

### Step 6: 데이터 빌더 (2일)

```
□ TransitDataBuilder
□ PatternBuilder (패턴 그룹화)
□ RoutesByStop 인덱스

테스트:
□ 데이터 변환 검증
```

### Step 7: 메인 엔진 (1일)

```
□ KoreanRaptor
□ KoreanRaptorBuilder
□ AccessEgressFinder
□ ResultMapper

테스트:
□ 전체 빌드 프로세스
```

### Step 8: CLI & 통합 테스트 (2일)

```
□ Main.java (CLI)
□ 통합 테스트

테스트 시나리오:
□ 서울역 → 강남역
□ 단일 노선 경로
□ 환승 1-2회 경로
□ 버스 + 지하철 복합
```

---

## 8. 예상 일정

| Step | 작업 | 기간 |
|------|------|------|
| 1 | 프로젝트 설정 | 0.5일 |
| 2 | GTFS 로더 | 2일 |
| 3 | OSM & 도로 그래프 | 2-3일 |
| 4 | 환승 생성 | 1일 |
| 5 | **Raptor SPI 구현** | 3-4일 |
| 6 | 데이터 빌더 | 2일 |
| 7 | 메인 엔진 | 1일 |
| 8 | CLI & 통합 테스트 | 2일 |
| **총계** | | **13-16일** |

---

## 9. 성능 최적화 포인트

### 9.1 메모리

```java
// ✅ 좋음: primitive 배열
private final int[] arrivalTimes;
private final int[] stopIndexes;

// ❌ 나쁨: 객체 배열
private final Integer[] arrivalTimes;
```

### 9.2 검색 속도

```java
// ✅ Flyweight 패턴
private final MutableBoardAlightEvent event = new MutableBoardAlightEvent();

public RaptorBoardOrAlightEvent search(...) {
    event.set(trip, index, stopPos, time);  // 재사용
    return event;
}

// ❌ 매번 새 객체 생성
public RaptorBoardOrAlightEvent search(...) {
    return new BoardAlightEvent(trip, index, stopPos, time);  // GC 부담
}
```

### 9.3 인덱스 구조

```java
// ✅ BitSet + int[] 조합
BitSet activeRoutes = new BitSet();
for (int route : routesByStop[stopIndex]) {
    activeRoutes.set(route);
}
```

---

## 10. 테스트 시나리오

### 10.1 기본 테스트

```
1. 서울역 (37.5547, 126.9707) → 강남역 (37.4979, 127.0276)
   - 예상: 지하철 1+2호선 환승
   - 소요시간: ~35분

2. 홍대입구 (37.5568, 126.9237) → 잠실역 (37.5133, 127.1001)
   - 예상: 2호선 직행
   - 소요시간: ~45분

3. 인천공항 → 서울역
   - 예상: 공항철도
```

### 10.2 성능 테스트

```
- 경로 탐색 1000회 반복
- 목표: 평균 < 500ms
```

---

## 11. 한국 OSM 다운로드

```bash
# 전국
wget https://download.geofabrik.de/asia/south-korea-latest.osm.pbf
# 크기: ~600MB

# 서울만 (BBBike)
# https://extract.bbbike.org/ 에서 추출
```

---

*마지막 업데이트: 2026-01-05*
