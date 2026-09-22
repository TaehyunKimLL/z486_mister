# User port를 통한 외장 ISA 버스 (LPC 브리지) 설계

상태: **설계 검토 / 미구현**
대상: 외장 ISA 사운드카드(Sound Blaster 계열, DMA + IRQ 필요)를
MiSTer User port에 연결해 z486 코어에서 사용하는 것

---

## 1. 목표와 범위

- z486 코어의 I/O 버스, 8237 DMA, 8259 PIC을 **LPC 호스트**로 외부에 노출한다.
- 외부에서는 LPC→ISA 브리지 보드 **dISAppointment**(Fintek F85226 LPC-to-ISA
  브리지)를 사용해 실제 ISA 카드를 장착한다.
- 1차 목표 카드: Sound Blaster 2.0 / Pro / 16 계열
  (I/O 220h–22Fh, OPL 388h, MPU-401 330h, IRQ 5/7, DMA 1 / 5).
- 필요한 기능:

  | 기능 | 필요한 이유 | 이 설계에서 |
  |---|---|---|
  | ISA I/O 읽기/쓰기 | DSP, 믹서, OPL, MPU | 지원 |
  | IRQ | DSP IRQ, MPU IRQ | 지원 (SERIRQ) |
  | 8-bit DMA | SB/SB Pro 디지털 음원 | 지원 (LDRQ + LPC DMA 사이클) |
  | 16-bit DMA | SB16 16-bit 음원 | 지원 설계, 브리지 동작은 실측 필요 |
  | ISA 메모리 사이클 | 옵션 ROM, NIC 공유 메모리 | 이번 범위에서 제외 (7절) |

- 외장 카드의 **아날로그 오디오 출력은 카드의 잭으로 직접 나간다.**
  MiSTer HDMI/아날로그 오디오 믹서로 되돌려 넣는 경로는 없다
  (남는 핀이 없고, I/O 보드에 라인 입력이 없다).

## 2. 핀 예산과 전체 구조

### 2.1 LPC 신호와 핀 예산

MiSTer User port는 `USER_IO[6:0]` 7핀이다
(`sys/sys.tcl:31-40`, 3.3-V LVTTL, weak pull-up, max current).

| LPC 신호 | 방향 | 필요 |
|---|---|---|
| LCLK | H→D | 필수 |
| LFRAME# | H→D | 필수 |
| LAD[3:0] | 양방향 | 필수 |
| LRESET# | H→D | 필수 |
| SERIRQ | 양방향(open-drain) | IRQ |
| LDRQ# | D→H | DMA |

합계 9핀. **User port 7핀 + Arduino 헤더의 보조 SD 핀 2개**로 9핀을 모두 FPGA에서
직접 구동한다(기본안). 보조 SD 슬롯은 쓸 수 없게 되며, I/O 보드의 버튼/LED(MCP23009,
`IO_SCL/IO_SDA`)는 그대로 쓸 수 있다.

- RTC 보드가 꽂히는 DE10-nano **LTC 커넥터는 HPS 핀**이라 FPGA에서 직접 쓸 수 없다.
  HPS Loan I/O는 MiSTer 전체 preloader 설정을 바꿔야 하고 RTC와 충돌하므로 제외한다.
- 추가 핀을 쓸 수 없는 환경을 위한 7핀 + 어댑터 CPLD 구성은 부록 A에 남겨 둔다.

### 2.2 핀 배치

| FPGA 포트 | 핀 | 신호 | FPGA 방향 | 비고 |
|---|---|---|---|---|
| `USER_IO[0]` | AG11 | LAD0 | 양방향 | push-pull, TAR 구간 tri-state |
| `USER_IO[1]` | AH9 | LAD1 | 양방향 | |
| `USER_IO[2]` | AH12 | LAD2 | 양방향 | SW[1]=1이면 HDMI I2S로 쓰임 → **SW[1]=0 필수** |
| `USER_IO[3]` | AH11 | LAD3 | 양방향 | |
| `USER_IO[4]` | AG16 | LFRAME# | 출력 | SW[1] 공유 핀 |
| `USER_IO[5]` | AF15 | LCLK | 출력 | DDIO 출력, SW[1] 공유 핀 |
| `USER_IO[6]` | AF17 | LRESET# | 출력 | 어댑터에 풀다운(4.4절) |
| `SD_SPI_MOSI` | U13 | SERIRQ | 양방향(open-drain) | 보조 SD MOSI 대체 |
| `SD_SPI_MISO` | AH8 | LDRQ# | 입력 | 보조 SD MISO 대체 |

- `SD_SPI_CS`(AE15)는 sync-on-green 제어에도 쓰이므로 건드리지 않는다.
  `SD_SPI_CLK`(AG8), `SDCD_SPDIF`(AH7)도 쓰지 않는다.
