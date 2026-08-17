# vani-optimize

Numerical optimization library for the [vāṇī compiler](https://github.com/enthusiasticgeek/vani-compiler).

Unconstrained and constrained continuous optimization, plus linear programming,
all built on plain `Vec<f64>` (points/vectors) in flat row-major layout for matrices/
Hessians, following the convention established by [vani-matrix](https://github.com/enthusiasticgeek/vani-matrix).
Depends on vani-matrix's `mat_solve` for the Newton-step and quadratic-specialty solvers.

**API reference / tutorial:** <https://enthusiasticgeek.github.io/vani-optimize/>

## Add to your project

```toml
# vani.toml
[deps]
optimize = { registry = "kosh", version = "^0.1" }
```

```sh
vanic add optimize
vanic build
```

## What's included (v0.1.0 — complete; v0.1.5 added Closure variants; see TODO.md)

| Module | Functions |
|---|---|
| Test objectives | `sphere_fn`, `sphere_grad`, `rosenbrock_fn`, `rosenbrock_grad` |
| Numerical differentiation | `grad_fd`, `hessian_fd` |
| Unconstrained (gradient-based) | `gradient_descent_fixed`, `armijo_line_search`, `gradient_descent_backtracking`, `newton_nd`, `newton_nd_fd`, `coordinate_descent` |
| Unconstrained, `Closure`-accepting (v0.1.5) | `gradient_descent_fixed_closure`, `armijo_line_search_closure`, `gradient_descent_backtracking_closure` -- same algorithms, but `f`/`grad_f` take `Closure(ref Vec<f64>, i64) -> ...` instead of a plain `fn` pointer, so the objective can capture data by reference (e.g. a training set) instead of needing it as a global or duplicated per call site. See "Closure variants" below. |
| Quadratic specialty | `quadratic_minimize`, `steepest_descent_quadratic` |
| Constrained (penalty method) | `penalty_value`, `penalty_grad_fd`, `penalty_line_search`, `penalty_method_minimize` |
| Linear programming | `simplex_solve`, `simplex_value` |

## Closure variants (v0.1.5)

The plain `fn`-typed functions above can't receive a closure that captures
data by reference -- `fn(...)->...` and `Closure(...)->...` are distinct
types with no implicit coercion. The `_closure`-suffixed variants accept
`Closure(ref Vec<f64>, i64) -> ...` instead, so you can write:

```vani
let target: Vec<f64> = vec(3.0, 4.0);
let f: Closure(ref Vec<f64>, i64) -> f64 = fn(x: ref Vec<f64>, n: i64) -> f64 [ref target] {
    // ... objective parameterized by `target`, captured by reference ...
};
let result: Vec<f64> = gradient_descent_fixed_closure(f, grad_f, ref x0, n, alpha, max_iter, tol);
```

Added additively -- the original `fn`-typed functions are unchanged, so
existing code (including this package's own tests/examples, which pass
plain top-level functions like `sphere_fn`) keeps compiling as-is.

## A note on finite differences

Functions ending in `_fd` (`grad_fd`, `hessian_fd`, `newton_nd_fd`) approximate
derivatives numerically and carry more error than their analytic-callback
counterparts -- `hessian_fd` in particular can differ from the true second
derivative by ~1e-6 to 1e-4 depending on step size `h` and problem scale. Use a
tolerance no tighter than ~1e-4 when comparing `hessian_fd` output against a
reference value; prefer the analytic-Hessian path (`newton_nd`) when you have one.

## Encoding

Points and vectors are `Vec<f64>` (length `n`). Matrices/Hessians are `Vec<f64>`
in flat row-major layout (length `n*n`), consistent with vani-matrix. Objective
and gradient functions are passed as function pointers:
`fn(ref Vec<f64>, i64) -> f64` and `fn(ref Vec<f64>, i64) -> Vec<f64>` respectively.

## What this library does NOT provide

These are already vāṇī compiler builtins — call them directly, no import needed:

`abs` `sqrt` `exp` `log` `pow`

`mat_solve` is provided by [vani-matrix](https://github.com/enthusiasticgeek/vani-matrix)
(a dependency of this package, not reimplemented here).

## License

MIT
