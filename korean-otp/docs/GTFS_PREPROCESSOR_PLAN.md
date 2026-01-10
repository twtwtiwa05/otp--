# 한국 GTFS 전처리기 개발 계획

> **목적**: 한국 KTDB GTFS 데이터를 OTP(OpenTripPlanner) 호환 표준 GTFS로 변환
> **작성일**: 2024-01-04
> **작업 위치**: `C:\Users\USER\OneDrive\Desktop\연구실\otp핵심\korean-otp\`

---

## 1. 현황 분석

### 1.1 한국 GTFS 데이터 현황

| 파일 | 레코드 수 | 크기 | 비고 |
|------|----------|------|------|
| agency.txt | 1 | 91B | KTDB 단일 기관 |/
| calendar.txt | 1 | 128B | 전일 운행 (B1) |
| stops.txt | 212,106 | 13MB | 전국 정류장 |
| routes.txt | 27,139 | 1.6MB | 전국 노선 |
| trips.txt | 349,581 | 17MB | 운행 회차 |
| stop_times.txt | 20,871,238 | 1.7GB | 정차 시간표 |

### 1.2 OTP 호환성 분석

| 파일 | 호환 상태 | 문제점 | 필요 작업 |
|------|----------|--------|----------|
| agency.txt | ✅ 호환 | BOM 포함 | BOM 제거 |
| calendar.txt | ✅ 호환 | BOM 포함 | BOM 제거 |
| stops.txt | ✅ 호환 | BOM 포함 | BOM 제거 |
| routes.txt | ❌ **불일치** | route_type 매핑 다름 | **route_type 변환 필수** |
| trips.txt | ✅ 호환 | BOM 포함 | BOM 제거 |
| stop_times.txt | ⚠️ 부분 호환 | stop_sequence가 float | **정수 변환 필수** |
| transfers.txt | ❌ 없음 | 파일 없음 | 도시철도환승정보 변환 (선택) |

---

## 2. 핵심 변환 작업

### 2.1 route_type 매핑 변환 (필수)

한국 KTDB의 route_type은 표준 GTFS와 **완전히 다른 의미**를 가집니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    route_type 매핑 테이블                        │
├──────────┬────────────────────┬──────────┬─────────────────────┤
│ 한국 코드 │ 한국 의미          │ 변환 코드 │ OTP 의미            │
├──────────┼────────────────────┼──────────┼─────────────────────┤
│    0     │ 시내/농어촌/마을버스│    3     │ BUS                 │
│    1     │ 도시철도/경전철    │    1     │ SUBWAY              │
│    2     │ 해운              │    4     │ FERRY               │
│    3     │ 시외버스          │    3     │ BUS                 │
│    4     │ 일반철도          │    2     │ RAIL                │
│    5     │ 공항리무진버스    │    3     │ BUS                 │
│    6     │ 고속철도          │    2     │ RAIL                │
│    7     │ 항공              │   1100   │ AIRPLANE (TPEG확장) │
└──────────┴────────────────────┴──────────┴─────────────────────┘
```

**변환하지 않으면 발생하는 문제:**
- 한국 시내버스(0) → OTP에서 TRAM(트램)으로 인식
- 한국 일반철도(4) → OTP에서 FERRY(페리)로 인식

### 2.2 stop_sequence 정수 변환 (필수)

```
현재 (한국 GTFS):
  stop_sequence = 1.000000000 (float 형태)

변환 후:
  stop_sequence = 1 (정수)
```

OTP의 StopTimeMapper는 정수형 stop_sequence를 기대합니다.

### 2.3 transfers.txt 생성 (선택)

도시철도환승정보.xlsx 파일을 GTFS transfers.txt로 변환합니다.

```
원본 (도시철도환승정보.xlsx):
  Fr_Stop_ID: RS_ACC1_S-1-0150
  To_Stop_ID: RS_ACC1_S-1-0426
  Time_Min: 2.208333

변환 후 (transfers.txt):
  from_stop_id,to_stop_id,transfer_type,min_transfer_time
  RS_ACC1_S-1-0150,RS_ACC1_S-1-0426,2,132

transfer_type=2: 최소 환승 시간 지정 (MIN_TIME)
min_transfer_time: 초 단위
```

