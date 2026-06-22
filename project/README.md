# ⚙️ Computer Architecture Final Project: Full Hazard RISC-V Processor

> **과목명:** 2026-1 컴퓨터구조 실습  
> **프로젝트 주제:** 5-stage Pipelined RISC-V 32-bit Processor 설계  
> **구현 Level:** Full Hazard  
> **작성자:** 2022067101 윤희찬  
> **사용 언어:** SystemVerilog  
> **사용 툴:** Vivado  

---

## 1. 🎯 프로젝트 목표

본 프로젝트의 목표는 5-stage pipelined RISC-V 32-bit processor를 SystemVerilog로 구현하고, pipeline 구조에서 발생하는 **data hazard**와 **control hazard**를 모두 해결하는 것이다.

기본적인 processor는 다음 5개의 stage로 구성된다.

```text
IF → ID → EXE → MEM → WB
```

각 stage의 역할은 다음과 같다.

| Stage | 이름 | 주요 역할 |
| :--- | :--- | :--- |
| IF | Instruction Fetch | PC를 이용하여 instruction memory에서 명령어를 fetch하고 PC+4 계산 |
| ID | Instruction Decode | instruction decode, register file read, immediate generation, control signal 생성 |
| EXE | Execute | ALU 연산, branch compare, branch/jump target address 계산 |
| MEM | Memory Access | load/store 명령어의 data memory 접근 |
| WB | Write Back | ALU result, memory data, PC+4 중 register file에 쓸 최종 결과 선택 |

이번 제출본은 **Full hazard processor**로, 다음 두 가지 hazard를 모두 처리한다.

* **Data hazard:** forwarding과 load-use stall을 통해 해결
* **Control hazard:** branch/jump 발생 시 잘못 fetch/decode된 instruction을 flush하여 해결

---

## 2. 📁 제출 ZIP 파일 구성

제출한 ZIP 파일의 전체 구조는 다음과 같다.

```text
2022067101_윤희찬_프로젝트/
├─ 2022067101_윤희찬_프로젝트 레포트.pdf
├─ clk.xdc
├─ data.mem
├─ instr.mem
├─ tb_riscv.sv
└─ full_hazard/
   ├─ alu.sv
   ├─ branch_compare.sv
   ├─ control_unit.sv
   ├─ data_mem.sv
   ├─ EXE.sv
   ├─ hazard_unit.sv
   ├─ ID.sv
   ├─ IF.sv
   ├─ imm_gen.sv
   ├─ instr_mem.sv
   ├─ load_aligner.sv
   ├─ MEM.sv
   ├─ REG_EXE_MEM.sv
   ├─ reg_file.sv
   ├─ REG_ID_EXE.sv
   ├─ REG_IF_ID.sv
   ├─ REG_MEM_WB.sv
   ├─ riscv.sv
   ├─ store_aligner.sv
   └─ WB.sv
```

---

## 3. 📌 전체 프로젝트 요약

이번 프로젝트에서는 RV32I subset을 기반으로 하는 5-stage pipelined RISC-V processor를 구현하였다.

기본 processor datapath는 instruction fetch, instruction decode, execute, memory access, write back 단계로 나누어 구성하였다. 각 stage 사이에는 pipeline register를 배치하여 한 cycle마다 instruction이 다음 stage로 이동하도록 하였다.

또한 pipeline 구조에서 발생하는 hazard를 해결하기 위해 `hazard_unit.sv`를 중심으로 forwarding, stall, flush logic을 구현하였다.

```text
Full Hazard Processor
├─ Standard 5-stage pipeline 동작
├─ Data hazard 해결
│  ├─ EX/MEM forwarding
│  ├─ MEM/WB forwarding
│  └─ Load-use stall
│
└─ Control hazard 해결
   ├─ PCSrc_E 발생 시 PC target address로 갱신
   ├─ IF/ID register flush
   └─ ID/EX register flush
```

---

## 4. 🧱 주요 구현 내용

### 4.1 Standard Processor 동작

기본 processor는 다음 흐름으로 동작한다.

```text
PC_F
 ↓
Instruction Memory
 ↓
IF/ID Register
 ↓
Instruction Decode + Register File + Immediate Generator
 ↓
ID/EX Register
 ↓
ALU + Branch Compare + Target Address Calculation
 ↓
EXE/MEM Register
 ↓
Data Memory + Load/Store Alignment
 ↓
MEM/WB Register
 ↓
Write Back Mux
 ↓
Register File
```

기본 명령어 실행을 위해 다음 기능을 구현하였다.

* R-type ALU instruction
* I-type ALU immediate instruction
* Load instruction
* Store instruction
* Branch instruction
* `jal`
* `jalr`
* `lui`
* `auipc`

---

### 4.2 Data Hazard 해결

Data hazard는 이전 instruction의 결과가 아직 register file에 writeback되기 전에 다음 instruction이 그 값을 필요로 할 때 발생한다.

이를 해결하기 위해 다음 두 가지 방식을 구현하였다.

#### 1) Forwarding

EXE stage에서 필요한 operand가 아직 WB stage까지 도달하지 않은 경우, EX/MEM stage 또는 MEM/WB stage의 결과를 ALU 입력으로 직접 전달한다.

```text
EX/MEM forwarding
MEM/WB forwarding
```

`hazard_unit.sv`에서는 source register와 destination register의 번호를 비교하여 forwarding 여부를 결정한다.

