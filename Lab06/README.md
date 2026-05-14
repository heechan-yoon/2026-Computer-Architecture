# ⚡ [Lab 06] Read-Write Synchronizer FSM Design

> **과목명:** 2026-1 컴퓨터구조 실습  
> **작성자:** 2022067101 윤희찬  

---

## 1. 🎯 프로젝트 목표

* `do_write` 입력 신호에 따라 READ 또는 WRITE 동작을 선택하는 FSM을 설계함.
* FSM이 `IDLE`, `READ`, `WRITE`, `DONE`의 4개 상태를 가지도록 구현함.
* `reset`이 active-high로 동작하여 FSM이 `IDLE` 상태로 초기화되는지 확인함.
* WRITE 요청과 READ 요청이 각각 상태도에 맞게 진행되는지 testbench와 waveform으로 검증함.
* DONE 상태에서만 실행 완료 신호 `exec`가 1로 출력되도록 설계함.
* waveform에서 상태 변화를 직접 확인하기 위해 `state[1:0]`을 debug output으로 추가함.

---

## 2. 🧱 Problem 1: Read-Write Synchronizer FSM

### 📊 설계 사양

* 입력 신호 `do_write`가 1이면 WRITE 요청, 0이면 READ 요청으로 판단함.
* FSM은 총 4개의 상태를 사용함.
  * `ST_IDLE`: 대기 상태
  * `ST_READ`: READ 동작 준비 상태
  * `ST_WRITE`: WRITE 동작 준비 상태
  * `ST_DONE`: 실행 완료 상태
* `reset`은 active-high 방식으로 동작하며, `reset = 1`이면 FSM은 `ST_IDLE` 상태로 초기화됨.
* `exec`는 DONE 상태에서만 1로 출력됨.
* `rd_wr`은 WRITE 동작일 때 1, READ 동작일 때 0을 출력함.
* DONE 상태에서는 직전 동작이 READ였는지 WRITE였는지를 유지하기 위해 `op_is_write` 레지스터를 사용함.

### 📌 State Encoding

| State | Encoding | 의미 |
| :--- | :--- | :--- |
| `ST_IDLE` | `2'd0` | 대기 상태 |
| `ST_READ` | `2'd1` | READ 동작 준비 상태 |
| `ST_WRITE` | `2'd2` | WRITE 동작 준비 상태 |
| `ST_DONE` | `2'd3` | 실행 완료 상태 |

---

## 3. 🔁 FSM 동작 원리

FSM은 `do_write` 입력에 따라 READ 또는 WRITE 방향을 결정한다.

### WRITE 동작

* `IDLE` 상태에서 `do_write = 1`이면 WRITE 요청으로 판단하여 `WRITE` 상태로 이동함.
* `WRITE` 상태에서 `do_write = 1`이 한 번 더 유지되면 `DONE` 상태로 이동함.
* `DONE` 상태에서 `exec = 1`이 출력되고, 이후 다시 `IDLE` 상태로 복귀함.

```text
IDLE → WRITE → DONE → IDLE
```

### READ 동작

* `IDLE` 상태에서 `do_write = 0`이면 READ 요청으로 판단하여 `READ` 상태로 이동함.
* `READ` 상태에서 `do_write = 0`이 한 번 더 유지되면 `DONE` 상태로 이동함.
* `DONE` 상태에서 `exec = 1`이 출력되고, 이후 다시 `IDLE` 상태로 복귀함.

```text
IDLE → READ → DONE → IDLE
```

즉, 이 FSM은 입력을 한 번만 보고 바로 완료 신호를 출력하는 구조가 아니라, 동일한 요청이 한 번 더 유지되는지를 확인한 뒤 `DONE` 상태에서 실행 완료 신호를 발생시키는 구조이다.

---

## 4. 📊 상태 천이표

| 현재 상태 | `do_write` | 다음 상태 | `exec` | `rd_wr` |
| :--- | :--- | :--- | :--- | :--- |
| `IDLE` | 1 | `WRITE` | 0 | 0 |
| `IDLE` | 0 | `READ` | 0 | 0 |
| `WRITE` | 1 | `DONE` | 0 | 1 |
| `WRITE` | 0 | `WRITE` | 0 | 1 |
| `READ` | 0 | `DONE` | 0 | 0 |
| `READ` | 1 | `READ` | 0 | 0 |
| `DONE` | X | `IDLE` | 1 | 이전 동작 방향 |

---

## 5. 💻 Implementation 1: `fsm_synchronizer.sv`