- USB3 커넥터 핀 번호와 `USER_IO` 대응은 MiSTer User port 핀아웃 문서를 따른다.
  LAD/LFRAME/LCLK 옆에 GND 핀을 최대한 둔다.
- SERIRQ/LDRQ#는 Arduino 헤더에서 점퍼선(또는 I/O 보드 아래 스태킹 헤더)으로 끌어낸다.
  두 신호도 LCLK에 동기화되어 있으므로 User port 쪽과 배선 길이 차이를 수 ns(수십 cm) 이내로 맞추고,
  GND 선을 함께 꼬아서 보낸다. I/O 보드의 SD 슬롯 쪽 배선은 스텁으로 남는다(카드를 꽂지 않는다).

### 2.3 블록 다이어그램

```
 z486 core (clk_sys 85 MHz)                                          dISAppointment
┌──────────────────────────────────────────────┐                   ┌──────────────┐
│ CPU ─ iobus_adapter ─┐                        │  User port        │              │
│                      ├─ lpc_host ─────────────┼─ LCLK ──────────► │              │
│ dma.v (ext ch) ──────┘  (arbiter, I/O + DMA)  ├─ LFRAME# ───────► │  F85226      │
│                                               ◄─ LAD[3:0] ──────► │  LPC→ISA     ├─ ISA 슬롯
│ core reset ───────────► lpc_reset ────────────┼─ LRESET# ───────► │              │  (SB 카드)
│                                               │  Arduino 헤더     │              │
│ pic ◄─ ext_irq ◄─ lpc_serirq_host ◄───────────┼─ SERIRQ ◄───────► │              │
│ dma ◄─ ext_drq ◄─ lpc_ldrq_rx ◄───────────────┼─ LDRQ# ◄───────── │              │
└──────────────────────────────────────────────┘                   └──────────────┘
```

어댑터는 능동 소자 없이 배선, 풀업/풀다운 저항, GND 연결만 한다.

## 3. 물리 계층

### 3.1 sys_top 수정 (필수)

현재 `sys/sys_top.v:1665-1671`은 User port를 **open-drain**으로만 구동한다
(`!user_out ? 1'b0 : 1'bZ`). weak pull-up(수십 kΩ)으로는 수 MHz도 어렵기 때문에
LPC에는 push-pull과 핀별 방향 제어가 필요하다.

- 매크로(예: `MISTER_USER_IO_PUSHPULL`)로 켜는 옵션 경로를 추가한다.
  - emu에서 `USER_OE[6:0]`(1 = push-pull 출력, 0 = 입력)를 새로 받는다.
  - `USER_OE[i]=1`이면 `USER_IO[i] = USER_OUT[i]`, 아니면 `Z`.
  - 매크로를 끄면 기존 open-drain 동작 그대로.
- LCLK는 `altddio_out`(DDIO 출력 레지스터)으로 만든다. DDIO는 패드에 붙어 있어야 하므로
  sys_top 안에 둔다. emu에서는 `USER_CLK_H/L`(DDIO 입력)만 넘긴다.
- LAD/LFRAME 출력과 LAD 입력은 `FAST_OUTPUT_REGISTER` / `FAST_INPUT_REGISTER`로
  IOE 레지스터에 고정해서 스큐를 줄인다.
- 출력 버퍼는 series OCT 사용을 검토한다
  (`OUTPUT_TERMINATION "SERIES 50 OHM WITHOUT CALIBRATION"`,
  Cyclone V 3.3-V LVTTL 지원 여부 확인). 안 되면 `CURRENT_STRENGTH`를 낮춰
  링잉을 줄인다.
- **보조 SD 핀 재할당**: 같은 매크로 아래에서 `SD_SPI_MOSI`를 SERIRQ(open-drain 양방향),
  `SD_SPI_MISO`를 LDRQ# 입력으로 emu에 넘긴다. `SD_MISO`는 `1'b1`로 고정하고
  `SD_SPI_CLK/MOSI` 구동을 끊는다(`sys/sys_top.v:133-143`). `SD_SPI_CS`와 SOG 경로는 그대로 둔다.
  두 포트가 `output`/`input`으로 선언되어 있으므로 `SD_SPI_MOSI`는 `inout`으로 바꾼다.
- MiSTer 공용 프레임워크(`sys/`) 수정이므로 업스트림 Template 머지 때 충돌 관리가 필요하다.

### 3.2 LCLK 주파수와 위상

LPC 엔진은 **clk_sys(85 MHz) 한 클럭 도메인**에서 돌리고, LCLK는 clk_sys를
나눠 만든다. 이렇게 하면 CPU/DMA와의 CDC가 없다.

| 설정 | LCLK | 비고 |
|---|---|---|
| 기본 | 85/3 = **28.33 MHz** | DDIO로 50% duty (반주기 = clk_sys 1.5클럭) |
| SI 문제 시 | 85/4 = 21.25 MHz | 파라미터로 선택 |

