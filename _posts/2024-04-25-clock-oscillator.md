---
layout: post
title:  "Two Clock Oscillators in SystemVerilog"
date:   2024-04-25 17:00:00 -0400
categories: jekyll update
---

Different from many programming languages, procedures in SystemVerilog are really syntactical sugar that register events. Roughly speaking, a SystemVerilog *event* is a thread that updates some variables.

For example, below shows two plausible ways[^1] to write a clock oscillator using `always`. In both ways,
the `initial` procedure registers an event at time 0, and the `always` procedure registers an event also at time 0, with a callback at the end of the event that registers the same event again. 


<div style="display: flex; justify-content: space-between;">
<div markdown="1" style="width: 48%; padding-right: 1%;">

```verilog
module osc1 (clk);
  initial #10 clk = 0;
  always @(clk) #10 clk = ~clk;
endmodule
```

</div>
<div markdown="1" style="width: 48%; padding-left: 1%;">

```verilog
module osc2 (clk);
  initial #10 clk = 0;
  always @(clk) #10 clk <= ~clk; 
endmodule
```


</div>
</div>

Below shows the simulation results for 30 time units. The simulation of osc1 ends early at time 20, which means `osc1` is not an oscillator. 

<div style="display: flex; justify-content: space-between;">
<div style="width: 48%; padding-right: 1%;">

<pre><code>// Simulation results for osc1
At time 0, Clock is x
At time 10, Clock is 0
At time 20, Clock is 1
</code></pre>

</div>
<div style="width: 48%; padding-left: 1%;">

<pre><code>// Simulation results for osc2
At time 0, Clock is x
At time 10, Clock is 0
At time 20, Clock is 1
At time 30, Clock is 0
</code></pre>

</div>
</div>

Let us consider what happens at each time slot.
Below shows how the simulator interprets osc1.

```python
# Simulation for osc1 using Python

def thread1():  # initial event
    Simulator.suspend(10)  # suspend this thread for 10 time units
    Simulator.change(clk, 0)  # change the value of clk to 0

def thread2():  # always event
    tmp_clk = not clk  # compute RHS at this time
    Simulator.suspend(10)  # suspend this thread for 10 time units
    Simulator.change(clk, tmp_clk)  # update LHS
    Simulator.register_event_on_change(clk, thread2)  # repeat

clk: Variable  # clk has value of 0, 1, or X (unknown)

time = 0

Simulator.register_event(thread1)
Simulator.register_event_on_change(clk, thread2)
Simulator.exec_events()  # take an event, deregister it, and execute it
```

And here is what happens during the simulation.

| Time | What Happens for osc1                                                                                                                                                                                                                                                                                                                  |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0    | `thread1` is executed and suspended for 10 time units. <br> The clock is unintialized, so its value is `x`.                                                                                                                                                                                                                                   |
| 10   | `thread1` is restored, and `Simulator.change(clk, 0)` is executed, which changes `clk` to `0` and then triggers `thread2`, in which the RHS `~ckl` is evaluated as `1`[^2], then `thread2` is suspended for 10 time units.                                                                                                                       |
| 20   | `thread2` is restored, and `Simulator.change(clk, tmp_clk)` is executed, which changes `clk` to `tmp_clk` and then triggers events that depend on `clk`. However, the only event `thread2` has been taken out and deregistered, so no event depends on `clk`. Finally, `thread2` is registered again, as a feature of the `always` procedure. |
|  >20 | At this time, although `thread2` is waiting for changes of `clk`, no statements will trigger the change. The simulation essentially ends here.                                                                                                                                                                                                            |

Therefore, osc1 does not describe a clock oscillator. Now, consider osc2.

```python
# Simulation for osc2 using Python

def thread1():  # the same as osc1
    Simulator.suspend(10)
    Simulator.change(clk, 0)

def thread2():  # always event
    tmp_clk = not clk  # compute RHS at this time
    Simulator.suspend(10)  # suspend this thread for 10 time units
    def assignment():
        Simulator.change(clk, tmp_clk)
    Simulator.register_NBA(assignment)  # register the update in to NBA
    Simulator.register_event_on_change(clk, thread2)  # repeat

clk: Variable  # clk has value of 0, 1, or X (unknown)

time = 0

Simulator.register_event(thread1)
Simulator.register_event_on_change(clk, thread2)
Simulator.exec_events()  # take an event, deregister it, and execute it
```

Below shows what happens at each time slot.

| Time | What Happens for osc2                                                                                                                                                                                                                                                                                                                         |
| ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0    | (The same as osc1) <br> `thread1` is executed and suspended for 10 time units. <br> The clock is unintialized, so its value is `x`.                                                                                                                                                                                                                                   |
| 10   | (The same as osc1) <br> `thread1` is restored, which changes `clk` to `0` and then triggers `thread2`, in which the RHS `~ckl` is evaluated as `1`[^3], then `thread2` is suspended for 10 time units.                                                                                                                       |
| 20   |  `thread2` is restored, and `assignment` is registered into the NBA region, which roughly means to do the assignment later–because the assignment is nonblocking. Then, `thread2` is registered again, as a feature of the `always` procedure. <br> Later, the events in NBA are moved to Active and gets executed. Here, `Simulator.change(clk, tmp_clk)` is executed, which changes `clk` to `tmp_clk` and then triggers events that depend on `clk`, which includes `thread2`. Therefore, `thread2` gets executed.  |
| >20  | Obviously, `thread2` will repeatedly be registered and executed every 10 time units.                                                                                                                                                                                                 |

Therefore, `osc2` indeed describes an oscillator. 


[^1]: [Nonblocking Assignments in Verilog Synthesis, Coding Styles That Kill!](http://www.sunburst-design.com/papers/CummingsSNUG2000SJ_NBA_rev1_2.pdf)

[^2]: [1800-2023 - IEEE Standard for SystemVerilog](https://ieeexplore.ieee.org/document/10458102), 4.9.3 Blocking assignment,
    > A blocking assignment statement (see 10.4.1) with an intra-assignment delay computes the right-hand side value using the current values, then causes the executing process to be suspended and scheduled as a future event.

[^3]: [1800-2023 - IEEE Standard for SystemVerilog](https://ieeexplore.ieee.org/document/10458102), 4.9.4 Nonblocking assignment, 
    > A nonblocking assignment statement (see 10.4.2) always computes the updated value and schedules the update as an NBA update event...