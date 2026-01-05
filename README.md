# DB에 수정 반영하기

전자결재 시스템 마이그레이션을 위한 데이터베이스 수정 및 검증 스크립트 모음입니다.

## 📋 목차
- [환경 설정](#환경-설정)
- [스크립트 상세 설명](#스크립트-상세-설명)
  - [조직도 데이터 관리](#1-조직도-데이터-관리)
  - [결재 데이터 변환](#2-결재-데이터-변환)
  - [첨부파일 및 이미지 처리](#3-첨부파일-및-이미지-처리)
  - [데이터 검증](#4-데이터-검증)
  - [문서 내보내기](#5-문서-내보내기)
- [데이터 파일](#데이터-파일)
- [실행 순서 가이드](#실행-순서-가이드)
- [주의사항](#주의사항)

---

## 환경 설정

### Python 설치
- Python 3.10 이상 권장
- [Python 공식 다운로드](https://www.python.org/downloads/)

### 필수 패키지 설치
```bash
pip install jupyter pandas pymysql tqdm beautifulsoup4 lxml openpyxl sqlalchemy
```

### Jupyter Notebook 실행
```bash
jupyter notebook
```

### DB 연결 설정
각 스크립트의 `DB_CONFIG` 섹션을 환경에 맞게 수정하세요:
```python
DB_CONFIG = {
    'host': 'localhost',
    'user': 'root',
    'password': '1234',
    'database': 'any_approval',
    'charset': 'utf8mb4'
}
```

---
## 추가정보
- etc 폴더는 cmds 수정 과정에서 사용된 일회성 스크립트입니다. 핵심 운영 코드에 포함되지 않으며, 향후 유사 작업 시 참조 목적으로 보존하였습니다.
- 관련하여 자세한 설명은 [📁etc 폴더](### 📁etc 폴더) 를 참조해주세요

## 스크립트 상세 설명

### 1. 조직도 데이터 관리

#### `재직자 조직도.ipynb`
**목적**: 현재 재직자 명단을 정리하여 DB 테이블로 관리

**기능**:
- 멤버목록 엑셀에서 '재직자' 그룹만 필터링
- 부서코드 → 부서명 자동 매핑 (부서목록 엑셀 기반)
- `in_employee` 테이블 생성 및 데이터 INSERT

**입력 파일**:
- `멤버목록_YYYY-MM-DD.xlsx`: 전체 사원 목록
- `부서목록_YYYY-MM-DD.xlsx`: 부서코드-부서명 매핑 정보
- 두 파일 모두 [애니파이브 타겟](https://anyfive.daouoffice.com/login?nextUrl=/manage/hr/org/batch-regist)에서 다운

**출력**:
- `in_employee.csv`: 재직자 명단 CSV
- `in_employee` 테이블: DB에 167명 재직자 정보 저장

**테이블 구조**:
| 컬럼 | 설명 |
|------|------|
| 사원명 | 직원 이름 |
| ID | 이메일 ID (emailId) |
| 부서 | 부서명 |
| 사원번호 | 사번 (empNo) |
| 직위 | 직급 |
| 부서코드 | 부서 코드 |

---

#### `퇴사자명단_DB업로드.ipynb`
**목적**: 퇴사자 명단을 DB에 업로드

**기능**:
- 멤버목록에서 퇴사자 그룹 필터링
- `out_employee` 테이블 생성 및 데이터 저장

**유의사항**
- 퇴사자 정보는 별도로 제공되지 않았습니다. 이에 따라 cmds에서 모든 사원의 이름을 추출한 후, 타겟 사이트에서 다운받은 현행 조직도와 대조하여 현행 조직도에 존재하지 않는 인원을 퇴사자로 분류하였습니다. 이 과정에서 많은 수정이 있었고 관련 코드는 [📁etc 폴더](### 📁etc 폴더)에 위치해있습니다.

---

#### `DB에_조직도_다시_반영하기_수정.ipynb`
**목적**: documents 테이블의 drafter, activities 정보를 최신 조직도 기준으로 업데이트

**기능**:
- `in_employee.csv` (재직자 167명) + `out_employee.csv` (퇴사자 336명) 로드
- 23,320개 문서의 drafter 정보 업데이트
- activities 배열 내 각 결재자 정보 업데이트

**매핑 로직**:
1. **재직자 우선**: 사원명이 재직자 목록에 있으면 해당 정보 사용
2. **퇴사자 차선**: 재직자에 없으면 퇴사자 목록에서 검색
3. **미등록자**: 둘 다 없으면 정보 공란 처리
**매핑 기준**: 사원명(이름) 기준 매칭
**업데이트 항목**: emailId, deptName, positionName, deptCode

**처리 결과**:
- drafter 업데이트: 23,320건
- activities 업데이트: 각 문서별 결재자 정보

---

#### `DB_조직도_반영_검증_v2.ipynb`
**목적**: 조직도 반영 결과 검증

**기능**:
- drafter 정보와 CSV 데이터 비교 검증
- activities 내 결재자 정보와 CSV 데이터 비교 검증
- referrers 정보 검증

**검증 항목**:
| 항목 | 검증 내용 |
|------|----------|
| emailId |  ID 일치 여부 |
| deptName | 부서명 일치 여부 |
| positionName | 직위 일치 여부 |
| deptCode | 부서코드 일치 여부 |

---

### 2. 결재 데이터 변환

#### `RETURN 변환.ipynb`
**목적**: 반려(RETURN) 결재 상태를 승인(APPROVAL)으로 변환하고 의견에 [반려] 표시
- 반려 상태 데이터 확인 후 RETURN으로 변경하였으나, 이관 시스템 호환성 이슈로 APPROVAL로 처리하고 의견 앞에 [반려] 표기 추가
- [verify_action_type.ipynb](### verify_action_type.ipynb): RETURN 상태로 변환  /  RETURN 변환.ipynb: RETURN → APPROVAL 변환 및 의견에 [반려] prefix 추가
- => ! verify_action_type.ipynb 실행 후 RETURN 변환.ipynb 실행해야합니다.!

**기능**:
- activities 컬럼에서 `type: "RETURN"` 항목 검색
- `actionLogType`과 `type`을 `APPROVAL`로 변경
- `actionComment` 앞에 `[반려]` 접두사 추가

**변환 예시**:
```json
// 변환 전
{"actionLogType": "RETURN", "type": "RETURN", "actionComment": "내용 보완 필요"}

// 변환 후
{"actionLogType": "APPROVAL", "type": "APPROVAL", "actionComment": "[반려] 내용 보완 필요"}
```

**처리 결과**: 38개 문서, 48건 변환 완료

---

#### `update_db_referrers.ipynb`
**목적**: referrers 필드의 데이터 구조 변환

**기능**:
- 기존 형식 `{name, empNo, deptCode}` → 신규 형식 `{name, emailId}` 변환
- empNo 기준 emailId 매핑 (in_employee + out_employee 테이블 활용)
- 매핑 실패 시 이름으로 2차 매핑

**변환 예시**:
```json
// 변환 전
{"name": "홍길동", "empNo": "201234567890", "deptCode": "DB30"}

// 변환 후  
{"name": "홍길동", "emailId": "gdhong"}
```

**처리 결과**: 7,423개 문서, 12,110건 referrer 변환

---

#### `fix_activities_order_db3.ipynb`
**목적**: activities 배열을 actionDate 기준으로 시간순 정렬 + 구조적 공백 제거

**기능**:
- `actionDate` (유닉스 타임스탬프) 기준 오름차순 정렬
- JSON 구조적 공백 제거 (`": "` → `":"`, `", "` → `","`)
- 문자열 값 내부 공백은 유지

**정렬 예시**:
```json
// 정렬 전 (공백 있음)
[{"name": "B", "actionDate": 300}, {"name": "A", "actionDate": 100}]

// 정렬 후 (공백 제거, 시간순)
[{"name":"A","actionDate":100},{"name":"B","actionDate":300}]
```

**보장 사항**:
- ✅ 키 순서 유지 (Python 3.7+)
- ✅ 문자열 내부 공백 유지 ("합의 합니다." 등)
- ✅ 구조적 공백만 제거

**처리 결과**: 23,320개 문서 처리 (순서 변경 0건, 공백만 변경 23,320건)

---

#### `add_year_column.ipynb`
**목적**: documents 테이블에 year 컬럼 추가

**기능**:
- `created_at` (유닉스 타임스탬프 밀리초) → 연도 추출
- UTC → KST 변환 후 연도 계산
- `year` 컬럼에 저장

**처리 결과**: 23,320개 문서 (2010~2025년 분포)

---

### 3-1. 첨부파일 처리
- 고객사가 제공한 다운로드 파일에 첨부파일 97개(문서로는 85건)가 없습니다.
- empty_path_export.ipynb로 path:""를 조회한 후 csv로 저장하여 (AnyFivePlusCrawler_attaches.java)[https://github.com/000Lee/any_crawling.git]를 통해 누락된 첨부파일을 크롤링 합니다.
- 데이터 오류 방지를 위해 99개 대상으로 모두 다운로드 시도하였으나 1건의 문서는 실패하여 수동으로 처리하였습니다. 
- 이는 apr11213102으로 해당 문서 총 2건중에 1건은 이전에 수집했던 approval_2020_attachments 경로에 있습니다.
- "path":"/PMS_SITE-U7OI43JLDSMO/approval/approval_plus_attachments/ 경로인것이 98개 
- "path":"/PMS_SITE-U7OI43JLDSMO/approval/approval_2020_attachments/ 경로인것이 1개입니다.
- 이후 attaches_transform.ipynb 실행하여 모든 첨부파일의 경로를 일정하게 맞춰줍니다.

#### `empty_path_export.ipynb`
**목적**: 첨부파일 경로가 비어있는 문서 추출

**기능**:
- attaches 필드에서 path가 빈 문자열인 항목 검색
- 해당 문서 목록 CSV로 내보내기

---

#### `attaches_transform.ipynb`
**목적**: 첨부파일 경로에 사이트 prefix 추가 및 JSON 공백 제거

**기능**:
- attaches 필드의 각 path에 `/PMS_SITE-U7OI43JLDSMO/approval/` prefix 추가
- JSON 구조적 공백 제거

**변환 예시**:
```json
// 변환 전
{"name": "file.pdf", "path": "attach/2020/file.pdf"}

// 변환 후
{"name":"file.pdf","path":"/PMS_SITE-U7OI43JLDSMO/approval/attach/2020/file.pdf"}
```

**처리 결과**: 10,148개 문서의 attaches 경로 변환

---

### 3-2. 이미지 처리

#### `convert_base64_images_db.ipynb`
**목적**
- doc_body 내 base64 인코딩 이미지를 파일 경로로 변환

❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗여기서부터 ❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗❗

**사전작업**
- base64 데이터를 실제 이미지 파일로 저장 ( 사이트에 접속해서 수동으로 다운받아야합니다. approval_2020_img/apr{source_id}/0.jpg 형식으로 저장해야합니다.)
- 2020년도 16개 이미지가 대상입니다. source_id(문서번호) 2978801, 2983943, 2985244, 3091219, 3097823, 3120072, 3187289, 3190258, 3202897, 3271155, 3288192, 3307760, 5637771, 5641483, 5649101, 5810171
- 각 source_id마다 이미지가 1개만 있습니다. approval_2020_img 폴더 안에 apr문서번호 폴더 안에 다운받은 이미지를 0.jpg로 저장해주세요

**기능**:
- `<img src="data:image/jpeg;base64,...">` 패턴 검색
- img 태그의 src를 프리픽스를 포함한 실제 저장한 파일 위치 경로로 대체 

**변환 예시**:
```html
<!-- 변환 전 -->
<img src="data:image/jpeg;base64,/9j/4AAQ...">

<!-- 변환 후 -->
<img src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_2020_img/apr2009123/0.jpg">
```

**처리 결과**: 16개 문서의 base64 이미지 변환

---

#### `cleanup_office_images.ipynb`
**목적**: doc_body에서 더 이상 접근 불가능한 office.anyfive.com 이미지 태그 제거

**기능**:
- `<img ... http://office.anyfive.com/... >` 패턴 검색
- 매칭된 이미지 태그 목록 txt 파일로 추출
- 검토 후 일괄 삭제 실행

**검색 패턴**: `<img[^>]+http://office\.anyfive\.com/[^>]+>`

**처리 결과**: 210개 문서에서 219개 이미지 태그 제거
---
### 수동으로 DB에서 수정해야하는 이미지 목록
- 수정 전 // 총 <img 태그 수1,473개/전체 문서 수23,320개/이미지가 포함된 문서 수1,042개
- 수정 후 // 총 <img 태그 수 1246개

#### 경로 변경

**2002936**
```html
<img height="13" src="http://image5.compuzone.co.kr/img/images/basket/basketcomp_litsT3.gif" width="57"/>
```
→ 경로변경
```html
<img height="13" src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_2015_img/apr2002936/0.jpg" width="57"/>
```

---

**2008214** (실제로는 이미지 없음)
```html
<img border="0" src="/PMS_SITE-U7OI43JLDSMO/approval/approval_2015_new_img/apr2008214/0.jpg">
```
→ 경로변경
```html
<img border="0" src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_2015_new_img/apr2008214/0.jpg">
```

---

**source_id 17893843** (너무 길어서 첨부할 수 없음. 이미지 태그 있는 순서대로 아래와같이 교체)
```html
<img src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_plus_img/apr17893843/0.jpg"/>
<img src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_plus_img/apr17893843/1.jpg"/>
```

---

**source_id 18056011** (너무 길어서 첨부할 수 없음. 이미지 태그 있는 순서대로 아래와같이 교체)
```html
<img src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_plus_img/apr18056011/0.jpg"/>
```

---

**source_id 3166524**
```html
<img alt=\"ISBN 안내 레이어 보기\" height=\"14\" src=\"http://static.naver.net/book/img3/btn_question.gif\" style=\"margin: -1px 0px 1px -1px; padding: 0px; border: 0px; position: relative; vertical-align: middle;\" title=\"ISBN 안내 레이어 보기\" width=\"14\"/>
```
→
```html
<img alt="ISBN 안내 레이어 보기" height="14" src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_plus_img/apr3166524/0.jpg" style="margin: -1px 0px 1px -1px; padding: 0px; border: 0px; position: relative; vertical-align: middle;" title="ISBN 안내 레이어 보기" width="14"/>
```

---

#### 삭제

**2008292**: 1개

**4건 (16403803, 16859308(2개), 18499144)** → 삭제
```html
<img alt=\"\" class=\"from\" src=\"\"/>
```

**1건 (25608213)** → 삭제
```html
<img alt=\"\" src=\"\" style=\"vertical-align: baseline; border: 0px solid rgb(0, 0, 0); width: 335.994px; height: 305.994px;\"/>
```

**1건 (25608213)** → 삭제
```html
<img alt=\"\" src=\"\" style=\"vertical-align: baseline; border: 0px solid rgb(0, 0, 0); width: 340px; height: 300px;\"/>
```

**1건 (12102242)** → 삭제
```html
<img class=\"txc-image\" id=\"tx_entry_64832_\" src=\"\" style=\"clear: none; float: none;\"/>
```

---
#### 이미지 경로 일괄 변경 (수동 수정 완료 후 실행)

> ⚠️ 위의 수동 변경/삭제 작업을 모두 완료한 후, 경로타입 확인 후 쿼리문 실행

**케이스 1: /approval/approval_ → /approval/attach/approval_**
```sql
UPDATE any_approval.documents
SET doc_body = REPLACE(doc_body,
                       '/PMS_SITE-U7OI43JLDSMO/approval/approval_',
                       '/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_')
WHERE doc_body LIKE '%/PMS_SITE-U7OI43JLDSMO/approval/approval_%';
```

**케이스 2: src="approval_ → src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_**
```sql
UPDATE any_approval.documents
SET doc_body = REPLACE(doc_body,
                       'src="approval_',
                       'src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_')
WHERE doc_body LIKE '%src="approval_%';
```



---

### 4. 데이터 검증

#### `verify_action_type.ipynb`
**목적**: activities의 actionLogType과 원본 approval_data 테이블 비교 검증

**기능**:
- documents.activities와 approval_data_XXXX 테이블 비교
- status → actionLogType 매핑 검증

**매핑 규칙**:
| approval_data.status | actionLogType |
|---------------------|---------------|
| 기안 | DRAFT |
| 승인 | APPROVAL |
| 합의 | AGREEMENT |
| 합의(대결) | AGREEMENT |
| 반려 | RETURN |

**검증 결과**:
- 매칭 성공: 80,114건
- 불일치 발견 시 자동 수정 기능 포함

---

#### `결재의견누락.ipynb`
**목적**: HTML 원본과 DB의 actionComment 비교하여 누락된 결재의견 복구

**기능**:
- HTML 파일에서 결재의견 파싱 (BeautifulSoup + lxml)
- DB의 actionComment와 비교
- 누락된 의견 자동 업데이트

**파싱 대상**:
- `<th>결재의견</th>` 이후 `<td>` 영역
- `<span class="user">` 내 결재자 정보
- 결재자별 의견 `<div>` 추출

**처리 결과**: 212건 누락 의견 복구

---

#### `comment_validation.ipynb`
**목적**: 결재의견 데이터 무결성 검증

---

### 5. 문서 내보내기

#### `export_documents_v2.ipynb`
**목적**: DB 문서를 addDocument 명령어 형식의 .cmds 파일로 내보내기

**기능**:
- 3가지 모드 지원: 전체(all), 연도별(year), 특정 source_id(source_ids)
- sourceId 형식: `doc_{source_id}_{시도횟수}` (예: `doc_15241300_06`)
- JSON 파싱 실패 시 상세 에러 로깅

**출력 형식**:
```
addDocument {"sourceId":"doc_2002144_09","docNum":"ANY-사업-0012","title":"...","activities":[...],...}
```

**설정 예시**:
```python
MODE = "year"
TARGET_YEAR = [2024, 2025]
ATTEMPT_NUMBER = 9
OUTPUT_FILE = "documents_09.cmds"
```

---

## 데이터 파일

| 파일명 | 설명 |
|--------|------|
| `in_employee.csv` | 재직자 명단 (167명) |
| `out_employee.csv` | 퇴사자 명단 (336명) |
| `attaches_backup_*.csv` | attaches 변환 전 백업 |
| `base64_images_before_*.txt` | base64 이미지 변환 전 목록 |
| `empty_path_documents_*.csv` | 빈 경로 첨부파일 문서 목록 |
| `office_anyfive_images_*.txt` | office.anyfive.com 이미지 목록 |
| `actionType_불일치목록.csv` | actionType 불일치 문서 목록 |
| `actionComment_누락목록.csv` | 결재의견 누락 문서 목록 |

---

## 실행 순서 가이드

### Phase 1: 조직도 데이터 준비
```
1. 재직자 조직도.ipynb        → in_employee 테이블 생성
2. 퇴사자명단_DB업로드.ipynb   → out_employee 테이블 생성
```

### Phase 2: 조직도 반영
```
3. DB에_조직도_다시_반영하기_수정.ipynb  → drafter/activities 정보 업데이트
4. DB_조직도_반영_검증_v2.ipynb         → 업데이트 결과 검증
```

### Phase 3: 데이터 구조 변환
```
5. update_db_referrers.ipynb    → referrers 형식 변환
6. attaches_transform.ipynb     → 첨부파일 경로 변환
7. fix_activities_order_db3.ipynb → activities 정렬 및 공백 제거
```

### Phase 4: 이미지 및 특수 처리
```
8. convert_base64_images_db.ipynb → base64 이미지 변환
9. cleanup_office_images.ipynb    → 레거시 이미지 태그 제거
10. RETURN 변환.ipynb             → 반려 상태 변환
```

### Phase 5: 검증 및 보정
```
11. verify_action_type.ipynb   → actionType 검증/수정
12. 결재의견누락.ipynb          → 누락 의견 복구
13. add_year_column.ipynb      → year 컬럼 추가
```

### Phase 6: 내보내기
```
14. export_documents_v2.ipynb  → .cmds 파일 생성
```

---

## 주의사항

### ⚠️ 필수 백업
모든 스크립트 실행 전 DB 백업을 권장합니다:
```sql
mysqldump -u root -p any_approval > backup_YYYYMMDD.sql
```

### ⚠️ Dry Run 모드
대부분의 스크립트는 실제 UPDATE 전 Dry Run(미리보기) 기능을 제공합니다:
```python
CONFIRM = False  # True로 변경 시 실제 UPDATE 실행
```

### ⚠️ CSV 경로 설정
각 스크립트 상단의 CSV 파일 경로를 실제 환경에 맞게 수정하세요:
```python
IN_EMPLOYEE_CSV = './in_employee.csv'
OUT_EMPLOYEE_CSV = './out_employee.csv'
```

### ⚠️ 배치 처리
대량 데이터 처리 시 배치 크기 조정 가능:
```python
BATCH_SIZE = 1000  # 기본값
```

### 📁etc 폴더
- etc 폴더는 cmds 변환 과정에서 사용된 일회성 스크립트와 개발 중 생성된 이전 버전 코드를 보존한 것입니다. 핵심 운영 코드에 포함되지 않으며, 향후 유사 작업 시 참조 목적으로 보존하였습니다.
-아래는 항목별로 파일의 기능을 분류한 정보입니다.

- 조직도 관리
| 파일명 | 설명 |
|--------|------|
| `DB 조직도 반영 검증.ipynb` | DB에 반영된 조직도 정보(drafter, activities)와 CSV 원본 데이터의 일치 여부 검증 |
| `DB에 조직도 다시 반영하기.ipynb` | 인사정보 CSV 기준으로 documents 테이블의 drafter/activities 정보 일괄 업데이트 |
| `DB에서 퇴사자 관련 이슈 수정.ipynb` | 퇴사자 관련 DB 데이터 이슈 수정 |

- 퇴사자 처리
| 파일명 | 설명 |
|--------|------|
| `퇴사자.ipynb` | 현재 조직도에 없는 인원을 퇴사자로 식별하여 명단 추출 (cmds 파일 내 이름과 인사정보 CSV 비교) |
| `퇴사자 직위 부서 포함.ipynb` | 퇴사자 정보에 직위, 부서 정보 포함하여 추출 |

> **참고**: 퇴사자 명단은 별도 제공되지 않아, 기존 cmds 데이터와 타겟 시스템 조직도를 대조하여 현행 조직도에 존재하지 않는 인원을 퇴사자로 분류하였습니다.

- 데이터 변환
| 파일명 | 설명 |
|--------|------|
| `convert_base64_images_v2.ipynb` | 문서 내 Base64 인코딩 이미지 태그를 파일 경로 참조로 변환 |
| `convert_referrers.ipynb` | referrers 필드 형식 변환 (`{name, empNo, deptCode}` → `{name, emailId}`) |
| `convert_referrers_safe.ipynb` | referrers 변환 (안전 버전, 공백 처리 개선) |
| `img_src_transform.ipynb` | documents 테이블의 img src 경로에 PMS 접두사 추가 |

- 데이터 정합성 검증
| 파일명 | 설명 |
|--------|------|
| `verify_action_type_with_time.ipynb` | actionType 데이터의 시간 기반 정합성 검증 |
| `리스트 차이.ipynb` | 두 데이터셋 간 차집합 비교 |
| `스타일태그 2개있는거 확인.ipynb` | HTML 문서 내 중복 style 태그 검출 |

- 데이터 수정
| 파일명 | 설명 |
|--------|------|
| `fix_activities_order(2).ipynb` | activities 배열 순서 정렬 |
| `fix_docnum(1).ipynb` | 문서번호(docNum) 형식 수정 |
| `referrers_업데이트.ipynb` | referrers 필드 업데이트 |

- 파일 처리
| 파일명 | 설명 |
|--------|------|
| `cmds maker.ipynb` | cmds 형식 파일 생성 |
| `split_yearly.ipynb` | cmds 파일을 createdAt 기준 연도별로 분리 (sourceId 오름차순 정렬) |
| `export_documents.ipynb` | 문서 데이터 내보내기 |

- 📊 데이터 파일
| 파일명 | 설명 |
|--------|------|
| `멤버목록_2025-12-24 (1).xlsx` | 사원 정보 목록 |
| `부서목록_2025-12-24.xlsx` | 부서 정보 목록 |
| `actionType_불일치목록_시간비교.csv` | actionType 검증 결과 |

---
