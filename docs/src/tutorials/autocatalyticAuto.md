# Fronts in 1d autocatalytic model (Automatic)

```@contents
Pages = ["autocatalyticAuto.md"]
Depth = 3
```

We consider the following model [^Balmforth][^Malham] which is also treated in [^Beyn]

$$\begin{array}{l}
u_{t}=a u_{x x}-u f(v), \quad a>0, u, v: \mathbb{R} \rightarrow \mathbb{R} \\
v_{t}=v_{x x}+u f(v)
\end{array}$$

where $f(u) = u^m 1_{u\geq 0}$. We chose the boundary conditions

$$\left(u_{-}, v_{-}\right)=(0,1),\quad \left(u_{+}, v_{+}\right)=(1,0)\tag{BC}.$$

It is straightforward to implement this problem as follows:

```@example TUTAUTOCATauto
using Revise
using ForwardDiff, SparseArrays
using BifurcationKit, LinearAlgebra, Plots, Setfield
const BK = BifurcationKit

# supremum norm
norminf(x) = norm(x, Inf)
f(u) = u^9 # solution are positive, so remove the heaviside

# helper function to plot solutions
function plotsol!(x; k...)
	u = @view x[1:end÷2]
	v = @view x[end÷2:end]
	plot!(u; label="u", k...)
	plot!(v; label="v", k...)
end
plotsol(x; k...) = (plot();plotsol!(x; k...))

# encode the nonlinearity
@views function NL!(dest, U, p, t = 0.)
	N = p.N
	u = U[1:N]
	v = U[N+1:2N]
	dest[1:N]    .= -u .* f.(v)
	dest[N+1:2N] .=  -dest[1:N]#u .* f.(v)
	return dest
end

# function for the differential with specific boundary conditions
# for fronts
@views function applyD_add!(f, U, p, a)
	uL = 0; uR = 1; vL = 1; vR = 0
	n = p.N
	u = U[1:n]
	v = U[n+1:2n]

	c1 = 1 / (2p.h)
	f[1]   += a * (uL      - u[2] ) * c1
	f[end] += a * (v[n-1]  - vR   ) * c1

	f[n]   += a * (u[n-1] - uR  ) * c1
	f[n+1] += a * (    vL - v[2] ) * c1

	@inbounds for i=2:n-1
		  f[i] += a * (u[i-1] - u[i+1]) * c1
		f[n+i] += a * (v[i-1] - v[i+1]) * c1
	end
	return f
end

# function which encodes the full PDE
@views function Fcat!(f, U, p, t = 0)
	uL = 0; uR = 1; vL = 1; vR = 0
	n = p.N
	# nonlinearity
	NL!(f, U, p)

	# Dirichlet boundary conditions
	h2 = p.h * p.h
	c1 = 1 / h2

	u = U[1:n]
	v = U[n+1:2n]

	f[1]   += p.a * (uL      - 2u[1] + u[2] ) * c1
	f[end] +=       (v[n-1]  - 2v[n] + vR   ) * c1

	f[n]   += p.a * (u[n-1] - 2u[n] +  uR  ) * c1
	f[n+1] +=       (vL - 2v[1] + v[2] ) * c1

	@inbounds for i=2:n-1
		  f[i] += p.a * (u[i-1] - 2u[i] + u[i+1]) * c1
		f[n+i] +=       (v[i-1] - 2v[i] + v[i+1]) * c1
	end
	return f
end
Fcat(x, p, t = 0.) = Fcat!(similar(x), x, p, t)
Jcat(x,p) = sparse(ForwardDiff.jacobian(x -> Fcat(x, p), x))
nothing #hide
```

We chose the following parameters:

```@example TUTAUTOCATauto
N = 200
lx = 25.
X = LinRange(-lx,lx, N)
par_cat = (N = N, a = 0.18, h = 2lx/N)

u0 = @. (tanh(2X)+1)/2
U0 = vcat(u0, 1 .- u0)
nothing #hide
```

## Freezing method

The problem may feature fronts, that is solutions of the form $u(x,t) = U(x-st)$ (same for $v$) for a fixed value of the profile $U$ and the speed $s$. The equation for the front profile is, up to an abuse of notations

