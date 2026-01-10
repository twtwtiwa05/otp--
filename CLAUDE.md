# CLAUDE.md

이 파일은 Claude Code가 이 프로젝트에서 작업할 때 참고하는 가이드입니다.

## Project Overview

OTP의 **Raptor 모듈(JAR)을 그대로 사용**하고, SPI 인터페이스만 구현하여 한국 GTFS 데이터를 연결한 **CLI 기반 대중교통 경로탐색 엔진**

**Tech Stack:** Java 21, Gradle, OTP Raptor JAR, 한국 GTFS, OpenStreetMap

**목표:** 출발/도착 좌표 입력 → Raptor 실행 → 경로 출력 (< 1초)

## Essential Commands

```bash
# ═══════════════════════════════════════════════════════════
# OTP Raptor JAR 빌드 (OpenTripPlanner-dev-2.x 폴더에서)
# ═══════════════════════════════════════════════════════════

cd OpenTripPlanner-dev-2.x/OpenTripPlanner-dev-2.x

# Raptor + Utils 모듈만 빌드 (가벼움, 권장)
mvn package -pl raptor,utils -am -DskipTests

# 결과물:
#   raptor/target/otp-raptor-*.jar
#   utils/target/otp-utils-*.jar

# 또는 전체 빌드 (무거움)
mvn package -DskipTests
# 결과물: otp-shaded/target/otp-shaded-*.jar

# ═══════════════════════════════════════════════════════════
# Korean Raptor 프로젝트 빌드 (korean-otp 폴더에서)
# ═══════════════════════════════════════════════════════════

cd korean-otp

# 빌드
./gradlew build

# 테스트
./gradlew test

# Fat JAR 생성
./gradlew fatJar

# 실행
java -Xmx8G -jar build/libs/korean-raptor-1.0.0-SNAPSHOT-all.jar

# ═══════════════════════════════════════════════════════════
# GTFS 전처리 (이미 완료됨, 참고용)
# ═══════════════════════════════════════════════════════════

cd korean-otp/scripts
python gtfs_preprocessor.py --validate
```

## Project Structure