LPC/PCI 클럭은 33 MHz 이하에서 동작하도록 정의되어 있지만,
**F85226이 ISA BCLK(보통 8.33 MHz)를 LCLK에서 분주해 만드는지**는 데이터시트로
확인해야 한다. 만약 그렇다면 BCLK가 7.08 MHz(28.33 MHz 기준)가 되는데,
ISA 카드는 보통 문제 없는 범위다.

타이밍 (LCLK 1주기 = clk_sys 3클럭, k = LCLK 상승 에지 clk_sys 에지):

| 이벤트 | clk_sys 에지 | LCLK 상승 기준 |
|---|---|---|
| LCLK 상승 | k | 0 ns |
| 호스트 출력(LAD/LFRAME) 변경 | k+1 | +11.8 ns |
| 디바이스 샘플 | 다음 k (k+3) | 셋업 ≈ 23.5 ns, 홀드 ≈ 11.8 ns |
| 호스트 입력 샘플 | k+3 (다음 상승) | 디바이스 Tval(2–11 ns) + 케이블 왕복 + IOE ≲ 20 ns → 여유 ≈ 15 ns |

호스트 출력을 LCLK 상승과 같은 에지에 바꾸면 디바이스 쪽 홀드가 0에 가까워지므로
**의도적으로 1/3주기 늦게 출력한다.** 입력 샘플 위상(k+2 / k+3)도 파라미터로
두고 실측으로 정한다.

### 3.3 어댑터 보드

- User port 커넥터에 **직접 꽂는 형태**(케이블 없이, 또는 10 cm 이하)를 권장한다.
- 풀업: LAD[3:0], LFRAME#, LDRQ#, SERIRQ에 LPC 규격 풀업(10–100 kΩ).
  FPGA 쪽 weak pull-up은 그대로 둔다.
- LRESET#: **2.2 kΩ 풀다운**. 다른 코어가 로드되어 핀이 weak pull-up(약 25 kΩ)만 걸린 상태에서도
  약 0.3 V로 리셋이 유지되게 한다. 10 kΩ 이상이면 중간 전압이 되므로 쓰지 않는다.
- 전원: ISA 카드용 +5 V/+12 V/−12 V와 dISAppointment 전원은 **외부 ATX/어댑터**에서 공급한다.
  GND는 반드시 공통으로 묶는다.
- dISAppointment의 LPC 헤더 핀아웃과 전원 입력 사양은 보드 문서로 확인한다.

## 4. 프로토콜

### 4.1 LPC 호스트 사이클 (FPGA `lpc_host`)

필드는 전부 LAD[3:0] 4비트 단위이고, 주소는 상위 니블부터, 데이터는 하위 니블부터 보낸다.

| 사이클 | 순서 |
|---|---|
| I/O Read | START `0000` · CYCTYP `0000` · ADDR×4 · TAR×2 · SYNC×n · DATA×2 · TAR×2 |
| I/O Write | START `0000` · CYCTYP `0010` · ADDR×4 · DATA×2 · TAR×2 · SYNC×n · TAR×2 |
| DMA Read (메모리→디바이스) | START · CYCTYP `1000` · CHANNEL · SIZE · { DATA×2 · TAR×2 · SYNC×n · TAR×2 } × 바이트 수 |
| DMA Write (디바이스→메모리) | START · CYCTYP `1010` · CHANNEL · SIZE · TAR×2 · { SYNC×n · DATA×2 } × 바이트 수 · TAR×2 |

- CHANNEL 필드: bit3 = **TC**, bit[2:0] = 채널 번호.
- SIZE 필드: `00` = 8-bit, `01` = 16-bit.
- DMA 방향 표기는 8237 관례(Read = 메모리 읽기 → I/O 쓰기)를 따른다.
  16-bit DMA에서 바이트마다 SYNC/TAR가 반복되는 세부 순서는 구현 전에
  Intel LPC Interface Specification 1.1로 재확인한다.

SYNC 처리:

| SYNC | 의미 | 호스트 동작 |
|---|---|---|
| `0000` | Ready | 계속 |
| `0101` | Short wait | 대기, 짧은 타임아웃(기본 64 LCLK) |
| `0110` | Long wait | 대기, 긴 타임아웃(기본 약 100 µs, 파라미터) |
| `1001` | Ready More (DMA 전용) | 완료, **DRQ 유지** (4.3절) |
| `1010` | Error | 완료 처리, 읽기 값 `FF`, 에러 카운터 증가 |
| `1111` 지속 | 응답 없음 | TAR 뒤 3 LCLK 안에 SYNC가 없으면 abort, 읽기 값 `FF` |

Abort: LFRAME#를 4 LCLK 이상 Low로, LAD=`1111`.

