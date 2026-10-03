# 소방특장컨트롤러(FSM uC) 상태 머신 설계 문서 (A단계)

> Remote Vehicle 소방특장(Fire-fighting Special eQuipment) 제어 게이트웨이 컨트롤러
> Target HW: Rexroth RC40 (BODAS) / Gateway Controller (GW2)
> 본 문서는 A(설계) → B(Simulink/Stateflow `.m` 자동생성) → C(RC40/C 구현) 중 **A단계** 산출물이다.

---

## 1. 개요 (Overview)

FSM uC는 **주 컨트롤러(Chassis uC)가 판단하여 CAN으로 전달하는 차량 상태**와 **원격/로컬 조작 명령**을 종합하여, 소방특장품(소방펌프 · 메인밸브 · 엔진시동)의 **High Side 파워 출력을 게이팅**하는 컨트롤러이다.

핵심 성격:

- **게이트웨이(Gateway)**: 복잡한 폐루프 제어보다는 "허가 조건 하에서 입력 명령을 해당 출력으로 통과(gate)"시키는 것이 주 기능.
- **다중 입력 중재**: 로컬 버튼 접점(수동운전) vs 원격 CAN 명령(원격운전) 사이를 **Mode Selection 스위치**로 명시 전환.
- **안전 감시**: 수압(Water Pressure) 저하 시 경고(Warning) 발생. (강제 차단 인터록 없음 — 운전자/원격 판단 유지)

---

## 2. 입출력 정의 (I/O — RC40 Pinmap GW2 기준)

Pinmap(`RC40_Pinmap.xlsx`, 시트 `GW2`)에서 **Pin Assignment 열이 채워진 핀만** 실제 포트로 생성한다 (MASAR TopModel 규칙).

### 2.1 명령 입력 (로컬 버튼 접점) — `DevInp_<pin>_D`

| 핀 | Type | In/Out | Pin Assignment | Connect to | 기능 | MASAR 매핑 |
|----|------|--------|----------------|------------|------|------------|
| A25 | AnalogSignal | IN | `SIG_Pmp_Start`    | PumpStart_Sig       | 펌프 시동 버튼       | `AnU_as[DevInp_A25_D]:HwInpAnU` |
| A26 | AnalogSignal | IN | `SIG_Pmp_Stop`     | PumpStop_Sig        | 펌프 정지 버튼       | `AnU_as[DevInp_A26_D]:HwInpAnU` |
| A27 | AnalogSignal | IN | `SIG_MV_Open`      | MainValve_Open_Sig  | 메인밸브 열림 버튼   | `AnU_as[DevInp_A27_D]:HwInpAnU` |
| A40 | AnalogSignal | IN | `SIG_MV_Close`     | MainValve_Close_Sig | 메인밸브 닫힘 버튼   | `AnU_as[DevInp_A40_D]:HwInpAnU` |
| A41 | AnalogSignal | IN | `SIG_Pmp_RPM_Up`   | Pump_RPM_UP_Sig     | 펌프 RPM 올림 버튼   | `AnU_as[DevInp_A41_D]:HwInpAnU` |
| A42 | AnalogSignal | IN | `SIG_Pmp_RPM_Down` | Pump_RPM_Down_Sig   | 펌프 RPM 내림 버튼   | `AnU_as[DevInp_A42_D]:HwInpAnU` |

> 입력은 Pull-Down 32V 아날로그 라인으로, 버튼 접점의 전압 레벨을 임계값(THR)으로 판정하여 디지털 눌림(pressed) 상태로 해석한다.

### 2.2 센서 입력 — `DevInp_<pin>_D`

| 핀 | Type | In/Out | Pin Assignment | Connect to | 기능 | MASAR 매핑 |
|----|------|--------|----------------|------------|------|------------|
| K38 | DigitalSignal | IN | `SIG_FS` | FSM-WP-F-#C | Water Pressure 센서 | `Dig_as[DevInp_K38_D]:HwInpDig` |

