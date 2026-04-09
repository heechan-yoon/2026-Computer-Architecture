# ⚡ [Lab 03] Sequential Logic with Flip-Flops, Counters, and Shift Registers

> **과목명:** 2026-1 컴퓨터구조 실습  
> **작성자:** 2022067101 윤희찬  

---

## 1. 🎯 프로젝트 목표
* `always @(posedge clk or negedge rst_n)` 구문을 사용하여 순차 논리 회로를 설계하는 방법을 익힘.
* non-blocking assignment(`<=`)의 사용 이유와 동작 방식을 이해함.
* active-low 비동기 reset, enable, load, up/down 제어와 같은 순차회로의 기본 기능을 구현하고 확인함.
* D flip-flop, counter, shift register와 같은 대표적인 순차회로의 동작을 파형으로 검증함.

---

## 2. 🧱 Problem 1: D Flip-Flop with Active-Low Asynchronous Reset

### 📊 설계 사양
* active-low 비동기 reset을 가지는 1-bit D flip-flop을 설계함.
* `rst_n=0`이면 clock과 관계없이 출력 `q`가 즉시 0으로 초기화됨.
* `rst_n=1`인 정상 상태에서는 clock의 상승엣지에서만 입력 `d`가 출력 `q`에 저장됨.
* 순차 논리 회로이므로 non-blocking assignment(`<=`)를 사용함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module dff (
    input  wire clk,
    input  wire rst_n,
    input  wire d,
    output reg  q
);

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        q <= 1'b0;
    else
        q <= d;

endmodule
```

<img width="752" height="144" alt="그림1" src="https://github.com/user-attachments/assets/458f2dd5-7ca9-4a96-943f-98a87131eccb" />

### ✅ 결과 요약
Problem 1에서는 active-low 비동기 리셋을 가지는 D flip-flop을 설계하였다.  
`rst_n`이 0이 되는 순간 clock과 관계없이 출력 `q`가 즉시 0으로 초기화되며, `rst_n`이 1인 정상 상태에서는 clock의 상승엣지에서만 입력 `d`의 값이 출력 `q`로 저장되도록 구현하였다.  
그 외의 시간에는 이전 값이 그대로 유지되므로, 이 회로는 데이터를 clock 타이밍에 맞추어 1비트 저장하면서도 필요 시 리셋 신호로 즉시 초기 상태로 되돌릴 수 있는 가장 기본적인 순차회로 소자임을 확인할 수 있었다.

---

## 3. 🧱 Problem 2: 4-bit Up Counter with Enable

### 📊 설계 사양
* enable 기능을 가진 4-bit up counter를 설계함.
* `rst_n=0`이면 비동기적으로 카운터 값 `count`가 즉시 0으로 초기화됨.
* 리셋이 해제된 상태에서는 clock의 상승엣지마다 동작함.
* `en=1`이면 현재 `count` 값이 1 증가하고, `en=0`이면 값을 그대로 유지함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module counter (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       en,
    output reg [3:0]  count
);

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        count <= 0;
    else if (en)
        count <= count + 1;

endmodule
```

<img width="752" height="139" alt="그림2" src="https://github.com/user-attachments/assets/3637e2dc-7787-4919-8747-1cc7a681767e" />

### ✅ 결과 요약
Problem 2에서는 enable 기능을 가진 4비트 up counter를 설계하였다.  
`rst_n`이 0이 되면 비동기적으로 카운터 값 `count`가 즉시 0으로 초기화되고, 리셋이 해제된 상태에서는 clock의 상승엣지마다 동작하도록 하였다.  
또한 `en`이 1이면 현재 `count` 값이 1 증가하고, `en`이 0이면 값을 변화시키지 않고 그대로 유지하도록 구현하였다.  
따라서 이 회로는 clock에 동기되어 0부터 15까지 순차적으로 증가하는 4비트 카운터이면서, enable 신호를 통해 카운트 진행 여부를 제어할 수 있는 기본적인 순차회로임을 확인할 수 있었다.

---

## 4. 🧱 Problem 3: 8-bit Shift Register (Serial-In, Parallel-Out)