중재: `lpc_host`는 한 번에 사이클 하나만 처리한다. 대기 중인 요청이 겹치면
**DMA 요청을 CPU I/O보다 먼저** 처리하고, 진행 중인 사이클은 중단하지 않는다.

### 4.2 LDRQ# 수신 (FPGA `lpc_ldrq_rx`)

- 메시지: START `0`, CHANNEL[2:0] (MSB 먼저), ACT 1비트, 이후 idle `1`. `lpc_host`와 같은
  샘플 위상에서 LCLK마다 1비트씩 읽는다.
- 입력은 2단 동기화 없이 IOE 입력 레지스터로 바로 받는다(LCLK에 동기화된 신호이므로
  `lpc_host`의 샘플 위상 규칙을 그대로 적용).

### 4.3 DRQ 래치 (LDRQ 의미 보존)

채널마다 LPC 호스트 규격과 같은 의미의 `ext_drq[ch]` 래치를 둔다.

```
set   : LDRQ 메시지 (ch, ACT=1)
clear : LDRQ 메시지 (ch, ACT=0)
        또는 ch에 대한 LPC DMA 사이클이 SYNC=Ready(0000)로 끝났을 때
keep  : SYNC=Ready More(1001)로 끝났을 때 (디맨드/블록 전송)
```

싱글 모드에서는 디바이스가 매 전송 뒤 ACT=1을 다시 보낸다. 사이클 완료로 clear한 뒤
새 메시지로만 set되므로 이미 서비스한 요청을 두 번 처리하지 않는다.

### 4.4 LRESET# (FPGA 직접 구동)

- 코어 리셋(`reset`, OSD Reset) 동안, 그리고 OSD에서 LPC를 끈 동안 Low.
- 리셋 해제 뒤에도 최소 1 ms(2^15 LCLK) 더 Low를 유지한다. LCLK는 리셋 중에도 계속 출력한다
  (LPC 규격상 리셋 중에도 클럭 필요).
- 코어가 로드되지 않았거나 다른 코어일 때는 어댑터의 2.2 kΩ 풀다운으로 리셋 상태를 유지한다.

### 4.5 SERIRQ (FPGA `lpc_serirq_host`)

- **continuous 모드**로 계속 동작한다: start frame(Low 8 LCLK), 슬롯당 3 LCLK
  (sample/recovery/turnaround), 뒤에 stop frame(Low 3 LCLK = continuous 유지).
  한 프레임 약 60 LCLK ≈ 2.1 µs.
- 슬롯 매핑: 0 = IRQ0, 1 = IRQ1, 2 = SMI#(무시), 3–15 = IRQ3–15, 16 = IOCHCK#(무시).
- 슬롯 극성(ISA active-high IRQ를 어떤 레벨로 싣는지)은 SERIRQ 규격과 F85226 데이터시트로
  확정한다. 파라미터로 둔다.
- 아무도 구동하지 않는 슬롯은 풀업으로 떠 있다. `ext_irq_mask`로 **사용할 IRQ만**
  통과시켜서 잘못된 인터럽트를 막는다.
- 출력은 open-drain(Low 또는 Z)이고, start/stop frame에서만 Low를 구동한다.

## 5. z486 코어 RTL 변경

### 5.1 새 모듈

| 파일 | 역할 | 예상 규모 |
|---|---|---|
| `src/lpc/lpc_host.sv` | LPC 사이클 FSM (I/O, DMA), SYNC/타임아웃/abort, LRESET#, LCLK DDIO 입력 생성 | 약 400 ALM |
| `src/lpc/lpc_serirq_host.sv` | SERIRQ continuous 모드 호스트, `ext_irq[15:0]` | 약 60 ALM |
| `src/lpc/lpc_ldrq_rx.sv` | LDRQ# 메시지 디코드 | 약 20 ALM |
| `src/lpc/lpc_drq.sv` | 채널별 `ext_drq` 래치 (4.3절) | 약 30 ALM |
| `tests/lpc/*` | LPC 디바이스 BFM(SERIRQ/LDRQ 포함) 테스트벤치 | – |

### 5.2 I/O 경로: `src/iobus_adapter.sv`, `src/system.sv`

현재 `iobus_adapter`는 바이트마다 ISSUE → WAIT → CAPTURE를 **고정 사이클**로 처리하고,
선택된 장치가 없는 포트는 `iobus_readdata8` mux 기본값 `8'hFF`로 응답한다
(`src/system.sv:808-822`).

변경:

1. `system.sv`에 `int_claimed`를 만든다. 내부 칩셀렉트를 모두 OR한 값이다(`ide0/1`, `floppy0`,
   `dma_*`, `pic_*`, `pit`, `ps2_*`, `rtc`, `fm`, `sb`, `mpu`, `joy`, `vga_*`,
   `debug_port`, `speedctl`, `zsst_pci`).
   `speedctl_cs`(8888h)는 읽기 mux에 없으므로 빠뜨리지 않도록 주의한다.
   `uart1_cs`/`uart2_cs`는 디코드만 되고 연결된 UART가 없으므로 **넣지 않는다**
   (외장 시리얼 카드가 3F8h/2F8h를 쓸 수 있게).
