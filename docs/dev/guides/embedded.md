# 임베디드 개발 지침 (HPC Linux + ECU/MCU)

공통 규칙은 [rules.md](../rules.md)를 따른다. 이 문서는 임베디드 역할에서 추가로 지킬 내용이다.

## 1. 목적

HPC와 ECU/MCU가 안정적으로 부팅하고, 업데이트에 실패해도 항상 복구되도록 시스템 소프트웨어를 만든다.

## 2. 적용 범위

| 대상 | 내용 |
| --- | --- |
| HPC (Linux) | 부트로더, 커널, rootfs 이미지(Yocto/Buildroot `TODO(확인 필요)`), RAUC 설정·훅·핸들러, 시스템 서비스 |
| ECU/MCU | 부트로더, RTOS 또는 베어메탈 펌웨어(`TODO(확인 필요)`: FreeRTOS, Zephyr, AUTOSAR Classic 등), 듀얼 뱅크 플래시 |
| 제외 | 차량 통신 설계(→ [automotive.md](automotive.md)), 응용 기능(→ 각 역할) |

## 3. 설계 원칙

### HPC

- **A/B 슬롯과 읽기 전용 rootfs.** 쓰기가 필요한 데이터는 별도 `data` 파티션에 둔다. rootfs를 런타임에 바꾸지 않는다.
- **부트 선택은 RAUC bootchooser에 맡긴다.** 부트로더(U-Boot, GRUB, EFI, barebox)의 부팅 시도 횟수와 fallback을 반드시 켠다.
- **헬스체크 후 mark-good.** 부팅 직후 systemd 서비스가 핵심 기능을 확인하고 `rauc status mark-good`을 호출한다. 실패하면 호출하지 않아 자동 롤백되게 한다.
- **MCU 업데이트는 HPC가 조정한다.** RAUC 번들에 MCU 이미지를 넣고, 설치 훅(`install-check`, slot handler)에서 UDS로 플래싱한다. 결정은 [ADR](../templates/adr.md)로 남긴다.

### ECU/MCU

- **듀얼 뱅크 또는 부트로더 + 앱 분리.** 실행 중인 뱅크에 쓰지 않는다. 새 뱅크는 CRC와 서명을 검증한 뒤에만 활성화한다.
- **워치독은 항상 켠다.** 메인 루프나 태스크 모니터가 주기적으로 리셋하고, 디버그 빌드에서도 끄지 않는다.
- **동적 메모리를 쓰지 않는다.** 스택 크기를 정적으로 정하고 사용량을 측정한다.
- **MISRA C:2012를 적용한다.** 예외는 ADR에 남긴다.

## 4. 구현 체크리스트

### HPC

- [ ] `system.conf`의 `compatible`이 [버전 규칙](../rules.md#3-버전-규칙)을 따른다
- [ ] keyring에는 운영 인증서만 있고, 개발 인증서는 개발 이미지에만 있다
- [ ] bootchooser 백엔드와 부팅 시도 횟수를 설정했다
- [ ] `data` 파티션, 로그, 상태 파일 위치를 정했다 (`data-directory`)
- [ ] mark-good 헬스체크 서비스가 있다
- [ ] 번들 형식이 `verity` 이상이다
- [ ] 이벤트 로그가 영속 저장소(journal)에 남는다

### ECU/MCU

- [ ] 뱅크 전환이 원자적이다 (플래그 한 번 쓰기로 전환)
- [ ] 플래싱 전 안전 상태 조건(REQ-SAF-…)을 확인한다
- [ ] 펌웨어 이미지에 버전, 빌드 ID, CRC가 들어 있다
- [ ] 부트로더가 앱 이미지를 검증하고, 실패하면 이전 뱅크로 부팅한다
- [ ] 스택 사용량과 최악 실행 시간을 측정했다

## 5. 테스트·검증

| 항목 | 방법 | 수준 |
| --- | --- | --- |
| RAUC 설치·롤백 흐름 | `./qemu-test`, `./qemu-test system` | 통합(가상) |
| 슬롯·부트로더 설정 | 실제 HPC에서 `rauc status`, 강제 롤백 | HIL |
| OTA 중 전원 차단 | 전원 릴레이로 쓰기 단계마다 차단, 최소 `TODO`회 반복 | HIL |
| MCU 뱅크 전환 | 플래싱 중 CAN 끊김·전원 차단 | HIL |
| MCU 로직 | 호스트 빌드 + Unity/Ceedling | SIL |
| 정적분석 | scan-build(HPC), MISRA 검사(MCU) | 매 커밋 |

## 6. 관련 표준 (참고·준용 수준)

| 표준 | 적용 부분 |
| --- | --- |
| UNECE R156 / ISO 24089 | 소프트웨어 식별(버전, `compatible`), 업데이트 기록, 실패 시 복구 |
| ISO 26262-6 | MCU 소프트웨어 코딩 지침(MISRA), 단위 테스트 |
| MISRA C:2012 | MCU C 코드 |

## 7. 흔한 실수

- 부트로더 환경 변수 영역을 A/B로 나누지 않아, 쓰기 중 전원이 끊기면 부트로더가 깨진다.
- mark-good을 부팅 직후 무조건 호출해 롤백이 동작하지 않는다.
- 개발용 인증서가 들어간 keyring을 운영 이미지에 그대로 넣는다.
- MCU 플래싱 중 HPC가 재부팅되면 MCU가 부트로더 상태에 머문다. 재개 절차가 없다.
- `/etc`에 런타임 설정을 써서 슬롯을 바꾸면 설정이 사라진다.

## 8. 다른 역할과의 인터페이스

| 상대 역할 | 주고받는 것 | 문서 |
| --- | --- | --- |
| 오토모티브 | UDS 플래싱 시퀀스, 안전 상태 조건 | [interface](../templates/interface.md), [HARA](../templates/hara.md) |
| 로봇 | ROS 2 런타임 이미지, 실시간 커널 설정 | [architecture](../templates/architecture.md) |
| MaaS | OTA 서버 API, 설치 결과 보고 | [interface](../templates/interface.md), [release-ota](../templates/release-ota.md) |
| AI | 모델 파일의 슬롯·artifact 배치, 추론 런타임 | [model-card](../templates/model-card.md) |
