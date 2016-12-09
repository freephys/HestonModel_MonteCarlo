Monte Carlo Methods applied to the Heston financial market model
============

## Overview
This project implements a Monte Carlo simulation of the Heston financial model, using both the European and the European barrier options. It contains an OpenCL C++ kernel, to be mapped to FPGA via SDAccel. It provides much better energy-per-operation than a GPU implementation, at a comparable performance level.

### Heston Model
The [Heston model][Heston Model], which was first published by Steven Heston in 1993, is a famous mathematical model describing the behaviour of investment instruments in financial markets. This model focuses on comparing the Return On Investment for one risky asset, whose price and volatility are subject to [geometric Brownian motion][geometric Brownian motion] and one riskless asset with a fixed interest rate.


### Call/Put Option
Entering more specifically into the financial sector operations, two styles of stock transaction [options][option] are considered in this project, namely the European vanilla option and European barrier option (which is one of the [exotic options][exotic options]).
[Call options][Call options] and [put options][put options] are defined reciprocally. Given the basic parameters for an option, namely expiration date and strike price, the call/put payoff price could be estimated.
[option]: https://en.wikipedia.org/wiki/Option_style
[exotic options]: https://en.wikipedia.org/wiki/Exotic_option

### The Monte Carlo Method
The [Monte Carlo Method][Monte Carlo] is one of the most widely used approaches to simulate stochastic processes, like the stock price and volatility modeled with Heston. This is especially true for exotic options, which are usually not solvable analytically. In this project, the Monte Carlo Method is used to estimate the payoff price of a given instrument using the Heston model.

The given time period has to be partitioned into M steps according to the style of the option. At each time point of a Monte Carlo simulation of this kind, the stock price and volatility are determined by the values at previous time point and a pair of correlated normally distributed random numbers. The expectation of the payoff price can thus be estimated by N parallel independent simulations.

The convergence of the result produced by the Monte Carlo method is ensured in this case by running a very large number of simulation steps, namely ![$C=M \cdot N$], (which should be a very large number, e.g. ![$10^9$]).
Other convergence criteria (e.g. checking the difference between successive iterations) could be added.


### Normally Distributed Random Number Generation
A key aspect of the quality of the results of the Monte Carlo method is the quality of the random numbers that it uses. The normally distributed random numbers that are used in this project are generated by the Mersenne-Twister algorithm followed by the Box-Muller transformation.

#### Mersenne-Twister

The [Mersenne Twister][Mersenne Twister] is an algorithm to generate uniformly distributed pseudo random numbers. Its very long periodicity ![$2^{19937}-1$] makes it a suitable algorithm for our application, since as discussed above the Monte Carlo method requires millions of random numbers.


#### Box-Muller transform
The [Box Muller transformation][Box Muller transformation] transforms a pair of
independent, uniformly distributed random numbers in the interval (0,1) into a
pair of independent normally distributed random numbers, which are required for
simulating the Heston model.
Given two independent ![$U_1$,$U_2 \sim U(0,1)$],

![$$Z_1=\sqrt{-2ln(U_1)}cos(2\pi U_2)\\Z_2=\sqrt{-2ln(U_1)}sin(2\pi U_2)$$]

then ![$Z_1$,$Z_2\sim N(0,1)$], also independent.





## Getting Started

The repository contains two directories, "hestonEuro" and "hestonBarrier", implementing the European option and the European barrier option respectively. They are written using C++, rather than OpenCL, because there is no workgroup-level memory that is worth sharing. All Monte Carlo simulations are independent.
In this implementation, since the complexity of the random number generation process is simpler than the complexity of the Monte Carlo simulation step, one random number generator feeds a group of NUM_SIMS simulations, by utilizing BRAMs storing the intermediate results. The iterations over NUM_RNGS are fully unrolled (as if they were executed by an independent work item in a work group) and the iterations over NUM_SIMS and NUM_SIMGROUPS are pipelined.
These two parameters should be chosen to fully utilize the resources on an FPGA.
In order to ensure that enough simulations of a given stock are performed, NUM_SIMGROUPS can then be tuned.

