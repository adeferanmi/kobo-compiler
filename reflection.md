# CA1 Reflection

## 1. Numbers and the fractional part

The check is at `src/scanner.rs:131`. I only treat a dot as part of a number if the character after it is a digit. I used `peek()` and `peek_next()` for this because I don't want the scanner consuming a character it isn't supposed to. For `5.`, it reads `5` as a `NUMBER`, but it doesn't include the dot because there is no digit after it. When the scanner gets to the dot on its next iteration, it reports `Character is not part of any token.` That's what section 1.4 requires: a decimal point needs digits on both sides to belong to a number. The same rule is why `.5` doesn't count as a single number either. I initially thought number scanning was just about consuming digits, but the lookahead is what stops it from accepting an incomplete decimal.

## 2. Line counting and EOF

I increment `line` in two places: `src/scanner.rs:95` for a newline encountered during normal scanning, and `src/scanner.rs:112` for a newline inside a string. I need both because a string can span multiple lines, and I still have to report where its token ends. EOF was the part that gave me problems. At first, I was adding it using the normal `add()` function, which used the scanner's current line and sliced the source for its lexeme. So files with blank lines or test comments at the end gave me the wrong EOF line and even a newline as its lexeme. I fixed that at `src/scanner.rs:34-39`: I get the line from the last actual token and push EOF with an empty string. If a file ends with two blank lines, EOF still gets the last real token's line, as section 6.1 requires. If there are no tokens, it defaults to line 1.

## 3. A test I failed and what I learned

I failed `tests/phase-1/valid/eof_line.kobo` while working on the scanner. My mistake was assuming EOF should use whatever line the scanner had reached when it finished reading the file. I hadn't accounted for trailing whitespace and comments: they move the scanner forward without creating another real token. That made several valid tests fail even though the other tokens were being recognised correctly. The fix is at `src/scanner.rs:34-39`. Instead of calling `self.add(TokenType::Eof)`, I take the line of the last token and create EOF separately with an empty lexeme. After fixing this and an earlier identifier-slicing mistake, all 14 published tests passed. This taught me that matching the token types isn't enough; the exact output format and line-number rules matter too.

**Git history note:** My available history shows the starter commit `711ec39` and completed scanner commit `6ad2e0d`. The intermediate failing version was not committed, so I cannot truthfully provide the two before-and-after bug-fix commits requested for this question. In the completed commit, the relevant fixed lines are:

```rust
let eof_line = self.tokens.last().map_or(1, |token| token.line);

self.tokens.push(Token {
    kind: TokenType::Eof,
    lexeme: String::new(),
    line: eof_line,
});
```
