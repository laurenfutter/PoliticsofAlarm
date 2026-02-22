# Error Analysis: `EqPUtil` Piecewise Utility Function

This report identifies three bugs in the construction of `EqPUtil` using `EQ1CONS`, `EQ2CONS`, and `EQ3CONS`.

---

## Bug 1 (Critical): `EQ3CONS` — Copy-Paste Error in First Condition Group

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

## Bug 2 (Critical): Argument Count Mismatch — `EUPEqXTot` Called with Too Many Arguments

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

**Fix:** Update the definitions of `EUPEq1Tot`, `EUPEq2Tot`, and `EUPEq3Tot` to accept the additional `γ` parameter (or remove `γ` from the calls in `EqPUtil` if it is not needed by those functions).

---

## Bug 3 (Secondary): `EUPEq2Tot` and `EUPEq3Tot` Internally Hardcode `θ = 1/2`

**Location:** Inside the body of `EUPEq2Tot` (around line 36617) and `EUPEq3Tot` (around line 24551)

Even though both functions accept a `θ_` parameter, their internal calls to the lower-level `EUPXtotEqY` functions pass the literal `1/2` for `θ` instead of forwarding the parameter:

```mathematica
(* Inside EUPEq2Tot — θ is silently ignored: *)
EUP1totEq2[pi, p, κ, c, 1/2]  (* should be: ..., c, θ] *)
EUP2totEq2[pi, p, κ, c, 1/2]  (* should be: ..., c, θ] *)

(* Inside EUPEq3Tot — same issue: *)
EUP2totEq3[pi, p, κ, c, 1/2]  (* should be: ..., c, θ] *)
```

(Note: `EUPEq1Tot` also contains this hardcoding pattern but was the original function and may be intentional there.)

This means that even after fixing Bug 2, passing any `θ ≠ 0.5` to `EqPUtil` will have no effect on the Eq2 and Eq3 utility values. In the current calls — e.g. `EqPUtil[pi, p0, k, 1, γ0, .5]` — `θ` is always `0.5`, so this is dormant. But if you ever vary `θ`, these functions will silently ignore it.

---

## Summary Table

| # | Location | Type | Effect |
|---|---|---|---|
| 1 | `EQ3CONS` definition | Copy-paste error (`St1` vs `Rst2`) | `EQ3CONS` evaluates wrong region; Eq3 classification is incorrect |
| 2 | `EqPUtil` calling `EUPEqXTot` | Argument count mismatch (5 vs 6) | All `EUPEqXTot` calls return unevaluated; `EqPUtil` always returns `0` |
| 3 | Body of `EUPEq2Tot`, `EUPEq3Tot` | Hardcoded `θ = 1/2` ignores parameter | `θ` has no effect on Eq2/Eq3 utility values |

**The most likely cause of any immediate runtime failure is Bug 2** — it will silently cause `EqPUtil` to always return `0`, making `RegionPlot`/`Plot` appear blank or flat. Bug 1 is a logical error that produces wrong results without an error message. Bug 3 is a latent issue that activates if `θ` is ever varied.
