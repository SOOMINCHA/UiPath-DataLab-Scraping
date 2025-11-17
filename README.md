# 한국관광데이터랩 웹 데이터 스크래핑 및 이메일 자동 발송 시스템 (UiPath REFramework)

## 1. 프로젝트 개요
한국관광데이터랩 웹사이트의 데이터를 자동으로 스크래핑하고  
정제된 데이터를 기반으로 Naver SMTP(SSL)로 이메일을 전송하는 개인 프로젝트입니다.  
REFramework 구조를 직접 적용하여 웹 자동화부터 메일 발송까지  
End-to-End 자동화를 구현했습니다.

---

## 2. 개발 환경 및 기술 스택
- UiPath Studio
- REFramework
- UIAutomation Activities
- DataTable / Excel Activities
- Naver SMTP (smtp.naver.com / SSL 465)
- Config.xlsx 기반 변수 관리
- JSON, Dynamic Selectors, Try-Catch, Retry 정책

---

## 3. AS-IS / TO-BE

### AS-IS
- 웹사이트에서 데이터를 수동으로 조회·정리해야 함  
- 선택자 불안정으로 스크래핑 오류 발생  
- 정리된 데이터를 매일 수동으로 이메일로 공유해야 함  

### TO-BE
- UiPath를 통한 자동 접속 및 데이터 스크래핑  
- 동적 Selector 및 Retry 로직 적용으로 안정성 확보  
- 정제된 데이터가 Naver SMTP로 자동 발송되는 완전 자동화 구조 구축  

---

## 4. 주요 기능

### 4.1 웹 스크래핑 자동화
- 한국관광데이터랩 페이지 자동 접속
- 지역/구 선택 자동화
- Extract Data Table 활용
- DataTable 기반 정제 및 오류 처리

### 4.2 REFramework 기반 프로세스 설계
- Init → Get Transaction Data → Process → End 단계 구성
- Config.xlsx로 URL, 지역, 파일 경로, 엑셀 시트, SMTP 계정, Retry 설정 관리, 메일 송신 여부
- 비즈니스/시스템 예외 처리 및 재시도 로직 적용
- 로그/상태 기반 흐름 설계

### 4.3 Naver SMTP 자동 이메일 발송
- smtp.naver.com / SSL 465 기반 메일 전송
- 스크래핑 데이터 기반 이메일 템플릿 생성
- SMTP 인증 변수화 및 보안 설정 분리
- 메일 발송 실패 시 예외 처리 및 재시도 적용

---

## 5. 트러블슈팅

### 5.1 REFramework 구조 이해 부족으로 인한 반복 실행 문제

**문제 상황**  
Config에 입력한 지역 리스트는 한 번만 처리되어야 하지만,  
프로세스가 Init → Process → End로 정상 종료되지 않고  
같은 트랜잭션이 2~3번 반복 실행되는 문제가 발생함.

**원인 분석**  
- REFramework의 Queue/Transaction 구조를 완전히 이해하지 못한 상태에서 개발을 진행함  
- QueueItem을 사용하지 않는데도 GetTransactionData가 반복 호출되는 구조였음  
- 반복 종료 조건이 맞지 않아 프로세스가 루프처럼 계속 돌았음

**해결 방법**  
- QueueItem 기능을 완전히 제거하고, 단일 데이터 처리 방식으로 프로세스 재설계  
- GetTransactionData에서 다음 트랜잭션을 반환하지 않도록 로직 수정  
- 조건 분기(If)와 Boolean flag 기반으로 트랜잭션 종료를 명확하게 설정

**배운 점**  
REFramework 사용 시 Queue 기반인지, 단일 프로세스 기반인지 먼저 설계해야 하며  
각 State 간 데이터 흐름과 종료 조건을 명확하게 정의해야 함을 깨달음.

### 5.2 Selector 불안정으로 지역 선택 실패 발생

**문제 상황**  
지역을 선택한 뒤 ‘조회’ 버튼을 누르면 결과 데이터가 여러 페이지로 나뉘는데,  
스크롤이 아래로 이동하면서 지역 선택 드롭다운이 화면에서 사라져  
다음 실행 시 지역 선택이 실패하거나 잘못된 요소를 클릭하는 문제가 반복됨.

**원인 분석**  
- 초기에 이미지 기반(Computer Vision)으로 지역 선택 요소를 설정함  
- 스크롤 이동 시 화면 위치가 바뀌어 CV 기반 요소가 인식되지 않음  
- 화면 상단으로 다시 이동하는 로직이 없어서 셀렉터 인식 실패 발생

**해결 방법**  
- 이미지 기반 요소 제거 후 안정적인 UI Selector로 재구성  
- Strict Selector와 Fuzzy Selector를 조합하여 안정성 테스트  
  - Strict: 정확하지만 스크롤 위치에 따라 인식 실패  
  - Fuzzy: 유연하지만 유사 요소 존재 시 오인식 가능 

**배운 점**  
Cv/Image 기반 자동화는 화면 이동에 취약하며,  
UI Selector 기반으로 요소를 고정시키고 화면 위치를 제어하는 것이 중요함.

### 5.3 페이징 처리(Next Page 이동) 불안정 문제

**문제 상황**  
데이터가 여러 페이지로 나뉘어 있어 ‘다음 페이지’로 이동해야 했으나,  
UI 요소 기반 페이징이 불안정하여 페이지 이동이 되지 않거나  
동일 페이지를 반복 스크래핑하는 문제가 발생함.

**원인 분석**  
- 페이지 번호가 Page 1, Page 2, Page 3처럼 변동되므로  
  고정된 Strict Selector로는 다음 페이지를 안정적으로 인식하지 못함  
- Fuzzy Selector를 적용해도 스크롤/화면 위치 변화 때문에  
  유사한 요소를 잘못 인식하거나 클릭 실패가 발생함  
- 즉, UI 요소 자체가 아니라 **페이지 번호 변화 + 스크롤 영향**이 핵심 문제였음

**해결 방법**  
- 페이지 번호를 직접 읽지 않고, `currentPage`, `nextPage` 변수를 이용해  
  **동적 Selector(Dynamic Selector)** 를 사용하도록 변경  
- Strict / Fuzzy Selector를 모두 테스트한 후,  
  최종적으로 *“Selector 튜닝 → X”, “논리적 페이징 → O”* 방식으로 해결  
- nextPage = currentPage + 1 방식으로 로직 기반 페이징 설계  
- "다음 페이지" 로직이 성공하면 currentPage를 nextPage로 업데이트하고 반복  
- 종료 조건을 명확히 설정하여 마지막 페이지에서 반복 중단

**배운 점**  
UI 기반 페이징은 화면 위치, 스크롤, 요소 변화에 매우 취약하며  
페이지 번호를 변수로 제어하는 **동적 Selector + 논리적 페이징 구조**가  
Web 스크래핑 자동화에서 훨씬 안정적이라는 것을 배움.

---

## 6. 느낀 점
REFramework 구조가 처음에는 낯설어 시간이 걸렸지만  
Init–Process–End 전체 흐름을 직접 분석하며 안정적인 자동화 시스템을 구현할 수 있었습니다.  
웹 스크래핑과 SMTP 이메일 발송을 하나의 프로세스로 자동화하면서  
예외 처리, Selector 안정화, 변수 관리 등 실무 핵심 역량을 습득했습니다.
