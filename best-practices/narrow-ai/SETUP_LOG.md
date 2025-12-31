# SAP HANA Cloud Setup 작업 일지

**날짜**: 2025-12-31
**목표**: SAP HANA Cloud 환경을 AI/ML 프로젝트(PAL)에 맞게 설정

---

## 1. 초기 문제 인식

### 문제점
- Time Series Forecasting 노트북 실행 시 테이블이 **DBADMIN 스키마**에 생성됨
- `.env`에 지정한 스키마(`BTP_AI_BP`)가 존재하지 않을 경우, 사용자의 기본 스키마로 생성됨
- DBADMIN 계정을 직접 사용하는 것은 보안상 권장되지 않음

### 해결 방향
1. 프로젝트 전용 **스키마 생성**
2. 별도의 **사용자 계정 생성** (DBADMIN은 관리용으로만 사용)
3. PAL(Predictive Analysis Library) 사용을 위한 **권한 설정**

---

## 2. 프로젝트 구조 개선

### 기존 구조의 문제점
- 각 프로젝트별로 스키마/사용자 생성 코드를 중복 작성할 뻔함

### 개선된 구조
```
/best-practices/narrow-ai/
├── 00_Setup_HANA_User_and_Schema.ipynb  ← 중앙 집중식 설정 노트북
├── time-series-forecasting/
├── classification/
├── clustering/
└── ...
```

**핵심 아이디어**:
- 스키마/사용자 관리는 **narrow-ai 레벨**에서 한 번만 수행
- 각 하위 프로젝트는 생성된 사용자 계정으로 작업

---

## 3. 스키마 및 사용자 생성

### 3.1 스키마 생성
```sql
CREATE SCHEMA BTP_AI_BP
```

**중요 사항**:
- 스키마는 **무료** (데이터 저장 시에만 스토리지 비용 발생)
- 스키마명은 **변경 불가** (삭제 후 재생성만 가능)

### 3.2 사용자 생성
```sql
CREATE USER PAL_ADMIN PASSWORD "PAL_ADNINa123!" NO FORCE_FIRST_PASSWORD_CHANGE
```

**비밀번호 정책**:
- SAP HANA는 **A1a 규칙** 적용
- 대문자 + 소문자 + 숫자 필수
- 처음에 약한 비밀번호로 에러 발생 → 수정

### 3.3 권한 부여

#### 성공한 권한 (필수)
```sql
-- 1. 스키마 권한 (가장 중요)
GRANT ALL PRIVILEGES ON SCHEMA BTP_AI_BP TO PAL_ADMIN

-- 2. PAL 실행 권한
GRANT AFL__SYS_AFL_AFLPAL_EXECUTE TO PAL_ADMIN
```

#### 실패한 권한 (선택적)
```sql
-- DBADMIN 권한으로는 부여 불가 (SYSTEM 계정 필요)
GRANT EXECUTE ON _SYS_AFL.PAL_UNIVARIATE_ANALYSIS TO PAL_ADMIN  -- 실패
GRANT CREATE ANY TO PAL_ADMIN  -- 실패
GRANT SELECT ON SCHEMA _SYS_BIC TO PAL_ADMIN  -- 실패
```

**해결 방법**:
- 실패한 권한들을 `try-except`로 감싸서 경고만 표시
- 핵심 권한(스키마 + AFL PAL)만으로도 PAL 사용 가능

---

## 4. 테이블 생성 오류 수정

### 문제
```python
# 잘못된 방법
table_name = f'{schema}.{table}'  # "BTP_AI_BP.OVERNIGHTSTAYS"
df_remote = dataframe.create_dataframe_from_pandas(
    connection_context=conn,
    pandas_df=df_data,
    table_name=table_name,  # 전체 문자열이 테이블명으로 처리됨
)
```
→ 결과: `DBADMIN.BTP_AI_BP.OVERNIGHTSTAYS` (잘못된 위치)

### 해결
```python
# 올바른 방법
df_remote = dataframe.create_dataframe_from_pandas(
    connection_context=conn,
    pandas_df=df_data,
    table_name='OVERNIGHTSTAYS',  # 테이블명만
    schema='BTP_AI_BP',  # 스키마는 별도 파라미터
    force=True,
    replace=False
)
```
→ 결과: `BTP_AI_BP.OVERNIGHTSTAYS` (올바른 위치)

---

## 5. Script Server 설정

