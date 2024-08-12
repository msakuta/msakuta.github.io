# Rusty-parser optimization

In developing multi-dimensional arrays in [rusty-parser](https://github.com/msakuta/rusty-parser) project, we stumbled upon an issue that parsing take **insanely** slow sometimes, even with a very simple script:

```
var a = array([[[1]]]);

print(a, type(a));
```

This seemingly naive script takes more than 2 seconds to parse even in release build.

```
real    0m2.447s
user    0m2.446s
sys     0m0.001s
```

This inefficiency quickly bloats as you add another level of nested arrays. Even replacing the function call with another extra level of array:

```
var a = [[[[1]]]];
```

makes it unreasonably slow.

```
real    0m28.689s
user    0m28.689s
sys     0m0.000s
```

## The cause

I knew recursive descent parser (or LL(k) parsers in general) needs to backtrack if one of the candidates does not match if there are multiple ways to parse a specific code. However, I did not expect exponential growth with simple expression like array literals.

This is important since we want to build array manipulation language and the level of nested arrays can be 3 to 4 levels quickly.

I found that we use a pattern like below pretty often.

```rust
fn array_rows(i: Span) -> IResult<Span, Vec<Vec<Expression>>> {
    let (r, (mut val, last)) = pair(
        many0(terminated(array_row, ws(tag(";")))), opt(array_row)
    )(i)?;

    if let Some(last) = last {
        val.push(last);
    }
    Ok((r, val))
}
```

This means a list of `array_row` separated by a semicolon, optionally with a trailing semicolon, so both `[1; 2]` and `[1; 2; ]` are accepted. However, this will call `array_row` twice if the input has no semicolons. In itself, this will only double the number of backtracks, but nesting this kind of parsers will quickly grow the number of backtracks.

The fix is easy. Instead of try `<array_row> ;` and try `<array_row>` again, we can parse `<array_row>` first, and concatenate `;` later.

```rust
fn array_rows(i: Span) -> IResult<Span, Vec<Vec<Expression>>> {
    terminated(separated_list0(char(';'), array_row), opt(ws(char(';'))))(i)
}
```

It's a bit more complicated code, but the effect is remarkable as we see later.
## Measuring invokes

I injected code like below to count the calls to each parser function. Since `nom` doesn't allow us to put context variables, we need to put a global variable to keep track of number of calls. Thankfully, it's not too difficult, even though Rust doesn't allow mutable globals:

```rust
static ARRAY_LIT: std::sync::atomic::AtomicUsize = std::sync::atomic::AtomicUsize::new(0);

pub(crate) fn array_literal(i: Span) -> IResult<Span, Expression> {
    ARRAY_LIT.fetch_add(1, Relaxed);
    // ...
    Ok((r, Expression::new(ExprEnum::ArrLiteral(val), span)))
}
```

Before the improvement to `array_rows`:

```
    array_rows: 225888
    array_literal_calls: 112968
    primary_expression_calls: 5421384
    postfix_expression_calls: 1807132
    fn_invoke_arg: 4
    cmp_expr: 903566
    expr_calls: 451783
```

After:

```
    array_rows: 28848
    array_literal_calls: 28872
    primary_expression_calls: 692424
    postfix_expression_calls: 230812
    fn_invoke_arg: 4
    cmp_expr: 115406
    expr_calls: 57703
```

We can apply the same optimization to `array_row`, which does almost the same thing as `array_rows` with `,` delimiter.

```
    array_rows: 7536
    array_literal_calls: 7560
    primary_expression_calls: 90504
    postfix_expression_calls: 30172
    fn_invoke_arg: 4
    cmp_expr: 15086
    expr_calls: 7543
```

We can further optimize `cmp_expr` parser to not repeat `expr`

```
    array_rows: 516
    array_literal_calls: 540
    primary_expression_calls: 3132
    postfix_expression_calls: 1048
    fn_invoke_arg: 4
    fn_invoke: 1048
    cmp_expr: 1048
    expr_calls: 524
```

I also optimized `postfix_expression` and `assign_expr` in a similar manner, and now the performance look optimal:

```
    array_rows: 3
    array_literal_calls: 5
    primary_expression_calls: 6
    postfix_expression_calls: 7
    fn_invoke_arg: 1
    fn_invoke: 7
    cmp_expr: 7
    assign_expr_calls: 7
```
