# Error Analysis: `EqPUtil` Piecewise Utility Function

This report identifies six bugs in the construction of `EqPUtil` using `EQ1CONS`, `EQ2CONS`, and `EQ3CONS`. All have been fixed.

---

## Bug 1 (Critical): `EQ3CONS` — Copy-Paste Error in First Condition Group ✅ FIXED

**Location:** The cell defining `EQ1CONS`, `EQ2CONS`, `EQ3CONS` (around line 36793 of the notebook)

`EQ3CONS` is structured as two `&&`-joined OR-groups. Comparing across all three:

| Variable | First `()` group | Second `()` group |
|---|---|---|
| `EQ1CONS` | `con1Eq1Rst2 \|\| con2Eq1Rst2 \|\| con3Eq1Rst2 \|\| con1Eq1Wst2 \|\| con2Eq1Wst2` | `con1Eq1St1 \|\| ... \|\| con4Eq1St1` |
| `EQ2CONS` | `con1Eq2Rst2 \|\| con2Eq2Rst2 \|\| con3Eq2Rst2 \|\| con1Eq2Wst2 \|\| ...` | `con1Eq2St1 \|\| ... \|\| con7Eq2St1` |
| **`EQ3CONS`** | **`con1Eq3St1 \|\| con2Eq3St1 \|\| con3Eq3St1 \|\| con1Eq3Wst2 \|\| ...`** ❌ | `con1Eq3St1 \|\| ... \|\| con7Eq3St1` |

In `EQ3CONS`, the first three conditions in the first group are `con1Eq3St1`, `con2Eq3St1`, `con3Eq3St1` — **`St1` conditions, not `Rst2` conditions.** By analogy with `EQ1CONS` and `EQ2CONS`, they should be `con1Eq3Rst2`, `con2Eq3Rst2`, `con3Eq3Rst2`.

Critically, those `Rst2` conditions **do exist** and are defined earlier in the notebook (around lines 12800, 13090, and 13426), but are **never referenced** in `EQ3CONS`. As a result:

- The first group of `EQ3CONS` is logically identical to a subset of its second group.
- The logical AND of two overlapping OR-groups collapses to just the larger group (the second one), so `EQ3CONS` does **not** properly encode the two independent equilibrium constraints it is supposed to.
- Any point satisfying the `St1` conditions will be incorrectly classified as Equilibrium 3, even if it fails the `Rst2` constraint.

**Fix:** Replace the first three terms in the first group of `EQ3CONS`:
```mathematica
(* WRONG *)
(con1Eq3St1 || con2Eq3St1 || con3Eq3St1 || con1Eq3Wst2 || con2Eq3Wst2 || con3Eq3Wst2)

(* CORRECT *)
(con1Eq3Rst2 || con2Eq3Rst2 || con3Eq3Rst2 || con1Eq3Wst2 || con2Eq3Wst2 || con3Eq3Wst2)
```

---

## Bug 2 (Critical): Argument Count Mismatch — `EUPEqXTot` Called with Too Many Arguments ✅ FIXED

**Location:** The `EqPUtil` definition (around lines 36978–37019)

`EUPEq1Tot`, `EUPEq2Tot`, and `EUPEq3Tot` are all defined with **5 parameters**:

```mathematica
EUPEq1Tot[pi_, p_, κ_, c_, θ_] := ...
EUPEq2Tot[pi_, p_, κ_, c_, θ_] := ...
EUPEq3Tot[pi_, p_, κ_, c_, θ_] := ...
```

But inside `EqPUtil`, all three are called with **6 arguments** — the new `γ` (gamma) parameter was inserted between `c` and `θ`:

```mathematica
EUPEq2Tot[pi, p, κ, c, γ, θ]   (* 6 args — no matching definition! *)
EUPEq1Tot[pi, p, κ, c, γ, θ]   (* 6 args — no matching definition! *)
EUPEq3Tot[pi, p, κ, c, γ, θ]   (* 6 args — no matching definition! *)
```

Because none of these calls match any defined pattern, Mathematica will return the expressions **unevaluated**, meaning `EqPUtil` will always return `0` (the `Piecewise` default) — not because no equilibrium applies, but because the utility value expressions simply never resolve to numbers.

