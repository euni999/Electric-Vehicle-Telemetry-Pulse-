# ⚡ EV-Pulse — Electric Vehicle Telemetry Pulse

전기차 배터리 텔레메트리를 실시간 수집·분석해 이상 전조를 조기 탐지하는 Azure 기반 모니터링 시스템. 자체 지표 BSI(Battery Stress Index) 로 차량 상태를 정량화한다.

- 5인 팀 프로젝트(2주) | 원본 저장소: ev-pulse/Electric-Vehicle-Telemetry-Pulse- 이 포크는 제가 담당한 파트(Text-to-SQL 챗봇 · 실시간 알림)를 중심으로 정리한 문서입니다.
---

## 프로젝트 목표

1. 배터리 열화 메커니즘을 반영한 EV 상태 실시간 모니터링 시스템 구축
2. 자체 기준(BSI)을 정의하고, 그 기준으로 항목별 이상 여부를 판단
3. 장기적으로 차량 생애주기 데이터를 축적해 추후 차세대 배터리 개선에 활용

---
## 내 담당 범위

영역	스택	산출물
Text-to-SQL 챗봇	Azure Functions(Python), Azure OpenAI gpt-4o-mini, Azure SQL	chatbot/
실시간 알림	Logic Apps, Azure SQL 폴링, Slack	Logic Apps 워크플로
DB 스키마 설계	Azure SQL	텔레메트리·알림 테이블

팀 공통 파트(시뮬레이터, Stream Analytics, Azure ML 모델링, 대시보드, Bicep IaC)는 아래 전체 아키텍처에 요약만 두었습니다.

---
1. Text-to-SQL 챗봇

Slack에서 자연어로 질문하면 SQL로 변환해 evpulse DB를 조회하고 답변한다.

사용자 질문 → 스키마 주입 프롬프트 → SQL 생성 → 실행 → 자연어 요약 → Slack 응답


구성

__init__.py	: Slack 이벤트 라우팅, 즉시 200 응답 후 비동기 처리

gpt_client.py :	스키마 주입 프롬프트 구성, 자연어 → SQL → 답변 생성

sql_query.py :SQL Server 연결 및 쿼리 실행



문제 1 — Slack 3초 타임아웃

Slack Events API는 3초 내 200 응답이 없으면 재시도한다. LLM 호출 + SQL 실행은 3초를 넘기므로 같은 질문이 중복 처리되는 문제가 발생했다.

해결: 이벤트 수신 즉시 200을 반환하고, 실제 처리는 비동기로 분리한 뒤 결과를 chat.postMessage로 후속 전송하는 구조로 변경.

결과: 중복 응답 제거, 사용자 체감 지연은 "처리 중" 메시지로 흡수.


  
문제 2 — 존재하지 않는 컬럼을 지어내는 환각

초기 프롬프트에 스키마를 넣지 않으니 모델이 battery_health 같은 없는 컬럼명을 만들어내 쿼리가 실패했다.

해결: 요청 시점에 대상 테이블의 컬럼명·타입을 조회해 프롬프트에 동적 주입. 하드코딩 대신 조회 방식으로 두어 스키마 변경 시 프롬프트 수정이 불필요하도록 함.

결과: 컬럼명 환각으로 인한 쿼리 실패가 사실상 사라짐.

한계: SELECT 외 구문 차단은 프롬프트 수준에서만 처리했고, 쿼리 파싱 기반 검증은 미구현.

---

2. 실시간 알림 (Logic Apps)

Azure ML이 기록한 [dbo].[ModelAlertTest] 테이블을 3분 간격으로 폴링해 이상/위험 상태 감지 시 Slack으로 알림을 전송한다. 알림에는 차량 VIN, 지역, BSI 수치, 이상 피처(위험 원인), 감지 시각이 포함되며 경고/위험으로 자동 분류된다.

디버깅하며 잡은 문제들:

UTC → KST: Azure SQL이 UTC로 기록해 알림 시각이 9시간 어긋남 → 워크플로 내 변환 처리

한글 깨짐: Slack 페이로드 인코딩 이슈 → 메시지 구성 방식 수정

null 처리: 이상 피처가 비어 있는 레코드에서 워크플로가 실패 → 조건 분기로 방어

---

## 아키텍처

![아키텍처](docs/architecture.png)

BSI: 위 수식으로 산출. 가중치는 NASA Battery Dataset 분석 기반, μ/σ 파라미터는 BMW i3 Dataset 기반. 실제 BSI 값은 Azure ML(LightGBM)이 추론해 SQL에 기록한다.
모델 목표: Danger Recall 0.80 / Macro F1 0.75 — 정확도보다 위험 차량 미탐지 최소화 우선.
Simulator: 전처리된 BMW CSV(약 200,000행·실차 70대)를 IoT Hub로 재생. 파생변수만 계산하고 BSI 산출은 Azure ML이 담당.
IaC: Bicep으로 핵심 리소스(IoT Hub·SQL·Stream Analytics·Storage·Logic Apps) 정의.

---