```systemverilog
if ((Rs1_E != 5'd0) && (Rs1_E == Rd_M) && RegWrite_M)
    ForwardA_E = FwdMem;
else if ((Rs1_E != 5'd0) && (Rs1_E == Rd_W) && RegWrite_W)
    ForwardA_E = FwdWb;
```

`Rs2_E`에 대해서도 동일한 방식으로 `ForwardB_E`를 생성한다.

#### 2) Load-use Stall

`lw` instruction 바로 다음 instruction이 load 결과를 사용하는 경우에는 forwarding만으로 해결할 수 없다.

이 경우 IF stage와 ID stage를 1 cycle 정지시키고, ID/EX register에 NOP을 넣어 bubble을 삽입한다.

```systemverilog
assign LoadUseStall = MemRead_E &&
                      (Rd_E != 5'd0) &&
                      ((Rd_E == Rs1_D) || (Rd_E == Rs2_D));

assign Stall_F = LoadUseStall;
assign Stall_D = LoadUseStall;
assign Flush_E = LoadUseStall || PCSrc_E;
```

---

### 4.3 Control Hazard 해결

Control hazard는 branch 또는 jump instruction에서 다음 PC가 아직 확정되지 않았는데 pipeline이 이미 다음 instruction을 fetch/decode했을 때 발생한다.

본 구현에서는 branch/jump target이 EXE stage에서 결정되므로, `PCSrc_E`가 1이 되면 다음 동작을 수행한다.

```text
1. PC_F를 PCTarget_E로 갱신
2. IF/ID register를 flush하여 잘못 fetch된 instruction 제거
3. ID/EX register를 flush하여 잘못 decode된 control signal 제거
```

핵심 control signal은 다음과 같다.

```systemverilog
assign Flush_D = PCSrc_E;
assign Flush_E = LoadUseStall || PCSrc_E;
```

`REG_IF_ID.sv`에서는 flush가 발생하면 instruction을 NOP으로 바꾼다.

```systemverilog
else if (Flush_D) begin
    PC_D <= 32'b0;
    PCPlus4_D <= 32'b0;
    Instr_D <= NOP;
end
```

`REG_ID_EXE.sv`에서는 flush가 발생하면 control signal을 모두 비활성화하여 잘못된 instruction이 이후 stage에 영향을 주지 않도록 한다.

---

## 5. 📄 최상위 파일 설명

| 파일명 | 설명 |
| :--- | :--- |
| `2022067101_윤희찬_프로젝트 레포트.pdf` | 프로젝트 보고서 파일. 구현 level을 Full로 명시하고, IF/ID/EXE/MEM/WB stage별 핵심 waveform 설명, testbench 결과, synthesis 결과를 정리한 문서 |
| `clk.xdc` | Vivado synthesis에서 사용하는 constraint 파일. clock period를 설정하여 timing analysis를 수행할 수 있도록 함 |
| `data.mem` | data memory 초기화 파일. simulation 및 synthesis에서 load/store test에 사용되는 RAM 초기값 포함 |
| `instr.mem` | instruction memory 초기화 파일. processor가 실행할 RISC-V machine code를 포함 |
| `tb_riscv.sv` | 전체 processor 검증용 testbench. BASE, DATA_HAZARD, CONTROL_HAZARD, FULL_HAZARD section을 검사하고 memory signature 값을 golden value와 비교 |

---

## 6. 📂 `full_hazard/` 폴더 파일 설명

### 6.1 Top Module

| 파일명 | 설명 |
| :--- | :--- |
| `riscv.sv` | 전체 5-stage pipelined RISC-V processor의 top module. IF, ID, EXE, MEM, WB stage와 pipeline register, hazard unit을 모두 연결함 |

`riscv.sv`는 전체 datapath를 연결하는 중심 파일이다.

주요 연결 구조는 다음과 같다.

```text
IF
 ↓
REG_IF_ID
 ↓
ID
 ↓
REG_ID_EXE
 ↓
EXE
 ↓
REG_EXE_MEM
 ↓
MEM
 ↓
REG_MEM_WB
 ↓
WB
```

또한 `hazard_unit`에서 생성한 forwarding, stall, flush signal을 각 stage와 pipeline register에 전달한다.

---

### 6.2 IF Stage 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `IF.sv` | Instruction Fetch stage 구현. PC register를 관리하고, instruction memory에서 instruction을 읽으며, PC+4를 계산함 |
| `instr_mem.sv` | instruction memory 구현. `instr.mem` 파일을 `$readmemh`로 읽어오고, PC address에 해당하는 instruction을 async read 방식으로 출력함 |
| `REG_IF_ID.sv` | IF/ID pipeline register. IF stage의 `PC_F`, `PCPlus4_F`, `Instr_F`를 ID stage로 전달하며, stall 및 flush 기능을 포함함 |

`IF.sv`의 핵심 동작은 다음과 같다.

```systemverilog
if (!rst_n)
    PC_F <= 32'b0;
else if (!Stall_F)
    PC_F <= PCSrc_E ? PCTarget_E : PCPlus4_F;
```

즉, stall이 없을 때만 PC를 갱신하고, branch/jump가 발생하면 sequential PC가 아니라 target address로 이동한다.

---

