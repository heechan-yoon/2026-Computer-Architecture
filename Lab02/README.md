# ⚡ [Lab 02] Combinational Logic with always and case

> **과목명:** 2026-1 컴퓨터구조 실습  
> **실험 일자:** 2026.04.01  
> **작성자:** 2022067101 윤희찬  
> **환경:** Vivado 2023.2, Icarus Verilog, GTKWave

---

## 1. 🎯 프로젝트 목표
* `always @(*)` 구문을 사용하여 조합 논리 회로를 설계하는 방법을 익힘.
* `case`, `if-else if`, blocking assignment(`=`)의 사용법과 의미를 이해함.
* MUX, Priority Encoder, 7-Segment Decoder, ALU, Barrel Shifter와 같은 기본 조합회로를 구현하고 동작을 확인함.

---

## 2. 🧱 Problem 1: 4:1 Multiplexer (8-bit)

### 📊 설계 사양
* 2-bit 선택 신호 `sel`에 따라 8-bit 입력 `a`, `b`, `c`, `d` 중 하나를 출력 `y`로 전달함.
* `always @(*)`와 `case`문을 사용하여 조합 논리 MUX를 구현함.
* 조합 논리 특성에 맞게 blocking assignment를 사용함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module mux4to1 (
    input  wire [7:0] a,
    input  wire [7:0] b,
    input  wire [7:0] c,
    input  wire [7:0] d,
    input  wire [1:0] sel,
    output reg  [7:0] y
);

    always @(*)
        case (sel)
            2'b00 : y = a;
            2'b01 : y = b;
            2'b10 : y = c;
            2'b11 : y = d;
            default : y = 8'b0;
        endcase

endmodule
```
<img width="752" height="174" alt="prob1" src="https://github.com/user-attachments/assets/eb2f89fe-097a-4684-a82e-5bbe6abff44c" />

### ✅ 결과 요약
Problem 1에서는 `always @(*)`와 `case`문을 사용하여 4:1 multiplexer를 설계하였다.  
2비트 선택 신호 `sel`에 따라 8비트 입력 `a`, `b`, `c`, `d` 중 하나가 출력 `y`로 전달되도록 구현하였으며, 조합논리 특성에 맞게 blocking assignment를 사용하였다.  
시뮬레이션 결과 waveform에서 `sel` 값이 변할 때마다 `y`가 올바른 입력값으로 선택되는 것을 확인하였고, `fail_count=0`으로 모든 테스트를 통과했음을 확인하였다.

---

## 3. 🧱 Problem 2: Priority Encoder (4-bit)

### 📊 설계 사양
* 4-bit 입력 `req[3:0]`에 대해 가장 높은 우선순위의 활성 요청을 선택함.
* `req[3]`이 가장 높은 우선순위를 가짐.
* 출력 `enc`는 선택된 요청의 위치를 나타내고, `valid`는 요청이 하나라도 존재할 때 1이 되도록 설계함.
* `always @(*)`와 `if-else if` 구조를 사용하며, default assignment를 두어 latch inference를 방지함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module priority_enc (
    input  wire [3:0] req,
    output reg  [1:0] enc,
    output reg        valid
);

    always @(*) begin
        enc   = 2'd0;
        valid = 1'b0;

        if (req[3] == 1) begin
            enc = 3;
            valid = 1;
        end
        else if (req[2] == 1) begin
            enc = 2;
            valid = 1;
        end
        else if (req[1] == 1) begin
            enc = 1;
            valid = 1;
        end
        else if (req[0] == 1) begin
            enc = 0;
            valid = 1;
        end
    end

endmodule
```

<img width="752" height="161" alt="prob2" src="https://github.com/user-attachments/assets/fb748af5-cdab-4dd8-8b70-90f063112684" />


### ✅ 결과 요약
Problem 2에서는 4비트 입력 `req[3:0]`에 대해 가장 높은 우선순위의 활성 요청을 선택하는 priority encoder를 설계하였다.  
`req[3]`이 가장 높은 우선순위를 가지며, 이에 따라 출력 `enc`는 선택된 요청의 위치를 나타내고 `valid`는 요청이 하나라도 존재할 때 1이 되도록 하였다.  
구현은 `always @(*)`와 `if-else if` 구조를 사용하였으며, 블록 시작 부분에 `enc=0`, `valid=0`의 default assignment를 두어 latch inference를 방지하였다.  
그 결과 입력 요청들 중 가장 높은 우선순위의 비트가 정상적으로 인코딩되도록 설계할 수 있었다.

---

## 4. 🧱 Problem 3: BCD to 7-Segment Decoder

### 📊 설계 사양
* 4-bit BCD 입력(`0~9`)을 7-segment 출력 패턴으로 변환함.
* `seg[6:0] = g,f,e,d,c,b,a` 순서를 따름.
* Active-high 방식을 사용함.
* `always @(*)`와 `case`문으로 각 숫자에 해당하는 패턴을 지정함.
* `0~9` 이외의 입력은 `default`를 사용하여 모든 세그먼트를 끄도록 처리함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module seven_seg (
    input  wire [3:0] bcd,
    output reg  [6:0] seg
);

    always @(*) begin
        case (bcd)
            4'd0:    seg = 7'b0111111;
            4'd1:    seg = 7'b0000110;
            4'd2:    seg = 7'b1011011;
            4'd3:    seg = 7'b1001111;
            4'd4:    seg = 7'b1100110;
            4'd5:    seg = 7'b1101101;
            4'd6:    seg = 7'b1111101;
            4'd7:    seg = 7'b0000111;
            4'd8:    seg = 7'b1111111;
            4'd9:    seg = 7'b1101111;
            default: seg = 7'b0000000;
        endcase
    end