2. `ext_io_sel = lpc_enable & ~int_claimed` (subtractive decode, 실제 칩셋의 ISA 브리지와 같은 방식).
   예외: 포트 80h는 `dma_page_cs`(80h–8Fh)에 들어가 있지만, 실제 칩셋처럼
   **80h 쓰기는 내부와 LPC에 동시에** 보낸다(POST 카드용, 옵션).
3. `iobus_adapter`에 `io_ext_sel`, `io_ext_ready` 포트를 추가한다.
   - ISSUE에서 `io_ext_sel`이면 `lpc_host`에 요청을 넣고, WAIT 상태에서
     `io_ext_ready`까지 머문다. 읽기는 `lpc_host` 데이터를 캡처한다.
   - 쓰기도 SYNC 완료까지 기다린다(non-posted). SB DSP처럼 쓰기 순서와 타이밍에
     민감한 장치를 고려한 선택이다. 성능이 문제되면 나중에 posted write를 검토한다.
4. IDE 데이터 포트(1F0h/170h) 32-bit 경로는 그대로 두고 LPC로 보내지 않는다.
5. 내부 장치 끄기 (OSD): 외장 사운드를 켜면 `sb_cs`, `fm_cs`, `mpu_cs`
   (필요하면 `joy_cs`)를 0으로 막는다. 그러면 해당 포트가 자동으로 LPC로 넘어간다.

LPC I/O는 8-bit 전용이다. CPU의 16/32-bit I/O는 `iobus_adapter`가 이미 바이트로 나누므로
그대로 동작한다. 다만 16-bit I/O 포트를 가진 카드(AWE32 EMU8000, 620h/A20h/E20h 등)는
바이트 분할 접근에서 동작하는지 확인이 필요하다.

주의: 내부 장치가 선택하지 않은 **모든** 포트 접근이 LPC/ISA 사이클(약 1–2 µs)로 바뀐다.
DOS 드라이버의 포트 스캔이 느려질 수 있지만 실제 ISA 시스템과 같은 수준이다.
ISA 카드는 대개 주소 10비트만 디코드하므로 400h 이상의 포트는 하위 10비트 별칭으로
카드에 닿는다. 이것도 실제 하드웨어와 같은 동작이다.

### 5.3 IRQ: `src/system.sv`

`always @*`의 `interrupt[]` 조립부(`src/system.sv:1355` 부근)에서:

```
interrupt[n] = irq_n | (ext_irq[n] & ext_irq_mask[n]);
interrupt[9] = irq_9 | irq_2 | (ext_irq[2] & ext_irq_mask[2]) | (ext_irq[9] & ext_irq_mask[9]);
```

외장 SB를 켜면 내부 `sound`의 `irq_5/7/10`은 끊는다(sb/fm 칩셀렉트가 막혀 있으면 자연히 조용해진다).
`ext_irq_mask`는 OSD 선택 IRQ(5/7/10 등)로만 연다.

### 5.4 DMA: `src/soc/dma.v`

현재 채널 FSM(`src/soc/dma.v:622-640`)은 다음 흐름이다.

- `transfer==2`(read, 메모리→디바이스): 1 → 2(mem read) → 3(data valid, **ack**) → 4(update) → 5 → 0
- `transfer==1`(write, 디바이스→메모리): 1 → 6(mem write 수락, **ack**) → 4 → 5 → 0
- 디바이스 쪽 데이터는 ack 1클럭 동안 동기적으로 주고받는다.

외장 채널은 한 바이트/워드에 1–2 µs가 걸리므로 **LPC 대기 상태를 추가**한다.
채널별 `ext[ch]`(OSD의 외장 DMA 채널 마스크)가 1일 때만 새 경로를 탄다.

```
state 레지스터를 3비트 → 4비트로 확장

transfer==2 (Read, mem→dev), ext:
  1 → 2 → 3 (mem data valid) → 8 : lpc_dma_req(dir=read, ch, tc, size, data)
                                   lpc_dma_done까지 대기 → 4
transfer==1 (Write, dev→mem), ext:
  1 → 9 : lpc_dma_req(dir=write, ch, tc, size)
          lpc_dma_done까지 대기, lpc_dma_rdata 캡처 → 6 (mem write) → 4
transfer==0 (Verify), ext:
  1 → 7 → 4  (LPC 사이클 없음)
```

- `req`: `ext[ch] ? ext_drq[ch] : 내부 req`. 외장 SB를 켜면 내부 SB의
  `dma_sb_req_8/16`은 끊는다.