### 6.3 ID Stage 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `ID.sv` | Instruction Decode stage 구현. instruction field 추출, control unit, register file, immediate generator를 연결함 |
| `control_unit.sv` | instruction opcode, funct3, funct7을 해석하여 control signal을 생성하는 조합논리 모듈 |
| `reg_file.sv` | RISC-V 32개 general-purpose register 구현. 2-read, 1-write 구조이며 x0 register는 항상 0으로 유지 |
| `imm_gen.sv` | instruction format에 따라 immediate 값을 32-bit로 sign-extension 또는 zero-extension하는 모듈 |
| `REG_ID_EXE.sv` | ID/EXE pipeline register. ID stage의 operand, immediate, register 번호, control signal을 EXE stage로 전달하며 flush 시 NOP 동작이 되도록 control signal을 초기화함 |

`ID.sv`에서는 instruction에서 register field를 다음과 같이 추출한다.

```systemverilog
assign Rs1_D    = Instr_D[19:15];
assign Rs2_D    = Instr_D[24:20];
assign Rd_D     = Instr_D[11:7];
assign Funct3_D = Instr_D[14:12];
```

`reg_file.sv`는 writeback과 decode stage read가 같은 cycle에 겹치는 경우 writeback data를 바로 read할 수 있도록 처리한다.

---

### 6.4 EXE Stage 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `EXE.sv` | Execute stage 구현. ALU operand 선택, forwarding mux, ALU 연산, branch compare, PC target address 계산, PCSrc 생성 |
| `alu.sv` | ALU 구현. add, sub, and, or, xor, shift, slt, sltu 등의 연산 수행 |
| `branch_compare.sv` | branch instruction의 taken 여부를 판단하는 비교 모듈. beq, bne, blt, bge, bltu, bgeu 지원 |
| `REG_EXE_MEM.sv` | EXE/MEM pipeline register. ALU result, store data, PC+4, destination register, memory control signal 등을 MEM stage로 전달함 |

`EXE.sv`는 forwarding mux를 통해 ALU operand를 선택한다.

```systemverilog
case (ForwardA_E)
    FwdMem:  SrcAForwarded = ForwardResult_M;
    FwdWb:   SrcAForwarded = Result_W;
    default: SrcAForwarded = RD1_E;
endcase
```

branch/jump target address는 다음과 같이 계산한다.

```systemverilog
if (Jalr_E)
    PCTarget_E = (SrcAForwarded + ImmExt_E) & 32'hffff_fffe;
else
    PCTarget_E = PC_E + ImmExt_E;
```

`jalr`의 경우 RISC-V 규칙에 따라 target address의 bit 0을 0으로 clear한다.

---

### 6.5 MEM Stage 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `MEM.sv` | Memory Access stage 구현. store aligner, data memory, load aligner를 연결하여 load/store 동작을 수행함 |
| `data_mem.sv` | byte-addressable data memory 구현. `data.mem` 파일로 초기화되며, async read와 sync write 구조를 사용함 |
| `store_aligner.sv` | `sb`, `sh`, `sw` store instruction에 맞게 write data와 byte strobe를 생성함 |
| `load_aligner.sv` | `lb`, `lh`, `lw`, `lbu`, `lhu` load instruction에 맞게 memory read data를 sign-extension 또는 zero-extension함 |
| `REG_MEM_WB.sv` | MEM/WB pipeline register. ALU result, memory read data, PC+4, destination register, writeback control signal을 WB stage로 전달함 |

`MEM.sv`에서는 store data를 먼저 byte lane에 맞춰 정렬한 뒤 data memory에 저장한다.

```text
WriteData_M
 ↓
store_aligner
 ↓
data_mem
 ↓
load_aligner
 ↓
ReadData_M
```

load instruction에서는 `funct3` 값에 따라 byte, halfword, word 단위 read를 처리한다.

---

### 6.6 WB Stage 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `WB.sv` | Write Back stage 구현. `ResultSrc_W`에 따라 ALU result, memory read data, PC+4 중 register file에 writeback할 최종 값을 선택함 |

`WB.sv`의 핵심 동작은 다음과 같다.

```systemverilog
case (ResultSrc_W)
    ResultMem:     Result_W = ReadData_W;
    ResultPcPlus4: Result_W = PCPlus4_W;
    default:       Result_W = ALUResult_W;
endcase
```

instruction 종류에 따라 writeback data는 다음과 같이 선택된다.

| Instruction 종류 | Writeback source |
| :--- | :--- |
| R-type / I-type ALU | `ALUResult_W` |
| Load | `ReadData_W` |
| `jal` / `jalr` | `PCPlus4_W` |
| Store / Branch | register writeback 없음 |

---

### 6.7 Hazard 관련 파일

| 파일명 | 설명 |
| :--- | :--- |
| `hazard_unit.sv` | Full hazard 해결의 핵심 모듈. forwarding signal, load-use stall, branch/jump flush signal을 생성함 |

`hazard_unit.sv`에서 생성하는 주요 signal은 다음과 같다.

| Signal | 역할 |
| :--- | :--- |
| `ForwardA_E` | EXE stage ALU operand A forwarding 선택 |
| `ForwardB_E` | EXE stage ALU operand B forwarding 선택 |
| `Stall_F` | load-use hazard 발생 시 IF stage PC 갱신 정지 |
| `Stall_D` | load-use hazard 발생 시 IF/ID register 갱신 정지 |
| `Flush_D` | branch/jump taken 시 IF/ID register flush |
| `Flush_E` | load-use stall 또는 branch/jump taken 시 ID/EXE register flush |

Full hazard 구현의 핵심은 data hazard와 control hazard를 동시에 고려하는 것이다.