The total number of simulations is ![$N=NUM\_SIMS \cdot NUM\_RNG \cdot NUM\_SIMGROUPS$] and each simulation group assigned to a given RNG runs NUM_SIMS simulations.
The best value for NUM_SIMS can be chosen by maximizing the use of BRAM blocks.  By default it is 512 (which fills one BRAM block with 512 float values).

### File Tree

```
hestonModel
│   README.md
│
└── common
│   │   defTypes.h
│   │   RNG.h
│   │   RNG.cpp
│   │   stockData.h
│   │   stockData.cpp
│   │   barrierData.h
│   │   barrierData.cpp
│   │   volatilityData.h
│   │   volatilityData.cpp
│   └─  ML_cl.h
│
└── hestonEuro
│   │   solution.tcl
│   │   main.cpp
│   │   hestonEuro.h
│   └─  hestonEuro.cpp
│
└── hestonBarrier
    │   solution.tcl
    │   main.cpp
    │   hestonBarrier.h
    └─  hestonBarrier.cpp
```

File/Dir name  |Information
-------------- | ---
hestonEuro.cpp  | Top function of the European option kernel
hestonBarrier.cpp | Top function of the European barrier option kernel
solution.tcl   | Script to run sdaccel
heston.h | It declares the Heston model object instantiated in the top functions (same methods for European and European barrier option).
heston.cpp | It defines the Heston model object instantiated in the top functions. Note that the definitions of the object methods are different between the European and European barrier options.
stockData.cpp	 | Basic stock datasets. It defines an object instantiated in the top functions
volatilityData.cpp | Basic volatility datasets used in Heston model. It defines an object instantiated in the top functions.
hestonEuroBarrier.cpp | Up and Down Barrier datasets.
RNG.cpp   | Random Number Generator class. It defines an object instantiated in
the Heston objects.
main.cpp  |  Host code calling the kernels, Input parameters for the kernels can be changed from the command line.
ML_cl.h | CL/cl.hpp for OpenCL 1.2.6

Note that in the repository we had to include the OpenCL header for version 1.2.6, instead of the version 1.1 installed by sdaccel, because the latter causes compile-time errors. SDAccel and Vivado HLS work perfectly well with this header.

### Parameters
The values of the parameters for a given stock and option are listed in the namespace Params in ***"main.cpp"***.
The values of these parameters, except for the name of the kernel, have a default value and can be changed via the corresponding command-line option at runtime. The kernel name must be "hestonBarrier" for the European barrier option and hestonEuro for the European one.
The call price and the put price are used for functional verification only (they are the expected output values for a given set of input values)

For example, the RTL emulation can be executed as follows to use the European
vanilla option with the default parameter values but a much shorter simulation
time (that would prevent convergence of the Monte Carlo simulation):

> run_emulation -flow hardware -args "-n hestonEuro -p 5.89 -c 8.71 -s 2"


Argument |  Meaning and default value
:-------- | :---
-n 	 | kernel name, to be passed to the OpenCL runtime (no default; must be hestonBarrier or hestonEuro)
-a 	 | OpenCl binary file name, to be passed to the OpenCL runtime (heston.xclbin)
-s       | number of simulation runs N (512)
-c       | expected call price (for verification)
-p       | expected put price (for verification)

The number of simulations N, and the number of time partitions M, as well as all the other parameters related to the simulation, are listed in ***"heston.cpp"***. They are compile-time constants in order to generate an optimized implementation. However, NUM_SIMGROUPS could also be set via a run-time command line option, like those above.

Parameter |  information
:-------- | :---
NUM_STEPS    | number of time steps (M)
NUM_RNGS | number of RNGs running in parallel, proportional to the area cost
NUM_SIMGROUPS  | number of simulation groups (each with ![$NUM\_RNG \cdot NUM\_SIMS$] simulations) running in pipeline, proportional to the execution time (512 optimizes BRAM usage)

The area cost is proportional to NUM_RNG.
### How to run an example
In each sub-directory, there is a script file called "solution.tcl". It can be used as follows:

>  sdaccel solution.tcl

The result of the call/put payoff price estimation will be printed on standard IO.

Please note that RTL simulation can take a very long time for the Heston model.
In order to obtain (imprecise) results quickly, the computation cost C can be
reduced. For instance, NUM_SIMS has been set to 2 in the TCL script. Better
results can be obtained with values like 512 or more.

## Performance Metrics


As discussed above, the computational cost ![$C=M \cdot N$] is a key factor that affects both the performance of the simulation and the quality of the result. The time complexity of the algorithm is O(C), so that we analyze the performance of our implementation as the total simulation time (number of clock cycles times clock period) per step: ![$t=T_s/C$]

The time taken by the algorithm is ![$$T=\alpha M \cdot N+\beta N+\gamma M+\theta$$] so for each step, ![$$t=T/C\approx\alpha$$]

**Basic Simulation procedure:**

>-  **Outer loop** (N iterations in total)

>>- **Inner loop** (M iterations)

>>> - Generate random numbers (unrolled loop)
>>> - Estimate stock price and volatility at each time partition point

>> - Calculate the payoff price
> - Count the sum of payoff prices
>
- Estimate the average

We can see that ![$\alpha$] is related to the latency of the inner loop. One of
the main factors that limit it is the latency of generating a random number.

### Performance Comparison
- Intel HD Graphics 4400 laptop GPU, with 80 cores, 1100MHz, 15W
- GeForce GTX 960 with 1024 cores, 1178MHz, 120W
- Quadro K4200 with 1344 cores, 784MHz, 108W
- Virtex 7 xc7vx690tffg1157-2, using the sin/cos functions, 160MHz
- Virtex 7 xc7vx690tffg1157-2, using the sinf/cosf functions, 133MHz

platform         |     t(ns)    | power(W)| energy/step(nJ)
:--------------- | ------------:| -------:| --------:
HD 4400          |     6.918    |  15     |  104    
GTX 960          |     0.604    |  88    |  53.15     
Quadro K4200     |     0.663    |  103    |  68.29     
Virtex 7 sin/cos |     1.424    |  12     |  17     
Virtex 7 sinf/cosf |     0.236    |  17.58     |  4.14     

[Heston Model]: https://en.wikipedia.org/wiki/Heston_model
[geometric Brownian motion]: https://en.wikipedia.org/wiki/Geometric_Brownian_motion
[Call options]: https://en.wikipedia.org/wiki/Call_option
[put options]: https://en.wikipedia.org/wiki/Put_option
[Mersenne Twister]: https://en.wikipedia.org/wiki/Mersenne_Twister
[Monte Carlo]: https://en.wikipedia.org/wiki/Monte_Carlo_method  
[Box Muller transformation]: https://en.wikipedia.org/wiki/Box%E2%80%93Muller_transform

[$\alpha$]:/figures/alpha.PNG
[$N=NUM\_SIMS \cdot NUM\_RNG \cdot NUM\_SIMGROUPS$]:figures/N.PNG
[$W_t$]:figures/wt.PNG
[$C=M \cdot N$]:figures/cmn.PNG
[$10^9$]:figures/109.PNG
[$2^{19937}-1$]:figures/19937.PNG
[$U_1$,$U_2 \sim U(0,1)$]:figures/u12.PNG
[$Z_1$,$Z_2\sim N(0,1)$]:figures/z12.PNG
[$NUM\_RNG \cdot NUM\_SIMS$]:figures/nn.PNG
[$t=T_s/C$]:figures/tstep.PNG
[$$Z_1=\sqrt{-2ln(U_1)}cos(2\pi U_2)\\Z_2=\sqrt{-2ln(U_1)}sin(2\pi U_2)$$]:/figures/boxm.PNG
[$t\approx\frac{clock\ period}{NUM\_RNGS}$]:/figures/tpro.PNG
[$$t=T/C\approx\alpha$$]:/figures/tmall.PNG
[$$T=\alpha M \cdot N+\beta N+\gamma M+\theta$$]:/figures/tall.PNG
