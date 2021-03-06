---
title: 'fastSF: A parallel code for computing the structure functions of turbulence'

tags:
  - C++
  - structure functions
  - turbulence
  - fluid dynamics

authors:
  - name: Shubhadeep Sadhukhan
    orcid: 0000-0002-7278-5041
    affiliation: 1
  - name: Shashwat Bhattacharya
    orcid: 0000-0001-7462-7680
    affiliation: 2
  - name: Mahendra K. Verma
    orcid: 0000-0002-3380-4561
    affiliation: 1
  

affiliations:
 - name: Department of Physics, Indian Institute of Technology Kanpur, Kanpur 208016, India
   index: 1
 - name: Department of Mechanical Engineering, Indian Institute of Technology Kanpur 208016, India
   index: 2

date: 18 February 2020

bibliography: paper.bib

---

# Summary

Turbulence is a complex phenomenon in fluid dynamics involving nonlinear interactions between multiple scales. Structure function is a popular diagnostics tool to study the statistical properties of turbulent flows [@Kolmogorov:Dissipation; @Kolmogorov:Structure; @Frisch:book]. Some of the earlier works comprising of such analysis are those of @Gotoh:PF2002, @Kaneda:PF2003, and @Ishihara:ARFM2009 for three-dimensional (3D) hydrodynamic turbulence; @Yeung:PF2005 and @Ray:NJP2008 for passive scalar turbulence; @Biferale:NJP2004 for two-dimensional (2D) hydrodynamic turbulence; and @Kunnen:PRE2008, @Kaczorowski:JFM2013, and @Bhattacharya:PF2019 for turbulent thermal convection. Structure functions are two-point statistical quantities; thus, an accurate computation of these quantities requires averaging over many points. However, incorporation of a large number of points makes the computations very expensive and challenging. Therefore, we require an efficient parallel code for accurate computation of structure functions. In this paper, we describe the design and validation of the results of ``fastSF``, a parallel code to compute the structure functions for a given velocity or scalar field. 

 ``fastSF``, written in C++, is an application for efficiently computing the structure functions of scalar and vector fields on Cartesian grids of a 2D or 3D periodic box, stored as HDF5 files. The code employs MPI (Message Passing Interface) parallelization with equal load distribution and vectorization for efficiency on SIMD architectures. The user can select the range of the orders of the structure functions to be computed and the computed structure functions are written to HDF5 files that can be further processed by the user.

We are not aware of any other open soure or commercial packages for computing structure functions; prior studies have relied on in-house software that was never publicly released.  As an open source package, `fastSF` provides a standard high-performance implementation and thus facilitates wider use of structure functions.

 The code uses the following libraries:
 
1.   blitz++ (version 1.0.2)
2.   hdf5 (version 1.8.20)
3.   h5si (version 1.1.1)
4.   yaml-cpp (version 0.3.0)
5.   mpich (version 3.3.2)
6.   cmake (version 3.16.4)


In the next section, we will briefly discuss the performance and scaling of ``fastSF`` in a Cray XC40 system.



# Velocity and scalar structure functions

We denote the velocity and scalar fields using $\mathbf{u}$ and $\theta$  respectively. The velocity difference between any two points $\mathbf{r}$ and $\mathbf{r+l}$ is $\delta \mathbf{u} = \mathbf{u(r+l)}-\mathbf{u(r)}$. The difference in the parallel components of the velocity field along $\mathbf{l}$ is $\delta u_\parallel=\delta \mathbf{u}\cdot \hat{\mathbf{l}}$.  The corresponding difference in the perpendicular component is $\delta u_\perp= |\delta \mathbf{u} - \delta u_\parallel \hat{\mathbf{l}}|$. Assuming statistical homogeneity, we define the longitudinal velocity structure functions of order $q$ as
$$ S_q^{u_\parallel}(\mathbf{l}) = \langle (\delta u_\parallel)^q \rangle = \langle [\{\mathbf{u(r+l)}-\mathbf{u(r)}\}\cdot \hat{\mathbf{l}}]^q \rangle, \quad \quad (1)$$ 
and the transverse velocity structure functions of order 
$q$ as 
$$ S_q^{u_\perp}(\mathbf{l}) = \langle (\delta u_\perp)^q \rangle = \langle |\delta \mathbf{u} - \delta u_\parallel \hat{\mathbf{l}}|^q \rangle. \quad \quad (2)$$ 
Here, $\langle \cdot \rangle$ denotes ensemble averaging. Similarly, we can define the scalar structure functions for the scalar field as 
$$ S_q^\theta(\mathbf{l}) = \langle (\delta \theta)^q\rangle = \langle [\theta (\mathbf{r+l}) - \theta(\mathbf{r})]^q \rangle. \quad \quad(3)$$

