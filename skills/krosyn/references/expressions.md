# Krosyn expressions

Every value in a connector (trigger, URLs, headers, query parameters, process fields, perform
body) is an expression. `validate_expression` checks syntax and `test_expression` evaluates
against sample data.

This page covers the syntax and the pitfalls. It is not the complete function list: call
`list_functions` for the live list, and see https://krosyn.com/docs/expressions for every
signature.

## Syntax

| Form | Example |
|---|---|
| Field reference | `$payload.user.name`, `$lookup.data[0].id` |
| Negative index (from the end) | `$payload.items[-1]` is the last element; out of range is null |
| String interpolation | `"Hello $payload.name, order $payload.order_id is confirmed"` |
| String literal | `'hello'` or `"hello"` |
| Function call | `uppercase($payload.name)` |
| Chained accessor on a field | `$payload.name.uppercase.trim` |
| Array literal | `["a", "b", $payload.x]` |

Accessors chain onto **field references only**, never onto a function result:
write `uppercase(concat($payload.first, ' ', $payload.last))`, not `concat(...).uppercase`.

### Operators

Operators are shorthand for functions; both forms always work.

| Operator | Function |
|---|---|
| `+ - * / %` | `add subtract multiply divide modulo` |
| `== !=` | `equals notEquals` (strict: `1 == "1"` is false) |
| `> < >= <=` | `greaterThan lessThan gte lte` (numeric) |
| `&& \|\| !` | `and or not` |

Precedence, high to low: unary `!` and `-`; `* / %`; `+ -`; `> < >= <=`; `== !=`; `&&`; `||`.
Parentheses group.

Operators are recognised only when the whole value is an expression, so bare literals stay
text: `2026-07-31`, `+1-800-555-0100` and `ORD-1001` are not arithmetic.

## Functions

| Category | Functions |
|---|---|
| string | `uppercase lowercase trim concat replace substring split startsWith endsWith padStart padEnd repeat` |
| conditional | `if` |
| comparison | `equals notEquals neq contains greaterThan gt lessThan lt gte lte` |
| logical | `and or not` |
| math | `add subtract multiply divide modulo round ceil floor abs min max` |
| decimal | `decimalAdd decimalSubtract decimalMultiply decimalDivide decimalRound decimalSum decimalCompare` |
| array | `join count first last includes map filter flatten unique sort sum groupBy toCsv` |
| date | `now formatDate timestamp dateDiff` |
| type | `toNumber toString toBoolean isNull isEmpty` |
| fallback | `default coalesce` |

Names are case-sensitive. Use the `decimal*` functions for money, where floating-point
rounding is not acceptable. This table was correct when written; `list_functions` is the live list.

### Argument order

Argument order is not uniform. These put the **subject last**, which trips up most first attempts:

| Function | Signature | Example |
|---|---|---|
| `replace` | `replace(search, replacement, subject)` | `replace('-', ' ', $payload.slug)` |
| `split` | `split(delimiter, value)` | `split(',', $payload.tags)` |
| `join` | `join(separator, array)` | `join(', ', $payload.tags)` |

The rest put the subject first:

| Function | Signature |
|---|---|
| `substring` | `substring(value, start, length?)` (zero-based) |
| `contains` | `contains(haystack, needle)` (text only; use `includes(array, value)` for arrays) |
| `startsWith` / `endsWith` | `startsWith(value, prefix)` |
| `padStart` / `padEnd` | `padStart(value, length, pad)` |
| `repeat` | `repeat(value, count)` |
| `round` | `round(value, precision?)` |
| `map` | `map(array, field)`: plucks one field from each element |
| `filter` | `filter(array, field, value)`: keeps elements whose field equals the value |
| `formatDate` | `formatDate(date, format)`: PHP date format such as `'Y-m-d'` |
| `dateDiff` | `dateDiff(date1, date2)`: seconds from date1 to date2, positive when date2 is later |

Other behaviour worth knowing:

- `if(condition, then, else?)`: `if(equals($payload.status, 'paid'), 'Paid', 'Open')`
- `default(value, fallback)` returns the fallback when the value is null or missing;
  `coalesce(a, b, ...)` returns the first non-null argument.
- `divide(a, 0)` returns null rather than failing.
- `isEmpty(value)` is true for null, an empty string, an empty array and false.
- `now()` returns the current time in ISO 8601.
- `formatDate` and `dateDiff` accept ISO strings and Unix timestamps (quoted or not). A missing
  date returns null, never today's date.

Combined: `join(', ', map(filter($payload.items, 'status', 'active'), 'sku'))` gives `"X, Z"`.

Array functions are limited to 10,000 elements. Fields with sensitive names (passwords and
similar) cannot be read through `map` and `filter`.

## Examples

Trigger only for new orders with a total above zero:

```
$payload.type == 'order.created' && $payload.total > 0
```

Lookup URL with a payload value:

```
https://api.example.com/customers/$payload.customer_id
```

Process fields, evaluated in order:

```json
{
  "full_name": "concat($lookup.first_name, ' ', $lookup.last_name)",
  "amount": "round($payload.total, 2)",
  "summary": "\"$processed.full_name paid $processed.amount\""
}
```

Keep expressions shallow. If one needs more than two levels of nesting, split it into process
fields and reference them with `$processed.NAME`.