```systemverilog
`timescale 1ns/1ps
`default_nettype none

module fsm_synchronizer (
    input  wire        clk,
    input  wire        reset,      // active-high reset
    input  wire        do_write,   // 1: write, 0: read

    output reg         exec,
    output reg         rd_wr,      // 1: write, 0: read
    output reg  [1:0]  state       // waveform에서 현재 상태 확인용
);

    localparam [1:0] ST_IDLE  = 2'd0;  // 00
    localparam [1:0] ST_READ  = 2'd1;  // 01
    localparam [1:0] ST_WRITE = 2'd2;  // 10
    localparam [1:0] ST_DONE  = 2'd3;  // 11

    reg [1:0] next_state;

    // DONE 상태에서 read/write 방향을 유지하기 위한 레지스터
    reg op_is_write;
    reg next_op_is_write;

    // Next state logic
    always @(*) begin
        next_state       = state;
        next_op_is_write = op_is_write;

        case (state)
            ST_IDLE: begin
                if (do_write) begin
                    next_state       = ST_WRITE;
                    next_op_is_write = 1'b1;
                end
                else begin
                    next_state       = ST_READ;
                    next_op_is_write = 1'b0;
                end
            end

            ST_WRITE: begin
                next_op_is_write = 1'b1;

                if (do_write) begin
                    next_state = ST_DONE;
                end
                else begin
                    next_state = ST_WRITE;
                end
            end

            ST_READ: begin
                next_op_is_write = 1'b0;

                if (!do_write) begin
                    next_state = ST_DONE;
                end
                else begin
                    next_state = ST_READ;
                end
            end

            ST_DONE: begin
                next_state = ST_IDLE;
            end

            default: begin
                next_state       = ST_IDLE;
                next_op_is_write = 1'b0;
            end
        endcase
    end

    // State register with active-high asynchronous reset
    always @(posedge clk or posedge reset) begin
        if (reset) begin
            state       <= ST_IDLE;
            op_is_write <= 1'b0;
        end
        else begin
            state       <= next_state;
            op_is_write <= next_op_is_write;
        end
    end

    // Output logic
    always @(*) begin
        exec  = 1'b0;
        rd_wr = 1'b0;

        case (state)
            ST_IDLE: begin
                exec  = 1'b0;
                rd_wr = 1'b0;
            end

            ST_READ: begin
                exec  = 1'b0;
                rd_wr = 1'b0;
            end

            ST_WRITE: begin
                exec  = 1'b0;
                rd_wr = 1'b1;
            end

            ST_DONE: begin
                exec  = 1'b1;
                rd_wr = op_is_write;
            end

            default: begin
                exec  = 1'b0;
                rd_wr = 1'b0;
            end
        endcase
    end

endmodule

`default_nettype wire
```

### ✅ 구현 설명

`fsm_synchronizer.sv`는 크게 세 부분으로 구성된다.

1. **Next State Logic**
   * 현재 상태 `state`와 입력 `do_write`를 이용해 다음 상태 `next_state`를 결정한다.
   * `always @(*)` 블록으로 작성하여 조합 논리임을 나타낸다.

2. **State Register**
   * `posedge clk`에서 현재 상태를 다음 상태로 갱신한다.
   * `reset`이 active-high이므로 `reset = 1`이면 `state`를 `ST_IDLE`로 초기화한다.

3. **Output Logic**
   * 현재 상태에 따라 `exec`와 `rd_wr` 값을 결정한다.
   * `READ` 상태에서는 `rd_wr = 0`, `WRITE` 상태에서는 `rd_wr = 1`, `DONE` 상태에서는 `exec = 1`이 출력된다.

---

## 6. 💻 Implementation 2: `fsm_synchronizer_tb.sv`

```systemverilog
`timescale 1ns/1ps
`default_nettype none