**Fix:** Removed `γ` from the calls to `EUPEq1Tot`, `EUPEq2Tot`, and `EUPEq3Tot` inside `EqPUtil`, since those functions do not use `γ` (it is already handled by `c1F`/`c2F`/`c3F` through the `EQ*CONS` conditions). All three calls now correctly pass 5 arguments.

---

## Bug 3 (Secondary): `EUPEq2Tot` and `EUPEq3Tot` Internally Hardcode `θ = 1/2` ✅ FIXED

**Location:** Inside the body of `EUPEq2Tot` (around line 36617) and `EUPEq3Tot` (around line 24551)

Even though both functions accept a `θ_` parameter, their internal calls to the lower-level `EUPXtotEqY` functions were passing the literal `1/2` for `θ` instead of forwarding the parameter.

**`EUPEq2Tot`** — all 7 value-function calls fixed:
```mathematica
(* Before *)                              (* After *)
EUP1totEq2[pi, p, κ, c, 1/2]    →    EUP1totEq2[pi, p, κ, c, θ]
EUP2totEq2[pi, p, κ, c, 1/2]    →    EUP2totEq2[pi, p, κ, c, θ]
EUP3totEq2[pi, p, κ, c, 1/2]    →    EUP3totEq2[pi, p, κ, c, θ]
EUP4totEq2[pi, p, κ, c, 1/2]    →    EUP4totEq2[pi, p, κ, c, θ]
EUP5totEq2[pi, p, κ, c, 1/2]    →    EUP5totEq2[pi, p, κ, c, θ]
EUP6totEq2[pi, p, κ, c, 1/2]    →    EUP6totEq2[pi, p, κ, c, θ]
EUP7totEq2[pi, p, κ, c, 1/2]    →    EUP7totEq2[pi, p, κ, c, θ]
```

**`EUPEq3Tot`** — 1 value-function call fixed (`EUP1totEq3`, `EUP3–6totEq3` already used `θ` correctly):
```mathematica
(* Before *)                              (* After *)
EUP2totEq3[pi, p, κ, c, 1/2]    →    EUP2totEq3[pi, p, κ, c, θ]
```

Note: Boundary functions (`cp2REq2`, `cp1WEq3`, etc.) intentionally retain `1/2`, consistent with `EUPEq1Tot`.

---

## Bug 4 (Critical): Stale `EQ*CONS` Values Due to Immediate Assignment ✅ FIXED

**Location:** Both cells that define `EQ1CONS`, `EQ2CONS`, `EQ3CONS` (lines ~8236 and ~36772)

`EQ1CONS`, `EQ2CONS`, and `EQ3CONS` are assigned with `=` (immediate evaluation), meaning they capture the current values of `con*` variables **at the moment the cell runs**. If the notebook had been partially evaluated before (e.g. a `Manipulate` slider set `κ` or `γ` to a numeric value globally), the `con*` conditions could have collapsed to `True` or `False` before being captured, making all three conditions permanently wrong regardless of input.

**Fix:** Added `ClearAll[EQ1CONS, EQ2CONS, EQ3CONS]` at the top of both cells that define them, ensuring stale values are always cleared before re-assignment:

```mathematica
ClearAll[EQ1CONS, EQ2CONS, EQ3CONS];
EQ1CONS = ...
EQ2CONS = ...
EQ3CONS = ...
```

---

## Bug 5 (Critical): κ Range in Second `Manipulate` Excludes All Valid Equilibria ✅ FIXED

**Location:** Second `Manipulate` block (around line 37054)

The second `Manipulate` plotted `EqPUtil` over `{k, 0, 1}` (κ ∈ [0, 1]). However, most equilibrium conditions require **κ > 1** — for example, `con1Eq1Wst2` requires `κ > 1`, and many `Rst2` conditions check `1 ≤ κ ≤ ...`. With κ always in [0, 1], every condition evaluated to `False`, `EqPUtil` always returned the default `0`, and the plot appeared blank.

The first `Manipulate` (RegionPlot) correctly used `{k, 1, 10}`.

**Fix:** Changed the second `Manipulate`'s plot range to match:

```mathematica
(* Before *)
Plot[EqPUtil[pi, p0, k, 1, γ0, .5], {k, 0, 1}]

(* After *)
Plot[EqPUtil[pi, p0, k, 1, γ0, .5], {k, 1, 10}]
```

---