For isotropic turbulence (in addition to being homogeneous), the structure functions become functions of $l$, where $l=|\mathbf{l}|$. The second-order velocity structure function $S_q^{u_{\parallel}}(l)$ provides an estimate for the energy in the eddies of size $l$ or less [@Davidson:book:Turbulence]. 

In the next section, we provide a brief description of the code.

# Design of the Code
First we present a sketch of the structure function computation for the velocity structure functions.  Typical structure function computations [Eqs (1-3)] in literature involve calculation of the velocity difference using loops over $\mathbf{r}$ and $\mathbf{l}$. These computations require six nested `for` loops for 3D fields that makes the computations very expensive for large grids. In our code, we employ vectorization and loops over only $\mathbf{l}$, thus requiring three loops instead of six for 3D fields. The new algorithm enhances the performance approximately 20 times over the earlier schemes due to vectorization.  In the following, we provide the algorithm for structure function computation for a 2D velocity field.

**Pseudo-code**

*Data*: Velocity field $\mathbf{u}$ in domain $(L_x, L_z)$; number of processors $P$.

*Procedure*:
 
* Divide $\mathbf{l}$'s among various processors. The process of data division among the processors  has been described later in this section. 
 
* For every processor:
     
    * for $\mathbf{l}= (l_x,l_z)$ assigned to the processor:
        
        * Compute $\delta \mathbf{u}(l_x,l_z)$ by taking the difference between two points with the same indices in pink and green subdomains as shown in Fig. \ref{Schematic}. This feature enables vectorized subtraction operation.
        
        * $\delta u_{\parallel}(l_x,l_z) = \delta \mathbf{u} \cdot \hat{\mathbf{l}}$ (Vectorized). 
        
        * $\delta u_{\perp}(l_x,l_z) = |\delta \mathbf{u} - \delta u_{\parallel} \hat{\mathbf{l}}$| (Vectorized). 
        
        * for order $q$:
        
            * $S_q^{u_{\parallel}}(l_x,l_z) =$ Average of $\delta u_{\parallel}^q$ (Vectorized).
            
            * $S_q^{u_{\perp}}(l_x,l_z) =$ Average of $\delta u_{\perp}^q$ (Vectorized).
            
            * Send the values of $S_q^{u_{\parallel}}(l_x,l_z)$, $S_q^{u_{\perp}}(l_x,l_z)$, $q$, $l_x$, and $l_z$ to the root process.
            
* The root process stores $S_q^{u_{\parallel}} (l_x, l_z)$ and $S_q^{u_{\perp}} (l_x, l_z)$.
            
* Stop

![The velocity difference $\delta \mathbf{u}(\mathbf{l})$ is computed by taking the difference between two points with the same indices in the pink and the green subdomains. For example, $\mathbf{u}(\mathbf{l}) - \mathbf{u}(0,0) = \mathbf{u}_B - \mathbf{u}_A$, where $B$ and $A$ are the origins of the green and the pink subdomains. This feature enables vecotrization of the computation. \label{Schematic}](docs/figs/Schematic.png)

Since $S_q^u(\mathbf{l})$ is important for intermediate scales (inertial range) only, we vary $\mathbf{l}$ upto half the domain size, that is, upto ($L_x/2, L_z/2$), to save computational cost. The $\mathbf{l}$'s are divided among MPI processors along $x$ and $z$ directions. Each MPI processor computes the structure functions for the points assigned to it and has access to the entire input data. Thus, we save considerable time that would otherwise be spent on communication between the processors during the calculation of the velocity difference. After computing the structure function for a given $\mathbf{l}$, each processor communicates the result to the root process, which stores the $S_q^{u_\parallel}(\mathbf{l})$ and $S_q^{u_\perp}(\mathbf{l})$ arrays.

