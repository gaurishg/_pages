---
title: Cholesky (LDLT) Decomposition of a symmetric positive definite matrix using MPI
date: 2023-06-16 12:26:00 +0900
categories: [C++, MPI]
tags: [c++, parallel-programming, mpi, distributed-computing]     # TAG names should always be lowercase
math: true
---

# Introduction
This project was done as the final assignment for course 4810-1115 Parallel Numerical Computation. The course dealt with parallelizing numerical algorithms using OpenMP (shared memory parallelization) and MPI (distributed systems).

## Problem statement
Given a symmetric positive definite matrix 

$$A=\begin{bmatrix}
a_{11} & a_{12} & a_{13} & \cdots & a_{1n}\\
a_{21} & a_{22} & a_{23} & \cdots & a_{2n}\\
\vdots & \vdots & \vdots & \ddots & \vdots\\
a_{n1} & a_{n2} & a_{n3} & \cdots & a_{nn}
\end{bmatrix}$$

factorize it in two matrices 

$$L = \begin{bmatrix}
1      & 0      & 0      & \cdots & 0\\
l_{21} & 1      & 0      & \cdots & 0\\
\vdots & \vdots & \vdots & \ddots & \vdots\\
l_{n1} & l_{n2} & l_{n3} & \cdots & 1
\end{bmatrix}$$

and

$$D = \begin{bmatrix}
d_{11} & 0      & 0      & \cdots & 0\\
0      & d_{22} & 0      & \cdots & 0\\
\vdots & \vdots & \vdots & \ddots & \vdots\\
0      & 0      & 0      & \cdots & d_{nn}
\end{bmatrix}$$

such that $$A = LDL^T$$

# Algorithm
Values of the $L$ and $D$ matrices can be calculated using the following formulae

$$l_{ij} = \frac{a_{ij} - \sum_{k=1}^{j-1}d_{kk}l_{jk}l_{ik}}{d_{jj}}$$

$$d_{ii} = a_{ii} - \sum_{k=1}^{i-1}d_{kk}l_{ik}^2$$

## Sequential Code
The formulae given above can be very easily implemented into code. The code that I used in my report is as follows

```cpp
void ldlt_sequential(const double* const A, double *L, double *d, const int n)
{
    // Fill upper part with zeroes
    for (int r=0; r < n - 1; ++r)
    {
        std::fill(L + r * n + (r + 1), L + (r + 1) * n, 0.0);
    }
    for (int j=0; j<n; ++j)
    {
        d[j] = A[j*n+j];
        for (int k=0; k<j; ++k)
            d[j] -= d[k] * L[j*n+k] * L[j*n+k];
        
        L[j*n+j] = 1;
        for (int i=j+1; i<n; ++i)
        {
            L[i*n+j] = A[i*n+j];
            for (int k=0; k<j; ++k)
                L[i*n+j] -= d[k] * L[j*n+k] * L[i*n+k];
            L[i*n+j] /= d[j];
        }
    }
}

```

# Parallelization using MPI

![Figure 1: Data needed from $L$ to calculate an element of $D$](/assets/images/cholesky-decomposition-mpi/dep1.png){: width="600"}
_Figure 1: Data needed from $L$ to calculate an element of $D$_


![Figure 2: Full rows $i$ and $j$ of $L$ are needed to calculate $L[i, j]$](/assets/images/cholesky-decomposition-mpi/dep2.png){: width="600"}
_Figure 2: Full rows $i$ and $j$ of $L$ are needed to calculate $L[i, j]$_

## Data Dependencies
We can see that in the calculations, every column of $L$ and $D$ needs the values of their previous columns already calculated, however calculation of rows can be done a bit independently.

### $D$ Matrix
For $D$, we can see from the equations that to get any element of $D$ we need all its previous elements and the complete row of the matrix $L$. We can see the dependency in figure 1, where dark blue columns are the ones which are needed.

### $L$ Matrix
If we are calculating say column $i$, than we need all the previous values of $D$ and the complete row $i$ of $L$. In the figure 2, we can see that for calculation of $L[r, 4]$ we need $L[4, :]$ and $L[r, :]$.

## How the parallelization is done
Since every column needs the values of its previous columns so there is not an easy way to parallelize that, so I have gone through all columns in sequential manner but in every column I have divided the work for rows to different processes.

