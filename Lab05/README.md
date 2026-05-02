# ⚡ [Lab 05] SystemVerilog Refactoring and Datapath Design

> **과목명:** 2026-1 컴퓨터구조 실습  
> **작성자:** 2022067101 윤희찬  

---

## 1. 🎯 프로젝트 목표
* 기존 Verilog RTL 코드를 SystemVerilog 문법으로 변환하고, 설계 의도를 더 명확하게 표현함.
* `logic`, `always_comb`, `always_ff`를 사용하여 조합 논리와 순차 논리를 구조적으로 분리함.
* `default_nettype none`을 적용하여 선언되지 않은 신호가 자동으로 wire로 생성되는 실수를 방지함.
* named port instantiation을 사용하여 하위 모듈 연결의 가독성과 안정성을 높임.
* ALU, Register Bank, Datapath, Counter 구조를 SystemVerilog 스타일로 구현하고 시뮬레이션을 통해 동작을 확인함.

---

## 2. 🧱 Problem 1: ALU with Accumulator

### 📊 설계 사양
* 8가지 연산(덧셈, 뺄셈, AND, OR, XOR, NOT, 좌시프트, 우시프트)을 수행하는 ALU와 Accumulator를 SystemVerilog로 구현함.
* 기존 Verilog의 `wire`, `reg` 구분 대신 `logic` 자료형을 사용하여 포트와 내부 신호를 선언함.
* ALU 연산은 `always_comb` 블록에서 수행하여 조합 논리임을 명확히 함.
* Accumulator 갱신은 `always_ff @(posedge clk or negedge rst_n)` 블록에서 수행하여 플립플롭 기반의 순차 논리임을 명확히 함.
* `default_nettype none`을 사용하여 암시적 net 생성을 방지하고, 코드 마지막에 `default_nettype wire`로 되돌림.

### 💻 Implementation (Code)
```systemverilog
`default_nettype none

module alu_acc (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] b,
    input  logic [2:0] op,
    input  logic       en,
    output logic [7:0] acc,
    output logic       zero
);

    logic [7:0] alu_result;

    // Combinational Logic: ALU
    always_comb begin
        alu_result = acc;

        case (op)
            3'b000:  alu_result = acc + b;
            3'b001:  alu_result = acc - b;
            3'b010:  alu_result = acc & b;
            3'b011:  alu_result = acc | b;
            3'b100:  alu_result = acc ^ b;
            3'b101:  alu_result = ~acc;
            3'b110:  alu_result = acc << 1;
            3'b111:  alu_result = acc >> 1;
            default: alu_result = acc;
        endcase
    end

    // Sequential Logic: Accumulator Register
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            acc <= 8'd0;
        end
        else if (en) begin
            acc <= alu_result;
        end
    end

    assign zero = (acc == 8'd0);

endmodule

`default_nettype wire
```

<img width="752" height="150" alt="그림1" src="https://github.com/user-attachments/assets/4aa097cc-8932-4dab-8d96-0f9591331f04" />


### ✅ 결과 요약
Problem 1에서는 기존 Verilog로 작성된 ALU with Accumulator를 SystemVerilog 문법으로 변환하였다. `logic` 자료형을 사용하여 `wire`와 `reg`를 구분해야 하는 번거로움을 줄였고, ALU 연산은 `always_comb`, Accumulator 저장 동작은 `always_ff`로 분리하였다. 이를 통해 조합 논리와 순차 논리의 역할이 명확해졌으며, 클럭 상승 에지에서 `en`이 활성화되었을 때만 `acc` 값이 갱신되는 것을 확인하였다. 또한 `zero` 신호는 `acc` 값이 0일 때 정상적으로 활성화되어 ALU 결과 상태를 올바르게 나타냈다.

---

## 3. 🧱 Problem 2: Register Bank Using MUX, Decoder, and Registers

### 📊 설계 사양
* 4개의 8-bit register를 이용하여 간단한 Register Bank를 구현함.
* `waddr`와 `we` 신호를 2:4 Decoder에 입력하여 one-hot write enable 신호 `wen[3:0]`을 생성함.
* 선택된 하나의 register만 `wdata`를 저장하도록 설계함.
* 3개의 2:1 MUX로 구성된 MUX tree를 사용하여 `raddr`에 해당하는 register 값을 `rdata`로 출력함.
* 하위 모듈 연결 시 named port 방식을 사용하여 포트 연결 오류를 줄임.

### 💻 Implementation (Code)
```systemverilog
`default_nettype none

module top_mux_dec_reg (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] wdata,
    input  logic [1:0] waddr,
    input  logic       we,
    input  logic [1:0] raddr,
    output logic [7:0] rdata
);

    logic [3:0] wen;

    logic [7:0] q0;
    logic [7:0] q1;
    logic [7:0] q2;
    logic [7:0] q3;

    logic [7:0] mux01_out;
    logic [7:0] mux23_out;

    // Write address decoder
    decoder2to4 u_decoder2to4 (
        .in  (waddr),
        .en  (we),
        .out (wen)
    );

    // Four 8-bit registers
    register8 u_reg0 (
        .clk   (clk),
        .rst_n (rst_n),
        .we    (wen[0]),
        .d     (wdata),
        .q     (q0)
    );

    register8 u_reg1 (
        .clk   (clk),
        .rst_n (rst_n),
        .we    (wen[1]),
        .d     (wdata),
        .q     (q1)
    );

    register8 u_reg2 (
        .clk   (clk),
        .rst_n (rst_n),
        .we    (wen[2]),
        .d     (wdata),
        .q     (q2)
    );

    register8 u_reg3 (
        .clk   (clk),
        .rst_n (rst_n),
        .we    (wen[3]),
        .d     (wdata),
        .q     (q3)
    );

    // Read MUX tree
    mux2to1 u_mux01 (
        .a   (q0),
        .b   (q1),
        .sel (raddr[0]),
        .y   (mux01_out)
    );

    mux2to1 u_mux23 (
        .a   (q2),
        .b   (q3),
        .sel (raddr[0]),
        .y   (mux23_out)
    );

    mux2to1 u_mux_final (
        .a   (mux01_out),
        .b   (mux23_out),
        .sel (raddr[1]),
        .y   (rdata)
    );