```systemverilog
assign Stall_F = LoadUseStall;
assign Stall_D = LoadUseStall;
assign Flush_D = PCSrc_E;
assign Flush_E = LoadUseStall || PCSrc_E;
```

이 구조를 통해 load-use 상황에서는 pipeline을 1 cycle 멈추고 bubble을 삽입하며, branch/jump 상황에서는 잘못 들어온 instruction을 제거한다.

---

## 7. 🧪 Testbench 설명

`tb_riscv.sv`는 processor 전체 동작을 검증하기 위한 integrated testbench이다.

검증 흐름은 다음과 같다.

```text
1. instr.mem에 저장된 RISC-V machine code를 instruction memory에 load
2. data.mem을 data memory에 load
3. processor 실행
4. 특정 DONE address에 DONE_MAGIC 값이 저장될 때까지 대기
5. data memory signature 영역의 결과값 확인
6. expected value와 비교
7. PASS / FAIL 출력
```

Testbench에서 사용하는 DONE 조건은 다음과 같다.

```systemverilog
DONE_MAGIC = 32'hcafe_babe
DONE_ADDR  = 32'h07fc
```

검증 section은 다음 네 가지로 구성된다.

| Section | 검증 내용 |
| :--- | :--- |
| `BASE` | 기본 RV32I instruction 동작 검증 |
| `DATA_HAZARD` | forwarding, load-use stall, load-to-store forwarding 등 data hazard 처리 검증 |
| `CONTROL_HAZARD` | branch/jump wrong-path flush 및 target 도달 여부 검증 |
| `FULL_HAZARD` | data hazard와 control hazard가 함께 발생하는 복합 상황 검증 |

Full hazard 제출본에서는 모든 section이 required test로 동작한다.

```systemverilog
if (!$value$plusargs("RV_VERSION=%s", rv_version)) begin
    rv_version = "full_hazard";
end
```

---

## 8. 🧪 Testbench 주요 검증 항목

### 8.1 BASE Section

BASE section에서는 기본 instruction 동작을 확인한다.

대표적인 검증 항목은 다음과 같다.

| 검증 항목 | 의미 |
| :--- | :--- |
| `add positive/negative` | add 연산 검증 |
| `sub signed operands` | sub 연산 검증 |
| `and/or/xor pattern` | bitwise logic 연산 검증 |
| `sll/srl/sra` | shift 연산 검증 |
| `slt/sltu` | signed/unsigned 비교 검증 |
| `lb/lh/lw/lbu/lhu` | load align 및 sign/zero extension 검증 |
| `sb/sh/sw` | store align 및 byte strobe 검증 |
| `beq/bne/blt/bge/bltu/bgeu` | branch instruction 검증 |
| `jal/jalr` | jump 및 link address 검증 |

---

### 8.2 DATA_HAZARD Section

DATA_HAZARD section에서는 forwarding과 stall이 올바르게 동작하는지 확인한다.

대표적인 검증 항목은 다음과 같다.

| 검증 항목 | 의미 |
| :--- | :--- |
| `EX/MEM forwarding` | 직전 instruction 결과를 EXE stage로 forwarding |
| `dual-source forwarding` | ALU operand A/B가 동시에 forwarding되는 상황 검증 |
| `forwarding priority` | EX/MEM forwarding이 MEM/WB forwarding보다 우선되는지 확인 |
| `MEM/WB forwarding` | WB stage 결과 forwarding 검증 |
| `load-use stall` | load 바로 다음 instruction이 load 결과를 사용할 때 stall 발생 |
| `load-to-store data` | load 결과를 store data로 사용하는 상황 검증 |
| `store address forwarding` | store address 계산에 forwarding이 필요한 상황 검증 |

---

### 8.3 CONTROL_HAZARD Section

CONTROL_HAZARD section에서는 branch/jump로 인해 잘못 fetch된 instruction이 flush되는지 확인한다.

대표적인 검증 항목은 다음과 같다.

| 검증 항목 | 의미 |
| :--- | :--- |
| `beq wrong-path flush` | beq taken 시 wrong-path instruction 제거 |
| `bne wrong-path flush` | bne taken 시 wrong-path instruction 제거 |
| `blt/bge/bltu/bgeu target marker` | branch target에 정상 도달했는지 확인 |
| `jal wrong-path flush` | jal 이후 잘못 fetch된 instruction 제거 |
| `jalr wrong-path flush` | jalr 이후 잘못 fetch된 instruction 제거 |

---

### 8.4 FULL_HAZARD Section

FULL_HAZARD section에서는 data hazard와 control hazard가 함께 발생하는 복합 상황을 검증한다.

| 검증 항목 | 의미 |
| :--- | :--- |
| `fwd branch + flush` | forwarding된 값을 branch compare에 사용하고, branch taken 시 flush 수행 |
| `branch target reached` | branch target instruction이 정상적으로 실행되는지 확인 |
| `load-use mixed` | load-use stall 이후 연산 결과가 정상인지 확인 |
| `load-to-store mixed` | load 결과를 store data로 사용하는 복합 상황 검증 |
| `load-to-branch stall+flush` | load 결과를 branch compare에 사용해야 하는 경우 stall과 flush가 함께 필요한 상황 검증 |
| `branch rs2 forwarding+flush` | branch의 두 번째 source operand에 forwarding이 필요한 상황에서 flush까지 정상 수행되는지 확인 |

---

## 9. 📈 Stage별 핵심 Waveform 분석

