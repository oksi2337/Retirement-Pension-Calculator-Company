# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

기업 인사·급여 담당자가 직원별 **퇴직연금 DC형 납입액**을 산출하는 웹 계산기. 단일 HTML 파일이며 빌드/번들 단계가 없다 — `index.html`을 브라우저로 직접 열면 동작한다.

## 아키텍처

**단일 파일 SPA** (`index.html`, 약 1300줄). 외부 의존성은 모두 CDN:
- Tailwind CSS (`cdn.tailwindcss.com`) — 스타일링
- `xlsx-js-style@1.2.0` — SheetJS의 무료 fork. 셀 스타일링(폰트/색/테두리/수식)을 지원해서 일반 `xlsx`가 아니라 이걸 쓴다. 라이브러리를 바꾸지 말 것.
- Pretendard (Google Fonts) — 화면 폰트

**상태 모델**: 전역 `persons` 배열 하나에 직원 객체(`createPerson()` 형태)들을 보관. 모든 mutation은 인덱스 기반 (`syncField(idx, field, value)`, `updateExtra(idx, ei, key, value)` 등). React/Vue 없이 `innerHTML` 재렌더 + 인라인 `oninput/onchange` 핸들러로 동작 — 새 UI를 추가할 때 이 패턴을 따른다.

**계산 흐름** (`calculatePerson(idx)` 기준):
```
임금총액 = 1년치 급여지급액
          - 첫 달 보정 (firstMode='prorated' → 월급여 빼고 일할안분액 더함 / 'deduction' → 차감액만 뺌)
          - 마지막 달 보정 (동일 분기)
          + 비일회성 과세별도기지급액 합계 (extraPays 중 included=true)

예상 납입액 = 임금총액 ÷ 12
최종 납입액 = round(예상 납입액 / 10) × 10  − 기지급 납입액(prevPaid)
            └── 10원 단위 반올림 (1원자리 반올림). 예) 3,049,198 → 3,049,200
```
입력값이 바뀌면 `calculatePerson` → `renderSummary` 가 즉시 호출돼 sticky 상단 요약 패널도 갱신된다.

**엑셀 추출** (`exportToExcel()`, line ~606): `skills/applications/retirement-pension-excel-skill.md`의 사양을 **그대로 JS로 구현**한 것. 두 파일은 한 쌍이다:
- 스킬 정의(`.md`)는 백엔드(Python/openpyxl) 포트의 사양서
- `exportToExcel()`은 그 사양의 프론트엔드 구현체

**엑셀 출력 규칙은 어디서 수정하든 둘 다 맞춰야 한다**:
- Sheet 구성: `요약` + `상세 내역` 2장
- 합계는 **`=SUM()` 수식**으로 (하드코딩 ❌)
- 금액 포맷: `#,##0"원"`, 0은 `-`로 표시
- 헤더 배경 `#1F4E78` 흰 글자 / 합계 행 `#FFF2CC` / 블록 헤더 `#D9E1F2` / 최종 납입액 `#C00000` 빨강 굵게
- 폰트: 맑은 고딕 11pt (헤더 12pt 굵게)
- 파일명: `퇴직연금_DC_납입액_YYYYMMDD.xlsx`

## 도메인 용어 (변경 시 전역 통일 필수)

과거 커밋에서 용어가 여러 번 통일됐다 (`calculator.md` 참고). 새 텍스트 추가 시:
- "부담금"이 아니라 **"납입액"**
- "퇴직급여"가 아니라 **"퇴직연금 납입액"**
- "교번"이 아니라 **"사번"**
- "인센티브"가 아니라 **"상여금"**
- 과세별도기지급액은 **비일회성 포함** 방식 (일회성 제외 ❌ — 과거에 뒤집힌 로직임)
- 퇴직일 / 기산일자는 토글로 라벨만 바꾸는 같은 필드

## 실행 / 개발

빌드도 테스트 러너도 없다. 변경 검증은:
```powershell
start index.html        # 기본 브라우저로 열기
```
UI 동작을 직접 눌러서 확인한다. 직원 추가/삭제, 일할↔차감 토글, 과세별도 항목 체크박스, 엑셀 추출 — 이 네 가지가 핵심 회귀 포인트.

## 파일

- `index.html` — 계산기 전체 (편집 대상)
- `calculator.md` — 커밋별 변경 이력. 새 기능 추가 후 여기에 한 줄 정리.
- `skills/applications/retirement-pension-excel-skill.md` — 엑셀 출력 사양서. `exportToExcel()`을 고칠 때 같이 본다.
- `.claude/settings.json` — git 명령과 Read/Edit/Write 권한이 사전 허용돼 있어 권한 프롬프트 없이 진행 가능.
