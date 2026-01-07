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
- 관련하여 자세한 설명은 [📁etc 폴더](#etc-폴더) 를 참조해주세요
- 새롭게 수정된 조직도는 전자결재 추가 수정 폴더의 DB에 수정 반영하기 폴더에 out_employee.csv,in_employee.csv 파일 입니다. 원하는 위치에 다운받은 후 해당 경로에 맞게 코드를 수정해주세요

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
- 퇴사자 정보는 별도로 제공되지 않았습니다. 이에 따라 cmds에서 모든 사원의 이름을 추출한 후, 타겟 사이트에서 다운받은 현행 조직도와 대조하여 현행 조직도에 존재하지 않는 인원을 퇴사자로 분류하였습니다. 이 과정에서 많은 수정이 있었고 관련 코드는 [📁etc 폴더](#etc-폴더)에 위치해있습니다.

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
- [verify_action_type.ipynb](#verify_action_typeipynb): RETURN 상태로 변환  /  RETURN 변환.ipynb: RETURN → APPROVAL 변환 및 의견에 [반려] prefix 추가
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
- empty_path_export.ipynb로 path:""를 조회한 후 csv로 저장하여 [any_crawling](https://github.com/000Lee/any_crawling.git)의 AnyFivePlusCrawler_attaches.java를 통해 누락된 첨부파일을 크롤링 합니다.
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
- 수정 전 // 총 <img 태그 수1,473개/전체 문서 수23,320개/이미지가 포함된 문서 수1,042개
- 수정 후 // 총 <img 태그 수 1246개
  
#### `convert_base64_images_db.ipynb`
**목적**
- doc_body 내 base64 인코딩 이미지를 파일 경로로 변환

**사전작업**
- base64 데이터를 실제 이미지 파일로 저장 ( 사이트에 접속해서 수동으로 다운받아야합니다. approval_2020_img/apr{source_id}/0.jpg 형식으로 저장해야합니다.)
- 2020년도 16개 이미지가 대상입니다. source_id(문서번호) 2978801, 2983943, 2985244, 3091219, 3097823, 3120072, 3187289, 3190258, 3202897, 3271155, 3288192, 3307760, 5637771, 5641483, 5649101, 5810171
- 각 source_id마다 이미지가 1개만 있습니다. approval_2020_img 폴더 안에 apr문서번호 폴더 안에 다운받은 이미지를 0.jpg로 저장해주세요
```
-- 1. DB에 검색
SELECT * FROM documents WHERE source_id = '2978801';
-- 2. 문서 파악 완료 후, 소스페이지 URL (https://auth.onnet21.com/?re=anyfive.onnet21.com/sso/login)에 접속하여 연도 설정 및 문서번호(doc_num) 기준 검색 진행. 이후 제목, 기안자, 날짜 등 주요 항목이 DB와 일치하는지 대조 확인 한 후 이미지 다운로드
-- 3. 16개의 문서를 같은방식으로 다운로드
-- 4. 다운받은 경로를  approval_2020_img/apr{source_id}/0.jpg 으로 변경
```

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
- 사이트에 해당 source_id 문서 상세페이지로 접속해서 이미지를 수동으로 다운받아야합니다
```
-- 1. DB에 검색
SELECT * FROM documents WHERE source_id = '2002936';
-- 2. 문서 파악 완료 후, 소스페이지 URL (https://auth.onnet21.com/?re=anyfive.onnet21.com/sso/login)에 접속하여 연도 설정 및 문서번호(doc_num) 기준 검색 진행. 이후 제목, 기안자, 날짜 등 주요 항목이 DB와 일치하는지 대조 확인 한 후 이미지 다운로드
-- 3. 다른 문서도 같은방식으로 다운로드
-- 4. 다운받은 경로를 안내와같이 변경
-- 경로 변경 후 태그가 <img height="13" src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_2015_img/apr2002936/0.jpg" width="57"/>이라면 approval_2015_img 폴더 안에 apr2002936폴더 안에 0.jpg로 해당 이미지가 저장되어있어야합니다.
```
#### 경로 변경

**source_id 2002936**
```html
<img height="13" src="http://image5.compuzone.co.kr/img/images/basket/basketcomp_litsT3.gif" width="57"/>
```
→ 경로변경
```html
<img height="13" src="/PMS_SITE-U7OI43JLDSMO/approval/attach/approval_2015_img/apr2002936/0.jpg" width="57"/>
```

---

**source_id 2008214** (확인 결과, 해당 이미지가 실제로 존재하지 않는 것으로 파악되었습니다. 해당 이미지 태그 삭제를 진행해도 무방할 것으로 판단됩니다.)
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
```
-- 1. DB에 검색
SELECT * FROM documents WHERE source_id = '2008292';
-- 2. DB에서 해당 문서 doc_body의 이미지태그 삭제 후 저장
```
**source_id 2008292**: 1개

**4건 (source_id 16403803, 16859308(2개), 18499144)** → 삭제
```html
<img alt=\"\" class=\"from\" src=\"\"/>
```

**1건 ( source_id 25608213)** → 삭제
```html
<img alt=\"\" src=\"\" style=\"vertical-align: baseline; border: 0px solid rgb(0, 0, 0); width: 335.994px; height: 305.994px;\"/>
```

**1건 (source_id 25608213)** → 삭제
```html
<img alt=\"\" src=\"\" style=\"vertical-align: baseline; border: 0px solid rgb(0, 0, 0); width: 340px; height: 300px;\"/>
```

**1건 (source_id 12102242)** → 삭제
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
### 3-3. 스타일태그 처리
```
DB에서 쿼리문 실행하여 처리합니다.

-- 스타일태그 (여러개 -> 한개) 수정
-- 1. 모든 style 태그 제거
UPDATE documents
SET doc_body = REGEXP_REPLACE(doc_body, '<style[^>]*>[^<]*</style>', '')
WHERE doc_body LIKE '%<style%';

-- 2. 올바른 CSS 추가 (문서 맨 끝에)
UPDATE documents
SET doc_body = CONCAT(doc_body, '<style>.content{width:80% !important;}.content table{width:100% !important;}@CHARSET \"UTF-8\";span,dl,dt,dd,ul,ol,li,h1,h2,h3,h4,h5,h6,blockquote,address,pre,cite,form,fieldset,caption,input,textarea,select{margin:0;padding:0;}p{font-size:12px;font-family:MalgunGothic;margin-top:0px;margin-bottom:0px;}h1,h2,h3,h4,h5,h6{font-size:100%;}fieldset,img,abbr,acronym{border:0 none}ul,ol{list-style:none !important;padding-left:0 !important;margin-left:0 !important;}address,caption,em,cite{font-weight:normal;font-style:normal}ins{text-decoration:none}del{text-decoration:line-through}hr{display:none}a{text-decoration:none;cursor:pointer color:#787878;}a:link{text-decoration:none;color:#787878;}a:visited{text-decoration:none;color:#787878;}a:active,a:focus{text-decoration:none;color:#787878;}a:hover{text-decoration:none;color:#787878}select{vertical-align:middle}.clear{display:block;float:none;clear:both;height:0;width:100%;font-size:0 !important;line-height:0 !important;overflow:hidden;margin:0 !important;padding:0 !important;}#p_wrapper{width:100%;}#header{width:100%;height:45px;background:url(../../images/apr/title_bg.gif) 0 bottom repeat-x;background-color:#FFFFFF;position:fixed;top:0px;z-index:10;}#header h1{margin:12px 0 0 10px;float:left;}#header .menu{margin:14px 0 0 20px;float:left;}#header .menu li{position:relative;float:left;font-family:\"\";font-size:11px;color:#5d5d5d;}#header .menu li label{position:absolute;top:0px;left:15px;top:2px\\0IE8+9;}#header .btn{font-family:\"\";font-size:12px;float:right;margin:8px 13px 0 0;}#header .btn a.print{display:inline-block;padding:4px;background:#fffff;border:1px solid #dadada;width:40px;height:14px;text-align:center;}#header .btn a.print:hover,#header .btn a.close:hover{display:inline-block;padding:4px;background:#949297;border:1px solid #7a7a7a;width:40px;height:14px;text-align:center;color:#FFF;}#header .btn a.close{display:inline-block;padding:4px;margin-left:4px;background:#fffff;border:1px solid #dadada;width:40px;height:14px;text-align:center;}.content{border:1px solid #ffffff;margin:15px 10px 0 10px;padding:10px 5px 6px 5px;min-width:850px;}#middle .content h1{width:100%;font-size:25px;display:inline-block;font-family:\"\";text-align:center;}#middle .content h2{width:100%;font-size:25px;display:inline-block;font-family:\"\";text-align:center;}#middle .content .company{float:left;font-family:\"\";font-size:14px;font-weight:bold;margin:10px 0 0 10px;}#middle .content .team{float:right;font-family:\"\";font-size:14px;font-weight:bold;margin:10px 10px 0 0;}.table01_box{border-top:1px #000000 solid;margin:5px 0 0 0;min-width:850px;}.table01{width:100%;border-top:1px #000000 solid;table-layout:fixed;border-collapse:collapse;font-family:\"\";font-size:12px;}.table01 td div.bg01{background:#FFF;height:25px;display:table-cell;vertical-align:middle;width:300px;}.table01 td div.bg02{background:#FFF;height:77px;position:relative;z-index:2;}.table01 td div.bg02 .sign{position:absolute;z-index:1;top:0px;left:50%;margin-left:-38px;height:77px;opacity:0.3;filter:alpha(opacity=30);}.table01 td div.bg02 .text{position:absolute;z-index:2;top:0px;height:77px;left:50%;width:74px;margin-left:-37px;display:table-cell;vertical-align:middle;}.table01 td div.bg02 .text ul{position:relative;height:77px;width:74px;display:table-cell;vertical-align:middle;}.table01 caption{display:none;}.table01 th{background:#e6e6e6;border-bottom:1px #000000 solid;border-left:1px #000000 solid;border-top:1px #000000 solid;text-align:center;height:25px;min-width:74px;}.table01 td{border-bottom:1px #000000 solid;border-top:1px #000000 solid;border-left:1px #000000 solid;text-align:center;background:#e6e6e6;line-height:16px;}.boeder_left{border-right:1px #000000 solid;}.boeder_top{border-top:1px #000000 solid;}#middle .boeder_bnone{border-bottom:none;}.table01 span{display:block;}.table01 span.red{color:#F00;font-weight:bold;font-size:12px;}.table02{table-layout:fixed;border-collapse:collapse;font-family:\"\";font-size:12px;width:100%;}.table02 td div.bg01{background:#FFF;height:25px;display:table-cell;vertical-align:middle;width:300px;}.table02 td div.bg02{background:#FFF;display:table-cell;vertical-align:middle;width:300px;overflow:hidden;}.table02 th div.bg02{background:#FFF;height:77px;display:table-cell;vertical-align:middle;width:300px;}.table02 caption{display:none;}.table02 th{background:#e6e6e6;border-bottom:1px #000000 solid;border-left:1px #000000 solid;text-align:center;height:25px;}.table02 td{border-top:1px #000000 solid;border-bottom:1px #000000 solid;border-left:1px #000000 solid;text-align:center;background:#e6e6e6;line-height:16px;}.table02 span{display:block;}.table02 span.red{color:#F00;font-weight:bold;font-size:12px;}.table02_box{border-top:1px #000000 solid;margin:5px 0 0 0;min-width:850px;}.table03{width:100%;border-top:1px #000000 solid;table-layout:fixed;border-collapse:collapse;font-family:\"\";font-size:12px;}.table03 caption{display:none;}.table03 th{background:#e6e6e6;border-bottom:1px #000000 solid;border-left:1px #000000 solid;border-top:1px #000000 solid;text-align:center;height:25px;}.table03 td{border-bottom:1px #000000 solid;border-top:1px #000000 solid;border-left:1px #000000 solid;border-right:1px #000000 solid;line-height:14px;padding-left:10px;}.table03_box{border-top:1px #000000 solid;margin:5px 0 0 5px;}.table04{width:100%;border-top:1px #000000 solid;table-layout:fixed;border-collapse:collapse;font-family:\"\";font-size:12px;}.table04 .user{color:#003399;}.table04 .user img{position:relative;top:2px;margin-right:5px;}.table04 .file img,.table04 .doc img{position:relative;top:1px;margin-right:5px;}.table04_box{border-top:1px #000000 solid;margin:5px 0 0 0;width:100%;min-width:850px;}.table04 caption{display:none;}.table04 th{background:#e6e6e6;border-bottom:1px #000000 solid;border-left:1px #000000 solid;border-top:1px #000000 solid;text-align:center;height:25px;}.table04 td{border-bottom:1px #000000 solid;border-top:1px #000000 solid;border-left:1px #000000 solid;border-right:1px #000000 solid;line-height:16px;padding:10px;}.table05_box{border-top:1px #000000 solid;margin:5px 0 0 0;width:100%;min-width:850px;}.content_box{border:1px #000000 solid;margin:5px 0 0 0;word-break:break-all;overflow:hidden;min-width:819px;}td.bgw{background:#FFF;}@media print{#header{display:none;}}.approvalOpinionCode0{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#f38405;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode1{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#69F;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode2{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#9FC93C;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode3{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#FFBB00;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode4{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#F66;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode5{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#9FC93C;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode6{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#FFBB00;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode7{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#A566FF;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode8{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#F66;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.approvalOpinionCode9{padding:0px 5px 2px 5px;border-radius:3px;color:#fff;background-color:#aeb2bd;font-family:Malgun Gothic,,dotum,,Gulim,Helvetica,AppleGothic,Tahoma,Verdana;}.F_12_black_b{color:#000;font-size:12px;font-weight:bold;}.F_11_gray{color:#999;font-size:12px;font-weight:100;}.fileIcon{display:inline-block;width:16px;height:16px;background-repeat:no-repeat;text-indent:-9999px;}.file_folder{background:url(../../resource/image/common/diskfile_ico.png) 0 -10px;}.file_folder_share{background:url(../resource/image/common/diskfile_ico.png) -258px -10px;}.file_image{background:url(../../resource/image/common/diskfile_ico.png) -21px -10px;}.file_zip{background:url(../../resource/image/common/diskfile_ico.png) -42px -10px;}.file_hwp{background:url(../../resource/image/common/diskfile_ico.png) -63px -10px;}.file_xls{background:url(../../resource/image/common/diskfile_ico.png) -83px -10px;}.file_txt{background:url(../../resource/image/common/diskfile_ico.png) -104px -10px;}.file_exe{background:url(../../resource/image/common/diskfile_ico.png) -125px -10px;}.file_pdf{background:url(../../resource/image/common/diskfile_ico.png) -147px -10px;}.file_html{background:url(../../resource/image/common/diskfile_ico.png) -170px -10px;}.file_ppt{background:url(../../resource/image/common/diskfile_ico.png) -193px -10px;}.file_ect{background:url(../../resource/image/common/diskfile_ico.png) -215px -10px;}.file_doc{background:url(../../resource/image/common/diskfile_ico.png) -237px -10px;}.o-i-min-fileList{background:url(../../resource/image/common/ico_notice.png) no-repeat;width:13px;height:17px;}.disInline{display:inline-block}.dispNone{display:none;}.verti_Top{vertical-align:top;}.o-i-view-minu{display:inline-block;background:url(../../images/mai/ico_btn_minu.png) no-repeat;width:14px;height:14px;text-indent:-9999px;cursor:pointer;margin-top:2px}.o-i-view-plus{display:inline-block;background:url(../../images/mai/ico_btn_plus.png) no-repeat;width:14px;height:14px;text-indent:-9999px;cursor:pointer;margin-top:2px}.abc-box{position:relative;padding:10px;border:1px solid #c5c5c5;-moz-border-radius:4px;-webkit-border-radius:4px;border-radius:4px;background-color:#fff;box-shadow:0 1px 2px rgba(0,0,0,0.6)}</style>')
WHERE doc_body IS NOT NULL;

-- 확인
SELECT
    (LENGTH(doc_body) - LENGTH(REPLACE(doc_body, '<style', ''))) / LENGTH('<style') AS style_tag_count,
    COUNT(*) as cnt
FROM documents
WHERE doc_body IS NOT NULL
GROUP BY style_tag_count;
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
5. update_db_referrers.ipynb      → referrers 형식 변환
6. fix_activities_order_db3.ipynb → activities 정렬 및 공백 제거
```

### Phase 4: 첨부파일 처리
```
7. empty_path_export.ipynb      → 누락 첨부파일 목록 추출 (CSV 저장)
   ↓ ⚠️ Java 크롤러(AnyFivePlusCrawler_attaches.java)로 누락 파일 다운로드
8. attaches_transform.ipynb     → 첨부파일 경로 변환
```

### Phase 5: 이미지 처리
```
   ⚠️ [수동] base64 이미지 16개 사전 다운로드 (상세: 3-2. 이미지 처리 참조)
9. convert_base64_images_db.ipynb  → base64 이미지 태그 경로 변환
10. cleanup_office_images.ipynb    → 불필요한 이미지 태그 제거
   ⚠️ [수동] 개별 이미지 태그 수정/삭제 (상세: 수동으로 DB에서 수정해야하는 이미지 목록 참조)
   ⚠️ [SQL] 이미지 경로 일괄 변경 쿼리 실행
```
### Phase 6: 스타일태그 처리
```
⚠️ [SQL] 스타일태그 일괄 변경 쿼리 실행
```

### Phase 7: 검증 및 보정
```
11. verify_action_type.ipynb   → actionType 검증/수정 (RETURN 상태로 변환)
12. RETURN 변환.ipynb          → RETURN → APPROVAL 변환 및 [반려] prefix 추가
    ※ 반드시 11번 실행 후 실행
13. 결재의견누락.ipynb          → 누락 의견 복구
14. add_year_column.ipynb      → year 컬럼 추가
```

### Phase 8: 내보내기
```
15. export_documents_v2.ipynb  → .cmds 파일 생성
```
---

### 댓글
- 댓글관련 코드는 [any_htmlver](https://github.com/000Lee/any_htmlver.git)과 [any_crawling](https://github.com/000Lee/any_crawling.git)에 있습니다.
- any_htmlver의 댓글크롤링폴더에서 "해당 기간 내에 있는 문서 ID만 txt파일로 가져오는 파이썬코드.ipynb"로 대상 문서 ID를 추출하고
- any_crawling의 AnyFiveCommentCrawler.java로 크롤링을 실시하고
- any_htmlver의 갯수세기.ipynb로 크롤링이 완료된 갯수를 확인하고 
- any_htmlver의 comments_to_cmds.ipynb로 DB의 comments 테이블을 addDocumentComment 형식의 cmds 파일로 변환합니다
- any_htmlver의 comment_validation.ipynb로 '크롤링한 결재댓글'과 '소스사이트에서 다운받은 html파일'을 서로 비교하여 정합성을 검사합니다.

### 📁etc 폴더
- etc 폴더는 cmds 변환 과정에서 사용된 일회성 스크립트와 개발 중 생성된 이전 버전 코드를 보존한 것입니다. 핵심 운영 코드에 포함되지 않으며, 향후 유사 작업 시 참조 목적으로 보존하였습니다.
- 아래는 항목별로 파일의 기능을 분류한 정보입니다.

#### 조직도 관리
| 파일명 | 설명 |
|--------|------|
| `DB 조직도 반영 검증.ipynb` | DB에 반영된 조직도 정보(drafter, activities)와 CSV 원본 데이터의 일치 여부 검증 |
| `DB에 조직도 다시 반영하기.ipynb` | 인사정보 CSV 기준으로 documents 테이블의 drafter/activities 정보 일괄 업데이트 |
| `DB에서 퇴사자 관련 이슈 수정.ipynb` | 퇴사자 관련 DB 데이터 이슈 수정 |

#### 퇴사자 처리
| 파일명 | 설명 |
|--------|------|
| `퇴사자.ipynb` | 현재 조직도에 없는 인원을 퇴사자로 식별하여 명단 추출 (cmds 파일 내 이름과 인사정보 CSV 비교) |
| `퇴사자 직위 부서 포함.ipynb` | 퇴사자 정보에 직위, 부서 정보 포함하여 추출 |

> **참고**: 퇴사자 명단은 별도 제공되지 않아, 기존 cmds 데이터와 타겟 시스템 조직도를 대조하여 현행 조직도에 존재하지 않는 인원을 퇴사자로 분류하였습니다.

#### 데이터 변환
| 파일명 | 설명 |
|--------|------|
| `convert_base64_images_v2.ipynb` | 문서 내 Base64 인코딩 이미지 태그를 파일 경로 참조로 변환 |
| `convert_referrers.ipynb` | referrers 필드 형식 변환 (`{name, empNo, deptCode}` → `{name, emailId}`) |
| `convert_referrers_safe.ipynb` | referrers 변환 (안전 버전, 공백 처리 개선) |
| `img_src_transform.ipynb` | documents 테이블의 img src 경로에 PMS 접두사 추가 |

#### 데이터 정합성 검증
| 파일명 | 설명 |
|--------|------|
| `verify_action_type_with_time.ipynb` | actionType 데이터의 시간 기반 정합성 검증 |
| `리스트 차이.ipynb` | 두 데이터셋 간 차집합 비교 |
| `스타일태그 2개있는거 확인.ipynb` | HTML 문서 내 중복 style 태그 검출 |

#### 데이터 수정
| 파일명 | 설명 |
|--------|------|
| `fix_activities_order(2).ipynb` | activities 배열 순서 정렬 |
| `fix_docnum(1).ipynb` | 문서번호(docNum) 형식 수정 |
| `referrers_업데이트.ipynb` | referrers 필드 업데이트 |

#### 파일 처리
| 파일명 | 설명 |
|--------|------|
| `cmds maker.ipynb` | cmds 형식 파일 생성 |
| `split_yearly.ipynb` | cmds 파일을 createdAt 기준 연도별로 분리 (sourceId 오름차순 정렬) |
| `export_documents.ipynb` | 문서 데이터 내보내기 |

#### 📊 데이터 파일
| 파일명 | 설명 |
|--------|------|
| `멤버목록_2025-12-24 (1).xlsx` | 사원 정보 목록 |
| `부서목록_2025-12-24.xlsx` | 부서 정보 목록 |
| `actionType_불일치목록_시간비교.csv` | actionType 검증 결과 |

---