### 9.1 IF Stage Waveform: PC Update

<img width="838" height="142" alt="if" src="https://github.com/user-attachments/assets/49925446-c708-4298-8c70-553656b7d8be" />

이 waveform은 IF stage의 핵심 동작인 **PC update**를 보여주는 사진이다.

IF stage에서는 현재 `PC_F`를 이용하여 instruction memory에서 명령어를 읽고, 동시에 다음 순차 주소인 `PCPlus4_F = PC_F + 4`를 계산한다.

waveform에서 `PCSrc_E`가 1일 때, branch 또는 jump target address인 `PCTarget_E`가 `0x000013fc`로 설정되어 있다. 이에 따라 `PC_F`는 기존 순차 흐름대로 `0x00001404` 이후 `PCPlus4_F = 0x00001408`로 진행하지 않고, target address인 `0x000013fc`로 변경된다.

```text
PCSrc_E = 0 → PC_F = PCPlus4_F
PCSrc_E = 1 → PC_F = PCTarget_E
```

이 동작을 IF stage의 핵심 동작으로 본 이유는 processor가 다음에 어떤 instruction을 fetch할지를 PC가 결정하기 때문이다.

RISC-V에서 일반 instruction은 `PC + 4` 순서로 실행되지만, branch, `jal`, `jalr`과 같은 control flow instruction은 다음 instruction 주소가 순차 주소가 아니라 target address가 되어야 한다. 따라서 `PCSrc_E`가 1일 때 `PC_F`를 `PCTarget_E`로 갱신하는 동작은 branch/jump 명령어를 올바르게 실행하기 위해 반드시 필요하다.

또한 pipelined processor에서는 잘못된 PC로 instruction을 fetch하면 이후 stage에 잘못된 명령어가 전달되므로, PC를 target address로 정확히 갱신하는 것은 control hazard 처리와도 직접적으로 관련된 중요한 동작이다.

---

### 9.2 ID Stage Waveform: Instruction Decode

<img width="752" height="254" alt="id" src="https://github.com/user-attachments/assets/9b1b0a8a-fd86-4800-b91e-d911d95a12f2" />

이 waveform은 ID stage에서 instruction decode가 수행되는 과정을 보여주는 사진이다.

waveform에서 `Instr_D`가 `0x00500093`일 때, instruction field로부터 다음 값들이 추출된다.

| Signal | Value | 의미 |
| :--- | :--- | :--- |
| `Instr_D` | `0x00500093` | 현재 decode 중인 instruction |
| `Rs1_D` | `0x00` | source register 1 = x0 |
| `Rd_D` | `0x01` | destination register = x1 |
| `ImmExt_D` | `0x00000005` | sign-extended immediate |
| `RD1_D` | `0x00000000` | x0에서 읽은 값 |
| `ALUSrc_D` | `1` | ALU 두 번째 operand로 immediate 사용 |
| `RegWrite_D` | `1` | register file writeback 수행 |
| `ALUControl_D` | `0` | add 연산 |

해당 instruction은 다음과 같은 I-type ALU immediate 명령어로 해석할 수 있다.

```assembly
addi x1, x0, 5
```

따라서 register file은 `Rs1_D = x0`에 해당하는 값을 읽어 `RD1_D = 0x00000000`을 출력한다. 또한 ALU의 두 번째 operand로 immediate를 사용해야 하므로 `ALUSrc_D`가 1로 설정되고, 연산 결과를 destination register에 저장해야 하므로 `RegWrite_D`가 1로 설정된다.

이 동작을 ID stage의 핵심 동작으로 본 이유는 ID stage가 instruction을 해석하여 source register, destination register, immediate, control signal을 생성하는 단계이기 때문이다.

RISC-V processor에서는 instruction 종류에 따라 ALU 입력 선택, memory 접근 여부, register writeback 여부가 달라지므로, ID stage에서 생성된 control signal은 이후 EXE, MEM, WB stage의 동작을 결정하는 기준이 된다.

---

### 9.3 EXE Stage Waveform: ALU Operation

<img width="752" height="252" alt="exe" src="https://github.com/user-attachments/assets/a353b9d5-2680-4dc4-80c8-f69aa2aae833" />

이 waveform은 EXE stage에서 ALU 연산이 수행되는 과정을 보여주는 사진이다.

waveform에서 `RD1_E`는 `0x00000000`이고, `ImmExt_E`는 `0x00000005`이다. `ALUSrc_E`가 1이므로 ALU의 두 번째 operand는 `RD2_E`가 아니라 immediate 값인 `ImmExt_E`로 선택된다.

또한 `ALUControl_E`가 0으로 설정되어 add 연산이 수행되며, 그 결과 `ALUResult_E`가 `0x00000005`로 출력된다.

| Signal | Value | 의미 |
| :--- | :--- | :--- |
| `RD1_E` | `0x00000000` | ALU 첫 번째 operand |
| `ImmExt_E` | `0x00000005` | immediate 값 |
| `ALUSrc_E` | `1` | 두 번째 operand로 immediate 선택 |
| `ALUControl_E` | `0` | add 연산 |
| `ALUResult_E` | `0x00000005` | ALU 연산 결과 |

이는 다음 명령어가 EXE stage에서 실제로 실행되는 동작을 보여준다.

```assembly
addi x1, x0, 5
```

이 동작을 EXE stage의 핵심 동작으로 본 이유는 EXE stage가 ID stage에서 전달된 operand와 control signal을 바탕으로 실제 산술/논리 연산을 수행하는 단계이기 때문이다.