### Calculating $d_{ij}$
Every _rank_ is resposnsible for calculating values of rows that are multiple of rank. In this way the value of $D$ can also be divided among processes because it already has the previous computations from that row so it does not need any data to be communicated to it for calculation of $d_{ii}$. After calculation the values of $d_{ii}$ are sent to all other processes since it will be needed for calculation of $L_{ij}$.

### Calculating $L_{ij}$
As soon as we start to calculate the values of a column, say column $j$, we broadcast the $j^{th}$ row to all the processes since it is needed as shown in figure 2. All the processes do their work independently and at the last they update the values when they receive the value of $d_{ii}$. They do not send the calculated values anywhere at this point of time, so there is no communication overhead.

# Parallelized code using MPI
```cpp
void ldlt_mpi(const double *const A, double *L, double *d, const int n)
{
    int rank, n_pr;
    MPI_Request req_for_row, req_for_dj;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &n_pr);

    // Fill with zeroes
    std::fill(L, L + n * n, 0.0);

    // Have to go column wise, can not be parallelized
    for (int j = 0; j < n; ++j)
    {
        L[j * n + j] = 1;

        // This row is needed by everyone so start sending it
        MPI_Ibcast(L + j * n, j, MPI_DOUBLE, j % n_pr, MPI_COMM_WORLD, &req_for_row);
        if (j % n_pr == rank)
        {
            d[j] = A[j * n + j];
            for (int k = 0; k < j; ++k) 
                d[j] -= d[k] * L[j * n + k] * L[j * n + k];
        }
        // Value for d[j] will be needed by every row so start sending it
        MPI_Ibcast(d + j, 1, MPI_DOUBLE, j % n_pr, MPI_COMM_WORLD, &req_for_dj);
        // Now the row is needed to compute values
        MPI_Wait(&req_for_row, MPI_STATUS_IGNORE);

        // Distribute the work for rows
        for (int i = j + 1; i < n; ++i)
        {
            if (i % n_pr == rank)
            {
                L[i * n + j] = A[i * n + j];
                for (int k = 0; k < j; ++k)
                    L[i * n + j] -= d[k] * L[j * n + k] * L[i * n + k];
            }
        } // work for each row ends

        MPI_Wait(&req_for_dj, MPI_STATUS_IGNORE);
        for (int i = j + 1; i < n; ++i)
            L[i * n + j] /= d[j];
    } // work for each column ends
} // function ldlt_mpi ends
```

# Verification of Results of Parallel Code
For verification that the result obtained using parallel algorithm are correct, I have simply used the product $LDL^T$ and checked it against $A$, which was found correct for all the combination of MPI processes and matrix sizes. Matrix multiplication was done sequentially.

# Performance
The code was run for various combination of matrix sizes and number of MPI processes and considerable speedup was seen both in terms of strong and weak scaling. Complete results are shown in table 1.


