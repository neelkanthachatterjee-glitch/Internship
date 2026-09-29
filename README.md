# Digital VLSI Design Projects: FIR Filter & 4-Bit Microprocessor

This repository contains two complete digital VLSI design projects implemented in Verilog HDL, verified using simulation testbenches, and analyzed for hardware synthesis trade-offs.

---

## Table of Contents
1. [Project 1: VLSI Design of FIR Filter for DSP Applications](#project-1-vlsi-design-of-fir-filter-for-dsp-applications)
   - [Overview](#overview-1)
   - [1. MATLAB Coefficient Generation & Quantization](#1-matlab-coefficient-generation--quantization)
   - [2. Verilog HDL Implementation](#2-verilog-hdl-implementation)
   - [3. Testbench Simulation](#3-testbench-simulation)
   - [4. Area, Speed, and Power Synthesis Trade-Off Analysis](#4-area-speed-and-power-synthesis-trade-off-analysis)
2. [Project 2: Design and Simulation of 4-bit Processor](#project-2-design-and-simulation-of-4-bit-processor)
   - [Overview](#overview-2)
   - [1. Custom Instruction Set Architecture (ISA)](#1-custom-instruction-set-architecture-isa)
   - [2. Complete Verilog CPU Core Architecture](#2-complete-verilog-cpu-core-architecture)
   - [3. CPU Testbench & Instruction Execution Flow](#3-cpu-testbench--instruction-execution-flow)
   - [4. Waveform Simulation & Verification Guide](#4-waveform-simulation--verification-guide)
3. [Repository Directory Structure](#repository-directory-structure)
4. [How to Run Simulations](#how-to-run-simulations)

---

# Project 1: VLSI Design of FIR Filter for DSP Applications

## Overview 1
This project presents the design and verification of a 5-tap Finite Impulse Response (FIR) digital filter. Filter coefficients are calculated in MATLAB, scaled to **8-bit signed fixed-point integers (Q1.7 format)**, and mapped into a parallel Direct-Form Verilog architecture.

### Key Specifications:
- **Filter Type:** 5-tap Symmetric Low-Pass FIR Filter
- **Input Sample Precision:** 8-bit Signed
- **Coefficient Precision:** 8-bit Signed (Q1.7 Fixed-Point Format)
- **Output Precision:** 16-bit Signed (Prevents MAC overflow)

---

## 1. MATLAB Coefficient Generation & Quantization

The MATLAB script below designs a 4th-order (5-tap) FIR filter using a Hamming window and converts floating-point filter coefficients to 8-bit integers:

```matlab
% MATLAB Script: FIR Filter Coefficient Generation & Quantization
clc; clear; close all;

N = 4;           % Filter Order (5 Taps: N + 1)
fc = 0.25;       % Normalized Cutoff Frequency

% Generate floating-point coefficients
h_float = fir1(N, fc, hamming(N+1));

% Scale and quantize to 8-bit signed integer (Q1.7 format)
scale_factor = 128; % 2^7
h_fixed = round(h_float * scale_factor);

fprintf('Floating Point Coefficients:\n');
disp(h_float);

fprintf('8-bit Quantized Integer Coefficients:\n');
disp(h_fixed);

% Plot frequency response
freqz(h_float, 1, 512);
title('FIR Filter Frequency Response');
```

---

## 2. Verilog HDL Implementation

```verilog
// File: fir_filter.v
// Top-Level 5-Tap Direct-Form FIR Filter
module fir_filter (
    input wire clk,
    input wire reset,
    input wire signed [7:0] x_in,     // 8-bit signed input sample
    output reg signed [15:0] y_out    // 16-bit signed output sample
);

    // Fixed-Point Quantized Coefficients (Generated via MATLAB)
    parameter signed [7:0] H0 = 8'sd5;   
    parameter signed [7:0] H1 = 8'sd31;  
    parameter signed [7:0] H2 = 8'sd57;  
    parameter signed [7:0] H3 = 8'sd31;  
    parameter signed [7:0] H4 = 8'sd5;   

    // Delay Line Registers
    reg signed [7:0] x [0:4];
    integer i;

    // MAC Intermediate Registers
    reg signed [15:0] p0, p1, p2, p3, p4;
    reg signed [15:0] sum;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            for (i = 0; i < 5; i = i + 1) begin
                x[i] <= 8'sd0;
            end
            y_out <= 16'sd0;
        end else begin
            // Shift Delay Chain
            x[0] <= x_in;
            x[1] <= x[0];
            x[2] <= x[1];
            x[3] <= x[2];
            x[4] <= x[3];

            // Multiply-Accumulate
            p0 = x[0] * H0;
            p1 = x[1] * H1;
            p2 = x[2] * H2;
            p3 = x[3] * H3;
            p4 = x[4] * H4;

            sum = p0 + p1 + p2 + p3 + p4;
            y_out <= sum;
        end
    end

endmodule
```

---

## 3. Testbench Simulation

```verilog
// File: tb_fir_filter.v
`timescale 1ns / 1ps

module tb_fir_filter;

    reg clk;
    reg reset;
    reg signed [7:0] x_in;
    wire signed [15:0] y_out;

    fir_filter uut (
        .clk(clk),
        .reset(reset),
        .x_in(x_in),
        .y_out(y_out)
    );

    // 100 MHz Clock Generation
    always #5 clk = ~clk;

    initial begin
        clk = 0;
        reset = 1;
        x_in = 0;

        #20 reset = 0;

        // Apply Impulse Input
        #10 x_in = 8'sd127; 
        #10 x_in = 8'sd0;
        #10 x_in = 8'sd0;
        #10 x_in = 8'sd0;
        #10 x_in = 8'sd0;
        
        // Apply Step Input
        #50 x_in = 8'sd10;
        #100 $finish;
    end

endmodule
```

---

## 4. Area, Speed, and Power Synthesis Trade-Off Analysis

| Architectural Choice | Area Utilization (LUTs/Gates) | Maximum Frequency ($F_{max}$) | Dynamic Power | Best Use-Case |
| :--- | :--- | :--- | :--- | :--- |
| **Direct-Form Standard** | Low | Moderate | Low | Small resource footprint |
| **Pipelined Direct-Form** | Moderate | **High** | Moderate | High-speed DSP pipelines |
| **Transposed-Form** | Moderate | **High** | Low–Moderate | High sample-rate processing |
| **Folded / Time-Multiplexed** | **Minimal** | Low | **Minimal** | Ultra-low power / IoT edge |

---

# Project 2: Design and Simulation of 4-bit Processor

## Overview 2
This project implements a complete custom 4-bit central processing unit (CPU). It contains a Control Unit, 4x4-bit Register File, Arithmetic Logic Unit (ALU), and a 16x4-bit Data Memory block connected through internal multiplexed buses.

---

## 1. Custom Instruction Set Architecture (ISA)

The CPU operates on **8-bit wide fixed instructions**:
- **Opcode `[7:6]`:** Identifies instruction type
- **Destination Register / Address `[5:4]`:** Primary target operand
- **Source Register / Immediate `[3:0]`:** Secondary operand or data payload

| Instruction | Opcode `[7:6]` | Operand 1 `[5:4]` | Operand 2 `[3:0]` | Action / RTL Operation |
| :--- | :--- | :--- | :--- | :--- |
| **ADD** | `2'b00` | Reg Dst | Reg Src | $R[Dst] \leftarrow R[Dst] + R[Src]$ |
| **SUB** | `2'b01` | Reg Dst | Reg Src | $R[Dst] \leftarrow R[Dst] - R[Src]$ |
| **LOADI** | `2'b10` | Reg Dst | Immediate `[1:0]` | $R[Dst] \leftarrow \text{ZeroExt}(Imm)$ |
| **STORE** | `2'b11` | Reg Src | Memory Address | $Memory[Addr] \leftarrow R[Src]$ |

---

## 2. Complete Verilog CPU Core Architecture

```verilog
// File: processor_4bit.v

// 1. Arithmetic Logic Unit
module alu (
    input wire [3:0] A,
    input wire [3:0] B,
    input wire [1:0] alu_op,
    output reg [3:0] result,
    output wire zero
);
    always @(*) begin
        case (alu_op)
            2'b00: result = A + B; // ADD
            2'b01: result = A - B; // SUB
            2'b10: result = B;     // Pass B (Immediate)
            default: result = 4'b0000;
        endcase
    end
    assign zero = (result == 4'b0000);
endmodule

// 2. 4x4 Register File
module register_file (
    input wire clk,
    input wire reset,
    input wire reg_write,
    input wire [1:0] read_reg1,
    input wire [1:0] read_reg2,
    input wire [1:0] write_reg,
    input wire [3:0] write_data,
    output wire [3:0] read_data1,
    output wire [3:0] read_data2
);
    reg [3:0] registers [0:3];
    integer i;

    assign read_data1 = registers[read_reg1];
    assign read_data2 = registers[read_reg2];

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            for (i = 0; i < 4; i = i + 1)
                registers[i] <= 4'b0000;
        end else if (reg_write) begin
            registers[write_reg] <= write_data;
        end
    end
endmodule

// 3. Control Unit
module control_unit (
    input wire [1:0] opcode,
    output reg reg_write,
    output reg mem_write,
    output reg alu_src,
    output reg [1:0] alu_op
);
    always @(*) begin
        case (opcode)
            2'b00: begin // ADD
                reg_write = 1'b1; mem_write = 1'b0; alu_src = 1'b0; alu_op = 2'b00;
            end
            2'b01: begin // SUB
                reg_write = 1'b1; mem_write = 1'b0; alu_src = 1'b0; alu_op = 2'b01;
            end
            2'b10: begin // LOADI
                reg_write = 1'b1; mem_write = 1'b0; alu_src = 1'b1; alu_op = 2'b10;
            end
            2'b11: begin // STORE
                reg_write = 1 me1'b0; mem_write = 1'b1; alu_src = 1'b0; alu_op = 2'b00;
            end
            default: begin
                reg_write = 1'b0; mem_write = 1'b0; alu_src = 1'b0; alu_op = 2'b00;
            end
        endcase
    end
endmodule

// 4. Data Memory Block
module data_memory (
    input wire clk,
    input wire mem_write,
    input wire [3:0] address,
    input wire [3:0] write_data,
    output wire [3:0] read_data
);
    reg [3:0] memory [0:15];
    assign read_data = memory[address];

    always @(posedge clk) begin
        if (mem_write) begin
            memory[address] <= write_data;
        end
    end
endmodule

// 5. Top-Level Processor Integration
module processor_4bit (
    input wire clk,
    input wire reset,
    input wire [7:0] instruction,
    output wire [3:0] alu_out,
    output wire [3:0] mem_out
);
    wire reg_write, mem_write, alu_src, zero;
    wire [1:0] alu_op;
    wire [3:0] reg_data1, reg_data2, alu_b_input;
    wire [3:0] immediate_ext = {2'b00, instruction[1:0]};

    control_unit ctrl (
        .opcode(instruction[7:6]),
        .reg_write(reg_write),
        .mem_write(mem_write),
        .alu_src(alu_src),
        .alu_op(alu_op)
    );

    register_file rf (
        .clk(clk),
        .reset(reset),
        .reg_write(reg_write),
        .read_reg1(instruction[5:4]),
        .read_reg2(instruction[3:2]),
        .write_reg(instruction[5:4]),
        .write_data(alu_out),
        .read_data1(reg_data1),
        .read_data2(reg_data2)
    );

    assign alu_b_input = (alu_src) ? immediate_ext : reg_data2;

    alu alu_inst (
        .A(reg_data1),
        .B(alu_b_input),
        .alu_op(alu_op),
        .result(alu_out),
        .zero(zero)
    );

    data_memory mem (
        .clk(clk),
        .mem_write(mem_write),
        .address(instruction[3:0]),
        .write_data(reg_data1),
        .read_data(mem_out)
    );
endmodule
```

---

## 3. CPU Testbench & Instruction Execution Flow

```verilog
// File: tb_processor_4bit.v
`timescale 1ns / 1ps

module tb_processor_4bit;

    reg clk;
    reg reset;
    reg [7:0] instruction;
    wire [3:0] alu_out;
    wire [3:0] mem_out;

    processor_4bit uut (
        .clk(clk),
        .reset(reset),
        .instruction(instruction),
        .alu_out(alu_out),
        .mem_out(mem_out)
    );

    always #5 clk = ~clk;

    initial begin
        clk = 0;
        reset = 1;
        instruction = 8'h00;

        #10 reset = 0;

        // 1. LOADI R0, 3  -> 10_00_00_11
        #10 instruction = 8'b10_00_00_11; 

        // 2. LOADI R1, 2  -> 10_01_00_10
        #10 instruction = 8'b10_01_00_10;

        // 3. ADD R0, R1   -> 00_00_01_00 (R0 = 3 + 2 = 5)
        #10 instruction = 8'b00_00_01_00;

        // 4. SUB R0, R1   -> 01_00_01_00 (R0 = 5 - 2 = 3)
        #10 instruction = 8'b01_00_01_00;

        // 5. STORE R0, Mem[4] -> 11_00_0100
        #10 instruction = 8'b11_00_0100;

        #20 $finish;
    end
endmodule
```

---

## 4. Waveform Simulation & Verification Guide

When loaded into ModelSim, Vivado, or EDA Playground:
1. **Reset Cycle:** Sets register values to zero on `reset = 1`.
2. **Immediate Load (`LOADI`):** `reg_write` asserts HIGH; immediate data moves through the ALU to write into target registers ($R0, R1$).
3. **Execution Stage (`ADD` / `SUB`):** Operands read from Register File, calculated in ALU, and saved back to destination register on the next clock edge.
4. **Memory Write (`STORE`):** `mem_write` toggles HIGH, writing $R0$ data directly into the specified memory address location.

---

## Repository Directory Structure

```
.
├── README.md
├── fir_filter/
│   ├── scripts/
│   │   └── fir_coef_gen.m
│   └── hdl/
│       ├── fir_filter.v
│       └── tb_fir_filter.v
└── processor_4bit/
    └── hdl/
        ├── processor_4bit.v
        └── tb_processor_4bit.v
```

---

## How to Run Simulations

1. **Clone the repository:**
   ```bash
   git clone https://github.com/YOUR_USERNAME/VLSI-Projects.git
   ```
2. **MATLAB Execution:** Run `fir_coef_gen.m` in MATLAB to plot frequency response and generate coefficients.
3. **Verilog Simulation:** Import `.v` files into ModelSim / Xilinx Vivado / EDA Playground, set `tb_fir_filter` or `tb_processor_4bit` as top-level modules, and run the simulation wave viewing tools.