### Script Server란?
- SAP HANA의 **시스템 레벨 서비스**
- PAL(Predictive Analysis Library) 실행에 **필수**
- ML/AI 함수들이 Script Server에서 동작

### 요구사항
1. **최소 3 vCPU** 인스턴스 필요
2. **Trial 계정에서는 사용 불가**
3. BTP Cockpit 또는 SAP Support를 통해서만 활성화

### 활성화 방법

#### Method 1: BTP Cockpit (권장)
```
BTP Cockpit
→ SAP HANA Cloud
→ 인스턴스 Actions (⋯)
→ Manage Configuration
→ Advanced Settings / Additional Features
→ Script Server 활성화
→ 인스턴스 재시작 (필요 시)
```

#### Method 2: SAP Support
- 컴포넌트: **HAN-CLS-HC**
- 요청 내용: Script Server 활성화
- 인스턴스 ID, vCPU 정보 제공

### 중요: 코드로 제어 불가 ❌
```python
# 이런 코드는 존재하지 않음 (불가능)
ALTER SYSTEM ENABLE SCRIPT SERVER  # ❌
```

**이유**:
- Script Server는 데이터베이스 설정이 아닌 **시스템 인프라 설정**
- 데이터베이스를 켜고 끄는 것과 비슷한 레벨
- 보안 및 안정성을 위해 BTP Cockpit으로만 제어 가능

### Script Server 상태 확인
```python
# M_SERVICES 시스템 뷰로 확인 가능
SELECT
    HOST,
    SERVICE_NAME,
    ACTIVE_STATUS,
    PROCESS_ID,
    PORT,
    COORDINATOR_TYPE
FROM M_SERVICES
WHERE SERVICE_NAME = 'scriptserver'
```

---

## 6. 비용 관련 정리

### 6.1 무료 항목
- ✅ 스키마 생성
- ✅ 사용자 생성
- ✅ Script Server 자체 (서비스 사용료 없음)

### 6.2 유료 항목
- 💰 **vCPU + 메모리** (시간당 또는 월 구독)
- 💰 **스토리지** (저장된 데이터 양)
- 💰 **Script Server를 위한 3+ vCPU 인스턴스** (더 큰 인스턴스 = 더 비쌈)

### 6.3 인스턴스 스펙 변경
**질문**: "64GB로 올렸다가 Script Server 끄고 48GB로 내리면?"

**답변**:
- ❌ Script Server는 On/Off가 자유롭지 않음 (BTP Cockpit/SAP Support 필요)
- ❌ 스펙 변경 시 인스턴스 재시작 필요
- ✅ **대신 Stop/Start 활용 권장**:
  - Stop = 과금 중단, 데이터/설정 보존
  - Start = 이전 설정 그대로 복원

### 6.4 비용 절약 전략

#### 전략 1: 필요할 때만 Stop/Start
```
평소: 인스턴스 Stop (과금 없음)
PAL 작업 시: Start → 작업 → Stop
```

#### 전략 2: 48GB로도 충분할 수 있음
- Script Server는 **3+ vCPU**만 있으면 작동
- 64GB까지 안 가도 48GB에서 가능

#### 전략 3: 개발/운영 분리
- 개발용: 작은 인스턴스 (테이블 생성, 데이터 확인)
- ML 작업용: 큰 인스턴스 (PAL 실행 시만 사용)

---

## 7. 주요 에러 및 해결

### 에러 1: 비밀번호 정책 위반
```
(412, "invalid password layout: password has to meet the rule ['A1a']")
```
**원인**: 대문자/소문자/숫자 중 하나 누락
**해결**: `PAL_ADNINa123!` (대문자 + 소문자 + 숫자 포함)

### 에러 2: PAL 권한 부여 실패
```
(258, "insufficient privilege: ...")
```
**원인**: DBADMIN이 특정 PAL 함수 권한을 부여할 권한 없음
**해결**: Try-except로 감싸고 핵심 권한(AFL__SYS_AFL_AFLPAL_EXECUTE)만 필수로 처리

### 에러 3: CREATE ANY 권한 부여 실패
```
(334, "invalid user privilege: CREATE ANY")
```
**원인**: DBADMIN이 시스템 권한 부여 불가
**해결**: Try-except로 감싸고 스키마 권한만으로 충분함을 안내

