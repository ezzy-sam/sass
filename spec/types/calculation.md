# Calculations

## Table of Contents

* [Syntax](#syntax)
  * [`CalculationExpression`](#calculationexpression)
  * [`MinMaxExpression`](#minmaxexpression)
* [Types](#types)
  * [Operations](#operations)
    * [Equality](#equality)
  * [Serialization](#serialization)
    * [Calculation](#calculation)
    * [`CalculationOperation`](#calculationoperation)
    * [`CalculationInterpolation`](#calculationinterpolation)
* [Procedures](#procedures)
  * [Simplifying a Calculation](#simplifying-a-calculation)
  * [Simplifying a `CalculationValue`](#simplifying-a-calculationvalue)
* [Semantics](#semantics)
  * [`CalcExpression`](#calcexpression)
  * [`ClampExpression`](#clampexpression)
  * [`CssMinMax`](#cssminmax)
  * [`CalcArgument`](#calcargument)
  * [`CalcSum`](#calcsum)
  * [`CalcProduct`](#calcproduct)
  * [`CalcValue`](#calcvalue)

## Syntax

### `CalculationExpression`

This production is parsed in a SassScript context when an expression is expected
and the input stream starts with an identifier with value `calc` or `clamp`
(ignoring case) followed immediately by `(`.

The grammar for this production is:

<x><pre>
**CalculationExpression** ::= CalcExpression | ClampExpression
**CalcExpression**        ::= 'calc('¹ CalcArgument ')'
**ClampExpression**       ::= 'clamp('¹ CalcArgument ( ',' CalcArgument ){2} ')'
**CalcArgument**²         ::= InterpolatedDeclarationValue† | CalcSum
**CalcSum**               ::= CalcProduct (('+' | '-')³ CalcProduct)\*
**CalcProduct**           ::= CalcValue (('\*' | '/') CalcValue)\*
**CalcValue**             ::= '(' CalcArgument ')'
&#32;                       | CalculationExpression
&#32;                       | MinMaxExpression
&#32;                       | FunctionExpression⁴
&#32;                       | Number
&#32;                       | Variable†
</pre></x>

1: The strings `calc(` and `clamp(` are matched case-insensitively.

2: A `CalcArgument` is only parsed as an `InterpolatedDeclarationValue` if it
includes interpolation, unless that interpolation is within a region bounded by
parentheses (a `FunctionExpression` counts as parentheses).

3: Whitespace is required around these `"+"` and `"-"` tokens.

4: This `FunctionExpression` cannot begin with `min(`, `max(`, or `clamp(`,
case-insensitively.

†: These productions are invalid in plain CSS syntax.

> The `CalcArgument` production provides backwards-compatibility with the
> historical use of interpolation to inject SassScript values into `calc()`
> expressions. Because interpolation could inject any part of a `calc()`
> expression regardless of syntax, for full compatibility it's necessary to
> parse it very expansively.

### `MinMaxExpression`

This production is parsed in a SassScript context when an expression is expected
and the input stream starts with an identifier with value `min` or `max`
(ignoring case) followed immediately by `(`.

<x><pre>
**MinMaxExpression**¹ ::= CssMinMax | FunctionExpression
**CssMinMax**         ::= ('min(' | 'max(')² CalcArgument (',' CalcArgument)* ')'
</pre></x>

1: If both `CssMinMax` and `FunctionExpression` could be consumed, `CssMinMax`
   takes precedence.

2: The strings `min(`, `max(`, `calc(`, `env(`, `var(`, and `clamp(` are matched
   case-insensitively.

## Types

The value type known as a "calculation" has the following structure:

```ts
interface Calculation {
  name: string;
  arguments: CalculationValue[];
}

type CalculationValue =
  | Number
  | UnquotedString
  | CalculationInterpolation
  | CalculationOperation
  | Calculation;
  
interface CalculationInterpolation {
  value: string;
}

interface CalculationOperation {
  operator: '+' | '-' | '*' | '/';
  left: CalculationValue;
  right: CalculationValue;
}
```

### Operations

A calculation follows the default behavior of all SassScript operations, except
that it throws an error if used as an operand of a unary or binary `+` or `-`
operation, and equality is defined as below.

> This helps ensure that if a user expects a number and receives a calculation
> instead, it will throw an error quickly rather than propagating as an
> unquoted string.

#### Equality

Two calculations are considered equal if their names are equal, they have the
same number of arguments, and each argument in one calculation is equal to the
corresponding argument in the other.

`CalculationOperation` and `CalculationInterpolation` values are equal if each
field in one value is equal to the corresponding field in the other.

### Serialization

#### Calculation

To serialize a calculation, emit its name followed by "(", then each of its arguments
separated by ",", then ")".

#### `CalculationOperation`

To serialize a `CalculationOperation`:

* Let `left` and `right` be the result of serializing the left and right values,
  respectively.

* If either:

  * the left value is a `CalculationInterpolation`, or
  * the operator is `"*"` or `"/"` and the left value is a
    `CalculationOperation` with operator `"+"` or `"-"`,

  emit `"("` followed by `left` followed by `")"`. Otherwise, emit `left`.

* Emit `" "`, then the operator, then `" "`.

* If either:

  * the right value is a `CalculationInterpolation`, or
  * the operator is `"*"` and the right value is a `CalculationOperation` with
    operator `"+"` or `"-"`, or
  * the operator is `"/"` and the right value is a `CalculationOperation`,

  emit `"("` followed by `right` followed by `")"`. Otherwise, emit `right`.

#### `CalculationInterpolation`

To serialize a `CalculationInterpolation`, emit its `value`.

## Procedures

### Simplifying a Calculation

This algorithm takes a calculation `calc` and returns a number or a calculation.

> This algorithm is intended to return a value that's CSS-semantically identical
> to the input.

* Let `arguments` be the result of [simplifying](#simplifying-a-calculationvalue) each
  of `calc`'s arguments.

* If `calc`'s name is `"calc"`, the syntax guarantees that `arguments` contain
  only a single argument. If that argument is a number or calculation, return
  it.

* If `calc`'s name is `"clamp"`, `arguments` has fewer than three elements, and
  none of those are unquoted strings or `CalculationInterpolation`s, throw an
  error.

  > It's valid to write `clamp(var(--three-args))` or `clamp(#{"1, 2, 3"})`, but
  > otherwise `clamp()` has to have three physical arguments.

* If `calc`'s name is `"min"`, `"max"`, or `"clamp"` and `arguments` are all
  numbers:

  * If those arguments' units are mutually [compatible], return the result of
    calling [`math.min()`], [`math.max()`], or `math.clamp()` (respectively)
    with those arguments.

    [compatible]: ../spec/types/number.md#compatible-units
    [`math.min()`]: ../spec/built-in-modules/math.md#min
    [`math.max()`]: ../spec/built-in-modules/math.md#max

  * Otherwise, if any two of those arguments are [definitely-incompatible],
    throw an error.

    [definitely-incompatible]: #possibly-compatible-numbers

* Otherwise, return a calculation with the same name as `calc` and `arguments`
  as its arguments.

### Simplifying a `CalculationValue`

This algorithm takes a `CalculationValue` `value` and returns a
`CalculationValue`.

> This algorithm is intended to return a value that's CSS-semantically identical
> to the input.

* If `value` is a number, unquoted string, or `CalculationInterpolation`, return
  it as-is.

* If `value` is a calculation:

    * Let `result` be the result of [simplifying] `value`.

    * If `result` is a calculation whose name is `"calc"`, return `result`'s
      single argument.

    * Otherwise, return `result`.

  [simplifying]: #simplifying-a-calculation

* Otherwise, `value` must be a `CalculationOperation`. Let `left` and `right` be
  the result of simplifying `value.left` and `value.right`, respectively.

* Let `operator` be `value.operator`.

* If `operator` is `"+"` or `"-"`:

  * If `left` and `right` are both numbers with [compatible] units, return
    `left + right` or `left - right`, respectively.

  * Otherwise, if either `left` or `right` is a number with more than one
    numerator unit or more than zero denominator units, throw an error.

  * Otherwise, if `left` and `right` are [definitely-incompatible] numbers,
    throw an error.

  * If `right` is a number whose value is fuzzy-less-than zero, set `right` to
    `right * -1` and set `operator` to `"-"` or `"+"`, respectively.

  * Return a `CalculationOperation` with `operator`, `left`, and `right`.

* If `operator` is `"*"` or `"/"`:

  * If `left` and `right` are both numbers, return `left * right` or
    `math.div(left, right)`, respectively.

  * Otherwise, return a `CalculationOperation` with `operator`, `left`, and
    `right`.

## Semantics

### `CalcExpression`

To evaluate a `CalcExpression`:

* Let `calc` be a calculation whose name is `"calc"` and whose only argument is
  the result of [evaluating the expression's `CalcArgument`](#calcargument).

* Return the result of [simplifying] `calc`.

### `ClampExpression`

To evaluate a `ClampExpression`:

* Let `clamp` be a calculation whose name is `"clamp"` and whose arguments are the
  results of [evaluating the expression's `CalcArgument`s](#calcargument).

* Return the result of [simplifying] `clamp`.

### `CssMinMax`

To evaluate a `CssMinMax`:

* Let `calc` be a calculation whose name is `"min"` or `"max"` according to the
  `CssMinMax`'s first token, and whose arguments are the results of [evaluating
  the expression's `CalcArgument`s](#calcargument).

* Return the result of [simplifying] `calc`.

### `CalcArgument`

To evaluate a `CalcArgument` production `argument` into a `CalculationValue` object:

* If `argument` is an `InterpolatedDeclarationValue`, evaluate it and return a
  `CalculationInterpolation` whose `value` is the resulting string.

* Otherwise, return the result of [evaluating `argument`'s
  `CalcValue`](#calcvalue).

### `CalcSum`

To evaluate a `CalcSum` production `sum` into a `CalculationValue` object:

* Left `left` be the result of evaluating the first `CalcProduct`.

* For each remaining "+" or "-" token `operator` and `CalcProduct` `product`:

  * Let `right` be the result of evaluating `product`.

  * Set `left` to a `CalcOperation` with `operator`, `left`, and `right`.

* Return `left`.

### `CalcProduct`

To evaluate a `CalcProduct` production `product` into a `CalculationValue`
object:

* Left `left` be the result of evaluating the first `CalcValue`.

* For each remaining "*" or "/" token `operator` and `CalcValue` `value`:

  * Let `right` be the result of evaluating `value`.

  * Set `left` to a `CalcOperation` with `operator`, `left`, and `right` as its
    values.

* Return `left`.

### `CalcValue`

To evaluate a `CalcValue` production `value` into a `CalculationValue` object:

* If `value` is a `CalcArgument`, `CssMinMax`, or `Number`, return the result of
  evaluating it.

* If `value` is a `FunctionExpression` or `Variable`, evaluate it. If the result
  is a number, an unquoted string, or a calculation, return it. Otherwise, throw
  an error.

  > Allowing variables to return unquoted strings here supports referential
  > transparency, so that `$var: fn(); calc($var)` works the same as
  > `calc(fn())`.