RISC-V processor에서 R-type/I-type ALU instruction의 결과, load/store의 address 계산, branch/jump target address 계산은 모두 EXE stage를 통해 만들어진다. 따라서 ALU 입력 선택과 `ALUResult_E` 생성은 EXE stage의 핵심 동작이다.

---

### 9.4 MEM Stage Waveform: Load Instruction Memory Access

<img width="752" height="223" alt="mem" src="https://github.com/user-attachments/assets/d713ab97-e7a3-4a40-b2fc-0ceddd1363b1" />

이 waveform은 MEM stage에서 load instruction이 data memory에 접근하는 과정을 보여주는 사진이다.

waveform에서 `ALUResult_M`은 `0x00000220`이며, 이 값이 data memory의 address로 사용된다. `MemRead_M`이 1이고 `MemWrite_M`이 0이므로 현재 instruction은 memory read를 수행하는 load instruction이다.

또한 `Funct3_M`은 2로, word 단위 load인 `lw` 동작을 의미한다. 따라서 data memory에서 읽은 `raw_read_data`는 `0x12345678`이고, word load에서는 별도의 sign extension이나 byte selection 없이 `aligned_read_data`도 `0x12345678`로 유지된다. 최종적으로 `ReadData_M`이 `0x12345678`로 출력되는 것을 확인할 수 있다.

| Signal | Value | 의미 |
| :--- | :--- | :--- |
| `ALUResult_M` | `0x00000220` | data memory address |
| `MemRead_M` | `1` | memory read 활성화 |
| `MemWrite_M` | `0` | memory write 비활성화 |
| `Funct3_M` | `2` | `lw` instruction |
| `raw_read_data` | `0x12345678` | data memory에서 읽은 원본 data |
| `aligned_read_data` | `0x12345678` | load aligner 통과 후 data |
| `ReadData_M` | `0x12345678` | MEM stage 최종 read data |

이 동작을 MEM stage의 핵심 동작으로 본 이유는 MEM stage가 load/store instruction에서 실제 data memory 접근을 수행하는 단계이기 때문이다.

RISC-V processor에서 ALU는 memory address를 계산하고, MEM stage는 그 address를 이용하여 memory read 또는 write를 수행한다. 또한 `lb`, `lh`, `lw`, `lbu`, `lhu`와 같이 load 크기와 sign extension 방식이 instruction마다 다르므로, MEM stage에서 load aligner를 통해 최종 `ReadData_M`을 올바르게 생성하는 동작이 중요하다.

---

### 9.5 WB Stage Waveform: Register Writeback Data Selection

<img width="752" height="170" alt="wb" src="https://github.com/user-attachments/assets/4610a987-6629-46f5-be8d-c593ba520f83" />

이 waveform은 WB stage에서 register file에 writeback할 최종 data가 선택되는 과정을 보여주는 사진이다.

waveform에서 `ResultSrc_W`는 1로 설정되어 있으며, 이는 writeback source로 memory read data를 선택한다는 의미이다. 이에 따라 `ReadData_W = 0x12345678`이 `Result_W = 0x12345678`로 선택된다.

또한 `RegWrite_W`가 1이므로 선택된 `Result_W` 값은 destination register인 `Rd_W = 0x0b`에 writeback된다.

| Signal | Value | 의미 |
| :--- | :--- | :--- |
| `ResultSrc_W` | `1` | memory read data를 writeback source로 선택 |
| `ReadData_W` | `0x12345678` | memory에서 읽은 data |
| `Result_W` | `0x12345678` | register file에 writeback될 최종 data |
| `RegWrite_W` | `1` | register writeback 활성화 |
| `Rd_W` | `0x0b` | destination register |

이 동작을 WB stage의 핵심 동작으로 본 이유는 WB stage가 앞선 stage에서 만들어진 결과 중 실제 register file에 저장할 최종 값을 결정하는 단계이기 때문이다.

RISC-V processor에서는 instruction 종류에 따라 writeback data가 달라진다.

| Instruction 종류 | Writeback data |
| :--- | :--- |
| R-type ALU instruction | `ALUResult_W` |
| I-type ALU instruction | `ALUResult_W` |
| Load instruction | `ReadData_W` |
| `jal` / `jalr` | `PCPlus4_W` |
| Store / Branch instruction | writeback 없음 |

따라서 `ResultSrc_W`에 따라 올바른 값을 `Result_W`로 선택하고, `RegWrite_W`가 활성화되었을 때 destination register에 기록하는 동작은 WB stage의 핵심 동작이다.

---

## 10. 🧪 Testbench 결과

### 10.1 BASE Test Result

<img width="535" height="446" alt="base" src="https://github.com/user-attachments/assets/44d034e3-adbc-4193-80b3-191c38185299" />

BASE section에서는 기본 RISC-V instruction이 정상적으로 실행되는지 확인하였다.

검증 항목에는 add, sub, and, or, xor, shift, set-less-than, immediate 연산, load/store, branch, jump instruction이 포함된다.

```text
[PASS] BASE 50/50
```

BASE test가 통과했다는 것은 hazard 처리 이전에 기본 5-stage RISC-V processor의 instruction 실행 기능이 정상적으로 구현되었음을 의미한다.

---

### 10.2 DATA_HAZARD Test Result

<img width="535" height="294" alt="data" src="https://github.com/user-attachments/assets/e3df5c3d-ab09-47db-8a1a-9565869b17a0" />

