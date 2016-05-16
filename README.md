# DeterministicPolicyGradient

[![Build Status](https://travis-ci.org/baggepinnen/DeterministicPolicyGradient.jl.svg?branch=master)](https://travis-ci.org/baggepinnen/DeterministicPolicyGradient.jl)

This package provides an implementation of the algorithm in the paper
"David Silver et al.. Deterministic Policy Gradient Algorithms. 2014."

# Usage
See file `second_order_sys.jl`, which requires `Plots.jl` and `PyPlot.jl` to display the results.

```julia
using ControlSystems
using DeterministicPolicyGradient
import DSP
try
  close("all")
catch
end
const G = tf(0.1,[1,0.01,0.1])

const T = 20
const t = 1:T
const m = 1
const n = 2
const p = 5
x0 = [0,1.]


# Cost function             ==========================================
const Q1 = diagm([5,1])
const Q2 = 1eye(m)
r(s,a) = (-(s)'Q1*(s) - a'Q2*a)[1]
L = lqr(ss(G),Q1,Q2)
μlqr(x) = -L*x


# Initialize solver options ==========================================
σβ      = 2
αΘ      = 0.005
αw      = 0.001
αv      = 0.001
αu      = 0.01
γ       = 0.999
τ       = 0.01
iters   = 30_000
rls_critic = true
λrls    = 0.99999
const opts = DPGopts(σβ,αΘ,αw,αv,αu,γ,τ,iters,m,rls_critic,λrls)

# Initialize functions      ==========================================
cp        = linspace(-5,5,p)
cv        = linspace(-5,5,p)
ca        = linspace(-10,10,p)
grid1     = meshgrid(cp,cv)
grid2     = meshgrid2(cp,cv,ca)
const c1  = [grid1[1][:] grid1[2][:]]
const c2  = [grid2[1][:] grid2[2][:] grid2[3][:]]
P         = size(c1,1)
P2        = size(c2,1)

function ϕ(s)
    a = exp(-1/(2*2)*sum((s'.-c1).^2,2))#exp(-1/2*(s.-c).^2)
    a ./= sum(a)
    a
    # [a; s[2]]
end

μ(s,Θ)          = Θ'ϕ(s)
∇μ(s)           = ϕ(s)
β(s,Θ,noise,i)  = Θ'ϕ(s) + noise[i]
ϕ(s,a,Θ)        = ∇μ(s)*(a-μ(s,Θ))
V(s,v)          = v'ϕ(s) # It's a good idea to bias V to some mean of the final landscape
Q(s,a,v,w,Θ)    = (ϕ(s,a,Θ)'w + V(s,v))[1]
simulate(Θ,x0, noise) = lsim(G, (i,s)->β(s,Θ,noise,i), t, x0)[3:4]
simulate(Θ,x0)  = lsim(G, (i,s)->μ(s,Θ), t, x0)[3:4]
exploration(σβ) = DSP.filtfilt(ones(5),[5],σβ*randn(T))
funs            = DPGfuns(μ,∇μ,β,ϕ,V,Q, simulate, exploration, r)


# Train the actor and critic ==========================================
@time cost, Θ, w, v = dpg(opts, funs, x0)

# Plots results              ==========================================
plot(cost[1:100:end])
gui()

lsimplot(G, (i,s)->[μ(s,Θ)], linspace(0,T,200),x0), plot!(title="After")
gui()
lsimplot(G, (i,s)->[μlqr(s)], linspace(0,T,200),x0), plot!(title="LQR")
gui()


Qpv_map = Float64[Q([sv,sp],0,v,w,Θ)[1] for sp in linspace(-5,5,100), sv in linspace(-5,5,100)]
PyPlot.matshow(Qpv_map); PyPlot.colorbar()
PyPlot.title("Q map");PyPlot.xlabel("Velocity");PyPlot.ylabel("Position")
PyPlot.yticks(0:10:100,-5:5);PyPlot.xticks(0:10:100,-5:5)


Qpa_map = Float64[Q([0,sp],a,v,w,Θ)[1] for sp in linspace(-5,5,100), a in linspace(-10,10,100)]
PyPlot.matshow(Qpa_map); PyPlot.colorbar()
PyPlot.title("Q map");PyPlot.xlabel("Control");PyPlot.ylabel("Position")
PyPlot.yticks(0:10:100,-5:5);PyPlot.xticks(0:10:100,-10:2:10)


control_map = Float64[μ([sv,sp],Θ)[1] for sp in linspace(-5,5,100), sv in linspace(-5,5,100)]
PyPlot.matshow(control_map); PyPlot.colorbar()
PyPlot.title("Control map");PyPlot.xlabel("Velocity");PyPlot.ylabel("Position")
PyPlot.yticks(0:10:100,-5:5);PyPlot.xticks(0:10:100,-5:5)


control_map = Float64[μlqr([sv,sp])[1] for sp in linspace(-5,5,100), sv in linspace(-5,5,100)]
PyPlot.matshow(control_map); PyPlot.colorbar()
PyPlot.title("Control map LQR");PyPlot.xlabel("Velocity");PyPlot.ylabel("Position")
PyPlot.yticks(0:10:100,-5:5);PyPlot.xticks(0:10:100,-5:5)

show()

```