```
otp핵심/
├── CLAUDE.md                                ← 이 파일
├── RAPTOR_ALGORITHM_COMPLETE_ANALYSIS.md    ← Raptor 알고리즘 상세 분석
│
├── OpenTripPlanner-dev-2.x/                 ← OTP 원본 (JAR 빌드용)
│   └── OpenTripPlanner-dev-2.x/
│       ├── raptor/                          ← Raptor 모듈 (우리가 사용)
│       └── utils/                           ← 유틸리티 모듈
│
└── korean-otp/                              ← ★ 우리 프로젝트
    ├── build.gradle
    ├── settings.gradle
    │
    ├── libs/                                ← OTP JAR 넣을 곳
    │   ├── otp-raptor-*.jar                 ← 빌드 후 복사
    │   └── otp-utils-*.jar
    │
    ├── data/
    │   ├── gtfs/                            ← 변환된 GTFS (완료)
    │   │   ├── stops.txt      (212,105)
    │   │   ├── routes.txt     (27,138)
    │   │   ├── trips.txt      (349,580)
    │   │   ├── stop_times.txt (20,871,237)
    │   │   └── ...
    │   └── osm/                             ← 한국 OSM (다운로드 필요)
    │       └── south-korea.osm.pbf
    │
    ├── scripts/                             ← Python 전처리기 (완료)
    │
    ├── docs/
    │   └── PHASE2_SPI_IMPLEMENTATION_PLAN.md ← ★ 구현 상세 계획
    │
    └── src/main/java/kr/otp/               ← Java 소스 (구현 예정)
        ├── Main.java                        ← CLI 진입점
        ├── core/
        │   └── KoreanRaptor.java            ← 메인 엔진
        ├── gtfs/                            ← GTFS 로더
        ├── osm/                             ← OSM + A* 도보 경로
        ├── transfer/                        ← 환승 생성기
        ├── raptor/spi/                      ← ★★★ SPI 구현체
        ├── result/                          ← 결과 모델
        └── util/                            ← 유틸리티
```

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Korean Raptor Engine                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  [한국 GTFS]           [한국 OSM]                           │
│       │                     │                              │
│       ▼                     ▼                              │
│  ┌─────────┐          ┌───────────┐                       │
│  │ GTFS    │          │ OSM + A*  │                       │
│  │ Loader  │          │ 도보 경로  │                       │
│  └────┬────┘          └─────┬─────┘                       │
│       └───────┬─────────────┘                              │
│               ▼                                            │
│  ┌──────────────────────────────────┐                     │
│  │ SPI 구현체 (우리가 구현)           │                     │
│  │ - KoreanTransitDataProvider      │                     │
│  │ - KoreanTripSchedule             │                     │
│  │ - KoreanTripPattern              │                     │
│  └───────────────┬──────────────────┘                     │
│                  │                                         │
│  ════════════════╪════════════════════════════════════    │
│                  ▼  SPI Interface                          │
│  ┌──────────────────────────────────┐                     │
│  │ OTP Raptor JAR (그대로 사용)      │                     │
│  │ RaptorService.route() 호출       │                     │
│  └──────────────────────────────────┘                     │
│                  │                                         │
│                  ▼                                         │
│            [경로 결과]                                      │
└────────────────────────────────────────────────────────────┘
```

## Current Status

| Phase | 작업 | 상태 |
|-------|------|------|
| 1 | GTFS 전처리 (Python) | ✅ 완료 |
| 2 | Gradle 프로젝트 설정 | ✅ 완료 |
| 2 | OTP Raptor JAR 빌드 | ⏳ 대기 |
| 2 | GTFS 로더 (Java) | ⏳ 대기 |
| 2 | OSM & A* 도보 경로 | ⏳ 대기 |
| 2 | **SPI 구현체** | ⏳ 대기 |
| 2 | 메인 엔진 & CLI | ⏳ 대기 |
| 2 | 통합 테스트 | ⏳ 대기 |

## Key Files

| 파일 | 위치 | 설명 |
|------|------|------|
| **구현 계획** | `korean-otp/docs/PHASE2_SPI_IMPLEMENTATION_PLAN.md` | SPI 구현 상세, 코드 예시 |
| **알고리즘 분석** | `RAPTOR_ALGORITHM_COMPLETE_ANALYSIS.md` | Raptor 알고리즘 이해 |
| **OTP Raptor SPI** | `OpenTripPlanner-dev-2.x/.../raptor/spi/` | 구현해야 할 인터페이스 |
| **OTP 테스트 구현체** | `OpenTripPlanner-dev-2.x/.../raptor/src/test/.../transit/` | SPI 구현 참조 |

## SPI Interfaces to Implement

```java
// 메인 - 모든 데이터 제공
RaptorTransitDataProvider<T>  → KoreanTransitDataProvider

// 트립/패턴
RaptorTripSchedule            → KoreanTripSchedule
RaptorTripPattern             → KoreanTripPattern
RaptorRoute<T>                → KoreanRoute
RaptorTimeTable<T>            → KoreanTimeTable
RaptorTripScheduleSearch<T>   → KoreanTripScheduleSearch  // 성능 핵심!

// 환승/접근
RaptorTransfer                → KoreanTransfer
RaptorAccessEgress            → KoreanAccessEgress

// 비용/슬랙
RaptorCostCalculator<T>       → KoreanCostCalculator
RaptorSlackProvider           → KoreanSlackProvider
```

## Next Steps

1. **OTP Raptor JAR 빌드** → `korean-otp/libs/`에 복사
2. **GTFS 모델 & 로더** 구현
3. **SPI 구현체** 작성
4. **메인 엔진** 완성
5. **테스트** (서울역 → 강남역)

## References

- **구현 계획:** `korean-otp/docs/PHASE2_SPI_IMPLEMENTATION_PLAN.md`
- **알고리즘 분석:** `RAPTOR_ALGORITHM_COMPLETE_ANALYSIS.md`
- **OTP 원본:** `OpenTripPlanner-dev-2.x/`
- **RAPTOR 논문:** "Round-Based Public Transit Routing" (Microsoft Research, 2012)

## OSM Data Download

```bash
# 한국 전체 (~600MB)
wget https://download.geofabrik.de/asia/south-korea-latest.osm.pbf -O korean-otp/data/osm/south-korea.osm.pbf
```

---
*마지막 업데이트: 2026-01-05*