**참고**: transfers.txt가 없어도 OTP는 동작합니다.
- OTP의 DirectTransferGenerator가 stops.txt의 좌표와 OSM 도로망을 기반으로 자동으로 도보 환승 경로를 생성합니다.
- transfers.txt는 도시철도 환승처럼 **정확한 환승 시간**이 필요할 때만 사용합니다.

### 2.4 BOM 제거 (필수)

모든 파일에 UTF-8 BOM(`\ufeff`)이 포함되어 있어 제거가 필요합니다.

---

## 3. 프로젝트 구조

```
korean-otp/
├── scripts/
│   ├── gtfs_preprocessor.py          ← 메인 실행 스크립트
│   │
│   ├── converters/                   ← 변환 모듈
│   │   ├── __init__.py
│   │   ├── base_converter.py         ← 기본 클래스 (BOM 제거, 파일 I/O)
│   │   ├── route_type_converter.py   ← route_type 변환
│   │   ├── stop_times_converter.py   ← stop_sequence 정수화
│   │   └── transfers_converter.py    ← 도시철도환승 → transfers.txt
│   │
│   └── config.py                     ← 설정 (경로, 매핑 테이블)
│
├── data/
│   ├── gtfs/                         ← 변환된 GTFS 출력
│   └── raw/                          ← 원본 데이터 심볼릭 링크 (선택)
│
└── docs/
    └── GTFS_PREPROCESSOR_PLAN.md     ← 이 문서
```

---

## 4. 구현 상세

### 4.1 config.py - 설정 파일

```python
# 경로 설정
INPUT_DIR = "C:/Users/USER/OneDrive/Desktop/연구실/otp핵심/raw/gtfs"
OUTPUT_DIR = "C:/Users/USER/OneDrive/Desktop/연구실/otp핵심/korean-otp/data/gtfs"
TRANSFER_XLSX = "C:/Users/USER/OneDrive/Desktop/연구실/otp핵심/raw/gtfs/202303_GTFS_도시철도환승정보.xlsx"

# route_type 매핑 테이블
ROUTE_TYPE_MAPPING = {
    0: 3,     # 시내/농어촌/마을버스 → BUS
    1: 1,     # 도시철도/경전철 → SUBWAY
    2: 4,     # 해운 → FERRY
    3: 3,     # 시외버스 → BUS
    4: 2,     # 일반철도 → RAIL
    5: 3,     # 공항리무진버스 → BUS
    6: 2,     # 고속철도 → RAIL
    7: 1100,  # 항공 → AIRPLANE (TPEG)
}
```

### 4.2 base_converter.py - 기본 변환기

```python
class BaseConverter:
    """파일 읽기/쓰기, BOM 제거 등 공통 기능"""

    def read_csv(self, filepath):
        """BOM 제거하며 CSV 읽기"""
        return pd.read_csv(filepath, encoding='utf-8-sig')

    def write_csv(self, df, filepath):
        """BOM 없이 CSV 쓰기"""
        df.to_csv(filepath, index=False, encoding='utf-8')

    def copy_with_bom_removal(self, input_path, output_path):
        """BOM만 제거하고 복사"""
        pass
```

### 4.3 route_type_converter.py - 노선 타입 변환

```python
class RouteTypeConverter(BaseConverter):
    """routes.txt의 route_type을 표준 GTFS로 변환"""

    def convert(self, input_path, output_path):
        df = self.read_csv(input_path)

        # 변환 전 통계 출력
        print("원본 route_type 분포:")
        print(df['route_type'].value_counts())

        # 매핑 적용
        df['route_type'] = df['route_type'].map(ROUTE_TYPE_MAPPING)

        # 저장
        self.write_csv(df, output_path)
```

### 4.4 stop_times_converter.py - 정차시간 변환

```python
class StopTimesConverter(BaseConverter):
    """stop_times.txt의 stop_sequence를 정수로 변환"""

    def convert(self, input_path, output_path):
        # 청크 단위 처리 (1.7GB 파일)
        chunk_size = 1_000_000

        for i, chunk in enumerate(pd.read_csv(input_path, chunksize=chunk_size)):
            # stop_sequence 정수 변환
            chunk['stop_sequence'] = chunk['stop_sequence'].astype(int)

            # 저장 (첫 청크만 헤더 포함)
            mode = 'w' if i == 0 else 'a'
            header = (i == 0)
            chunk.to_csv(output_path, mode=mode, header=header, index=False)
```