- **TC**: 지금은 update(state 4) 시점의 `!current_counter`로 만든다. LPC는 CHANNEL 필드에
  TC를 실어야 하므로 **state 1에서 `current_counter == 0`을 미리 계산**해 요청에 넣는다.
- 채널 0–3은 SIZE=8-bit, 채널 5–7은 SIZE=16-bit (`mem_16bit`와 같다).
- 사이클 결과(Ready/Ready More)는 `lpc_drq`에 전달해 `ext_drq` clear 여부를 정한다(4.3절).
- 8237 master(ch4 cascade)와 slave의 우선순위, 마스크, 모드 레지스터는 그대로 쓴다.
  외장 전송 중에는 해당 dma 컨트롤러가 busy로 남아서 다른 채널 요청은 그 뒤에 처리된다.
- DMA 메모리 쪽(`dma_held_*`, `src/system.sv:730-801`)은 바뀌지 않는다.

### 5.5 z486_mister.sv / OSD

- User port 소유권: LPC 모드를 켜면 `USER_OUT/USER_OE`를 `lpc_host`가 쓰고
  **MT32-pi(`mt32pi` 인스턴스)와 User port UART/MIDI는 비활성화**한다.
- 보조 SD: LPC 빌드(매크로 사용)에서는 보조 SD 슬롯을 쓸 수 없다. 이 기능은 빌드 옵션으로 둔다.
- OSD 항목(제안):
  - `ISA via User port: Off / On`
  - `External sound card: Off / SB (220h, IRQ 5, DMA 1/5) / Custom`
    → 내부 SB/OPL/MPU 끄기, `ext_irq_mask`, `ext_dma` 마스크 설정
  - `LPC clock: 28 MHz / 21 MHz`
- `status` 비트 할당은 구현할 때 정한다.

### 5.6 내부 장치와의 충돌

현재 코드 기준 내부 장치의 점유 자원(`src/system.sv:880-906`, `1355-1372`, `dma` 인스턴스):

| 자원 | 내부 점유 | 외장 카드에 쓸 수 있는 것 |
|---|---|---|
| I/O | 000–00F, 020–021, 040–043, 060–067, 061, 070–071, 080–08F, 090–09F, 0A0–0A1, 0C0–0DF, 170–177, 1F0–1F7, 201, 220–22F, 330–331, 376, 388–38B, 3B0–3DF, 3F0–3F7, 3F6, 402, 8888–8889, (Voodoo 빌드: CF8–CFF) | 나머지 전부. 3F8/2F8(COM)은 디코드만 되고 장치가 없으므로 외장 가능 |
| IRQ | 0 PIT, 1 KBD, 2→9 VGA, 5/7/10 내장 사운드, 6 FDC, 8 RTC, 12 마우스, 14/15 IDE | **3, 4, 11** 은 비어 있음. 9는 VGA IRQ2와 공유. 내장 SB를 끄면 5/7/10 |
| DMA | 1 / 5 내장 SB, 2 FDC | **0, 3, 6, 7** 은 비어 있음. 내장 SB를 끄면 1/5 |
| 메모리 | A0000–BFFFF VGA, BIOS 영역 | ISA 메모리 사이클 미지원(7절) |

충돌 형태와 대응:

1. **I/O 포트가 겹치는 경우**: subtractive decode에서는 내부 장치가 이기므로
   외장 카드는 읽기도 쓰기도 **받지 못한다**(버스 충돌이나 하드웨어 손상은 없고,
   외장 카드가 안 보이는 것뿐이다). Sound Blaster를 쓸 때는 OSD에서 외장 사운드를
   켜서 `sb_cs`, `fm_cs`(OPL 388h), `mpu_cs`(330h)를 막아야 한다. SB 카드의 게임 포트를
   쓰려면 `joy_cs`(201h)도 막는다. 내장 CMS는 220h 범위라 `sb_cs`와 함께 꺼진다.
2. **10-bit 주소 별칭**: ISA 카드는 대개 A0–A9만 디코드한다. 400h 이상의
   외부로 나가는 포트는 하위 10비트 별칭으로 카드에 도착한다(실제 하드웨어와 동일).
   예를 들어 내부가 쓰는 8888h(speedctl)는 LPC로 나가지 않으므로 별칭 문제가 없다.
3. **IRQ 공유**: z486 PIC은 에지 트리거라서, 같은 IRQ를 내부와 외부가 OR로 함께 구동하면
   한쪽이 High인 동안 다른 쪽의 에지가 사라진다. `ext_irq_mask`로 **외장 카드에 준 IRQ만** 열고,
   같은 IRQ를 쓰는 내부 장치는 반드시 끈다. OSD에서 내부 SB가 켜진 상태로
   외장 IRQ 5/7/10을 여는 조합은 막는다.