endmodule

`default_nettype wire
```

<img width="752" height="206" alt="그림2" src="https://github.com/user-attachments/assets/ffdcc782-f2ab-4a4b-9a2a-c1f7b82ead8b" />


### ✅ 결과 요약
Problem 2에서는 Decoder, Register, MUX를 조합하여 4개의 8-bit register를 갖는 Register Bank를 구현하였다. 쓰기 동작에서는 `decoder2to4`가 `waddr`를 해석하여 하나의 register만 선택적으로 활성화하고, 읽기 동작에서는 MUX tree가 `raddr`에 맞는 register 값을 선택하여 `rdata`로 출력한다. 기존 Verilog 코드와 비교했을 때, SystemVerilog 버전은 대부분의 신호를 `logic`으로 선언할 수 있어 코드가 더 간결해졌으며, named port instantiation을 사용하여 각 신호가 어떤 포트에 연결되는지 쉽게 확인할 수 있었다. 따라서 동일한 기능을 수행하면서도 가독성과 유지보수성이 더 높은 구조로 구현되었다.

---

## 4. 🧱 Problem 3: Simple Datapath

### 📊 설계 사양
* Register File, MUX, ALU를 결합하여 간단한 Datapath를 구성함.
* Register File에서 두 개의 데이터를 동시에 읽고, ALU 연산 결과를 다시 Register File에 write-back함.
* `src_b_sel` 제어 신호를 사용하여 ALU의 B 입력으로 register 값 또는 immediate 값을 선택함.
* ALU 결과가 0이면 `zero` 신호가 활성화되도록 설계함.
* 하위 모듈을 named port 방식으로 연결하여 datapath 내부 신호 흐름을 명확히 표현함.

### 💻 Implementation (Code)
```systemverilog
`default_nettype none

module datapath (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [1:0] raddr1,
    input  logic [1:0] raddr2,
    input  logic [1:0] waddr,
    input  logic       we,
    input  logic [2:0] alu_op,
    input  logic [7:0] imm,
    input  logic       src_b_sel,
    output logic [7:0] alu_result,
    output logic       zero
);

    logic [7:0] rdata1;
    logic [7:0] rdata2;
    logic [7:0] alu_b;
    logic [7:0] alu_y;

    // Register file
    reg_file u_reg_file (
        .clk    (clk),
        .rst_n  (rst_n),
        .raddr1 (raddr1),
        .raddr2 (raddr2),
        .waddr  (waddr),
        .wdata  (alu_y),
        .we     (we),
        .rdata1 (rdata1),
        .rdata2 (rdata2)
    );

    // Select ALU input B
    mux2to1_8bit u_src_b_mux (
        .a   (rdata2),
        .b   (imm),
        .sel (src_b_sel),
        .y   (alu_b)
    );

    // ALU
    alu_unit u_alu_unit (
        .a    (rdata1),
        .b    (alu_b),
        .op   (alu_op),
        .y    (alu_y),
        .zero (zero)
    );

    assign alu_result = alu_y;