module fsm_synchronizer_tb;

    reg clk;
    reg reset;
    reg do_write;

    wire       exec;
    wire       rd_wr;
    wire [1:0] state;

    integer errors;

    // Testbench에서도 상태값 해석용으로 동일하게 정의
    localparam [1:0] ST_IDLE  = 2'd0;  // 00
    localparam [1:0] ST_READ  = 2'd1;  // 01
    localparam [1:0] ST_WRITE = 2'd2;  // 10
    localparam [1:0] ST_DONE  = 2'd3;  // 11

    fsm_synchronizer dut (
        .clk      (clk),
        .reset    (reset),
        .do_write (do_write),
        .exec     (exec),
        .rd_wr    (rd_wr),
        .state    (state)
    );

    // Clock generation: 10 ns period
    initial begin
        clk = 1'b0;
    end

    always #5 clk = ~clk;

    task tick;
        begin
            @(posedge clk);
            #1;
        end
    endtask

    task check;
        input [1:0] expected_state;
        input       expected_exec;
        input       expected_rd_wr;
        input [8*100-1:0] message;
        begin
            if ((state != expected_state) ||
                (exec  != expected_exec)  ||
                (rd_wr != expected_rd_wr)) begin

                $display("[FAIL] time=%0t : %0s", $time, message);
                $display("       expected state=%b exec=%b rd_wr=%b",
                         expected_state, expected_exec, expected_rd_wr);
                $display("       got      state=%b exec=%b rd_wr=%b",
                         state, exec, rd_wr);

                errors = errors + 1;
            end
            else begin
                $display("[PASS] time=%0t : %0s | state=%b exec=%b rd_wr=%b",
                         $time, message, state, exec, rd_wr);
            end
        end
    endtask

    initial begin
        $dumpfile("fsm_synchronizer.vcd");
        $dumpvars(0, fsm_synchronizer_tb);

        errors   = 0;
        reset    = 1'b1;
        do_write = 1'b1;

        // ============================================================
        // Reset test
        // reset은 active-high이므로 reset=1이면 state는 IDLE
        // ============================================================

        tick();
        check(ST_IDLE, 1'b0, 1'b0, "reset active: state should be IDLE");

        reset = 1'b0;

        // ============================================================
        // WRITE operation test
        // do_write=1이면 IDLE -> WRITE -> DONE -> IDLE
        // ============================================================

        // IDLE -> WRITE
        tick();
        check(ST_WRITE, 1'b0, 1'b1, "IDLE to WRITE when do_write=1");

        // WRITE -> DONE
        tick();
        check(ST_DONE, 1'b1, 1'b1, "WRITE to DONE when do_write remains 1");

        // DONE -> IDLE
        tick();
        check(ST_IDLE, 1'b0, 1'b0, "DONE to IDLE after WRITE operation");

        // ============================================================
        // READ operation test
        // do_write=0이면 IDLE -> READ -> DONE -> IDLE
        // ============================================================

        do_write = 1'b0;

        // IDLE -> READ
        tick();
        check(ST_READ, 1'b0, 1'b0, "IDLE to READ when do_write=0");

        // READ -> DONE
        tick();
        check(ST_DONE, 1'b1, 1'b0, "READ to DONE when do_write remains 0");

        // DONE -> IDLE
        tick();
        check(ST_IDLE, 1'b0, 1'b0, "DONE to IDLE after READ operation");

        // ============================================================
        // WRITE self-loop test
        // WRITE 상태에서 do_write=0이면 WRITE에 머물러야 함
        // ============================================================

        do_write = 1'b1;

        // IDLE -> WRITE
        tick();
        check(ST_WRITE, 1'b0, 1'b1, "IDLE to WRITE for WRITE self-loop test");

        // WRITE 상태에서 do_write=0이면 WRITE 유지
        do_write = 1'b0;
        tick();
        check(ST_WRITE, 1'b0, 1'b1, "WRITE stays WRITE when do_write=0");

        // 다시 do_write=1이면 DONE
        do_write = 1'b1;
        tick();
        check(ST_DONE, 1'b1, 1'b1, "WRITE to DONE when do_write returns to 1");

        // DONE -> IDLE
        tick();
        check(ST_IDLE, 1'b0, 1'b0, "DONE to IDLE after WRITE self-loop test");

        // ============================================================
        // READ self-loop test
        // READ 상태에서 do_write=1이면 READ에 머물러야 함
        // ============================================================

        do_write = 1'b0;

        // IDLE -> READ
        tick();
        check(ST_READ, 1'b0, 1'b0, "IDLE to READ for READ self-loop test");

        // READ 상태에서 do_write=1이면 READ 유지
        do_write = 1'b1;
        tick();
        check(ST_READ, 1'b0, 1'b0, "READ stays READ when do_write=1");

        // 다시 do_write=0이면 DONE
        do_write = 1'b0;
        tick();
        check(ST_DONE, 1'b1, 1'b0, "READ to DONE when do_write returns to 0");

        // DONE -> IDLE
        tick();
        check(ST_IDLE, 1'b0, 1'b0, "DONE to IDLE after READ self-loop test");

        // ============================================================
        // Result
        // ============================================================

        if (errors == 0) begin
            $display("==============================================");
            $display("ALL TESTS PASSED");
            $display("==============================================");
        end
        else begin
            $display("==============================================");
            $display("TEST FAILED: errors = %0d", errors);
            $display("==============================================");
        end

        $finish;
    end

endmodule

`default_nettype wire
```

### ✅ Testbench 구성

Testbench에서는 FSM의 주요 동작을 다음과 같이 검증하였다.

| Test 항목 | 검증 내용 |
| :--- | :--- |
| Reset test | `reset = 1`일 때 상태가 `IDLE`이 되는지 확인 |
| WRITE operation | `do_write = 1`일 때 `IDLE → WRITE → DONE → IDLE` 순서로 동작하는지 확인 |
| READ operation | `do_write = 0`일 때 `IDLE → READ → DONE → IDLE` 순서로 동작하는지 확인 |
| WRITE self-loop | `WRITE` 상태에서 `do_write = 0`이면 `WRITE` 상태에 머무르는지 확인 |
| READ self-loop | `READ` 상태에서 `do_write = 1`이면 `READ` 상태에 머무르는지 확인 |

