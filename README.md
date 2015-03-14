Computer Architecture: Parallelism & Locality Lab Description
=============================================================
Instructor: Prof. Mattan Erez

From smart phones, to multi-core CPUs and GPUs, to the world's largest supercomputers and web sites, parallel processing is ubiquitous in modern computing. The course provide a deep understanding of the fundamental principles and engineering trade-offs involved in designing modern parallel computing systems and teach parallel programming techniques necessary to effectively utilize these machines. Because writing good parallel programs requires an understanding of key machine performance characteristics, this course will cover both parallel hardware and software design.

Lab1. Locality in a CPU
-----------------------
In this lab, we use dense matrix-matrix multiplication to study the effects of locality on performance and power dissipation. We explore two programming styles in the domain of dense linear algebra: iterative and recursive with cache-aware and cache-oblivious algorithm and learn about 3 tools that can help with architecture optimizations (performance counters, CACTI and PIN).
    
    // Baseline matrix-matrix multiplication
    #include <stdlib.h>
    
    // define the matrix dimensions A is MxP, B is PxN, and C is MxN
    #define M 512
    #define N 512
    #define P 512
    
    // calculate C = AxB
    void matmul(float **A, float **B, float **C) {
      float sum;
      int   i;
      int   j;
      int   k;
    
      for (i=0; i<M; i++) {
        // for each row of C
        for (j=0; j<N; j++) {
          // for each column of C
          sum = 0.0f; // temporary value
          for (k=0; k<P; k++) {
            // dot product of row from A and column from B
            sum += A[i][k]*B[k][j];
          }
          C[i][j] = sum;
        }
      }
    }
    
    // function to allocate a matrix on the heap
    // creates an mXn matrix and returns the pointer.
    //
    // the matrices are in row-major order.
    void create_matrix(float*** A, int m, int n) {
      float **T = 0;
      int i;
    
      T = (float**)malloc( m*sizeof(float*));
      for ( i=0; i<m; i++ ) {
         T[i] = (float*)malloc(n*sizeof(float));
      }
      *A = T;
    }
    
    int main() {
      float** A;
      float** B;
      float** C;
    
      create_matrix(&A, M, P);
      create_matrix(&B, P, N);
      create_matrix(&C, M, N);
    
      // assume some initialization of A and B
      // think of this as a library where A and B are
      // inputs in row-major format, and C is an output
      // in row-major.
    
      matmul(A, B, C);
    
      return (0);
    }

Step1: Improve the performance of the baseline version above on a uni-processor. Most of the performance to be gained is with locality optimizations, followed by converting the code to use modern processor's short-vector SIMD units. 

Step2: Use the performance counters to measure the number of cycles, instructions and cache hits and misses required by our application to calculate GFLOPS. (Although PAPI package is the best supported, we use perf tool instead for the convenience of teaching.) 

Step3: Try both cache-aware and cache-oblivious methods to improve locality and hence performance (optimizing registers as well as the memory hierarchy).

Step4: Modify existing PIN tools to implement a read- and write-allocate, write-back, inclusive 2-level cache model with LRU replacement. [[dcache.H](./lab1/dcache.H), [dcache.cpp](./lab1/dcache.cpp)]

Step5: Use CACTI tool to estimate the power and energy consumed by the matrix multiplication and the potential power savings of locality optimizations.

[Here](./lab1) are codes of Lab1.

Lab2.  A Simple CUDA Renderer
-----------------------------
100 points total

![image](https://github.com/sparkfiresprairie/capl/blob/master/lab2/lab2.png)

In this lab, we write a parallel renderer in CUDA that draws colored circles. While this renderer is very simple, parallelizing the renderer will require us to design and implement data structures that can be efficiently constructed and manipulated in parallel.

###Part1 - CUDA Warm-Up 1: SAXPY (5 pts)
Warm-up task to implement the SAXPY in CUDA. Compare the performance with the sequential CPU-based implementation of SAXPY (time, bandwidth, etc) [[saxpy.cu](./lab2/saxpy.cu)]

###Part2 - CUDA Warm-Up 2: Parallel Prefix-Sum (10 pts)
In this part, we are asked to come up with parallel implementation of the function find_repeats which, given a list of integers A, returns a list of all indices i for which A[i] == A[i+1]. For example, given the array {1,2,2,1,1,1,3,5,3,3}, our program should output the array {1,3,4,8}. We implement find_repeats by first implementing parallel exclusive prefix-sum operation. 

The following "C-like" code is an iterative version of scan. We use parallel_for to indicate potentially parallel loops.
 
     void exclusive_scan_iterative(int* start, int* end, int* output)
    {
        int N = end - start;
        memmove(output, start, N*sizeof(int));
        // upsweep phase.
        for (int twod = 1; twod < N; twod*=2)
        {
         int twod1 = twod*2;
         parallel_for (int i = 0; i < N; i += twod1)
         {
             output[i+twod1-1] += output[i+twod-1];
         }
        }
    
        output[N-1] = 0;
    
        // downsweep phase.
        for (int twod = N/2; twod >= 1; twod /= 2)
        {
         int twod1 = twod*2;
         parallel_for (int i = 0; i < N; i += twod1)
         {
             int t = output[i+twod-1];
             output[i+twod-1] = output[i+twod1-1];
             output[i+twod1-1] += t; // change twod1 to twod to reverse prefix sum.
         }
        }
    }
    
Code correctness and performance are tested on random input arrays. For reference, a scan score table is provided below, showing the performance of a simple CUDA implementation on a stampede cluster with a K20. [[scan.cu](./lab2/scan.cu)]
   
    -------------------------
    Scan Score Table:
    -------------------------
    -------------------------------------------------------------------------
    | Element Count   | Fast Time       | Your Time       | Score           |
    -------------------------------------------------------------------------
    | 10000           | 0.387           | 0.010 (F)       | 0               |
    | 100000          | 0.770           | 0.070 (F)       | 0               |
    | 1000000         | 1.771           | 0.167 (F)       | 0               |
    | 2000000         | 2.799           | 0.150 (F)       | 0               |
    -------------------------------------------------------------------------
    |                                   | Total score:    | 0/5             |
    -------------------------------------------------------------------------

###Part3 - A Simple Circle Renderer (85 pts)