endmodule

`default_nettype wire
```

<img width="752" height="228" alt="그림3" src="https://github.com/user-attachments/assets/c7679e6c-1049-4564-b517-e6f7e24ce0d2" />


### ✅ 결과 요약
Problem 3에서는 Register File에서 데이터를 읽고, MUX를 통해 ALU 입력을 선택한 뒤, ALU 연산 결과를 다시 Register File에 저장하는 Simple Datapath를 구현하였다. 시뮬레이션 결과 `raddr1`로 선택된 값이 ALU의 A 입력으로 전달되고, `src_b_sel` 값에 따라 `raddr2`의 register 값 또는 `imm` 값이 ALU의 B 입력으로 정상 선택되는 것을 확인하였다. 또한 `alu_op`에 따라 `alu_result`가 올바르게 변화하였고, ALU 결과가 0이 되는 경우 `zero` 신호가 1로 활성화되었다. SystemVerilog의 `logic`, `always_comb`, `always_ff`, named port 방식을 사용함으로써 기존 Verilog 구현보다 설계 의도가 더 명확하고 포트 연결 오류를 줄일 수 있는 구조가 되었다.

---

## 5. 🧱 Problem 4: RTL Refactoring (Good Counter)

### 📊 설계 사양
* 기존 `bad_counter.v`의 기능을 유지하면서 SystemVerilog 스타일로 리팩토링함.
* CamelCase 형태의 신호 이름을 `clk`, `rst_n`, `en`, `load_val`과 같은 snake_case 형식으로 수정함.
* active-low reset 신호에는 `_n` 접미사를 사용하여 신호 의미를 명확히 함.
* `always_comb`에서 `next_count`를 계산하고, `always_ff`에서 `count`를 갱신하여 조합 논리와 순차 논리를 분리함.
* `next_count = count`라는 기본값을 먼저 지정하여 hold 동작을 명확히 하고 latch inference를 방지함.

### 💻 Implementation (Code)
```systemverilog
`default_nettype none

module good_counter (
    input  logic       clk,
    input  logic       rst_n,
    input  logic       en,
    input  logic [1:0] mode,      // 00=hold, 01=up, 10=down, 11=load
    input  logic [7:0] load_val,
    output logic [7:0] count,
    output logic       is_zero
);

    logic [7:0] next_count;

    // Combinational logic: next-state generation
    always_comb begin
        next_count = count;   // default: hold, prevents latch inference

        if (en) begin
            case (mode)
                2'b00:   next_count = count;          // hold
                2'b01:   next_count = count + 8'd1;   // up
                2'b10:   next_count = count - 8'd1;   // down
                2'b11:   next_count = load_val;       // load
                default: next_count = count;
            endcase
        end
    end

    // Sequential logic: counter register
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            count <= 8'd0;
        end
        else begin
            count <= next_count;
        end
    end

    // Output logic
    assign is_zero = (count == 8'd0);

endmodule

`default_nettype wire
```

<img width="752" height="193" alt="그림4" src="https://github.com/user-attachments/assets/40868e70-ec5e-43e4-8dac-1571c270881b" />


### ✅ 결과 요약
Problem 4에서는 기존의 좋지 않은 counter 코드를 SystemVerilog 스타일로 리팩토링하였다. 시뮬레이션 파형에서 reset 이후 `mode=01`일 때 `count`가 0에서 1, 2, 3, 4, 5로 증가하고, `mode=10`일 때 4, 3, 2, 1, 0으로 감소하는 것을 확인하였다. 또한 `mode=11`에서는 `load_val` 값이 `count`에 저장되었으며, `count`가 0이 되는 구간에서 `is_zero`가 활성화되어 zero flag도 정상적으로 동작하였다. 이 구현은 `always_comb`와 `always_ff`를 사용하여 next-state logic과 register update logic을 분리했고, 기본 hold 값을 명시하여 latch 발생 가능성을 줄였다. 따라서 기존 Verilog 코드보다 설계 의도가 분명하고, SystemVerilog의 코딩 스타일을 적용하여 가독성과 유지보수성을 높인 구현이라고 볼 수 있다.

---
