# BifurcationKit.jl

This Julia package aims at performing **automatic bifurcation analysis** of possibly large dimensional equations F(u,λ)=0 where λ∈ℝ.  

It incorporates continuation algorithms (PALC, deflated continuation, ...) which provide a *predictor* $(u_1,p_1)$ from a known solution $(u_0,p_0)$. A Newton-Krylov method is then used to correct this predictor and a Matrix-Free eigensolver is used to compute stability and bifurcation points.

> Despite initial focus on large scale problems, the package can easily handle low dimensional problems.

By leveraging on the above method, the package can also seek for periodic orbits of Cauchy problems by casting them into an equation $F(u,p)=0$ of high dimension. **It is by now, one of the only softwares which provides shooting methods AND methods based on finite differences or collocation to compute periodic orbits.**

The current package focuses on large scale nonlinear problems and multiple hardwares. Hence, the goal is to use Matrix Free methods on **GPU** (see [PDE example](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials2b/index.html#The-Swift-Hohenberg-equation-on-the-GPU-1) and [Periodic orbit example](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorialsCGL/#Continuation-of-periodic-orbits-on-the-GPU-(Advanced)-1)) or on a **cluster** to solve non linear PDE, nonlocal problems, compute sub-manifolds...

One design choice is that we try not to require `u` to be a subtype of an `AbstractArray` as this would forbid the use of spectral methods like the one from `ApproxFun.jl`. For now, our implementation does not allow this for Hopf continuation and computation of periodic orbits.

## Installation

This package requires Julia >= v1.3.0 because of the use of methods added to abstract types (see [#31916](https://github.com/JuliaLang/julia/pull/31916)).

To install it, please run

`] add BifurcationKit`

To install the bleeding edge version, please run

`] add BifurcationKit#master`

## Citing this work
If you use this package for your work, we ask that you **cite** the following paper!! Open source development strongly depends on this. It is referenced on HAL-Inria as follows:

```
@misc{veltz:hal-02902346,
  TITLE = {{BifurcationKit.jl}},
  AUTHOR = {Veltz, Romain},
  URL = {https://hal.archives-ouvertes.fr/hal-02902346},
  INSTITUTION = {{Inria Sophia-Antipolis}},
  YEAR = {2020},
  MONTH = Jul,
  KEYWORDS = {pseudo-arclength-continuation ; periodic-orbits ; floquet ; gpu ; bifurcation-diagram ; deflation ; newton-krylov},
  PDF = {https://hal.archives-ouvertes.fr/hal-02902346/file/354c9fb0d148262405609eed2cb7927818706f1f.tar.gz},
  HAL_ID = {hal-02902346},
  HAL_VERSION = {v1},
}
```

## Other softwares

There are many good softwares already available.

- For continuation in small dimension, most softwares are listed on [DSWeb](https://dsweb.siam.org/Software). One can mention the widely used AUTO-07p, or also, [XPPAUT](http://www.math.pitt.edu/~bard/xpp/xpp.html), [MATCONT](http://www.matcont.ugent.be/), [PyDSTool](https://github.com/robclewley/pydstool) and [COCO](https://sourceforge.net/projects/cocotools/). All these are very reliable and some address high codimensional bifurcations.

- For large scale problems, there is the versatile and feature full [pde2path](http://www.staff.uni-oldenburg.de/hannes.uecker/pde2path/) but also [Trilinos](https://trilinos.org/), [CL_MATCONTL](https://github.com/careljonkhout/cl_matcontL), [COCO](https://sourceforge.net/projects/cocotools/), and the python libraries [pyNCT](https://pypi.org/project/PyNCT/) and [pacopy](https://github.com/nschloe/pacopy).

- For *deflated continuation*, there is [defcont](https://bitbucket.org/pefarrell/defcon/src/master/) (by the inventor of the algo. P. E. Farrell) and this [code](https://github.com/evstigneevnm/deflated_continuation) by N. M. Evstigneev.

In Julia, we have for now a [wrapper](https://github.com/JuliaDiffEq/PyDSTool.jl) to PyDSTools, and also [Bifurcations.jl](https://github.com/tkf/Bifurcations.jl).

## A word on performance

The examples which follow have not **all** been written with the goal of performance but rather simplicity (except maybe [2d Ginzburg-Landau equation (finite differences)](@ref) and [1d Langmuir–Blodgett transfer model (advanced)](@ref)). One could surely turn them into more efficient codes. The intricacies of PDEs make the writing of efficient code highly problem dependent and one should take advantage of every particularity of the problem under study.

For example, in one of the simplest tutorials, [Temperature model (Simplest example)](@ref), one could use `BandedMatrices.jl` for the jacobian and an inplace modification when the jacobian is called ; using a composite type would be favored. Porting them to GPU would be another option.

## Main features

- Newton-Krylov solver with generic linear / eigen *preconditioned* solver. Idem for the arc-length continuation.
- Continuation written as an [iterator](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/iterator/)
- Monitoring user functions along curves computed by continuation, see [events](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/EventCallback/)
- Newton-Krylov solver with nonlinear deflation and preconditioner. It can be used for branch switching for example.
- Continuation methods: PALC, Moore Spence, Deflated continuation
- Bifurcation points are located using a bisection algorithm
- Branch, Fold, Hopf bifurcation point detection of stationary solutions.
- Automatic branch switching at branch points (whatever the dimension of the kernel)
- Automatic branch switching at simple Hopf points to periodic orbits
- **Automatic bifurcation diagram computation of equilibria**
- Fold / Hopf continuation based on Minimally Augmented formulation, with Matrix Free / Sparse Jacobian.
- detection of Bogdanov-Takens, Bautin and Cusp bifurcations
- Periodic orbit computation and continuation using Shooting, Finite Differences or Orthogonal Collocation.
- Branch, Fold, Neimark-Sacker, Period Doubling bifurcation point detection of periodic orbits.
- Computation and Continuation of Fold of periodic orbits

Custom state means, we can use something else than `AbstractArray`, for example your own `struct`.

**Note that you can combine most of the solvers, like use Deflation for Periodic orbit computation or Fold of periodic orbits family.**


|Features|Matrix Free|Custom state| [Tutorial](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/) | GPU |
|---|---|---|---|---|
| (Deflated) Krylov-Newton| Yes| Yes| All| Y|
| Continuation (Natural, Secant, Tangent, Polynomial) | Yes| Yes| All |Y  |
| Deflated Continuation | Yes| Yes| [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorialCarrier/#Deflated-Continuation-in-the-Carrier-Problem-1) |Y  |
| Branching / Fold / Hopf point detection | Yes| Yes| All / All / [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/#Bifurcation-diagrams-with-periodic-orbits-1) | Y |
| Fold Point continuation | Yes| Yes| [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials1/#Temperature-model-(simplest-example-for-equilibria)-1), [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorialsCGL/#Complex-Ginzburg-Landau-2d-1) | Y |
| Hopf continuation | Yes| `AbstractArray` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials3/#Continuation-of-Hopf-points-1) | |
| Branch switching at Branch / Hopf points | Yes| `AbstractArray` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/#Bifurcation-diagrams-with-periodic-orbits-1) | Y |
| **Automatic bifurcation diagram computation of equilibria** | Yes| `AbstractArray` |  [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/#Automatic-bifurcation-diagram-1) | |
| Periodic Orbit (Trapezoid) Newton / continuation | Yes| `AbstractVector` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials3/#Brusselator-1d-(automatic)-1), [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorialsCGL/#Complex-Ginzburg-Landau-2d-1) | Y|
| Periodic Orbit (Collocation) Newton / continuation | Yes| `AbstractVector` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/ode/tutorialsODE/#Neural-mass-equation-(Hopf-aBS)) | |
| Periodic Orbit with Parallel Poincaré / Standard Shooting Newton / continuation | Yes| `AbstractArray` |  [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/#Bifurcation-diagrams-with-periodic-orbits-1) | |
| Fold, Neimark-Sacker, Period doubling detection | Yes| `AbstractVector` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorials/#Bifurcation-diagrams-with-periodic-orbits-1)  | |
| Continuation of Fold of periodic orbits | Yes| `AbstractVector` | [Link](https://bifurcationkit.github.io/BifurcationKitDocs.jl/dev/tutorials/tutorialsCGL/#Complex-Ginzburg-Landau-2d-1) | Y |


## Requested methods for Custom State
Needless to say, if you use regular arrays, you don't need to worry about what follows.

We make the same requirements as `KrylovKit.jl`. Hence, we refer to its [docs](https://jutho.github.io/KrylovKit.jl/stable/#Package-features-and-alternatives-1) for more information. We additionally require the following methods to be available:

- `Base.length(x)`: it is used in the constraint equation of the pseudo arclength continuation method (see [`continuation`](@ref) for more details). If `length` is not available for your "vector", define it `length(x) = 1` and adjust tuning the parameter `theta` in `ContinuationPar`.
- `Base.copyto!(dest, in)` this is required to reduce the allocations by avoiding too many copies

## Citations
The papers citing this work are collected on [google scholar](https://scholar.google.fr/scholar?hl=fr&as_sdt=2005&cites=159498619004863176%2C8662907770106865595&scipsc=&as_ylo=&as_yhi=).
