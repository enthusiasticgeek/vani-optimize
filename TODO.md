# vani-optimize — TODO

> Compiler builtins that already exist and must NOT be reimplemented:
> `abs` `sqrt` `exp` `log` `pow`
>
> Depends on [vani-matrix](https://github.com/enthusiasticgeek/vani-matrix) (`mat_solve`)
> — vendored via `vanic add matrix`, see `[deps]` in `vani.toml`.

---

## v0.1.0 — Implemented ✓

### Test objective functions (4 functions)
- [x] `sphere_fn`, `sphere_grad` — convex, global min 0 at the origin
- [x] `rosenbrock_fn`, `rosenbrock_grad` — nonconvex, the classic hard test case

### Numerical differentiation (2 functions)
- [x] `grad_fd` — central-difference gradient
- [x] `hessian_fd` — central-difference Hessian (diagonal + mixed-partial formulas)

### Unconstrained gradient-based optimization (6 functions)
- [x] `gradient_descent_fixed` — fixed step size
- [x] `armijo_line_search`, `gradient_descent_backtracking` — adaptive step via
      Armijo backtracking; validated on Rosenbrock from the standard hard
      starting point (-1.2, 1)
- [x] `newton_nd` — analytic gradient+Hessian, quadratic convergence near the optimum
- [x] `newton_nd_fd` — fully derivative-free Newton's method
- [x] `coordinate_descent` — derivative-free axis-sweep baseline

### Quadratic specialty solvers (2 functions)
- [x] `quadratic_minimize` — closed-form `x = -Q⁻¹b` via `mat_solve`
- [x] `steepest_descent_quadratic` — exact optimal step size each iteration

### Constrained optimization — penalty method (4 functions)
- [x] `penalty_value`, `penalty_grad_fd`
- [x] `penalty_line_search` — backtracking line search specialized to the
      penalized objective (a fixed step size reliably diverges once the outer
      penalty weight `mu` grows large — see doc comment in `src/lib.vani`)
- [x] `penalty_method_minimize` — validated against a textbook equality-constrained
      quadratic (minimize x²+y² s.t. x+y≥1 → exact solution (0.5, 0.5))

### Linear programming — tableau simplex (2 functions)
- [x] `simplex_solve` — validated against two independent textbook LPs
      (including the classic Wyndor Glass Co. problem: max 3x+5y s.t. x≤4,
      2y≤12, 3x+2y≤18 → x=2, y=6, objective=36)
- [x] `simplex_value`

### Tests and examples
- [x] `tests/test_gradient_methods.vani` — `grad_fd`/`hessian_fd` cross-checked
      against analytic gradients/Hessians, `gradient_descent_fixed`/`_backtracking`
      on sphere and Rosenbrock, `coordinate_descent`
- [x] `tests/test_newton_quadratic.vani` — `newton_nd` (analytic) and `newton_nd_fd`
      on Rosenbrock, `quadratic_minimize` and `steepest_descent_quadratic` against
      a hand-solved 2×2 SPD system
- [x] `tests/test_penalty_simplex.vani` — `penalty_method_minimize` on the
      constrained sphere problem, `simplex_solve`/`simplex_value` on two
      independent LPs
- [x] `examples/rosenbrock_showdown.vani` — gradient descent vs. Newton's method
      vs. coordinate descent, side by side, on the same hard starting point
- [x] `examples/constrained_demo.vani` — penalty method and simplex, one demo each

### Safety annotations
- [x] `#[bounded_stack(bytes=N)]` on all functions, budgets set to `vanic check`'s
      exact reported worst-case (largest: `newton_nd` at 784 bytes)

### Known compiler quirk encountered (not a library bug)
- A failing `assert` under `vanic run`/`vanic test` can surface as an apparent
  native stack overflow (Windows `STATUS_STACK_BUFFER_OVERRUN`) instead of a
  clean assertion-failure exit — `vanic build`'s AOT binary reports the same
  failure correctly and cleanly. If a test looks like it crashed, build it to a
  binary first (`vanic build tests/foo.vani -o foo.exe`) to get an honest
  pass/fail signal before assuming a real crash. Tracked as MATH-3 in
  vani-compiler's `docs/TODO_CURRENT.md`.

---

## v0.1.5 — Implemented ✓ (2026-07-25)

### Closure-accepting variants (3 functions)
- [x] `gradient_descent_fixed_closure`, `armijo_line_search_closure`,
      `gradient_descent_backtracking_closure` — same algorithms as the
      v0.1.0 originals, but `f`/`grad_f` take `Closure(ref Vec<f64>,
      i64) -> ...` instead of a plain `fn` pointer, so the objective can
      capture data by reference (e.g. a training set) instead of needing
      it as a global. Added additively — the original `fn`-typed
      functions are untouched, every existing caller keeps compiling
      unchanged (no implicit coercion exists between `fn(...)->...` and
      `Closure(...)->...`, confirmed by direct test, so an in-place
      signature change would have broken every current caller).
      Motivated by vani-ml's `logreg_fit` needing to pass a closure
      capturing training data by reference — see vani-compiler's
      `docs/ref_capturing_closures_design.md`.
- [x] `tests/test_closure_variants.vani` — minimizes
      `f(x) = sum((x_i - target_i)^2)` with `target` captured by
      reference (not hand-coded into a top-level function), via both
      the fixed-step and backtracking closure variants, verified on
      both backends.
- [x] `#[bounded_stack(bytes=N)]` on all 3 new functions, exact
      `vanic check`-computed values.

**Found and fixed two real vanic compiler bugs along the way** (not this
package's bugs — filed and fixed in `vani-compiler`, not tracked further
here): a function merely *taking* a `Closure`-typed parameter failed to
compile if no matching closure literal existed anywhere in the calling
program (would have broken every existing consumer of this package the
moment these functions were added); and the C backend could order a
closure's struct typedef before a `Vec<T>` type it referenced. Both
fixed in `vani-compiler` before this version was finalized here.

**Published** 2026-07-26 (this note was stale until 2026-07-27 -- v0.1.5
was already live in kosh-index by the time the Phase I simplex work
below started).

---

## v0.1.6 (2026-07-27)

- [x] `simplex_solve_phase1(c, A, b, n_vars, n_constraints)` -- two-phase
      tableau simplex for LPs that aren't feasible at the origin (any
      sign of `b`, unlike `simplex_solve`'s `b >= 0` restriction). Every
      row gets both a slack and an artificial variable (uniform, simpler
      to get right than a variable-sized artificial block, same
      "correctness > speed" tradeoff as `vani-bignum`'s long division);
      rows with `b[i] < 0` are negated first so every RHS is >= 0, which
      also flips that row's own slack coefficient to -1 -- **the one real
      bug found and fixed during development**: an early version
      hardcoded the slack coefficient to +1 regardless of the row's
      sign, which silently corrupted Phase 2's represented LP for any
      negated row (Phase 1 still "converged" to a wrong answer instead
      of erroring, since nothing about the bug was inconsistent enough
      to trip an internal check) -- caught by hand-verifying the very
      first test case's expected optimum, not by the type checker.
      Factored `_simplex_pivot` (shared by both phases) and
      `_simplex_basic_row` (finds which column is currently basic in a
      row, used to rebuild Phase 2's objective against whatever Phase 1
      left behind) as private helpers; `simplex_solve` itself is
      untouched. No dedicated infeasibility signal (returns an all-zero
      Vec), same convention as `simplex_solve`'s unbounded-LP case.
- [x] `tests/test_penalty_simplex.vani` extended: a classic textbook
      two-phase example (minimize 2x+3y s.t. x+y>=4, x+2y>=6 -- both
      constraints bind at the optimum (2,2), objective 10, verified by
      hand against every other vertex of the feasible region) and a
      genuinely infeasible LP (x+y<=1 AND x+y>=3 simultaneously). Full
      suite + `vanic audit-safety` re-verified on both backends. No
      unrelated WCET/stack drift found in this package.

## Future

No v0.2.0 is currently planned. Candidates if a concrete need shows up:
constrained optimization beyond the single-inequality penalty method
(augmented Lagrangian, or a proper active-set/interior-point method), quasi-Newton
methods (BFGS/L-BFGS) to avoid `newton_nd`'s full-Hessian cost, and stochastic
gradient variants once a use case with noisy gradients appears.
