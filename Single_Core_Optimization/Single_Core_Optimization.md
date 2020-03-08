# Single Core Optimization

## Introduction: *Performance of Computing Platforms.*

In the last few years the exponential increase in CPU frequency results in a "free" speedup for numerical software. In other words, legacy codes written for a predecessor will run faster without any extra effort. However, the performance of computers has evolved due to increases in the processors' parallelism (vector processing, multi-threading, SIMD, multicore, etc). 

Ideally, compilers would take care of this porblem by automatically vectorizing and parallelizing existing source code. However, this not usually happens. Most time taking advantage of platform's available parallelism often requires an algorithm structured differently than
the one that would be used in the corresponding sequential code.

Similar problems are caused by the computer's memory hierarchy (see Fig 1.), independently of the available parallelism. Moving data from and to memory has become the bottleneck. The memory hierarchy, aims to address this problem, but can only work if data is accessed in a suitable order. Compilers are inherently limited in optimizing for the memory hirarchy since this optimizations may require algorithm restructuring. 

![image](./images/memory-hierarchy.png)

Adding to these problems is the fact that CPU frequency scaling is approaching its en due to limits to the chip's possible power density (see Fig 2.). This implies the end of "automatic speedup"; then, in order to obtain performance gains, we need to parallelize. 

![image](./images/intel_performance.png)


The first step before parallelize code is to optimize our sequential code. This type of optimizations involves not only how to use well the memory resources but also how to avoid, when it is possible,  compute more instructions than the ones we need.

Before start with some examples we are going to introduce some helpful tools which allow us:

* To know a new tool to meassure time (in sec) in an easy way.
* To know information about our cpu resources
* To know where is the bottleneck in our code (target to optimize)
* To know information about cache-references, cache-misses, time, etc.

## Functions to measure elapsed time

### gettimeofday

This function can be used to measure wall-clock time on Unix-based operating 
systems. It requires the header file `sys/time.h`. The following is an example 
of its usage:

```
#include<stdio.h>
#include<sys/time.h>
int main()
{
    struct timeval start, end;
    double timeTaken;

    gettimeofday(&start, NULL);
    do_something();
    gettimeofday(&end, NULL);

    timeTaken  = (end.tv_sec - start.tv_sec);         // seconds elapsed
    timeTaken += (end.tv_usec - start.tv_usec)/1e6;   // additional microseconds elapsed
    
    printf("Time taken: %f seconds", &timeTaken);
}
```

### omp_get_wtime

(Need to install libgomp1, the GCC OpenMP (GOMP) support library)

Performs the same function as the previous command: gives the elapsed wall clock time in seconds. The time is measured per thread, no guarantee can be made that two distinct threads measure the same time. Time is measured from some "time in the past", which is an arbitrary time guaranteed not to change during the execution of the program.

C/C++:
    Prototype:     double omp_get_wtime(void); 

To use this tool in our code we should add:

```
	#include <omp.h>
```

When we compile:

```
	gcc -o our_program our_program.c -fopenmp -lgomp 
```