<table>
    <caption>Table 1: Running time (in ms) for different values of MPI process and matrix sizes</caption>
    <thead>
        <tr>
            <th rowspan=2>Mat size</th>
            <th rowspan=2>Sequential</th>
            <th colspan=11><center>Parallelized using MPI (number of processes)</center></th>
        </tr>
        <tr>
            <th>2</th>
            <th>4</th>
            <th>8</th>
            <th>16</th>
            <th>32</th>
            <th>40</th>
            <th>64</th>
            <th>80</th>
            <th>128</th>
            <th>256</th>
            <th>512</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>100</td>
            <td>0.7</td>
            <td>1.1</td>
            <td>1.27</td>
            <td>2</td>
            <td>2.96</td>
            <td>4.7</td>
            <td>6.7</td>
            <td>16.3</td>
            <td>20.4</td>
            <td>23.75</td>
            <td>27.4</td>
            <td>28.2</td>
        </tr>
        <tr>
            <td>200</td>
            <td>5.6</td>
            <td>4.63</td>
            <td>3.58</td>
            <td>4.1</td>
            <td>4.95</td>
            <td>6.65</td>
            <td>7.9</td>
            <td>13.67</td>
            <td>13</td>
            <td>26.83</td>
            <td>54.2</td>
            <td>53.65</td>
        </tr>
        <tr>
            <td>500</td>
            <td>86.8</td>
            <td>50</td>
            <td>30</td>
            <td>24</td>
            <td>22.2</td>
            <td>23.85</td>
            <td>25</td>
            <td>49.1</td>
            <td>75.3</td>
            <td>95.34</td>
            <td>101.7</td>
            <td>163</td>
        </tr>
        <tr>
            <td>750</td>
            <td>293.4</td>
            <td>159</td>
            <td>89</td>
            <td>60</td>
            <td>47.7</td>
            <td>45.24</td>
            <td>48</td>
            <td>48.27</td>
            <td>46</td>
            <td>70.3</td>
            <td>214.7</td>
            <td>159</td>
        </tr>
        <tr>
            <td>1000</td>
            <td>696.2</td>
            <td>367</td>
            <td>199</td>
            <td>123</td>
            <td>85.9</td>
            <td>75.3</td>
            <td>75</td>
            <td>76.3</td>
            <td>72</td>
            <td>81.4</td>
            <td>137</td>
            <td>332</td>
        </tr>
        <tr>
            <td>1500</td>
            <td>2490</td>
            <td>1215</td>
            <td>638</td>
            <td>363</td>
            <td>227.8</td>
            <td>180</td>
            <td>178</td>
            <td>169.14</td>
            <td>148</td>
            <td>160</td>
            <td>159</td>
            <td>437</td>
        </tr>
        <tr>
            <td>2000</td>
            <td>6234</td>
            <td>2901</td>
            <td>1477</td>
            <td>806</td>
            <td>477</td>
            <td>361</td>
            <td>348</td>
            <td>300</td>
            <td>257</td>
            <td>273.5</td>
            <td>261</td>
            <td>472</td>
        </tr>
        <tr>
            <td>2500</td>
            <td>12124</td>
            <td>6057</td>
            <td>2876</td>
            <td>1537</td>
            <td>905</td>
            <td>660</td>
            <td>622</td>
            <td>494</td>
            <td>412</td>
            <td>416</td>
            <td>397</td>
            <td>672</td>
        </tr>
        <tr>
            <td>3000</td>
            <td>20767</td>
            <td>10482</td>
            <td>5062</td>
            <td>2715</td>
            <td>1569</td>
            <td>1056</td>
            <td>981</td>
            <td>763</td>
            <td>612</td>
            <td>618</td>
            <td>579</td>
            <td>940</td>
        </tr>
        <tr>
            <td>3500</td>
            <td>32733</td>
            <td>16588</td>
            <td>8217</td>
            <td>4357</td>
            <td>2425</td>
            <td>1536</td>
            <td>1397</td>
            <td>1111</td>
            <td>860</td>
            <td>869</td>
            <td>786</td>
            <td>1241</td>
        </tr>
        <tr>
            <td>4000</td>
            <td>48573</td>
            <td>24454</td>
            <td>12376</td>
            <td>6527</td>
            <td>3614</td>
            <td>2239</td>
            <td>1956</td>
            <td>1551</td>
            <td>1193</td>
            <td>1197</td>
            <td>1053</td>
            <td>1604</td>
        </tr>
        <tr>
            <td>5000</td>
            <td>93340</td>
            <td>47194</td>
            <td>23773</td>
            <td>12335</td>
            <td>6658</td>
            <td>4015</td>
            <td>3512</td>
            <td>2840</td>
            <td>2248</td>
            <td>2251</td>
            <td>1769</td>
            <td>2504</td>
        </tr>
    </tbody>
</table>

## Weak Scaling
From table 1 we see that upto a certain level (approximately till $64$ processes) that increasing the number of processes allows us to solve bigger and bigger problems. For example in the sequential algorithm we can solve the problem with matrix size of $750 \times 750$ in around $300ms$, while with $64$ processes we can solve the problem for matrix size of $2000 \times 2000$ in the same time. In the figure 3 it is clearly seen that execution time decreases with increasing parallelism.

## Strong Scaling
It is clearly seen that offering more processors helps us solving the problems faster (if they are big enough to be benefitted from parallelization). For the matrix size of $5000 \times 5000$ the sequential algorithm takes $93340ms$, which is around $1.6$ minute, which using $80$ processes in parallel, the same problem is solved in less than $2.3s$. This can be seen in table 2 and figure 4.


