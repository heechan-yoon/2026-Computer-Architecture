# ⚡ [Lab 04] ALU, Register Bank, and Datapath Design

> **과목명:** 2026-1 컴퓨터구조 실습  
> **작성자:** 2022067101 윤희찬  

---

## 1. 🎯 프로젝트 목표
* ALU(Arithmetic Logic Unit)와 Accumulator가 결합된 구조를 이해하고 Verilog로 설계함.
* MUX, Decoder, Register 서브모듈을 활용하여 4-word Register Bank를 구성함.
* Register File과 ALU를 결합하여 데이터의 읽기, 연산, 쓰기가 이루어지는 미니 데이터패스(Datapath)를 구현함.
* 잘못된 코딩 스타일의 RTL을 리팩토링하여 합성 친화적이고 가독성 높은 코드로 개선함.

---

## 2. 🧱 Problem 1: ALU with Accumulator

### 📊 설계 사양
* 8가지 연산(덧셈, 뺄셈, AND, OR, XOR, NOT, 좌시프트, 우시프트)을 수행하는 ALU를 설계함.
* 조합 논리(`always @(*)`)로 연산 결과를 미리 계산하고, 순차 논리(`always @(posedge clk or negedge rst_n)`)로 `acc` 값을 갱신함.
* 비동기 active-low 리셋(`rst_n`)과 출력 `acc`가 0인지 검사하는 `zero` 플래그 기능을 포함함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module alu_acc (
    input  wire       clk,
    input  wire       rst_n,
    input  wire [7:0] b,
    input  wire [2:0] op,
    input  wire       en,
    output reg  [7:0] acc,
    output wire       zero
);

reg [7:0] alu_result;

