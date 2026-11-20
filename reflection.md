# CA3 Reflection — The Complete Compiler

### Question 1: Short-Circuiting `and` and `or`

In `src/interpreter.rs:146-156`, `and` and `or` evaluate their left operand and return early without evaluating the right operand whenever the outcome is already determined:

```rust
Expr::Logical { left, op, right } => {
    let l = evaluate(left, env, program)?;
    if op.kind == TokenType::Or {
        if l.is_truthy() {
            return Ok(l);
        }
    } else if !l.is_truthy() {
        return Ok(l);
    }
    evaluate(right, env, program)
}
```

Per section 4.1 of the specification, logical operators evaluate to one of their operands rather than coercing to a boolean. For `nil or "fallback"`, `nil` is falsey (`!l.is_truthy()`), so the evaluator proceeds to the right operand and returns `"fallback"`. For `0 and 1`, `0` is truthy in Kobo (section 2.1), so `and` does not short-circuit and evaluates and returns `1`. If the implementation returned `Value::Bool(true)` or `Value::Bool(false)` instead, tests using `and`/`or` strictly inside `if` or `while` conditions would still pass (because truthiness behaves identically), whereas tests printing or storing the resulting operand value directly (such as `print nil or "fallback";`) would fail.

### Question 2: Compile-Time Resolution and Stack Discipline

In `src/resolver.rs:64-74`, `lookup` resolves local variables statically before execution:

```rust
fn lookup(&self, name: &str) -> Resolved {
    for (slot, local) in self.locals.iter().enumerate().rev() {
        if local.initialised && local.name == name {
            return Resolved::Local {
                depth: self.depth - local.scope,
                slot,
            };
        }
    }
    Resolved::Global
}
```

Resolving a variable to a fixed stack `slot` at compile time makes the VM substantially faster than the tree-walking interpreter because `Op::GetLocal(slot)` performs a direct $O(1)$ vector index lookup (`self.stack[base + slot]`) rather than traversing a chain of hash maps and hashing string identifiers on every variable access.

In `src/compiler.rs:172-177`, `end_scope` emits one `Op::Pop` per local declared in a block:

```rust
fn end_scope(&mut self, locals: usize) {
    let line = self.chunk.last_line();
    for _ in 0..locals {
        self.chunk.write(Op::Pop, line);
    }
}
```

If `end_scope` emits one `Pop` too few, a dead local value remains orphaned on the VM stack after the block exits, shifting the stack offsets (`base + slot`) of all subsequent local declarations in enclosing scopes or loops. The `phase-4` suite catches this bug (e.g., in loop and slot-allocation tests where leaked values accumulate or misalign slots), whereas the `phase-3` suite does not catch it because `phase-3` runs exclusively on the tree-walking interpreter, which manages scopes via `Environment::pop_scope()` rather than bytecode stack offsets.

### Question 3: Extension — Conditional Expression (`c ? a : b`)

To implement the right-associative conditional expression (`c ? a : b`) binding just above assignment, I added the following code across the compiler pipeline:

* **Lexing (`src/token.rs:26-27`, `src/token.rs:69-70`, `src/scanner.rs:73-74`):** Added `TokenType::Question` (`?`) and `TokenType::Colon` (`:`).
* **AST & Printing (`src/ast.rs:84-88`, `src/printer.rs:42-46`):** Added `Expr::Conditional { condition, then_expr, else_expr }` and formatted it as `(?: cond then else)`.
* **Parsing (`src/parser.rs:224`, `src/parser.rs:244-257`):** Updated `assignment` to call `conditional()`, which parses `or()`, matches `?`, parses `then_expr` with `expression()`, consumes `:`, and recursively calls `conditional()` for `else_expr` to enforce right-associativity.
* **Resolution & Interpretation (`src/resolver.rs:138-145`, `src/interpreter.rs:212-220`):** Resolved all three sub-expressions and evaluated only the selected branch based on `condition.is_truthy()`.
* **Bytecode Compilation (`src/compiler.rs:155-169`):** Emitted `JumpIfFalse` around the `then` branch, `Pop` before each branch so the condition value does not leak onto the stack, and an unconditional `Jump` after the `then` branch to skip the `else` branch.

The bytecode path was harder than the tree walk because the stack machine requires manual control-flow jump patching (`patch(else_jump)` and `patch(end_jump)`) and explicit stack discipline (`Op::Pop` on both control paths) to ensure the condition value is discarded before evaluating the chosen branch, leaving a net stack effect of $+1$. In the tree walk, Rust's native `if/else` handles both control flow and value return automatically without manual stack cleanup.

* **Broken commit (`f832703`):** `if self.frames.len() >= MAX_FRAMES { return Err(error(line, "Stack overflow.")); }` (and before `conditional()` was added to `src/parser.rs`)
* **Fixed commit (`4158844`):** `if self.frames.len() > MAX_FRAMES { return Err(error(line, "Stack overflow.")); }` *(Note: after committing your extension files, replace `4158844` with that final extension commit hash and quote `let expr = self.conditional()?;`)*