4. **DMA 채널 공유**: 같은 채널에 내부/외부 요청을 동시에 두면 전송이 섞인다.
   채널별 `ext[ch]`는 내부 요청을 대체하는 방식(OR 아님)으로 하고, ch2(FDC)는
   외장으로 지정하지 못하게 한다.
5. **ISA PnP 카드**: 설정 포트 279h(쓰기), A79h(쓰기)는 내부에서 쓰지 않는다.
   READ_DATA 포트(203h–3FFh 중 소프트웨어가 고름)가 내부 점유 포트와 겹치지 않게
   PnP 설정 도구에서 지정해야 한다(예: 20Bh는 비어 있음).
6. **SB16의 IDE/CD-ROM 인터페이스**: 170h/1F0h 계열로 설정하면 내부 IDE와 겹친다.
   비활성화하거나 1E8h/168h로 둔다.

## 6. 성능 검토

| 항목 | 값 (LCLK 28.33 MHz 기준) |
|---|---|
| I/O 사이클 LPC 오버헤드 | 약 13 LCLK ≈ 0.46 µs + ISA 사이클(브리지 대기) |
| I/O 1바이트 총 시간 | 약 1–1.5 µs (실제 ISA 8 MHz와 비슷) |
| 8-bit DMA 1전송 | 약 12 LCLK + ISA DMA 사이클 ≈ 1–1.5 µs |
| SB Pro 44.1 kHz 스테레오 8-bit | 88.2 k전송/s → 11.3 µs 간격 → LPC 점유율 약 13% |
| SB16 44.1 kHz 스테레오 16-bit (DMA5) | 88.2 k워드/s → 점유율 약 15% |
| IRQ 지연 | SERIRQ 1프레임 ≈ 2.1 µs 이하 |
| DRQ 지연 | LDRQ 메시지 5 LCLK ≈ 0.2 µs |

DSP 폴링, DMA 오토이닛, IRQ 기반 재생에 모두 충분하다.

## 7. 제외 항목과 확장

- **ISA 메모리 사이클**(C8000h–EFFFFh 옵션 ROM, NIC 공유 메모리, MDA): CPU 메모리 경로
  (`cpu_mem_valid`, valid/ready)에서 설정된 창을 LPC 메모리 사이클(CYCTYP `0100`/`0110`)로 보내고,
  L1에서 캐시하지 않게 하면 된다. 사운드카드에는 필요 없어서 제외한다.
- **Bus master DMA**: LPC/F85226 모두 지원하지 않는다.
- **외장 카드 오디오를 MiSTer로 믹싱**: 경로 없음(1절).
- **ISA PnP 카드**(SB16 PnP, AWE64): 279h / A79h / READ_DATA 포트가 내부에서 선택되지 않으므로
  그대로 LPC로 나간다. DOS에서는 CTCM, UNISOUND 등으로 설정한다.
- **16-bit DMA가 브리지에서 안 될 경우**: SB16은 16-bit 데이터를 8-bit DMA 채널(High DMA = Low DMA)로
  보낼 수 있으므로 우회할 수 있다.

## 8. 위험 요소