assign zero = (acc == 8'b0);

always @(*) begin
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

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        acc <= 8'b0;
    else if (en)
        acc <= alu_result;
end

endmodule
```

<img width="752" height="148" alt="그림1" src="https://github.com/user-attachments/assets/0c87dc15-6317-41ae-9adf-75ed13e54d35" />


### ✅ 결과 요약
Problem 1에서는 ALU와 Accumulator가 통합된 구조를 설계하였다. 조합 회로부에서 ALU 연산을 수행하고, 순차 회로부에서 clock에 맞춰 연산 결과를 acc 레지스터에 업데이트하도록 구현하였다. 파형을 통해 리셋 및 각 연산 모드(ADD, SUB 등)가 정상 동작함을 확인하였다.

---

## 3. 🧱 Problem 2: Register Bank (MUX, Decoder, and Registers)

### 📊 설계 사양
* 2:4 Decoder를 통해 쓰기 주소(waddr)에 해당하는 레지스터만 선택적으로 활성화함.
* 4개의 8-bit Register를 개별 인스턴스화하여 데이터를 저장함.
* 2:1 MUX 3개를 계층적으로 연결(MUX Tree)하여 읽기 주소(raddr)에 맞는 데이터를 선택 출력함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module top_mux_dec_reg (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       we,
    input  wire [7:0] wdata,
    input  wire [1:0] waddr,
    input  wire [1:0] raddr,
    output wire [7:0] rdata
);

wire [3:0] we_vec;
wire [7:0] reg0_q, reg1_q, reg2_q, reg3_q;
wire [7:0] mux_low, mux_high;

decoder2to4 u_dec (.in(waddr), .en(we), .out(we_vec));

register8 u_reg0 (.clk(clk), .rst_n(rst_n), .we(we_vec[0]), .d(wdata), .q(reg0_q));
register8 u_reg1 (.clk(clk), .rst_n(rst_n), .we(we_vec[1]), .d(wdata), .q(reg1_q));
register8 u_reg2 (.clk(clk), .rst_n(rst_n), .we(we_vec[2]), .d(wdata), .q(reg2_q));
register8 u_reg3 (.clk(clk), .rst_n(rst_n), .we(we_vec[3]), .d(wdata), .q(reg3_q));

mux2to1 u_mux0 (.a(reg0_q), .b(reg1_q), .sel(raddr[0]), .y(mux_low));
mux2to1 u_mux1 (.a(reg2_q), .b(reg3_q), .sel(raddr[0]), .y(mux_high));
mux2to1 u_mux2 (.a(mux_low), .b(mux_high), .sel(raddr[1]), .y(rdata));

endmodule
```

<img width="752" height="166" alt="그림2" src="https://github.com/user-attachments/assets/72521384-3f25-4537-bd4f-d9f67fc301fb" />


### ✅ 결과 요약
Problem 2에서는 Decoder와 MUX를 사용하여 4개의 레지스터를 효율적으로 관리하는 Register Bank를 구현하였다. 특정 주소에만 데이터를 쓰고, 선택된 주소의 데이터만 읽어오는 논리 구조를 통해 CPU 내부 레지스터 파일의 하드웨어적 구성 원리를 파악하였다.
Problem 2에서는 enable 기능을 가진 4비트 up counter를 설계하였다.  

---

## 4. 🧱 Problem 3: Simple Datapath

### 📊 설계 사양
* Register File, ALU, MUX를 결합하여 하나의 연산 경로를 구축함.
* raddr1, raddr2 두 개의 읽기 포트를 통해 레지스터 데이터를 동시에 불러옴.
* src_b_sel 제어 신호를 이용하여 ALU의 입력값으로 레지스터 값 또는 즉시값(imm)을 선택할 수 있도록 설계함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module datapath (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       we,
    input  wire       src_b_sel,
    input  wire [1:0] raddr1,
    input  wire [1:0] raddr2,
    input  wire [1:0] waddr,
    input  wire [2:0] alu_op,
    input  wire [7:0] imm,
    output wire [7:0] alu_result,
    output wire       zero
);

wire [7:0] rdata1, rdata2, alu_b;

reg_file u_rf (
    .clk(clk), .rst_n(rst_n), .raddr1(raddr1), .raddr2(raddr2),
    .waddr(waddr), .wdata(alu_result), .we(we),
    .rdata1(rdata1), .rdata2(rdata2)
);

assign alu_b = src_b_sel ? imm : rdata2;

alu_unit u_alu (
    .a(rdata1), .b(alu_b), .op(alu_op),
    .y(alu_result), .zero(zero)
);

endmodule
```

<img width="752" height="207" alt="그림3" src="https://github.com/user-attachments/assets/2ad08d59-93a8-47c5-bd89-03d07223bbb2" />


### ✅ 결과 요약
Problem 3에서는 하위 모듈들을 통합하여 기본적인 데이터패스를 완성하였다. 레지스터에서 값을 읽어 ALU에서 연산하고, 그 결과를 다시 레지스터에 저장하는 일련의 과정이 clock에 맞춰 순차적으로 일어나는 것을 확인하였으며, 이는 프로세서 설계의 핵심 기반이 됨을 이해하였다.

---

## 5. 🧱 Problem 4: RTL Refactoring (Good Counter)

### 📊 설계 사양
* 가독성이 낮고 합성 오류 가능성이 높은 기존 코드를 업계 표준 RTL 규칙에 맞춰 리팩토링함.
* default_nettype none을 사용하여 의도치 않은 wire 선언을 방지함.
* always 문 내에서 non-blocking assignment(<=)를 사용하고, 모든 제어 경로를 명확히 하여 Latch 발생을 방지함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module good_counter (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       en,
    input  wire [1:0] mode, // 00:hold, 01:up, 10:down, 11:load
    input  wire [7:0] load_val,
    output reg  [7:0] count,
    output wire       is_zero
);

assign is_zero = (count == 8'd0);

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        count <= 8'd0;
    else if (en) begin
        if (mode == 2'b01)
            count <= count + 8'd1;
        else if (mode == 2'b10)
            count <= count - 8'd1;
        else if (mode == 2'b11)
            count <= load_val;
        else
            count <= count; // explicit hold to prevent latches
    end
end

endmodule
```

<img width="752" height="174" alt="그림4" src="https://github.com/user-attachments/assets/7f0eb3b6-44cd-42ae-af65-05eaa8d5d522" />


### ✅ 결과 요약
Problem 4를 통해 기능 구현뿐만 아니라 하드웨어 합성(Synthesis)을 고려한 코딩 스타일의 중요성을 학습하였다. 제어 신호 en과 mode에 따른 상태 전이를 명확하게 기술함으로써, 안정적인 순차 회로를 설계하는 방법을 익혔다.


---