$$\begin{array}{l}
0=a u_{\xi\xi}+su_{\xi}-u f(v)\\
0=v_{\xi\xi}+sv_{\xi}+u f(v)
\end{array}$$

with unknowns $u,v,s$. The front is solution of these equations but it is not uniquely determined because of the phase invariance. Hence, we add the phase condition (see [^Beyn])

$$0 = \left\langle U, \partial_\xi U_0	\right\rangle$$

where $U_0$ is some fixed profile. This is encoded in the problem [`TWProblem`](@ref)

```@example TUTAUTOCATauto
using LinearAlgebra

# this structure encode the Lie generator
struct Advection{T}
	p::T
end

function LinearAlgebra.mul!(f, aD::Advection, U, α, β)
	rmul!(f, β)
	applyD_add!(f, U, aD.p, α)
end

uold = vcat(u0, (1 .- u0))

# we create a TW problem
probTW = BK.TWProblem(Fcat, Jcat, Advection(par_cat), copy(uold))
nothing #hide
```

We now define the $U_0$ profile

```@example TUTAUTOCATauto
uold = vcat(u0, (1 .- u0))
Duold = zero(uold); applyD_add!(Duold, uold, par_cat,1)

# update problem parameters for front problem
par_cat_wave = (par_cat..., u0Du0 = dot(uold, Duold), Du0 = Duold, uold = uold)
nothing #hide
```

Let us find the front using `newton`

```@example TUTAUTOCATauto
front, = newton(probTW, vcat(U0, 0.1), par_cat, NewtonPar(verbose = true), jacobian = :AutoDiff)
println("norm front = ", front[1:end-1] |> norminf, ", speed = ", front[end])
```

```@example TUTAUTOCATauto
plotsol(front[1:end-1], title="front solution")
```

## Continuation of front solutions

Following [^Malham], the modulated fronts are solutions of the following DAE

$$\begin{array}{l}\tag{DAE}
u_{t}=a u_{x x}+su_x-u f(v)\\
v_{t}=v_{x x}+s v_x+u f(v)\\
0 = \left\langle U, \partial_\xi U_0	\right\rangle
\end{array}$$

which can be written with a PDE $M_aU_t = G(u)$ with mass matrix $M_a = (Id, Id, 0)$. We have already written the vector field of (MF) in the problem `probTW`.

Having found a front $U^f$, we can continue it as function of the parameter $a$ and detect instabilities. The stability of the front is linked to the eigenelements $(\lambda, V)$ solution of the generalized eigenvalue problem:

$$\lambda M_a\cdot V = dG(U^f)\cdot V.$$

This is handled automatically when calling `continuation` on a `TWProblem`.

```@example TUTAUTOCATauto
optn = NewtonPar(tol = 1e-8, verbose = true)
opt_cont_br = ContinuationPar(pMin = 0.015, pMax = 0.18, newtonOptions = optn, ds= -0.001, plotEveryStep = 2, detectBifurcation = 3, nev = 10, nInversion = 6)
br_TW, front2, = continuation(probTW, front, par_cat, (@lens _.a), opt_cont_br;
	jacobian = :AutoDiff, recordFromSolution = (x, p) -> (s = x[end], b=0), )
plot(br_TW, legend = :topright)
```

We have detected a Hopf instability in front dynamics, this will give rise of modulated fronts.


## References

[^Balmforth]:> N. J. Balmforth, R. V. Craster, and S. J. A. Malham. Unsteady fronts in an autocatalytic system. R. Soc. Lond. Proc. Ser. A Math. Phys. Eng. Sci., 455(1984):1401–1433, 1999.

[^Malham]:> S. J. A. Malham and M. Oliver. Accelerating fronts in autocatalysis. R. Soc. Lond. Proc. Ser. A Math. Phys. Eng. Sci., 456(1999):1609–1624, 2000.

[^Beyn]:> Beyn, Wolf-Jürgen, and Vera Thümmler. “Phase Conditions, Symmetries and PDE Continuation.” In Numerical Continuation Methods for Dynamical Systems: Path Following and Boundary Value Problems Springer Netherlands, 2007. https://doi.org/10.1007/978-1-4020-6356-5_10.