| 위험 | 영향 | 대응 |
|---|---|---|
| F85226 동작 세부(설정 레지스터 초기화 필요 여부, 16-bit DMA, BCLK 분주) | 기능 제한 | 데이터시트 확보, 로직 분석기로 실측 |
| 28 MHz 신호 품질 (커넥터, 크로스토크, 링잉) | 간헐 오류 | 직결 어댑터, GND 확보, OCT/전류 설정, 21 MHz 폴백, 샘플 위상 파라미터 |
| sys_top 수정 | 업스트림 머지 충돌 | 매크로로 분리, 변경 최소화 |
| User port 공유 (MT32-pi, UART, SW[1] HDMI I2S) | 동시 사용 불가 | OSD 배타 선택, 문서화 |
| SERIRQ 슬롯 극성 해석 | IRQ 반전 | 극성 파라미터 |
| Arduino 헤더 점퍼 배선(SERIRQ/LDRQ#) | 간헐 IRQ/DMA 오류 | 짧은 배선, GND 동반, 21 MHz 폴백 |
| 보조 SD 핀 재할당 | 보조 SD 사용 불가 | 매크로로 분리, OSD/문서에 명시 |
| subtractive decode로 모든 미사용 포트가 외부로 나감 | 포트 스캔이 느려짐 | 실제 ISA와 동일, 필요하면 포트 창 제한 옵션 |
| DMA 싱글 모드 요청 재발행 경쟁 | 누락 또는 중복 전송 | 완료 시 clear 규칙(4.3절), 시뮬레이션으로 검증 |

## 9. 검증 계획

### 9.1 시뮬레이션 (Verilator, `tests/lpc/`)

1. LPC 디바이스 BFM: I/O 레지스터 파일, SYNC 대기(short/long/none/error) 주입.
2. BFM에 SERIRQ 슬롯 구동, LDRQ# 메시지 생성 기능을 넣는다.
3. 테스트:
   - I/O 읽기/쓰기, 32-bit `IN`/`OUT` 바이트 분할, 타임아웃 시 `FF`
   - 내부 칩셀렉트 포트는 LPC로 나가지 않는지 (subtractive decode)
   - 8-bit DMA read/write, 오토이닛, TC 비트, Ready More
   - 16-bit DMA (ch5)
   - 싱글 모드 연속 요청에서 전송 수 = 요청 수
   - SERIRQ 프레임 해석과 `ext_irq_mask`
   - LRESET# 최소 유지 시간
4. 기존 `verilator/` 부팅 시뮬레이션 회귀: LPC 모드 Off에서 동작이 바뀌지 않는지.

### 9.2 하드웨어 bring-up

1. 스코프로 LCLK 파형과 duty, LAD/LFRAME 링잉 확인.
2. LRESET# 동작, SERIRQ start/stop frame 파형, LDRQ# 레벨 확인(점퍼 배선 포함).
3. POST 카드(80h, 5.2절의 80h 동시 쓰기 옵션 필요)를 ISA 슬롯에 꽂고 POST 코드가 보이는지 → I/O 쓰기 경로 확인.
4. OPL 검출(388h AdLib 타이머 테스트) → I/O 읽기/쓰기 확인.
5. SB DSP 리셋(2x6h) 후 2xAh = `AAh` → DSP I/O 확인.
6. DSP 명령 `F2h`(IRQ 트리거) → SERIRQ → PIC 경로 확인.
7. `DIAGNOSE` / `SBTEST`의 8-bit DMA 재생 → LDRQ → DMA 경로 확인.
8. SB16 16-bit 재생(DMA5), 게임(Doom, Duke3D, Monkey Island)으로 장시간 재생 테스트.

## 10. 구현 단계

| 단계 | 내용 | 완료 기준 |
|---|---|---|
| P0 | sys_top push-pull/OE/DDIO 옵션, 보조 SD 핀 재할당, LCLK 출력 | 스코프로 28.33 MHz 확인 |
| P1 | `lpc_host` I/O, `iobus_adapter` wait, subtractive decode | POST 카드, OPL 검출 |
| P2 | `lpc_serirq_host`, IRQ 연결 | DSP F2h IRQ |
| P3 | `lpc_ldrq_rx`, `lpc_drq`, `lpc_host` DMA 사이클, `dma.v` ext 경로 | SB 8-bit DMA 재생 |
| P4 | 16-bit DMA, OSD 정리, 문서화 | SB16 16-bit 재생 |
| (선택) P5 | ISA 메모리 사이클 | 옵션 ROM 인식 |

## 부록 A. 7핀 구성 (추가 핀을 쓸 수 없는 경우)

보조 SD 핀을 쓸 수 없으면 어댑터에 작은 CPLD(MachXO2-256/640, iCE40 UL/LP 등, 약 150 LUT)를 둔다.

- User port: LAD[3:0], LFRAME#, LCLK, **SIDEBAND**(CPLD → FPGA 입력).
- CPLD가 LRESET#를 생성한다: 전원 인가 시 약 1 ms, 호스트의 "리셋 명령"
  (LFRAME#=0, LAD=`1111`이 64 LCLK 이상 지속, 정상 abort는 4–8 LCLK라 오검출 없음),
  내부 오실레이터로 LCLK 소실(약 10 µs)을 감지했을 때.
- CPLD가 SERIRQ 호스트와 LDRQ# 디코더를 맡고, 결과를 SIDEBAND 프레임으로 보낸다.

| 비트 | 길이 | 내용 |
|---|---|---|
| IDLE | 2 | `1` |
| START | 1 | `0` |
| IRQ[15:0] | 16 | IRQ 레벨 |
| DRQ_ACT[7:0] | 8 | 채널별 마지막 LDRQ ACT 값 |
| DRQ_SEQ[7:0][1:0] | 16 | 채널별 ACT=1 메시지 수 (mod 4) |
| PARITY | 1 | 앞 40비트 짝수 패리티 |
| 합계 | 44 LCLK | 28.33 MHz에서 약 1.55 µs |

FPGA는 DRQ_SEQ가 바뀌면 `ext_drq`를 set하고, 나머지 규칙은 4.3절과 같다.
패리티 오류 프레임은 버리고, 레벨/카운터 구조라 다음 프레임에서 복구된다.

## 참고

- Intel Low Pin Count (LPC) Interface Specification, Rev 1.1
- Serialized IRQ Support for PCI Systems, Rev 6.0
- Fintek F85226 데이터시트 (LPC-to-ISA bridge)
- dISAppointment 보드 문서 (LPC 헤더 핀아웃, 전원)
- MiSTer User port 핀아웃 문서
