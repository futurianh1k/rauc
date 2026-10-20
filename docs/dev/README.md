# 개발 문서 인덱스

1인 개발 프로젝트의 규칙, 역할별 지침, 산출물 양식을 모은 곳이다.

- 프로젝트: `TODO(확인 필요)` 한 줄 설명
- 타깃: HPC(Linux + RAUC), ECU/MCU 제어기(RTOS/베어메탈)
- 역할: 임베디드, 오토모티브, 로봇, MaaS·모빌리티, AI (모두 1인이 맡음)

이 폴더는 Sphinx 문서 빌드(`docs/conf.py`, `.rst`만 읽음)에 포함되지 않는다. upstream RAUC에 기여할 때는 이 폴더를 빼고 보낸다.

## 문서 지도

### 규칙

| 문서 | 내용 |
| --- | --- |
| [rules.md](rules.md) | 브랜치·버전·커밋, 코드 품질, 테스트, 안전, 보안, 셀프 리뷰, 완료 정의 |
| [../../AGENTS.md](../../AGENTS.md) | AI 에이전트용 저장소 규칙과 차량용 가드레일 |
| [../contributing.rst](../contributing.rst) | upstream RAUC 기여 규칙 (최우선) |

### 역할별 지침

모두 같은 목차를 쓴다: 목적 / 적용 범위 / 설계 원칙 / 구현 체크리스트 / 테스트·검증 / 관련 표준 / 흔한 실수 / 다른 역할과의 인터페이스

| 역할 | 문서 | 핵심 |
| --- | --- | --- |
| 임베디드 | [guides/embedded.md](guides/embedded.md) | HPC A/B 슬롯과 RAUC, MCU 듀얼 뱅크, 전원 차단 대비 |
| 오토모티브 | [guides/automotive.md](guides/automotive.md) | CAN/SOME/IP, UDS, E2E, HARA·TARA |
| 로봇 | [guides/robotics.md](guides/robotics.md) | ROS 2 계약, 독립 비상정지, 시뮬레이션에서 실제 장비로 |
| MaaS·모빌리티 | [guides/mobility-maas.md](guides/mobility-maas.md) | 장치 인증, 단계적 OTA, 개인정보 |
| AI | [guides/ai-ml.md](guides/ai-ml.md) | 재현성, 타깃 장치 측정, 모델 OTA·롤백 |

### 산출물 양식

| 양식 | 파일 | ID 형식 | 주로 쓰는 역할 |
| --- | --- | --- | --- |
| 요구사항 명세 | [templates/requirements.md](templates/requirements.md) | `REQ-<영역>-<번호>` | 전체 |
| 아키텍처/설계 | [templates/architecture.md](templates/architecture.md) | `ARC-<영역>-<번호>` | 전체 |
| 결정 기록 | [templates/adr.md](templates/adr.md) | `ADR-<0000>` | 전체 |
| 인터페이스 정의서 | [templates/interface.md](templates/interface.md) | `IFD-<번호>` | 오토모티브, 로봇, MaaS |
| 간이 HARA | [templates/hara.md](templates/hara.md) | `HZ-`, `SG-` | 오토모티브, 로봇 |
| 간이 TARA | [templates/tara.md](templates/tara.md) | `AS-`, `TS-` | 오토모티브, MaaS, 임베디드 |
| 테스트 계획서 | [templates/test-plan.md](templates/test-plan.md) | `TP-`, `TC-<영역>-<번호>` | 전체 |
| 테스트 결과 보고서 | [templates/test-report.md](templates/test-report.md) | `TR-` | 전체 |
| 릴리스·OTA 기록 | [templates/release-ota.md](templates/release-ota.md) | 버전 | 임베디드, MaaS |
| 모델 카드 | [templates/model-card.md](templates/model-card.md) | 모델 버전 | AI |
| 데이터셋 카드 | [templates/dataset-card.md](templates/dataset-card.md) | `DS-` | AI, MaaS |
| 이슈/장애 보고서 | [templates/incident.md](templates/incident.md) | `INC-` | 전체 |
| 주간 회고 | [templates/weekly-retro.md](templates/weekly-retro.md) | `YYYY-Www` | 나 |

## 개발 단계별 사용 문서