### 4.5 transfers_converter.py - 환승정보 변환

```python
class TransfersConverter(BaseConverter):
    """도시철도환승정보.xlsx → transfers.txt 변환"""

    def convert(self, xlsx_path, output_path):
        # 엑셀 읽기 (헤더가 7번째 행)
        df = pd.read_excel(xlsx_path, header=6)
        df.columns = ['Xfer_SEQ', 'Fr_Stop_ID', 'To_Stop_ID', 'Time_Min', 'Data_Ref']
        df = df.iloc[1:]  # 헤더 중복 제거

        # transfers.txt 형식 변환
        transfers = pd.DataFrame({
            'from_stop_id': df['Fr_Stop_ID'],
            'to_stop_id': df['To_Stop_ID'],
            'transfer_type': 2,  # MIN_TIME
            'min_transfer_time': (df['Time_Min'].astype(float) * 60).astype(int)
        })

        self.write_csv(transfers, output_path)
```

### 4.6 gtfs_preprocessor.py - 메인 실행

```python
def main():
    print("=" * 60)
    print("한국 GTFS 전처리 시작")
    print("=" * 60)

    # 1. 단순 복사 (BOM 제거만)
    copy_files(['agency.txt', 'calendar.txt', 'stops.txt', 'trips.txt'])

    # 2. routes.txt 변환 (route_type)
    RouteTypeConverter().convert(...)

    # 3. stop_times.txt 변환 (stop_sequence)
    StopTimesConverter().convert(...)

    # 4. transfers.txt 생성 (선택)
    TransfersConverter().convert(...)

    # 5. 결과 검증
    validate_output()

    print("완료!")
```

---

## 5. 실행 방법

```bash
# korean-otp 폴더에서 실행
cd C:\Users\USER\OneDrive\Desktop\연구실\otp핵심\korean-otp

# 전처리 실행
python scripts/gtfs_preprocessor.py

# 출력 확인
ls data/gtfs/
```

---

## 6. 검증 체크리스트

변환 완료 후 확인 사항:

- [ ] 모든 파일에 BOM이 제거되었는지 확인
- [ ] routes.txt의 route_type이 올바르게 변환되었는지 확인
- [ ] stop_times.txt의 stop_sequence가 정수인지 확인
- [ ] 레코드 수가 원본과 동일한지 확인
- [ ] OTP에서 그래프 빌드 성공 여부 확인

---

## 7. 참고 자료

### 7.1 OTP GTFS 매핑 코드 위치

```
OpenTripPlanner-dev-2.x/application/src/main/java/org/opentripplanner/gtfs/mapping/
├── TransitModeMapper.java    ← route_type → TransitMode 변환
├── RouteMapper.java          ← routes.txt 처리
├── StopMapper.java           ← stops.txt 처리
├── StopTimeMapper.java       ← stop_times.txt 처리
└── TransferMapper.java       ← transfers.txt 처리
```

### 7.2 표준 GTFS route_type 정의

| 코드 | 의미 |
|------|------|
| 0 | TRAM (트램/경전철) |
| 1 | SUBWAY (지하철) |
| 2 | RAIL (철도) |
| 3 | BUS (버스) |
| 4 | FERRY (페리) |
| 5 | CABLE_CAR |
| 6 | GONDOLA |
| 7 | FUNICULAR |
| 11 | TROLLEYBUS |
| 12 | MONORAIL |
| 100-199 | RAIL (TPEG 확장) |
| 200-299 | COACH (TPEG 확장) |
| 700-799 | BUS (TPEG 확장) |
| 1100-1199 | AIRPLANE (TPEG 확장) |

### 7.3 GTFS transfers.txt transfer_type 정의

| 코드 | 의미 | 설명 |
|------|------|------|
| 0 | RECOMMENDED | 권장 환승 |
| 1 | GUARANTEED | 보장 환승 (차량이 기다림) |
| 2 | MIN_TIME | 최소 환승 시간 지정 |
| 3 | FORBIDDEN | 환승 금지 |
| 4 | STAY_SEATED | 동일 차량 환승 |
| 5 | STAY_SEATED_NOT_ALLOWED | 좌석 유지 불가 |

---

## 8. 변경 이력

| 날짜 | 내용 |
|------|------|
| 2024-01-04 | 초안 작성 |
