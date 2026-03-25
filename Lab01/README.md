# ⚡ [Lab 01] Combinational Logic with assign

> **과목명:** 2026-1 컴퓨터구조 실습
> **실험 일자:** 2026.03.25 (Week 3)
> **작성자:** 2022067101 윤희찬
> **환경:** Vivado 2023.2, Icarus Verilog, GTKWave

---

## 1. 🎯 프로젝트 목표
* `wire`와 `assign` 구문을 사용하여 기본적인 조합 논리 회로 설계 능력을 배양함.
* Bitwise, Arithmetic, Relational, Reduction 연산자의 활용법 및 하드웨어적 의미를 이해함.
* Ternary operator(`? :`)와 `parameter`를 사용하여 효율적이고 재사용 가능한 모듈을 설계함.

---

## 2. 🧱 Problem 1: Half Adder (1-bit)

### 📊 설계 사양
* 1-bit 입력 2개를 더하여 Sum과 Carry out을 산출함.
* **논리식:** $sum = a \oplus b$, $cout = a \cdot b$

### 💻 Implementation (Code)
```
`default_nettype none
module HalfAdder(
    input  wire a,
    input  wire b,
    output wire sum,
    output wire cout
);

    // TODO: Implement half adder using assign statements
    // sum  = a XOR b
    // cout = a AND b
    assign sum = a ^ b;
    assign cout = a & b;

endmodule
```
<img width="747" height="435" alt="ca_lab1_p1" src="https://github.com/user-attachments/assets/1381c0e0-cb06-4e70-a1a9-b707fae20688" />

---

## 3. 🧱 Problem 2: Full Adder (1-bit)

### 📊 설계 사양
* 하위 비트의 Carry in($cin$)을 포함하여 3개의 비트를 연산함.
* **논리식:** $sum = a \oplus b \oplus cin, cout = (a \cdot b) + (cin \cdot (a \oplus b))$

### 💻 Implementation (Code)
```
`default_nettype none

module fulladder (
    input  wire a,
    input  wire b,
    input  wire cin,
    output wire sum,
    output wire cout
);
    assign sum = a ^ b ^ cin;
    assign cout = (a & b) | (cin & (a ^ b));
endmodule
```
<img width="1570" height="916" alt="ca_lab1_p2" src="https://github.com/user-attachments/assets/a98981fd-99ea-4678-b257-a94d6dbd19d9" />

---

## 4. 🧱 Problem 3: 8-bit Bitwise ALU

### 📊 설계 사양
* 2-bit op 신호에 따라 4가지(AND, OR, XOR, NOT) 비트 연산을 수행함.
* 중첩 삼항 연산자를 사용하여 MUX 구조를 설계함.

| op | 연산 | 설명 |
| :---: | :---: | :--- |
| **00** | `a & b` | AND 연산 |
| **01** | `a \| b` | OR 연산 |
| **10** | `a ^ b` | XOR 연산 |
| **11** | `~a` | NOT A 연산 |

### 💻 Implementation (Code)
```
`default_nettype none

module alu8bit (
    input  wire [7:0] a,
    input  wire [7:0] b,
    input  wire [1:0] op,
    output wire [7:0] y
);
    assign y = (op==2'b00) ? a&b:
    (op==2'b01) ? (a|b):
    (op==2'b10) ? (a^b): ~a;
endmodule
```
<img width="1571" height="692" alt="ca_lab1_p3" src="https://github.com/user-attachments/assets/9b10b3a2-9107-4031-956b-6716328ddd37" />

---

## 5. 🧱 Problem 4: Zero and Sign Detector

### 📊 설계 사양
* 8-bit 데이터를 2의 보수로 해석하여 $is\_zero, is\_neg, is\_pos$ 상태를 판별함.
* Reduction OR 연산자를 통해 모든 비트가 0인지 효율적으로 확인함.

### 💻 Implementation (Code)
```
`default_nettype none
module detector (
    input  wire [7:0] data,
    output wire       is_zero,
    output wire       is_neg,
    output wire       is_pos
);

    assign is_zero = (data == 8'b0);
    assign is_neg  = (data[7] == 1'b1);
    assign is_pos  = (data != 8'b0 && data[7] == 1'b0);

endmodule
```
<img width="1571" height="916" alt="ca_lab1_p4" src="https://github.com/user-attachments/assets/e57ea0a4-da5e-46c9-b65e-0a69a9c2dab9" />

---

## 6. 🧱 Problem 5: Parameterized Adder

### 📊 설계 사양
* parameter를 사용하여 비트 폭을 조절 가능한 가산기를 설계함 (Default: 8-bit).
* Concatenation({ })을 사용하여 Carry out과 Sum을 동시에 처리함.

### 💻 Implementation (Code)
```
`default_nettype none

module param_adder #(
    parameter WIDTH = 8
)(
    input  wire [WIDTH-1:0] a,
    input  wire [WIDTH-1:0] b,
    input  wire             cin,
    output wire [WIDTH-1:0] sum,
    output wire             cout
);
    assign {cout, sum}=a+b+cin;
    // TODO: Implement parameterized adder using assign
    // Hint: Use concatenation on the left-hand side
    //       {cout, sum} = a + b + cin;

endmodule
```
<img width="1547" height="582" alt="ca_lab1_p5" src="https://github.com/user-attachments/assets/d33bae3b-338d-40bf-ae71-a9f6726fb935" />