Testbench에서는 `errors` 변수를 사용하여 예상값과 실제 출력값이 다를 경우 error count를 증가시키도록 하였다. 시뮬레이션 결과 `errors[31:0] = 00000000`으로 유지되어 모든 test case가 통과한 것을 확인하였다.

---

## 7. 🌊 Waveform

<img width="752" height="183" alt="그림1" src="https://github.com/user-attachments/assets/59baf1b2-e60e-4875-a8d1-413c13e49b8d" />

### ✅ Waveform 결과 분석

시뮬레이션 waveform에서 `state[1:0]`은 다음과 같은 순서로 변화하였다.

```text
0 → 2 → 3 → 0 → 1 → 3 → 0
```

이는 각각 다음 상태 변화를 의미한다.

```text
IDLE → WRITE → DONE → IDLE → READ → DONE → IDLE
```

따라서 WRITE 동작과 READ 동작이 모두 정상적으로 포함되어 있음을 확인할 수 있다.

WRITE 동작 구간에서는 `state = 2`, 즉 `ST_WRITE` 상태에서 `rd_wr = 1`이 출력된다. 이후 `state = 3`, 즉 `ST_DONE` 상태가 되면 `exec = 1`이 출력되며 write 동작이 완료된다.

READ 동작 구간에서는 `state = 1`, 즉 `ST_READ` 상태로 이동하고, `rd_wr = 0`이 유지된다. 이후 `state = 3`, 즉 `ST_DONE` 상태가 되면 `exec = 1`이 출력되며 read 동작이 완료된다.

또한 DONE 상태 이후에는 항상 `IDLE` 상태로 복귀하는 것을 waveform에서 확인할 수 있었다.

---

## 8. 🔍 트러블슈팅 및 고찰

초기에는 waveform에서 FSM의 현재 상태를 직접 확인하기 어려웠다. `exec`와 `rd_wr` 출력만으로는 FSM이 실제로 `IDLE`, `READ`, `WRITE`, `DONE` 중 어느 상태에 있는지 명확히 판단하기 어려웠기 때문이다.

이를 해결하기 위해 `state[1:0]`을 output port로 추가하였다. 그 결과 waveform에서 다음과 같이 상태 encoding을 직접 확인할 수 있었다.

| Waveform 값 | FSM 상태 |
| :--- | :--- |
| 0 | `IDLE` |
| 1 | `READ` |
| 2 | `WRITE` |
| 3 | `DONE` |

또한 DONE 상태는 READ 이후에도 들어가고 WRITE 이후에도 들어가기 때문에, DONE 상태에서 `rd_wr` 값을 올바르게 유지하기 위한 별도의 레지스터가 필요했다. 이를 위해 `op_is_write` 레지스터를 추가하여 직전 동작이 WRITE였는지 READ였는지를 저장하도록 설계하였다.

그 결과 DONE 상태에서도 이전 동작 방향에 따라 `rd_wr`이 올바르게 유지되었고, `exec`는 DONE 상태에서만 1로 출력되었다.

---

## 9. ✅ 결과 요약

설계한 Read-Write Synchronizer FSM은 주어진 상태도와 동일하게 동작하였다. Reset이 active-high로 동작하여 reset 구간에서 FSM이 `IDLE` 상태로 초기화되었고, `do_write` 입력에 따라 WRITE 또는 READ 상태로 정상적으로 전이되었다.

시뮬레이션 결과 WRITE 동작은 다음 순서로 진행되었다.

```text
IDLE → WRITE → DONE → IDLE
```

READ 동작은 다음 순서로 진행되었다.

```text
IDLE → READ → DONE → IDLE
```

또한 WRITE self-loop와 READ self-loop 조건도 정상적으로 검증되었다. `exec` 신호는 DONE 상태에서만 1로 출력되었고, `rd_wr` 신호는 WRITE 동작 중에는 1, READ 동작 중에는 0으로 출력되어 read/write 방향을 올바르게 나타냈다.

---

## 10. 📌 결론

본 과제에서는 `do_write` 입력에 따라 READ와 WRITE 동작을 선택하고, 조건이 유지되었을 때 DONE 상태에서 `exec` 신호를 출력하는 Read-Write Synchronizer FSM을 설계하였다.

시뮬레이션 결과, reset 이후 FSM은 정상적으로 `IDLE` 상태로 초기화되었으며, WRITE 동작은 `IDLE → WRITE → DONE → IDLE`, READ 동작은 `IDLE → READ → DONE → IDLE` 순서로 진행되었다. 또한 self-loop 조건도 정상적으로 동작하였고, testbench의 error count가 0으로 유지되어 설계가 요구사항을 만족함을 확인하였다.

---