It is clear from Fig. \ref{Schematic} that the sizes of the pink or green subdomains are $(L_x-l_x)(L_z-l_z)$, which are function of $\mathbf{l}$'s.  This function decreases with increasing $\mathbf{l}$ leading to larger computational costs for small $l$ and less cost of larger $l$.   Hence, a straightforward division of the domain among the processors along $x$ and $z$ directions will lead to a load imbalance.   Therefore, we assign both large and small $\mathbf{l}$'s to each processor to achieve equal load distribution. We illustrate the above idea  using the following example.

Consider a one-dimensional domain of size $L=15$, for which the possible $l$'s are
$$l=\{0, 1, 2, 3 ... 15\}.$$ 
We need to compute the structure functions for $l$ ranging from 0 to 7. We divide the task among four processors, with two $l$'s assigned to each processor. The following distribution of $l$'s ensures equal load distribution:
$$\mbox{Processor 0: } \quad l=\{0,7\}, \quad \sum(L-l)=(15-0)+(15-7) = 23,$$
$$\mbox{Processor 1: } \quad l=\{1, 6\}, \quad \sum(L-l)=(15-1)+(15-6) = 23,$$
$$\mbox{Processor 2: } \quad l=\{2,5\}, \quad \sum(L-l)=(15-2)+(15-5) = 23,$$
$$\mbox{Processor 3: } \quad l=\{3, 4\}, \quad \sum(L-l)=(15-3)+(15-4) = 23.$$
Similarly, if two processors are used, then the following distribution results in load balance. 
$$\mbox{Processor 0: } \quad l=\{0, 7, 2, 5\},$$
$$\mbox{Processor 1: } \quad l=\{1, 6, 3, 4\}.$$
 This idea of load distribution has been implemented in our program and has been extended to higher dimensions. 

Note that for 2D, $l_x>0$, but $l_z$ can take both positive and negative values. However, for isotropic turbulence, the structure functions for $+l_z$ and $-l_z$ are statistically equal. Therefore, in our computations, we restrict  to $l_x>0$, $l_z>0$. For anisotropic turbulence, not discussed here, the structure functions will depend on $(l_x,l_z)$ rather than $l$; our code will be extended to such systems in future. 

For 3D turbulence, the structure functions will depend on $(l_x,l_y,l_z)$. We divide the tasks among processors over $l_x$ and $l_y$ as done  above for 2D turbulence. The aforementioned algorithm can be easily extended to the 3D case. We employ a similar method for the computation of scalar structure functions as well.
 
In the next section, we discuss the scaling of our code.

# Scaling of `fastSF`

`fastSF` is scalable over many processors due to vectorization and equal load distribution. We demonstrate the scaling of `fastSF` for the third-order longitudinal structure function for an idealized velocity field on a $128^3$ grid.  For our computation we employ a maximum of 1024 cores. We take the velocity field as
$$\mathbf{u} = 
\begin{bmatrix} 
x \\ y \\z
\end{bmatrix}.$$
We perform four runs on a Cray XC40 system (Shaheen II of KAUST) for this problem using a total of 16, 64, 256, and 1024 cores. We used 16 cores per node for each run. In Fig. \ref{Scaling}, we plot the inverse of time taken in seconds versus the number of cores. The best fit curve for these data points yields
$$T^{-1} \sim p^{0.986 \pm 0.002},$$
Thus, the data-points follow $T^{-1} \sim p$ curve to a good approximation. Hence, we conclude that our code exhibits good scaling. 

![Scaling of `fastSF` for the computation of third-order longitudinal velocity structure function using 16, 64, 256, and 1024 processors of Shaheen II. All the runs were conducted on a $128^3$ grid.  We observe a linear scaling. \label{Scaling}](docs/figs/SF_scaling.png)


# Conclusions

This paper provides a brief description of ``fastSF``, an efficient parallel C++ code that computes structure functions for given velocity and scalar fields. This code is shown to be scalable over many processors. We are currently using this code to investigate the structure functions of two-dimensional turbulence with large-scale forcing. An earlier version of the code was used by @Bhattacharya:PF2019 for analyzing the structure functions of turbulent convection.  We believe that ``fastSF`` will be useful to turbulence community because of its efficiency and scalability.  


# Acknowledgements

We thank R. Samuel, A. Chatterjee, S. Chatterjee, and M. Sharma for useful discussions during the development of ``fastSF``. Our computations were performed on Shaheen II at KAUST supercomputing laboratory, Saudi Arabia, under the project k1416. 

---

# References