endmodule
```

<img width="752" height="134" alt="prob3" src="https://github.com/user-attachments/assets/abac086d-7a5a-4fda-847b-c251dfb0abbc" />


### ✅ 결과 요약
Problem 3에서는 4비트 BCD 입력(`0~9`)을 7-segment 출력 패턴으로 변환하는 decoder를 설계하였다.  
문제 조건에 따라 `seg[6:0]=g,f,e,d,c,b,a` 순서와 active-high 방식을 사용하였으며, `always @(*)`와 `case`문으로 각 숫자에 해당하는 세그먼트 패턴을 지정하였다.  
예를 들어 0은 `0111111`, 1은 `0000110`, 8은 `1111111`로 표현하였다.  
또한 `0~9` 이외의 입력에 대해서는 `default`를 사용하여 모든 세그먼트를 끄도록 하였다.  
이를 통해 7-segment decoder의 조합논리 구현 방법을 확인할 수 있었다.

---

## 5. 🧱 Problem 4: Simple ALU (8-bit)

### 📊 설계 사양
* 3-bit `op` 신호에 따라 8-bit 입력 `a`, `b`에 대해 산술 및 논리 연산을 수행함.
* `always @(*)`와 `case`문을 사용하여 연산을 구현함.
* 출력 결과는 `y`로 나타내며, `zero` 신호는 `assign`문을 통해 `y==0`일 때 1이 되도록 정의함.

| op | 연산 | 설명 |
| :---: | :---: | :--- |
| **000** | `a + b` | 덧셈 |
| **001** | `a - b` | 뺄셈 |
| **010** | `a & b` | AND |
| **011** | `a \| b` | OR |
| **100** | `a ^ b` | XOR |
| **101** | `~a` | NOT A |
| **110** | `a << 1` | 왼쪽 시프트 |
| **111** | `a >> 1` | 오른쪽 시프트 |

### 💻 Implementation (Code)
```verilog
`default_nettype none

module alu (
    input  wire [7:0] a,
    input  wire [7:0] b,
    input  wire [2:0] op,
    output reg  [7:0] y,
    output wire       zero
);

    always @(*) begin
        case (op)
            3'b000 : y = a + b;
            3'b001 : y = a - b;
            3'b010 : y = a & b;
            3'b011 : y = a | b;
            3'b100 : y = a ^ b;
            3'b101 : y = ~a;
            3'b110 : y = a << 1;
            3'b111 : y = a >> 1;
            default: y = 8'b0;
        endcase
    end

    assign zero = (y == 8'b0);

endmodule
```

<img width="752" height="154" alt="prob4" src="https://github.com/user-attachments/assets/002ef1f8-6642-43ac-95c3-7680eb086b68" />


### ✅ 결과 요약
Problem 4에서는 3비트 `op` 신호에 따라 8비트 입력 `a`, `b`에 대해 산술 및 논리 연산을 수행하는 간단한 ALU를 설계하였다.  
`always @(*)`와 `case`문을 사용하여 덧셈, 뺄셈, AND, OR, XOR, NOT, 왼쪽 시프트, 오른쪽 시프트 연산을 각각 구현하였고, 출력 결과는 `y`로 나타내었다.  
또한 `zero` 신호는 `assign`문을 통해 `y`가 0일 때 1이 되도록 정의하였다.  
이를 통해 연산 선택 신호에 따라 다양한 기본 연산을 수행하는 조합논리 ALU의 동작을 확인할 수 있었다.

---

## 6. 🧱 Problem 5: Barrel Shifter (8-bit Left Rotate)

### 📊 설계 사양
* 8-bit 입력 `data`를 지정된 비트 수만큼 왼쪽으로 회전시키는 barrel shifter를 설계함.
* 입력 `shamt`에 따라 회전할 비트 수를 결정함.
* `case`문과 비트 연결(concatenation)을 사용하여 각 경우의 출력을 구현함.
* `shamt=0`인 경우에는 입력을 그대로 출력하도록 하여 회전이 없는 경우도 처리함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module barrel_shifter (
    input  wire [7:0] data,
    input  wire [2:0] shamt,
    output reg  [7:0] y
);

    always @(*) begin
        case (shamt)
            3'd1: y = {data[6:0], data[7]};
            3'd2: y = {data[5:0], data[7:6]};
            3'd3: y = {data[4:0], data[7:5]};
            3'd4: y = {data[3:0], data[7:4]};
            3'd5: y = {data[2:0], data[7:3]};
            3'd6: y = {data[1:0], data[7:2]};
            3'd7: y = {data[0],   data[7:1]};
            default: y = data;
        endcase
    end

endmodule
```

<img width="752" height="163" alt="prob5" src="https://github.com/user-attachments/assets/4ab0c9ff-2870-42f6-bcee-27a0638d3f13" />


### ✅ 결과 요약
Problem 5에서는 8비트 입력 `data`를 지정된 비트 수만큼 왼쪽으로 회전시키는 barrel shifter를 설계하였다.  
입력 `shamt`에 따라 회전할 비트 수를 결정하며, `case`문과 비트 연결(concatenation)을 사용하여 각 경우의 출력을 구현하였다.  
예를 들어 `shamt=1`일 때는 `{data[6:0], data[7]}`로 1비트 left rotate를 수행하고, `shamt=2`일 때는 `{data[5:0], data[7:6]}`로 2비트 left rotate를 수행한다.  
`shamt=0`인 경우에는 입력을 그대로 출력하도록 하여 회전이 없는 경우도 처리하였다.  
이를 통해 barrel shifter의 기본 동작과 비트 재배열 방식의 조합논리 구현을 확인할 수 있었다.