### 📊 설계 사양
* serial input을 받아 병렬 데이터로 변환하는 8-bit shift register를 설계함.
* `rst_n=0`이면 비동기적으로 `parallel_out`이 모두 0으로 초기화됨.
* 리셋이 해제된 후에는 clock의 상승엣지마다 기존 값이 한 비트씩 왼쪽으로 이동함.
* 새로운 입력 `serial_in`은 최하위 비트에 저장되도록 구현함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module shift_reg (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       serial_in,
    output reg [7:0]  parallel_out
);

always @(posedge clk or negedge rst_n)
    if (!rst_n)
        parallel_out <= 8'b00000000;
    else
        parallel_out <= {parallel_out[6:0], serial_in};

endmodule
```

<img width="752" height="99" alt="그림3" src="https://github.com/user-attachments/assets/b1cd616e-ecfb-4de7-80e2-843c2dba8105" />

### ✅ 결과 요약
Problem 3에서는 serial input을 받아 병렬 데이터로 변환하는 8비트 shift register를 설계하였다.  
`rst_n`이 0이 되면 비동기적으로 `parallel_out`이 모두 0으로 초기화되고, 리셋이 해제된 후에는 clock의 상승엣지마다 기존 값이 한 비트씩 왼쪽으로 이동하며 새로운 입력 `serial_in`이 최하위 비트에 저장되도록 하였다.  
따라서 시간이 지날수록 직렬로 들어오는 비트들이 순서대로 누적되어 `parallel_out`에 8비트 병렬 형태로 나타나게 된다.  
파형에서도 clock마다 값이 `00`, `01`, `02`, `05`, `0B`처럼 변화하면서 shift 동작이 정상적으로 이루어지는 것을 확인할 수 있었다.

---

## 5. 🧱 Problem 4: 8-bit Up/Down Counter with Load and Enable

### 📊 설계 사양
* 비동기 active-low reset, enable, 방향 제어, 병렬 load 기능을 모두 갖는 8-bit up/down counter를 설계함.
* `rst_n=0`이면 clock과 관계없이 `count`가 즉시 0으로 초기화됨.
* 리셋 해제 후에는 `load=1`일 때 입력 `data` 값이 우선적으로 `count`에 저장됨.
* `load=0`이고 `en=1`이면 `up` 신호에 따라 카운터가 증가하거나 감소함.
* `en=0`이면 현재 값을 그대로 유지함.

### 💻 Implementation (Code)
```verilog
`default_nettype none

module updown_counter (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       load,
    input  wire       en,
    input  wire       up,
    input  wire [7:0] data,
    output reg [7:0]  count
);

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        count <= 8'd0;
    end
    else if (load) begin
        count <= data;
    end
    else if (en) begin
        if (up)
            count <= count + 8'd1;
        else
            count <= count - 8'd1;
    end
    else begin
        count <= count;
    end
end

endmodule
```

<img width="752" height="163" alt="그림4" src="https://github.com/user-attachments/assets/8c88f4e9-93e4-42ca-b0c4-fe129ebde7fa" />

### ✅ 결과 요약
Problem 4에서는 비동기 active-low reset, enable, 방향 제어, 그리고 병렬 load 기능을 모두 갖는 8비트 up/down counter를 설계하였다.  
`rst_n`이 0이면 clock과 관계없이 `count`가 즉시 0으로 초기화되고, 리셋이 해제된 뒤에는 우선적으로 `load`가 1일 때 입력 `data` 값이 그대로 `count`에 저장되도록 하였다.  
그다음으로 `en`이 1이면 `up` 신호에 따라 카운터가 증가하거나 감소하며, `en`이 0이면 현재 값을 그대로 유지하도록 구현하였다.  
파형에서도 초기 리셋 후 카운터가 순차적으로 증가하다가 `up=0`일 때 감소하고, `load`가 활성화되면 `64`, `C8`, `FE` 같은 값이 바로 저장되며, 이후 다시 증가·감소와 overflow/underflow까지 정상적으로 수행되는 것을 확인할 수 있었다.

---
