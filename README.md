# UnitfulCurrencies

A supplemental units package for [Unitful.jl](https://github.com/PainterQubits/Unitful.jl) adding units for all currently active currencies in the world, along with tools to perform conversions based on exchange market rates.

**This package is not registered and is current under development.**

## Summary

Currency dimensions are created for each currency, along with its reference
unit. Being an extension of [Unitful.jl](https://github.com/PainterQubits/Unitful.jl), currency units play nicely along with Unitful's quantities.

An `ExchangeMarket` type is also defined as an alias for
`Dict{Tuple{String,String},Real}`, in which the tuple key contains the
quote-ask currency pair (e.g. `("EUR", "USD")`) and the value is the
exchange rate for the pair.

Based on an given exchange market instance of `ExchangeMarket`, a conversion
can be made from the "quote" currency to the "base" currency. This conversion
is implemented as an extended dispatch for `Unitful.uconvert`.

## Examples

Let us see some examples using `UnitfulCurrencies.jl`.

### Cost of production

As an example, let us say we have a product P that depends on four materials, whose costs C₁, ... C₂ are given as

```julia
julia> using Unitful

julia> using UnitfulCurrecies

julia> C₁ = 0.45u"USD/lb"
0.45 USD lb⁻¹

julia> C₂ = 0.78u"USD/lb"
0.78 USD lb⁻¹
```

Besides the cost

Suppose we need one pound of P₁ and a half pound of P₂ to produce one unit of P. Thus, the cost of producing 100 pieces of product P is

```julia
julia> 100 * 1.0u"lb" * C₁ + 0.5u"lb" * C₂
45.39 USD
```

### Continuously varying interest rate

Now, let us suppose we have a £1,000 in a savings account in a British bank, with an expected variable interest rate for the next ten years of the form

![formula](https://render.githubusercontent.com/render/math?math=\qquad\qquad\text{rate}(t)=\left(0.015%2B0.5\frac{(t/\text{yr})^2}{(1%2B(t/\text{yr})^3)}\right)/yr),

and suppose we want to estimate how much we will have after ten years. This can be implemented as follows.

```julia
julia> using Unitful, UnitfulCurrencies, DifferentialEquations

julia> rate(t) = (1.5 + 5(t * u"1/yr")^2 * ( 1 + (t * u"1/yr")^3)^-1)*u"percent/yr"
rate (generic function with 1 method)

julia> f(u,rate,t) = rate(t) * u
f (generic function with 1 method)

julia> tspan = Tuple([0.0,10.0]*u"yr")
(0.0 yr, 10.0 yr)

julia> u₀ = 1000.0u"GBP"
1000.0 GBP

julia> prob = ODEProblem(f,u₀,tspan,rate)
ODEProblem with uType Quantity{Float64,GBPCURRENCY,Unitful.FreeUnits{(GBP,),GBPCURRENCY,nothing}} and tType Quantity{Float64,𝐓,Unitful.FreeUnits{(yr,),𝐓,nothing}}. In-place: false
timespan: (0.0 yr, 10.0 yr)
u0: 1000.0 GBP

julia> savings = solve(prob);

julia> println("After $(savings.t[end]), we expect to have $(savings.u[end])")
After 10.0 yr, we expect to have 1303.6211777402004 GBP
```

Thus, we expect to have about £1,303.62 in our savings account, after ten years.

Now let us suppose I want to see the evolution of this savings in terms of Brazillian Reais, and suppose this happened ten years ago, so we can use some real exchange rates. In this case, I use an exchange rate time series, as follows.

```julia
julia> BRLGBP_timeseries = Dict(
           "2011-01-01" => ExchangeMarket(("BRL","GBP") => 0.38585),
           "2012-01-01" => ExchangeMarket(("BRL","GBP") => 0.34587),
           "2013-01-01" => ExchangeMarket(("BRL","GBP") => 0.29998),
           "2014-01-01" => ExchangeMarket(("BRL","GBP") => 0.25562),
           "2015-01-02" => ExchangeMarket(("BRL","GBP") => 0.24153),
           "2016-01-03" => ExchangeMarket(("BRL","GBP") => 0.17093),
           "2017-01-02" => ExchangeMarket(("BRL","GBP") => 0.24888),
           "2018-01-02" => ExchangeMarket(("BRL","GBP") => 0.22569),
           "2019-01-04" => ExchangeMarket(("BRL","GBP") => 0.21082),
           "2020-01-04" => ExchangeMarket(("BRL","GBP") => 0.18784)
       );

julia> uconvert.(u"BRL", 1000u"GBP", values(BRLGBP_timeseries), extended=true)'
1×10 LinearAlgebra.Adjoint{Quantity{Float64,BRLCURRENCY,Unitful.FreeUnits{(BRL,),BRLCURRENCY,nothing}},Array{Quantity{Float64,BRLCURRENCY,Unitful.FreeUnits{(BRL,),BRLCURRENCY,nothing}},1}}:
 2591.68 BRL  2891.26 BRL  4018.0 BRL  4743.38 BRL  …  4140.27 BRL  3912.06 BRL  4430.86 BRL  5323.68 BRL
```

Notice we used the optional argument `extended=true`, so it uses the inverse rate for the conversion. This is different than using the rate for the pair `("GBP", "BRL")` since we don't want to buy `GBP` with `BRL`, and neither do we want the direct rate for `("BRL", "GBP")` since we don't want to buy a specific amount of `BRL` with `GBP`. Instead, we want to find out how much `BRL` we can buy with a given amount of `GBP`.

### Exchange markets

For exchanging money, consider, for example, the following exchange market:

```julia
julia> using Unitful

julia> using UnitfulCurrencies

julia> exch_mkt = ExchangeMarket(
           ("EUR","USD") => 1.19172, ("USD","EUR") => 0.839125,
           ("USD","CAD") => 1.30015, ("CAD","USD") => 0.769144,
           ("USD","BRL") => 5.41576, ("BRL","USD") => 5.41239
       )
Dict{Tuple{String,String},Real} with 6 entries:
  ("USD", "BRL") => 5.41576
  ("BRL", "USD") => 5.41239
  ("EUR", "USD") => 1.19172
  ("USD", "EUR") => 0.839125
  ("CAD", "USD") => 0.769144
  ("USD", "CAD") => 1.30015
```

Then, the conversions between these currencies can be done as follows:

```julia
julia> uconvert(u"BRL", 100u"USD", test_mkt)
541.576 BRL
```

## License

This package is licensed under the [MIT license](https://opensource.org/licenses/MIT) (see file [LICENSE](LICENSE) in the root directory of the project).