<table>
    <thead>
        <tr>
            <th rowspan=2>Matrix Size</th>
            <th colspan=9><center>Speedup by parallelization for different number of MPI processes</center></th>
        </tr>
        <tr>
            <th>2</th>
            <th>4</th>
            <th>8</th>
            <th>32</th>
            <th>64</th>
            <th>80</th>
            <th>128</th>
            <th>256</th>
            <th>512</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>100</td>
            <td>0.64</td>
            <td>0.55</td>
            <td>0.35</td>
            <td>0.15</td>
            <td>0.04</td>
            <td>0.03</td>
            <td>0.03</td>
            <td>0.03</td>
            <td>0.02</td>
        </tr>
        <tr>
            <td>200</td>
            <td>1.21</td>
            <td>1.56</td>
            <td>1.37</td>
            <td>0.84</td>
            <td>0.41</td>
            <td>0.43</td>
            <td>0.21</td>
            <td>0.10</td>
            <td>0.10</td>
        </tr>
        <tr>
            <td>500</td>
            <td>1.74</td>
            <td>2.89</td>
            <td>3.62</td>
            <td>3.64</td>
            <td>1.77</td>
            <td>1.15</td>
            <td>0.91</td>
            <td>0.85</td>
            <td>0.53</td>
        </tr>
        <tr>
            <td>750</td>
            <td>1.85</td>
            <td>3.30</td>
            <td>4.89</td>
            <td>6.49</td>
            <td>6.08</td>
            <td>6.38</td>
            <td>4.17</td>
            <td>1.37</td>
            <td>1.85</td>
        </tr>
        <tr>
            <td>1000</td>
            <td>1.90</td>
            <td>3.50</td>
            <td>5.66</td>
            <td>9.25</td>
            <td>9.12</td>
            <td>9.67</td>
            <td>8.55</td>
            <td>5.08</td>
            <td>2.10</td>
        </tr>
        <tr>
            <td>1500</td>
            <td>2.05</td>
            <td>3.90</td>
            <td>6.86</td>
            <td>13.83</td>
            <td>14.72</td>
            <td>16.82</td>
            <td>15.56</td>
            <td>15.66</td>
            <td>5.70</td>
        </tr>
        <tr>
            <td>2000</td>
            <td>2.15</td>
            <td>4.22</td>
            <td>7.73</td>
            <td>17.27</td>
            <td>20.78</td>
            <td>24.26</td>
            <td>22.79</td>
            <td>23.89</td>
            <td>13.21</td>
        </tr>
        <tr>
            <td>2500</td>
            <td>2.00</td>
            <td>4.22</td>
            <td>7.89</td>
            <td>18.37</td>
            <td>24.54</td>
            <td>29.43</td>
            <td>29.14</td>
            <td>30.54</td>
            <td>18.04</td>
        </tr>
        <tr>
            <td>3000</td>
            <td>1.98</td>
            <td>4.10</td>
            <td>7.65</td>
            <td>19.67</td>
            <td>27.22</td>
            <td>33.93</td>
            <td>33.60</td>
            <td>35.87</td>
            <td>22.09</td>
        </tr>
        <tr>
            <td>3500</td>
            <td>1.97</td>
            <td>3.98</td>
            <td>7.51</td>
            <td>21.31</td>
            <td>29.46</td>
            <td>38.06</td>
            <td>37.67</td>
            <td>41.65</td>
            <td>26.38</td>
        </tr>
        <tr>
            <td>4000</td>
            <td>1.99</td>
            <td>3.92</td>
            <td>7.44</td>
            <td>21.69</td>
            <td>31.32</td>
            <td>40.72</td>
            <td>40.58</td>
            <td>46.13</td>
            <td>30.28</td>
        </tr>
        <tr>
            <td>5000</td>
            <td>1.98</td>
            <td>3.93</td>
            <td>7.57</td>
            <td>23.25</td>
            <td>32.87</td>
            <td>41.52</td>
            <td>41.47</td>
            <td>52.76</td>
            <td>37.28</td>
        </tr>
    </tbody>
</table>


# Conclusion
The problem was successfully parallelized using MPI and almost linear speedup was seen for sufficiently
large problem where computations were more than communication overheads.

![Figure 3: Matrix size vs execution time for various number of MPI processes](/assets/images/cholesky-decomposition-mpi/matrix_vs_time.png){: width="600"}
_Figure 3: Matrix size vs execution time for various number of MPI processes_


![Figure 4: Speedup of $5000 \times 5000$ matrix with different number of MPI processes](/assets/images/cholesky-decomposition-mpi/processes_vs_speedup.png){: width="600"}
_Figure 4: Speedup of $5000 \times 5000$ matrix with different number of MPI processes_

![Figure 5: Snapshot of submission using pjstat](/assets/images/cholesky-decomposition-mpi/pjstat.png){: width="600"}
_Figure 5: Snapshot of submission using pjstat_