DATA_HAZARD section에서는 forwarding과 load-use stall이 정상적으로 동작하는지 확인하였다.

주요 검증 항목은 다음과 같다.

| 항목 | 의미 |
| :--- | :--- |
| `EX/MEM forwarding` | 직전 instruction 결과를 EXE stage로 forwarding |
| `dual-source forwarding` | ALU 두 operand가 동시에 forwarding되는 상황 검증 |
| `forwarding priority` | EX/MEM forwarding이 MEM/WB forwarding보다 우선되는지 확인 |
| `MEM/WB forwarding` | WB stage 결과 forwarding 검증 |
| `load-use stall` | load 바로 다음 instruction이 load 결과를 사용하는 경우 stall 발생 |
| `load-to-store data` | load 결과를 store data로 사용하는 상황 검증 |
| `store address forwarding` | store address 계산에 forwarding이 필요한 상황 검증 |

```text
[PASS] DATA_HAZARD 10/10
```

이 결과는 data hazard 상황에서 forwarding과 stall이 모두 정상적으로 동작했음을 의미한다.

---

### 10.3 CONTROL_HAZARD Test Result

<img width="557" height="561" alt="control" src="https://github.com/user-attachments/assets/1812a6d6-2f00-4500-b449-1bd2b7732a4b" />

CONTROL_HAZARD section에서는 branch/jump instruction으로 인해 잘못 fetch/decode된 instruction이 flush되는지 확인하였다.

주요 검증 항목은 다음과 같다.

| 항목 | 의미 |
| :--- | :--- |
| `beq wrong-path flush` | beq taken 시 wrong-path instruction 제거 |
| `bne wrong-path flush` | bne taken 시 wrong-path instruction 제거 |
| `blt/bge/bltu/bgeu target marker` | branch target에 정상적으로 도달하는지 확인 |
| `jal wrong-path flush` | jal 이후 잘못 fetch된 instruction 제거 |
| `jalr wrong-path flush` | jalr 이후 잘못 fetch된 instruction 제거 |

```text
[PASS] CONTROL_HAZARD 22/22
```

이 결과는 branch/jump 발생 시 IF/ID 및 ID/EXE register flush가 정상적으로 수행되어 control hazard가 해결되었음을 의미한다.

---

### 10.4 FULL_HAZARD Test Result

<img width="557" height="197" alt="full" src="https://github.com/user-attachments/assets/5167f7cd-0633-4d9f-958b-657e0a90879a" />

FULL_HAZARD section에서는 data hazard와 control hazard가 동시에 발생하는 복합 상황을 검증하였다.

주요 검증 항목은 다음과 같다.

| 항목 | 의미 |
| :--- | :--- |
| `fwd branch + flush` | forwarding된 값을 branch compare에 사용하고 branch taken 시 flush 수행 |
| `branch target reached` | branch target instruction이 정상적으로 실행되는지 확인 |
| `load-use mixed` | load-use stall 이후 연산 결과가 정상인지 확인 |
| `load-to-store mixed` | load 결과를 store data로 사용하는 상황 검증 |
| `load-to-branch stall+flush` | load 결과를 branch compare에 사용하는 상황에서 stall과 flush가 함께 필요한 경우 검증 |
| `branch rs2 forwarding+flush` | branch의 두 번째 source operand에 forwarding이 필요한 상황에서 flush까지 정상 수행되는지 확인 |

```text
[PASS] FULL_HAZARD 6/6
[PASS] tb_riscv RV_VERSION=full_hazard required=4/4 xfail=0
```

이 결과는 최종 제출본이 data hazard와 control hazard를 모두 해결하는 Full hazard processor로 정상 동작했음을 보여준다.

---

## 11. 🛠 Synthesis 결과

### 11.1 LUT Slice Utilization

Synthesis 결과, Slice Logic 사용량은 다음과 같이 나타났다.

| Resource | Used | Available | Utilization |
| :--- | :--- | :--- | :--- |
| Slice LUTs | 22731 | 47200 | 48.16% |
| LUT as Logic | 22731 | 47200 | 48.16% |
| LUT as Memory | 0 | 19000 | 0.00% |
| Slice Registers | 17988 | 94400 | 19.06% |
| Register as Flip Flop | 17988 | 94400 | 19.06% |
| Register as Latch | 0 | 94400 | 0.00% |
| F7 Muxes | 2544 | 31700 | 8.03% |
| F8 Muxes | 1080 | 15850 | 6.81% |

이 결과를 통해 구현한 Full hazard processor가 Vivado synthesis에서 실제 hardware resource로 변환 가능함을 확인하였다.

---

### 11.2 Timing Report

Timing report에서 max delay path와 min delay path의 slack은 다음과 같이 나타났다.

| Timing Path | Slack | 의미 |
| :--- | :--- | :--- |
| Max Delay Path | `11.462ns` | setup timing 만족 |
| Min Delay Path | `0.132ns` | hold timing 만족 |

Max delay path의 slack이 양수이므로 setup time violation이 발생하지 않았다.  
Min delay path의 slack도 양수이므로 hold time violation이 발생하지 않았다.

따라서 본 설계는 주어진 clock constraint 조건에서 timing requirement를 만족하며, synthesizable한 Full hazard RISC-V processor로 확인된다.

---

## 12. 📊 보고서 반영 요약

본 마크다운 문서는 제출한 프로젝트 보고서의 핵심 파형과 설명을 기반으로 구성하였다.