| 단계 | 할 일 | 쓰는 문서 | 지켜야 할 규칙 |
| --- | --- | --- | --- |
| 기획 | 기능 정의, 위험 파악 | 요구사항, 간이 HARA, 간이 TARA | rules 1, 7.5 |
| 설계 | 구조·인터페이스 결정 | 아키텍처, ADR, 인터페이스 정의서 | rules 9.4 |
| 구현 | 코드·테스트 작성 | 역할별 지침 체크리스트 | rules 2, 4, 5 |
| 검증 | 단위 → QEMU/SIL → HIL | 테스트 계획서, 테스트 결과 보고서 | rules 6, 9 |
| 배포 | 번들 생성·서명, 단계적 OTA | 릴리스·OTA 기록, (AI) 모델 카드 | rules 3, 7, 8 |
| 운영 | 장애 대응, 취약점 대응, 재학습 | 이슈/장애 보고서, 데이터셋 카드 | rules 8.5 |
| 매주 | 돌아보기 | 주간 회고 | — |

## 문서 작성 규칙

- 양식을 복사해 쓴다. 저장 위치: `TODO(확인 필요)` (예: `project/docs/<종류>/<ID>.md`)
- 파일 이름은 문서 ID로 한다. 예: `REQ-OTA.md`, `ADR-0001-mcu-flashing.md`
- 한국어로 쓰고, 코드·명령어·식별자·표준 이름은 원문 그대로 쓴다.
- [필수] 칸은 비워 두지 않는다. 해당 없으면 "해당 없음"과 이유를 쓴다.
- 모르는 값은 지어내지 않고 `TODO(확인 필요)`로 남긴다.
- 안전·보안 표준은 "참고" 또는 "준용"으로만 쓴다. "준수", "인증"이라고 쓰지 않는다.

## 결정이 필요한 항목 (TODO 모음)

정하면 해당 문서의 `TODO(확인 필요)`를 고치고 이 목록에서 지운다.

| # | 항목 | 관련 문서 |
| --- | --- | --- |
| 1 | 프로젝트 한 줄 설명 | 이 문서 |
| 2 | 프로젝트 전용 코드·문서 위치 (fork 내부 `project/` 또는 별도 저장소) | [rules 1](rules.md#1-적용-범위와-우선순위), 이 문서 |
| 3 | HPC 이미지 빌드 시스템 (Yocto / Buildroot)과 SoC | [embedded](guides/embedded.md) |
| 4 | MCU 종류와 RTOS (FreeRTOS / Zephyr / AUTOSAR Classic / 베어메탈) | [embedded](guides/embedded.md) |
| 5 | MISRA 검사 도구 | [rules 5](rules.md#5-코드-품질-기준) |
| 6 | mark-good 헬스체크 항목 | [rules 7](rules.md#7-안전-규칙) |
| 7 | MCU 플래싱 허용 안전 상태 조건 (차속, 기어, 전압) | [rules 7](rules.md#7-안전-규칙), [HARA](templates/hara.md) |
| 8 | 운영 서명 키 보관 방법 (HSM, 오프라인) | [rules 8](rules.md#8-보안-규칙) |
| 9 | HIL 벤치 구성 (HPC 보드, MCU 보드, CAN 인터페이스, 전원 릴레이) | [automotive](guides/automotive.md), [test-plan](templates/test-plan.md) |
| 10 | 전원 차단 테스트 반복 횟수, CAN 버스 부하 목표 | [embedded](guides/embedded.md), [automotive](guides/automotive.md) |
| 11 | 로봇 종류, ROS 2 배포판, 시뮬레이터, 적용 로봇 안전 표준 | [robotics](guides/robotics.md) |
| 12 | 클라우드·인프라, 서비스 범위, OTA 배포 중단 실패율, 목표 장치 수 | [mobility-maas](guides/mobility-maas.md) |
| 13 | AI 프레임워크, 엣지 런타임, 실험 관리 도구, 데이터 저장소 | [ai-ml](guides/ai-ml.md), [dataset-card](templates/dataset-card.md) |
| 14 | 위치·운행 데이터 수집의 법적 근거 | [interface](templates/interface.md), [dataset-card](templates/dataset-card.md) |