## Bug 6 (Critical): Pattern Variable Shadowing in `c1F`, `c2F`, `c3F` ✅ FIXED

**Location:** Definitions of `c1F`, `c2F`, `c3F` (near the `EqPUtil` definition)

`c1F`, `c2F`, and `c3F` check whether a given point satisfies `EQ1CONS`, `EQ2CONS`, or `EQ3CONS` respectively by substituting numeric values into the symbolic expression. Their pattern variables were named identically to the global symbols they were meant to substitute:

```mathematica
(* BROKEN *)
c1F[pi_?NumericQ, p_?NumericQ, \[Kappa]_?NumericQ, c_?NumericQ,
    \[Gamma]_?NumericQ, \[Theta]_?NumericQ] :=
  c1F[pi, p, \[Kappa], c, \[Gamma], \[Theta]] =
    TrueQ@Quiet@Check[EQ1CONS /. {\[Kappa] -> \[Kappa], \[Gamma] -> \[Gamma],
        \[Theta] -> \[Theta], pi -> pi, p -> p, c -> c}, False]
```

Inside the function body, the pattern variable `\[Kappa]` has already been bound to the numeric argument (e.g. `2`). So the replacement rule `{\[Kappa] -> \[Kappa]}` evaluates to `{2 -> 2}` — a no-op. `EQ1CONS` remains fully symbolic after the substitution, and `TrueQ` of any symbolic expression returns `False`. As a result, `c1F` (and `c2F`, `c3F`) **always returned `False`** regardless of the input values, meaning `EqPUtil` could never identify any equilibrium region.

This was confirmed by the diagnostic test `c1F[0.5, 0.7, 2, 1, 1.0, 0.5]` returning `False` for a point that should satisfy `EQ1CONS`.

**Fix:** Renamed all pattern variables to use a `V` suffix (`piV_`, `pV_`, `κV_`, `cV_`, `γV_`, `θV_`). The global symbols on the left-hand side of each replacement rule now correctly refer to the unbound global symbol, not the local pattern variable:

```mathematica
(* FIXED *)
c1F[piV_?NumericQ, pV_?NumericQ, \[Kappa]V_?NumericQ, cV_?NumericQ,
    \[Gamma]V_?NumericQ, \[Theta]V_?NumericQ] :=
  c1F[piV, pV, \[Kappa]V, cV, \[Gamma]V, \[Theta]V] =
    TrueQ@Quiet@Check[EQ1CONS /. {\[Kappa] -> \[Kappa]V, \[Gamma] -> \[Gamma]V,
        \[Theta] -> \[Theta]V, pi -> piV, p -> pV, c -> cV}, False]
```

The same rename was applied to `c2F` (substituting into `EQ2CONS`) and `c3F` (substituting into `EQ3CONS`).

---

## Summary Table

| # | Severity | Location | Type | Effect | Status |
|---|---|---|---|---|---|
| 1 | Critical | `EQ3CONS` definition | Copy-paste error (`St1` vs `Rst2`) | `EQ3CONS` evaluates wrong region; Eq3 classification is incorrect | ✅ Fixed |
| 2 | Critical | `EqPUtil` calling `EUPEqXTot` | Argument count mismatch (5 vs 6) | All `EUPEqXTot` calls return unevaluated; `EqPUtil` always returns `0` | ✅ Fixed |
| 3 | Secondary | Body of `EUPEq2Tot`, `EUPEq3Tot` | Hardcoded `θ = 1/2` ignores parameter | `θ` has no effect on Eq2/Eq3 utility values | ✅ Fixed |
| 4 | Critical | `EQ*CONS` definitions (both cells) | Stale immediate assignment | Prior evaluations corrupt condition logic; `c1F`/`c2F`/`c3F` return wrong values | ✅ Fixed |
| 5 | Critical | Second `Manipulate` plot range | κ range excludes valid equilibria | No equilibrium ever detected; plot always blank | ✅ Fixed |
| 6 | Critical | `c1F`, `c2F`, `c3F` definitions | Pattern variable shadowing global symbols | Replacement rules become no-ops; `TrueQ` always returns `False`; `EqPUtil` never finds any equilibrium | ✅ Fixed |

**To apply all fixes:** Restart the Mathematica kernel (`Kernel → Quit Kernel`) and evaluate the notebook top-to-bottom.