| 구분 | 첨부할 이미지 | 설명 내용 |
| :--- | :--- | :--- |
| IF stage | `./images/if-stage-waveform.png` | `PCSrc_E`에 따른 `PC_F` 갱신 및 branch/jump target 반영 |
| ID stage | `./images/id-stage-waveform.png` | `Instr_D = 0x00500093`에서 register field, immediate, control signal 생성 |
| EXE stage | `./images/exe-stage-waveform.png` | `ALUSrc_E = 1`일 때 immediate를 operand로 선택하고 `ALUResult_E = 5` 생성 |
| MEM stage | `./images/mem-stage-waveform.png` | `ALUResult_M = 0x220` 주소에서 `0x12345678` load |
| WB stage | `./images/wb-stage-waveform.png` | `ResultSrc_W = 1`에 따라 `ReadData_W`를 `Result_W`로 선택 |
| Testbench | `./images/full-hazard-test-result.png` | BASE, DATA_HAZARD, CONTROL_HAZARD, FULL_HAZARD all pass |
| Synthesis | `./images/synthesis-lut-slice.png`, `./images/synthesis-timing-report.png` | LUT Slice 사용량 및 max/min delay slack 확인 |

---

## 13. 📌 제출 파일별 한 줄 요약

| 경로 | 한 줄 설명 |
| :--- | :--- |
| `2022067101_윤희찬_프로젝트 레포트.pdf` | Full hazard 구현 내용을 stage별 waveform, simulation, synthesis 결과와 함께 정리한 보고서 |
| `clk.xdc` | Vivado synthesis를 위한 clock constraint |
| `data.mem` | data memory 초기값 및 signature 영역 검증용 memory file |
| `instr.mem` | processor가 실행할 RISC-V machine code가 저장된 instruction memory file |
| `tb_riscv.sv` | BASE/DATA/CONTROL/FULL hazard 검증을 수행하는 integrated testbench |
| `full_hazard/riscv.sv` | 전체 processor top module |
| `full_hazard/IF.sv` | PC update 및 instruction fetch stage |
| `full_hazard/instr_mem.sv` | instruction memory |
| `full_hazard/REG_IF_ID.sv` | IF/ID pipeline register, stall/flush 지원 |
| `full_hazard/ID.sv` | decode stage top module |
| `full_hazard/control_unit.sv` | instruction decode 및 control signal 생성 |
| `full_hazard/reg_file.sv` | 32개 register file, x0 write 방지 및 bypass read 지원 |
| `full_hazard/imm_gen.sv` | I/S/B/U/J-type immediate 생성 |
| `full_hazard/REG_ID_EXE.sv` | ID/EXE pipeline register, flush 시 control signal 제거 |
| `full_hazard/EXE.sv` | ALU, forwarding mux, branch compare, target address 계산 |
| `full_hazard/alu.sv` | 산술/논리 연산 수행 |
| `full_hazard/branch_compare.sv` | branch taken 여부 판단 |
| `full_hazard/REG_EXE_MEM.sv` | EXE/MEM pipeline register |
| `full_hazard/MEM.sv` | memory access stage top module |
| `full_hazard/data_mem.sv` | byte-addressable data memory |
| `full_hazard/store_aligner.sv` | sb/sh/sw store data alignment |
| `full_hazard/load_aligner.sv` | lb/lh/lw/lbu/lhu load data alignment |
| `full_hazard/REG_MEM_WB.sv` | MEM/WB pipeline register |
| `full_hazard/WB.sv` | writeback result mux |
| `full_hazard/hazard_unit.sv` | forwarding, load-use stall, branch/jump flush를 담당하는 full hazard 핵심 모듈 |

---

## 14. ✅ 최종 결과 정리

본 제출본은 Full hazard processor 구현을 목표로 작성되었다.

구현 결과는 다음과 같이 정리할 수 있다.

* 5-stage pipelined RISC-V processor 구조 구현
* IF, ID, EXE, MEM, WB stage 구현
* stage 사이 pipeline register 구현
* RV32I 주요 instruction decode 및 실행 지원
* register file 구현
* immediate generation 구현
* ALU 및 branch compare 구현
* load/store aligner 구현
* data memory 및 instruction memory 구현
* writeback mux 구현
* forwarding logic 구현
* load-use stall 구현
* branch/jump flush 구현
* full hazard 상황 검증용 testbench 포함
* Vivado synthesis용 clock constraint 포함
* 프로젝트 보고서 PDF 포함

---

## 15. 🔍 프로젝트 의의

이번 프로젝트를 통해 단순한 single-cycle processor가 아니라 실제 pipeline 구조에서 발생하는 문제를 직접 해결하는 경험을 하였다.

특히 pipeline processor에서는 각 instruction이 동시에 서로 다른 stage에 존재하기 때문에, 단순히 stage별 기능만 구현해서는 올바른 실행 결과를 보장할 수 없다.

Data hazard를 해결하기 위해서는 register writeback 이전의 값을 EXE stage로 forwarding해야 하며, load-use 상황에서는 1 cycle stall을 삽입해야 한다. 또한 branch/jump instruction에서는 target address가 결정되기 전 잘못 fetch/decode된 instruction을 flush해야 한다.

최종적으로 본 구현은 forwarding, stall, flush를 모두 포함하여 data hazard와 control hazard를 함께 처리하는 Full hazard RISC-V processor로 구성되었다.

---
