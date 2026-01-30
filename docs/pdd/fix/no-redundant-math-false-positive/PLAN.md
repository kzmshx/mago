# fix/no-redundant-math-false-positive

## Problem

`no-redundant-math` rule flags ALL instances of `expr * -1` as "redundant, use `-expr`".
This is technically correct for simple variables (`$x * -1` → `-$x`), but produces misleading
suggestions for constants and structured expressions where `* -1` conveys intent more clearly
than unary negation.

### Examples flagged (all 31 instances are `* -1`)

```php
// 1. Sign-flip of a named constant — intent is "negate the term"
VDateUtils::getCalcDate(VDateUtils::getToday(), self::ACTIVE_TERM_DAYS * -1)
// Suggested: -self::ACTIVE_TERM_DAYS  (arguably less readable)

// 2. Structural symmetry — return +1 or -1 with coefficient
return -1 * $coefficient;
return  1 * $coefficient;  // `1 * $coefficient` also flagged (by * 1 rule)

// 3. Ternary sign toggle
$this->dateInterval->invert ? $seconds * -1 : $seconds;
```

## Analysis

The rule at `crates/linter/src/rule/redundancy/no_redundant_math.rs` lines 178-204 matches
`(Some(-1), _) | (_, Some(-1))` unconditionally for multiplication. Unlike division and modulo
which have clear mathematical reasons to simplify, `* -1` as sign negation is a common idiom
that is not always less readable than unary `-`.

However, this is a **style preference issue**, not a correctness bug. The suggestion
`-$x` is always semantically equivalent to `$x * -1` in PHP. The question is whether the rule
should be less aggressive.

## Decision Point

This is borderline between "bug" and "style disagreement". Options:

1. **Lower severity**: Change default level from `Help` to something lower, or leave as-is
   (it is already `Help` level, the least severe)
2. **Exclude constant operands**: Only flag `$variable * -1`, not `CONSTANT * -1`
3. **No change**: Accept that `Help` level is advisory and doesn't fail builds

## Recommendation

Since the rule is already at `Help` level (advisory, non-failing) and the suggestion is
technically correct, this is **low priority**. The real-world impact is noise, not broken code.

If we proceed, option (2) is the most targeted fix: skip the report when the non-`-1` operand
is a constant/class-constant expression.

## Tasks

- [ ] Decide whether to proceed (this is advisory-level, not a correctness bug)
- [ ] If yes: add check to skip `* -1` report when non-`-1` operand is a constant expression
- [ ] Add test cases for constant `* -1` (should not be flagged) vs variable `* -1` (still flagged)
- [ ] Run existing tests

## Verification

- [ ] `cargo test -p mago-linter` passes
- [ ] `CONST * -1` is no longer reported
- [ ] `$var * -1` is still reported