### 에러 4: 테이블이 잘못된 스키마에 생성
```
DBADMIN.BTP_AI_BP.OVERNIGHTSTAYS  (X)
BTP_AI_BP.OVERNIGHTSTAYS  (O)
```
**원인**: `table_name=f'{schema}.{table}'` 형식 사용
**해결**: `schema` 파라미터 분리

---

## 8. 생성된 파일

### [00_Setup_HANA_User_and_Schema.ipynb](00_Setup_HANA_User_and_Schema.ipynb)

**구조**:

#### Part 1: CREATE
1. 환경 변수 로드 및 DBADMIN 연결
2. 설정 (스키마명, 사용자명, 비밀번호)
3. 스키마 생성
4. 사용자 생성 + 권한 부여
5. 권한 확인
6. Script Server 상태 확인
7. Script Server 활성화 가이드
8. 새 사용자로 연결 테스트
9. `.env` 업데이트 가이드

#### Part 2: DELETE
1. 사용자 삭제 (CASCADE)
2. 스키마 삭제 (CASCADE)

#### Part 3: Final Cleanup
- Admin 연결 종료

---

## 9. 최종 워크플로우

### 초기 설정 (1회만)
1. **DBADMIN 계정으로 연결** → `00_Setup_HANA_User_and_Schema.ipynb` 실행
2. **스키마 생성**: `BTP_AI_BP`
3. **사용자 생성**: `PAL_ADMIN` + 권한 부여
4. **Script Server 확인** 및 활성화 (필요 시 BTP Cockpit)
5. **`.env` 파일 업데이트**:
   ```env
   hana_user=PAL_ADMIN
   hana_password=PAL_ADNINa123!
   HANA_SCHEMA=BTP_AI_BP
   ```

### 프로젝트 실행 (이후)
1. Jupyter Kernel 재시작
2. 각 프로젝트 노트북 실행:
   - `time-series-forecasting/`
   - `classification/`
   - `clustering/`
   - 등등
3. 자동으로 `PAL_ADMIN` 계정, `BTP_AI_BP` 스키마 사용

---

## 10. 남은 확인 사항

### Script Server 관련
- [ ] 현재 인스턴스 스펙 확인 (vCPU, 메모리)
- [ ] Script Server 활성화 여부 확인
- [ ] 필요 시 BTP Cockpit에서 Script Server 활성화

### 인스턴스 스펙 조정
- [ ] 64GB 필요 여부 재검토 (48GB로 충분할 수 있음)
- [ ] PAL 작업 시에만 인스턴스 Start, 평소엔 Stop으로 비용 절감

### 테스트
- [ ] `PAL_ADMIN` 계정으로 테이블 생성 테스트
- [ ] PAL 함수 실행 테스트 (Script Server 활성화 후)

---

## 11. 참고 문서

### SAP 공식 문서
- [SAP KBA 3216010 - Enable/Disable Script Server](https://userapps.support.sap.com/sap/support/knowledge/en/3216010)
- [SAP HANA Cloud - Script Server Configuration](https://help.sap.com/docs/SAP_DATASPHERE/9f804b8efa8043539289f42f372c4862/287194276a7d4d778ec98fdde5f61335.html)
- [Getting Started with PAL and APL](https://community.sap.com/t5/technology-blog-posts-by-sap/getting-started-with-sap-hana-cloud-pal-and-apl/ba-p/13482157)

### 핵심 개념
- **DBADMIN vs SYSTEM**: DBADMIN은 일반 관리자, SYSTEM은 최상위 권한
- **스키마 vs 사용자**: 스키마는 객체 컨테이너, 사용자는 접속 계정
- **PAL vs AFL**: PAL은 ML 라이브러리, AFL은 Application Function Library (PAL 포함)
- **Script Server**: PAL 실행을 위한 시스템 서비스 (최소 3 vCPU)

---

## 12. 핵심 교훈

1. **보안**: DBADMIN 직접 사용 X → 프로젝트 전용 계정 생성 O
2. **구조**: 공통 설정은 상위 폴더에서 중앙 관리
3. **권한**: 핵심 권한만 확보, 실패한 권한은 try-except 처리
4. **비용**: Stop/Start로 비용 절감, Script Server는 인스턴스 스펙에 포함
5. **제약**: Script Server는 코드로 제어 불가, BTP Cockpit 필수

---

**작성자**: Claude Code
**작업 기간**: 2025-12-31
**상태**: ✅ 설정 완료, Script Server 활성화 대기 중