In the examples you can see how we use this tool to meassure time. However, [here](http://msdn.microsoft.com/en-us/library/x721b5yk.aspx) is a general example of how to use this function.

## CPU Information

### Linux

`lscpu` and `cat /proc/cpuinfo`: If you type these two commands in your terminal in Linux, you can obtain useful information about your CPU resources. Try it and experiment by yourself!!!

Example:

```
> lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                8
On-line CPU(s) list:   0-7
Thread(s) per core:    2
Core(s) per socket:    4
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 45
Stepping:              7
CPU MHz:               3599.827
BogoMIPS:              7200.05
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              10240K
NUMA node0 CPU(s):     0-7
```

### Mac OS X

The command `sysctl hw` displays information about the CPU on Mac computers.

## Profiling code

### gprof (GNU profiler)

Profiling allows you to learn where your program spent its time and which functions called which other functions while it was executing. This information can show you which pieces of your program are slower than you expected, and might be candidates for rewriting to make your program execute faster.

How to use it: 

**Step 1:**


Compile and link your program with profiling enabled, when we compile we should introduce the flag -pg

```
	gcc -o our_program our_program.c -pg
```

**Step 2:**

Execute your program to generate a profile data file


```
	./our_program 
```

You program will write the profile data into a file called `gmon.out' just before exiting.

**Step 3:**

Run `gprof` to analyze the profile data

After you have a profile data file `gmon.out'`, you can run gprof to interpret the information in it. The gprof program prints a flat profile and a call graph on standard output. Typically you would redirect the output of `gprof` into a file with `>`. 

```
	 gprof our_program gmon.out > profile.txt
```

Then `profile.txt` is going to contain lot of useful information but the first lines should look like this:

```
    %       cumulative    self            self     total
    time     seconds    seconds   calls  ms/call  ms/call   name

    92.68      0.38       0.38      1    380.00    380.00   function2
    7.32       0.41       0.03      1     30.00     30.00   function1
```

You can try using gprof in your own program and see where is the bootleneck of your code. 

For more information about `gprof`, [here](https://www.cs.utah.edu/dept/old/texinfo/as/gprof.html) there is a manual.


### perf (Linux profiling with performance counters)

The perf tool offers a rich set of commands to collect and analyze performance and trace data.

First we need to install perf so:

```
    sudo apt-get install linux-tools-common
```

Common comands:

 * perf stat: obtain event counts
 * perf record: record events for later reporting
 * perf report: break down events by process, function, etc.
 * perf annotate: annotate assembly or source code with event counts

If we want to measure more than one event, simply provide a comma separated list with no space, e.g: 

```
    perf stat -e instructions,cache-misses,cache-references ./our_program
```

To learn more about perf, click [here](https://perf.wiki.kernel.org/index.php/Tutorial)


## Let's start with optimizations

1. Without modifying code - using compiler flags

    See what happens when you use `-O0`, `-O1`, `-O2`, `-O3`, `-Ofast`.
    To see which flags are enabled and disabled with these optimization levels, 
    run the following in the command line:
    
    ```
    gcc -c -Q -O3 --help=optimizers
    ```
    
    If you want to add a flag that is not included in `-O<number>`, you can 
    choose to add it directly as a flag when you compile.
    
    In the folder `examples`, you can find the code `example_1.c`. Try to run it
    and see what happens using different flags.
    Also available are the assembly codes. So you can see the magic 
    of the compiler.

2. If you can, avoid unnecessary `if` statements.

    For example: non-periodic boundary conditions
    
    ```C
        1 void compute1(float a[], const unsigned int N) {
        2     for (unsigned int i=0; i<N; ++i) {
        3         if (i==0) {
        4             a[i] = a[i+1]/2;
        5         } else if (i==N-1) {
        6             a[i] = a[i-1]/2;
        7         } else { // 0<i<N-1
        8             a[i] = (a[i-1]+a[i+1])/2;
        9         }
        10     }
        11 }
    
    ``` 
    
    Is better if we write it like this:
    

    ```C
    1 void compute2(float a[], const unsigned int N) {
    2     a[0]=a[1]/2;
    3     for (unsigned int i=1; i<N-1; ++i) {
    4         a[i] = (a[i-1]+a[i+1])/2;
    5     }
    6     a[N-1]=a[N-2]/2;
    7 }
    ``` 

    In folder `examples`, you can find the code `example_2.c` which justifies this. 
    Besides, you can try it with different flags of compilation and see what happens.

    > **Note:**
    >
    > You will see a difference in the run-time when you use the `-O0` flag. But the 
    > difference between the functions `compute1` and `compute2` is negligible when 
    > using the `-O3` flag. This is because the compiler optimizes branch prediction.

These codes are easy codes for the compiler, most times happens that the compiler does not optimize automatically so you need to work in the code. 
    
--------------------------------------------------------------------------
### What makes sense to do?

* A little of loop unrolling: 
        Disassemble dependencies using records to achieve more Instruction 
        Level Parallelism (ILP).
        It has a limit: *register spilling*.

* Expose paralellism, when is possible, so the compiler can vectorize 
automatically.

* Cache Optimizations:
 - Rearranging loops. 
 - Cache blocking.
 - Memory alignment.

* Indicate there is no aliasing.

--------------------------------------------------------------------------

### Interesting examples:

#### B = A^t (transposed)

    * Arithmetic intensity = number of FLOPs per memory access = 0/ (2 ∗ L^2) = 0

    * We have a memory bound problem.

The matrix in the memory looks like:

![image](./images/mat_in_mem.jpg)

### Naïve code:

```C 
    1 #define L (1<<11)
    2 
    3 float a[L][L], b[L][L];
    4 
    5 int main(void) {
    6     unsigned int i=0, j=0;
    7     for (i=0; i<L; ++i)
    8             for (j=0; j<L; ++j)
    9                 b[j][i] = a[i][j];
    10     return (int)b[(int)b[0][2]][(int)b[2][0]];
    11 }
```
Code available in folder examples as `mtxtransp1.c` 

If we run this code with perf (-r 4 means run 4 times and do an average), we obtain:

We also compile with -O0 to avoid optimizations and see the raw behaviour.

```
perf stat -r 4 -e instructions,cycles,cache-references,cache-misses ./mtxtransp1
```

```
Performance counter stats for './mtxtransp1' (4 runs):

       67.429.332 instructions              #    0,14  insns per cycle          ( +-  8,75% ) [75,45%]
       483.118.384 cycles                     ( +-  1,50% ) [75,39%]
       4.657.399 cache-references                                              ( +-  0,30% ) [75,73%]
       4.545.049 cache-misses              #   97,588 % of all cache refs      ( +-  1,11% ) [49,25%]

       0,233285113 seconds time elapsed                                          ( +-  3,62% )

```

### Doing cache blocking:

Modify the portion that performs the transpose in code `mtxtransp2.c` in the
folder `examples`:
```C
for (i=0; i<L; i+=BY)
        for (j=0; j<L; j+=BX)
            for (ii=i; ii<i+BY; ++ii)
                for (jj=j; jj<j+BX; ++jj)
                    b[jj][ii] = a[ii][jj];
```

Run `perf` as before:

```
Performance counter stats for './mtxtransp2' (4 runs):

        83.762.004 instructions              #    0,59  insns per cycle          ( +-  1,02% ) [75,36%]
       142.685.092 cycles                     ( +-  1,08% ) [76,10%]
         4.792.723 cache-references                                              ( +-  1,41% ) [77,15%]
           650.754 cache-misses              #   13,578 % of all cache refs      ( +-  2,30% ) [48,54%]

       0,067984763 seconds time elapsed  
```

*Speedup*: 0.233/0.067 = 3.5x :D

If now we run both codes with `-O3` we obtain: 

*Speedup*: 0.19/0.046 = 4.1x :D :D

You can try different values of `BX` and `BY` and see how the performance varies. But we are lucky because someone ran different possible combinations and found the best blocking for this problem is `BX=2^1` and `BY=2^8`.

![image](./images/best_blocking.png)

*Speedup*: 0.19/0.035 = 5.4x :D :D :D

> **Note:**
>
> The above optimal values of `BX` and `BY` are specific to the hardware. 
> The values will depend on the processor on which the code is run (specifically, the properties of the cache.)

## Limits and Problems:

* [Roof line model](http://www.eecs.berkeley.edu/Pubs/TechRpts/2008/EECS-2008-134.pdf)

## References
[1] [Optimizing software in C++](http://www.agner.org/optimize/optimizing_cpp.pdf). Agner Fog. Technical University of Denmark.

[2] [How To Write Fast Numerical Code: A Small Introduction](http://users.ece.cmu.edu/~franzf/papers/gttse07.pdf). Srinivas Chellappa, Franz Franchetti, and Markus Puschel

[3] Classes notes from "Computación paralela - 2014 FaMAF-UNC". Nicolás Wolowick


















 

