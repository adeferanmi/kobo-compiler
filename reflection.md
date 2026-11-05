# CA1 Reflection

## 1. Dangling Else

Lines `src/parser.rs:193` to `src/parser.rs:197` resolve the dangling else.

``` rust
    let else_branch = if self.matches(&[TokenType::Else]) {
    Some(Box::new(self.statement()?))
    } else {
        None
    };
```
Recursive descent resolves the ambiguity because the inner if_statement() call eagerly consumes an else token if one is present before returning control back to the outer if_statement(). The grammar itself is ambiguous, but the call stack enforces that the nearest matching if swallows the else first.   For if (false) if (true) print "a"; else print "b";, the parser outputs:

``` rust
    (if false (if true (print "a")) (print "b"))
```

If the else bound to the outer if instead, it would output:

``` rust
    (if false (if true (print "a") (print "b")))
```

Git Commit Anchors:

Wrong/Incomplete (Commit `f2e4d6c` / initial starter): `todo!("if_statement")`

Fixed (Commit `18aedd9`): `let else_branch = if self.matches(&[TokenType::Else]) { Some(Box::new(self.statement()?)) } else { None };`

## 2. Assignment Function and LL1

The check verifying the target of an assignment is at `src/parser.rs:266` to `src/parser.rs:272`:

``` rust
    if let Expr::Variable {name, at} = expr{
        return Ok(Expr::Assign {
            name,
            value: Box::new(value),
            at,
        });
    }
```

This check cannot happen before parsing the left side because assignment is not LL(1). Given an input like a = 1; versus a + 1;, reading the single token a does not provide enough lookahead to distinguish between a variable assignment target and the beginning of a larger arithmetic expression. The parser must parse the entire left-hand side as an ordinary expression first, and then retrospectively inspect whether the evaluated AST node turned out to be a bare variable identifier.   Executing 1 = 2; produces the exact error message specified in Section 5.1:

`"Cannot assign to this expression."`

Git Commit Anchors:

Wrong/Incomplete (Commit `be4d59b`): `self.or()`

Fixed (Commit `1622563`): `if let Expr::Variable { name, at } = expr {`

## 3. Synchronisation

The synchronise method is implemented at `src/parser.rs:497-524`. It advances past the error token and discards tokens until `self.previous().kind` is `TokenType::Semicolon` or `self.peek().kind` matches one of the statement-beginning keywords: `Fun`, `Var`, `If`, `While`, `Print`, or `Return`. These tokens are targeted because a semicolon marks the end of a broken statement, and these keywords reliably signal the start of a new top-level declaration or statement, re-aligning the parser safely.   If the call to `synchronise()` at line 18 is deleted, the invalid test `tests/phase-2/invalid/expect_semicolon_after_value.kobo` changes output. Instead of producing a single error:

`[line 2] Error at 'print': Expect ';' after value.`

Wrong/Incomplete (Commit `0630a6e`): Infinite step / double increment logic inside recovery loop: `self.depth = 0; while !self.at_end() { self.current += 1; ... }`

Fixed (Commit `33b7403`): `if !self.at_end() { self.current += 1; }`