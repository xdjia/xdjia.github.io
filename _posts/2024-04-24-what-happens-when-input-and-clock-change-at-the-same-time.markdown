---
layout: post
title:  "What Happens When Input and Clock Change at the Same Time in SystemVerilog"
date:   2024-04-24 16:12:03 -0400
categories: jekyll update
---

```Verilog
// Declare a register module
module register(
  input logic clk,
  input logic in,
  output reg out
);
  initial begin
    out = 0;  // Initial state of the output
  end

  // Capture input at the rising edge of the clock
  always @(posedge clk) begin
    out <= in;
  end
endmodule

// Testbench to simulate the register behavior
module testbench();
  logic clk = 0;  // Initial clock state
  logic in = 1;   // Initial input state
  logic out;      // Output to be driven by the register
  
  register dut (clk, in, out);  // Instantiate the device under test
  
  initial begin
    // Display initial states
    $display("Before: clk=%b, in=%b, out=%b", clk, in, out);
    
    #10;
    clk = 1; // Rising edge of the clock
    in = 0;  // Change input at the same time
    
    #1;
    // Display states after changes
    $display("After: clk=%b, in=%b, out=%b", clk, in, out);
  end
endmodule
```

In real hardware, an input that changes right at the clock edge can potentially lead to metastability due to violations of setup and hold times.
In the simulation, particularly at the point where the input (`in`) and the clock (`clk`) change at the same simulation time, SystemVerilog handles the events based on its simulation model. Here’s what happens:

#### Event Scheduling

The two events of interests are scheduled into the Active region:

   - Event 1: Begin-End Block Evaluation.
   - Event 2: Nonblocking Statement Evaluation. The right-hand side (RHS) of the nonblocking statement `out <= in`, denoted as `in`, is evaluated.

#### Event Execution

The language reference manual[^1] (LRM) states that
(1) the simulator can execute the events in any order, and (2) within a block, the simulator can suspend statements execution and turn to another event.
Therefore, the simulator can execute the events in three ways.

   - Evaluate Event 2, then Event 1. Consequently, `in` is captured as `1`.
   - Or, evaluate event 1, and,
     - Evaluate `clk = 1`, suspend, then evaluate `in` in Event 2, then restore Event 1 and evaluate `in = 0`. Consequently, `in` is captured as `1`.
     - Or, evaluate `clk = 1` then `in = 0`, then evaluate `in` in Event 2. Consequently, `in` is captured as `0`.

Using Synopsys VCS 2021.09, the observed output is as follows, which indicates the third way.

```
Before: clk=0, in=1, out=0
After: clk=1, in=0, out=0
```

Below shows a famous way[^2] to write a clock oscillator. It works because the RHS of `clk = ~clk;` is evaluated in the Active region, which is before the update of the LHS in the NBA region.

```Verilog
module osc1 (clk);
  output clk;
  reg clk;

  initial #10 clk = 0;

  always @(clk) #10 clk = ~clk;
endmodule 
```

[^1]: [1800-2023 - IEEE Standard for SystemVerilog](https://ieeexplore.ieee.org/document/10458102)
[^2]: [Nonblocking Assignments in Verilog Synthesis, Coding Styles That Kill!](http://www.sunburst-design.com/papers/CummingsSNUG2000SJ_NBA_rev1_2.pdf)