> 센서 전원/GND: K71 `PWR_FS`(VSS_1 5V, FSM-WP-F-#B), K18 `GND_FS`(SensorGND, FSM-WP-F-#A). (포트 아님 — 배선 전용)

### 2.3 액추에이터 출력 (High Side 파워) — `DevOutp_<pin>HS_D`

| 핀 | Type | HS/LS | Pin Assignment | Connect to | 기능 | MASAR 매핑 |
|----|------|-------|----------------|------------|------|------------|
| A34 | PWMSignal     | HS | `Out_Pmp_Start`    | Pump_Start_Out       | 펌프 시동 출력       | `PropPwr_as[DevOutp_A34HS_D]:HwOutpPropPwr` |
| A35 | PWMSignal     | HS | `Out_Pmp_Stop`     | Pump_Stop_Out        | 펌프 정지 출력       | `PropPwr_as[DevOutp_A35HS_D]:HwOutpPropPwr` |
| A51 | PWMSignal     | HS | `OUT_MV_Open`      | MainValve_Open_Out   | 메인밸브 열림 출력   | `PropPwr_as[DevOutp_A51HS_D]:HwOutpPropPwr` |
| A52 | PWMSignal     | HS | `OUT_MV_Close`     | MainValve_Close_Out  | 메인밸브 닫힘 출력   | `PropPwr_as[DevOutp_A52HS_D]:HwOutpPropPwr` |
| A53 | PWMSignal     | HS | `OUT_Pmp_RPM_Up`   | Pump_RPM_UP_Out      | 펌프 RPM 올림 출력   | `PropPwr_as[DevOutp_A53HS_D]:HwOutpPropPwr` |
| A54 | PWMSignal     | HS | `OUT_Pmp_RPM_Down` | Pump_RPM_Down_Out    | 펌프 RPM 내림 출력   | `PropPwr_as[DevOutp_A54HS_D]:HwOutpPropPwr` |
| A50 | DigitalSignal | HS | `OUT_Eng_Start_HS` | Engine_Start_HS_OUT  | 엔진 시동 출력       | `DigSig_as[DevOutp_A50HS_D]:HwOutpDigSig` |

### 2.4 통신 (CAN)

| 포트 | 핀(H/L) | 용도 | 본 설계에서의 역할 |
|------|---------|------|---------------------|
| CAN1 | K88/K66 | Wake-up capable, not selective | (예비) |
| CAN2 | K89/K67 | For programming | 플래싱 전용 |
| CAN3 | K90/K68 | ISOBUS | (예비) |
| **CAN4** | **K91/K69** | Wake-up capable, selective | **Chassis uC state 및 원격 명령 수신 (주 통신)** |

> Chassis state(작동 허가)와 원격 조작 명령(Remote Command), 그리고 FSM 상태정보 송신(Status/FSM Info)은 모두 **CAN4** 를 통한다. 구체 메시지/시그널 매핑은 DBC 수령 후 바인딩(아래 10절 참조).

---

## 3. 운전 모드 (Mode Selection — 최상위 상태)

물리 셀렉터(또는 CAN/패널 설정)로 결정되는 3-포지션 최상위 모드.

```
        ┌─────────────────────────────────────────────┐
        │                   OFF                        │
        │   모든 소방특장 출력 비활성 (안전 기본 상태)   │
        └───────────────┬───────────────┬─────────────┘
                 selector│               │selector
          ┌──────────────▼──┐        ┌───▼───────────────┐
          │     LOCAL        │        │     REMOTE        │
          │  (수동운전모드)   │        │   (원격운전모드)   │
          │ 버튼 접점 입력이  │        │ CAN4 Remote Cmd가  │
          │ 출력을 직접 게이팅 │        │ 출력을 구동        │
          └──────────────────┘        └───────────────────┘
```

| 모드 | 명령 소스 | 출력 동작 | 상태 피드백 |
|------|-----------|-----------|-------------|
| **OFF** | 없음 | 전 출력 OFF | — |
| **LOCAL** | 로컬 버튼 접점 (A25~A42) | 버튼 → 대응 출력 게이팅 | `FSM Info` → FSM Control Panel |
| **REMOTE** | CAN4 Remote Command | CAN 명령 → 대응 출력 구동 | `Status` → Control Computer |

**모드 전이 규칙**
- 모드 전환 시 **먼저 전 출력을 OFF로 리셋**한 뒤 새 모드 진입(bumpless, 안전 우선).
- `OFF ↔ LOCAL ↔ REMOTE` 전환은 셀렉터 입력으로만. 소프트웨어가 임의로 모드를 바꾸지 않는다(단, 안전 상태 진입 예외 — 5절).

---

## 4. 상위 게이트 — Chassis Enable (CAN4 수신)

Chassis uC가 CAN4로 전달하는 **작동 허가 상태**를 추상 신호 `CHASSIS_ENABLE`로 둔다.

- `CHASSIS_ENABLE == TRUE`  : LOCAL/REMOTE에서 출력 구동 허용
- `CHASSIS_ENABLE == FALSE` : 모드와 무관하게 **전 출력 OFF 유지** (소방작업 비허가 — 예: 주행중)

> `CHASSIS_ENABLE`의 실제 CAN 시그널/enum은 DBC 수령 후 바인딩(10절). 다중 state enum이면 "작동 허가에 해당하는 값 집합 → TRUE"로 매핑.

---

## 5. 안전 / 통신 감시 (상시)

| 조건 | 감지 | 반응 |
|------|------|------|
| **CAN4 타임아웃** (Chassis state 또는 Remote Command 수신 끊김) | 수신 주기 감시 (워치독) | `SAFE_STATE` 진입 → 전 출력 OFF, 경고 래치 |
| **Water Pressure 저하** (K38 `SIG_FS`) | 임계값(WP_LOW_THR) 비교 | **경고(Warning)만 발생** (출력 차단 없음), `FSM Info`/`Status`로 통지 |
| **출력 에러 반응** (단락/과전류 등) | RC40 HW `stErrDetn` / `stErrReactn` | 해당 출력 보호 동작(HW 기본), 상태 보고 |

- `SAFE_STATE`는 통신 복구 + 명시적 해제(또는 OFF 경유 재진입) 시에만 벗어난다.
- Remote 모드에서 통신 두절 → 데드맨(dead-man) 개념으로 **전 출력 OFF**가 기본(Remote Vehicle 안전 원칙).

---

## 6. 상태 머신 계층 구조 (Hierarchical State Machine)

```
FSM_uC (root)
│
├── [SUPERVISOR]  ── 상시 병렬(parallel) 감시
│     ├─ CommMonitor:  CAN4 수신 워치독 → ok / timeout
│     └─ WpMonitor:    Water Pressure → normal / low(warning)
│
└── [OPERATING_MODE]  ── 배타(exclusive) 상태
      ├── OFF            (초기 상태, 전 출력 0)
      ├── SAFE_STATE     (통신두절 등 → 전 출력 0, 복구 대기)
      ├── LOCAL
      │     └── (Chassis Enable 가드) ──> 버튼→출력 게이팅 서브로직
      └── REMOTE
            └── (Chassis Enable 가드) ──> CAN Remote Cmd→출력 서브로직
```

병렬 SUPERVISOR는 모드와 독립적으로 항상 동작하며, `timeout` 발생 시 OPERATING_MODE를 `SAFE_STATE`로 강제 전이시킨다.

---

## 7. 상태 전이 테이블 (Transition Table)

| # | From | Event / Guard | To | Action (진입 시) |
|---|------|---------------|----|------------------|
| T1 | (init) | 전원 On | OFF | 전 출력 OFF |
| T2 | OFF | selector = LOCAL | LOCAL | 출력 리셋 후 LOCAL 진입 |
| T3 | OFF | selector = REMOTE | REMOTE | 출력 리셋 후 REMOTE 진입 |
| T4 | LOCAL | selector = OFF | OFF | 전 출력 OFF |
| T5 | REMOTE | selector = OFF | OFF | 전 출력 OFF |
| T6 | LOCAL | selector = REMOTE | REMOTE | 출력 리셋 후 전환 |
| T7 | REMOTE | selector = LOCAL | LOCAL | 출력 리셋 후 전환 |
| T8 | LOCAL / REMOTE | CAN4 timeout | SAFE_STATE | 전 출력 OFF, 경고 래치 |
| T9 | SAFE_STATE | CAN4 ok 복구 AND selector 재확인(또는 OFF 경유) | OFF | 전 출력 OFF 유지, 경고 해제 |
| G1 | LOCAL / REMOTE | `CHASSIS_ENABLE == FALSE` (가드) | (동일 모드 유지) | 전 출력 강제 OFF |

> 수압 저하(Wp low)는 상태 전이를 일으키지 않으며 Warning 플래그만 세팅한다.

---

## 8. 입력 → 출력 매핑 (게이팅 매트릭스)

각 기능은 입력(버튼 or 원격명령)과 출력이 1:1 대응한다. 출력 구동 조건:
`출력 ON ⟺ (모드 활성) AND (CHASSIS_ENABLE) AND (해당 명령 active) AND (SUPERVISOR != timeout)`

| 기능 | LOCAL 입력(버튼) | REMOTE 입력(CAN4) | 출력 핀 | 출력 종류 |
|------|------------------|--------------------|---------|-----------|
| 펌프 시동        | A25 `SIG_Pmp_Start`    | Cmd.PumpStart    | A34 `Out_Pmp_Start`    | PropPwr (HS) |
| 펌프 정지        | A26 `SIG_Pmp_Stop`     | Cmd.PumpStop     | A35 `Out_Pmp_Stop`     | PropPwr (HS) |
| 펌프 RPM 올림    | A41 `SIG_Pmp_RPM_Up`   | Cmd.PumpRpmUp    | A53 `Out_Pmp_RPM_Up`   | PropPwr (HS) |
| 펌프 RPM 내림    | A42 `SIG_Pmp_RPM_Down` | Cmd.PumpRpmDown  | A54 `Out_Pmp_RPM_Down` | PropPwr (HS) |
| 메인밸브 열림    | A27 `SIG_MV_Open`      | Cmd.MainValveOpen| A51 `OUT_MV_Open`      | PropPwr (HS) |
| 메인밸브 닫힘    | A40 `SIG_MV_Close`     | Cmd.MainValveClose| A52 `OUT_MV_Close`    | PropPwr (HS) |
| 엔진 시동        | (해당 버튼 미할당*)     | Cmd.EngineStart  | A50 `OUT_Eng_Start_HS` | DigSig (HS)  |

> *엔진 시동: GW2에서 입력측 버튼 핀 할당이 명시되지 않음(출력 A50만 할당). LOCAL 모드 시동 트리거 소스는 확인 필요(10절 Q1).

**상호배타(Interlock) 권고**: 펌프 Start/Stop, 밸브 Open/Close, RPM Up/Down 은 **동시 ON 금지**(동일 쌍 중 하나만). 동시 입력 시 우선순위(예: Stop/Close 우선) 적용 — 구현 시 가드로 처리.

---

## 9. HW 신호 접근 (C단계 참고 — RC40 BSW)

C단계에서 사용할 구조체 접근 패턴 (헤더 근거):

**출력(설정은 `.inp_s`):**
- PropPwr(펌프/밸브): `HwOutp_s.PropPwr_as[DevOutp_<pin>HS_D].inp_s` → `stErrReactn_e`, `iSp_mA_u16`(0..4000), `dutyCycSp_perml_u16`(0..1000), `flgSp_l`
- DigSig(엔진시동): `HwOutp_s.DigSig_as[DevOutp_A50HS_D].inp_s` → `stErrReactn_e`, `flgSp_l`

**입력(측정은 `.outp_s`):**
- AnU(버튼 접점): `HwInp_s.AnU_as[DevInp_<pin>_D].outp_s` → `u_mV_u16`(0..40000), `stErrDetn_u8`
- Dig(수압센서): `HwInp_s.Dig_as[DevInp_K38_D].outp_s` → `flg_l`, `stErrDetn_u8`

> 버튼 접점(AnU)은 `u_mV_u16`를 임계값과 비교해 pressed/released로 디지털화.

---

## 10. 미해결 / 후속 바인딩 (Open Items)

DBC·시스템아키텍처 수령 후 확정:

- **Q1 (엔진시동 LOCAL 소스)**: A50 출력에 대응하는 LOCAL 버튼 입력이 핀맵에 없음. 로컬 시동 트리거를 어떻게 받는지(별도 핀 / 패널 전용 / Remote 전용) 확인 필요.
- **Q2 (CHASSIS_ENABLE 매핑)**: CAN4 상의 Chassis state enum 값 집합 → DBC로 바인딩.
- **Q3 (Remote Command 메시지)**: CAN4 Remote Command 각 비트/시그널 → 기능 매핑 (DBC).
- **Q4 (Status/FSM Info 송신)**: FSM 상태·경고를 담아 CAN4로 송신할 메시지 레이아웃.
- **Q5 (수압 임계값 WP_LOW_THR)**: 경고 발생 수압 임계값(물리 단위) 수치.
- **Q6 (버튼 임계 THR)**: AnU 입력 pressed 판정 전압 임계값.
- **Q7 (타임아웃 값)**: CAN4 수신 워치독 타임아웃(ms).

---

## 11. 다음 단계 (B / C)

- **B단계**: 본 설계를 바탕으로 MATLAB `.m` 스크립트를 작성하여 Simulink/Stateflow 상태 머신 + GW2 Inport/Outport + CAN4 Rx/Tx 포트를 자동 생성. 포트 명명·데이터타입은 2절·9절 규칙 적용.
- **C단계**: RC40 BSW 통합 — `Os10msProc.c` 패턴으로 상태 머신 출력을 `HwOutp_s` 구조체에 매핑, 입력은 `HwInp_s`에서 읽기.

> B·C단계는 코드 산출물이므로 설계 승인 후 별도로 진행한다.

---

### 부록 A. 상태 다이어그램 (ASCII)

```
                    ┌────────────────────────────┐
         power on   │                            │
      ───────────▶  │           OFF              │ ◀───────┐
                    │   (all outputs = 0)        │         │ selector=OFF
                    └───┬───────────────┬────────┘         │
          selector=LOCAL│               │selector=REMOTE   │
                        ▼               ▼                  │
              ┌──────────────┐   ┌──────────────┐          │
              │    LOCAL     │   │    REMOTE    │──────────┤
              │ 버튼→출력     │◀─▶│ CAN4 Cmd→출력 │          │
              └──────┬───────┘   └──────┬───────┘          │
                     │  CAN4 timeout     │                 │
                     └─────────┬─────────┘                 │
                               ▼                           │
                    ┌────────────────────────────┐         │
                    │        SAFE_STATE          │─────────┘
                    │ (all outputs=0, warn latch)│  CAN4 ok → OFF
                    └────────────────────────────┘

   [parallel] SUPERVISOR: CommMonitor(CAN4 watchdog) + WpMonitor(수압 저하 경고)
   [guard]    CHASSIS_ENABLE==FALSE → 모든 모드에서 출력 강제 OFF
```
