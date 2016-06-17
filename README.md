# Zynq specific RTKLIB

A optimzed Version of the RTKLIB by reaplcement of some/most time consuming calculations with VHDL logic for the PL of a Xilinx Zynq 7020. 

All core position calculations are in src/rtkpos.c. The most interesting starting point to optimize is obviously udbias(). Based on the findings at [rtklibexplorer](https://rtklibexplorer.wordpress.com/2016/03/13/improving-rtklib-solution-phase-bias-sum-error/) there is a lot of room for improvements. (@rtklibexplorer also have some demos to a basic understanding of the famous RTKLIB)   

In udbias() you will find nested for loops with different calculations and checks. In every loop sdobs() (single-differenced observable) will called six times. Also the macro SQR (defined as x * y) will be called.

Interestingly inside the loop there is no recursion. Which mean at – least for the main loop – a FPGA can do all the calculations at the same time. 

This leads us to a bold solution. What if we map the hole structure of the "rtk_t *rtk" variable to a well known memory range? The FPGA could access this memory doing the needed calculations and write the results to a different target memory. For every for-loop there is a separate state machine. Given the inner loop body (with at least 100 x86 instructions) is called approximal 40 times a classic CPU needs more than 8000 clocks. In every 5 CPU clock a FPGA (700MHz vs. 150MHz at Xilinx Zynq or Altera Cyclone) can only do one operation. But with all loop iterations parallel and a state machine with not more than seven states ([Line 753: bias …](/https://github.com/rtklibexplorer/RTKLIB/blob/demo2/src/rtkpos.c#L785)) via pipelining the same instructions only need the same time like arround 40 CPU clock cycles.

The drawback is the additional time to get the the content of the memory into the FPGA and out. At an Zynq 7020 the time to read a memory block of 1MByte to the processing logic needs around 1000 CPU clock cycles. To write the results back to the target memory through the MPU needs even ~20% more. But even with the total needed 2240 clocks it is much fewer than the 8000 clocks of the CPU. With a careful organisation on a multicore CPU it is possible to suspend the calling thread.

###TODO

- [ ] Describe how to map the "rtk_t *rtk" (with a variable length) to a fix memory range? 
- - [ ] How to inform the FPGA to use only the valid data?
- [ ] Profiling of str2str.c
- [ ] Decision about FP encoding


###ORIGINAL DOCUMENTATION

The original readme is [here](readme.txt).

The home of RTKLIB is [here](http://www.rtklib.com).
