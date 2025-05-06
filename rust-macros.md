---
layout: page
title: Rust Macros
permalink: /rust-macros/
---

#### How I Fell in Love with Rust Procedural Macros

I consider myself a junior Rust developer. I have been learning Rust for a few months now, and I have thoroughly enjoyed
the process. Recently, we started writing the  [next generation](https://github.com/FalkorDB/falkordb-rs-next-gen) of
Falkordb using Rust. We chose Rust because of its
performance, safety, and rich type system.

One part we are implementing by hand is the scanner and parser. We do this to optimize performance and to maintain a
clean AST (abstract syntax tree). We are working with
the [Antlr4 Cypher grammar](https://github.com/FalkorDB/falkordb-rs-next-gen/blob/main/graph/src/Cypher.g4), where each
derivation in the grammar
maps to a Rust function.

For example, consider the parse rule for a NOT expression:

```antlr
oC_NotExpression
             :  ( NOT SP? )* oC_ComparisonExpression ;
```

This corresponds to the Rust function:

```rust 
    fn parse_not_expr(&mut self) -> Result<QueryExprIR, String> {
    let mut not_count = 0;

    while self.lexer.current() == Token::Not {
        self.lexer.next();
        not_count += 1;
    }

    let expr = self.parse_comparison_expr()?;

    if not_count % 2 == 0 {
        Ok(expr)
    } else {
        Ok(QueryExprIR::Not(Box::new(expr)))
    }
}
```

Here, we compress consecutive NOT expressions during parsing, but otherwise, the procedure closely resembles the Antlr4
grammar. The function first consumes zero or more NOT tokens, then calls parse_comparison_expr

While working on the parser, a recurring pattern emerged. Many expressions follow the form:

```antlrv4
oC_ComparisonExpression
             :  oC_OrExpression ( ( SP? COMPARISON_OPERATOR SP? ) oC_OrExpression )* ;
```

which translates roughly to:

```rust
fn parse_comparison_expr(&mut self) -> Result<QueryExprIR, String> {
    let mut expr = self.parse_or_expr()?;

    while self.lexer.current() == Token::ComparisonOperator {
        let op = self.lexer.current();
        self.lexer.next();
        let right = self.parse_or_expr()?;
        expr = QueryExprIR::BinaryOp(Box::new(expr), op, Box::new(right));
    }

    Ok(expr)
}
```

Similarly, for addition and subtraction:

```antlrv4
oC_AddOrSubtractExpression
                       :  oC_MultiplyDivideModuloExpression ( ( SP? '+' SP? oC_MultiplyDivideModuloExpression ) | ( SP? '-' SP? oC_MultiplyDivideModuloExpression ) )* ;

```

which looks like this in Rust:

```rust
fn parse_add_sub_expr(&mut self) -> Result<QueryExprIR, String> {
    let mut vec = Vec::new();
    vec.push(self.parse_mul_div_modulo_expr()?);
    loop {
        while Token::Plus == self.lexer.current() {
            self.lexer.next();
            vec.push(self.parse_mul_div_modulo_expr()?);
        }
        if vec.len() > 1 {
            vec = vec!(QueryExprIR::Add(vec));
        }
        while Token::Dash == self.lexer.current() {
            self.lexer.next();
            vec.push(self.parse_mul_div_modulo_expr()?);
        }
        if vec.len() > 1 {
            vec = vec!(QueryExprIR::Sub(vec));
        }
        if ![Token::Plus, Token::Dash].contains(&self.lexer.current()) {
            return Ok(vec.pop().unwrap());
        }
    };
}
```

This pattern appeared repeatedly with one, two, or three operators. Although the code is not very complicated, it would
be nice to have a macro that generates this code for us.

We envisioned a macro that takes the expression parser and pairs of (token, AST constructor) like this:

```rust
parse_binary_expr!(self.parse_mul_div_modulo_expr()?, Plus => Add, Dash => Sub);
```

So I started exploring how to write procedural macros in Rust, and I must say it was a very pleasant experience. With
the help of the crates quote and syn, I was able to write a procedural macro that generates this code automatically. The
quote crate lets you generate token streams from templates, and syn allows parsing Rust code into syntax trees and token
streams. Using these two crates makes writing procedural macros in Rust feel like writing a compiler extension.

Let's dive into the code.

The first step is to model your macro syntax using Rust data structures. In our case, I used two structs:

```rust
struct BinaryOp {
    parse_exp: Expr,
    binary_op_alts: Vec<BinaryOpAlt>,
}

struct BinaryOpAlt {
    token_match: syn::Ident,
    ast_constructor: syn::Ident,
}
```

The leaves of these structs are data types from the syn crate. Expr represents any Rust expression, and syn::Ident
represents an identifier.

Next, we parse the token stream into these data structures. This is straightforward with syn by implementing the Parse
trait:

```rust
impl Parse for BinaryOp {
    fn parse(input: ParseStream) -> Result<Self> {
        let parse_exp = input.parse()?;
        _ = input.parse::<syn::Token![,]>()?;
        let binary_op_alts =
            syn::punctuated::Punctuated::<BinaryOpAlt, syn::Token![,]>::parse_separated_nonempty(
                input,
            )?;
        Ok(Self {
            parse_exp,
            binary_op_alts: binary_op_alts.into_iter().collect(),
        })
    }
}

impl Parse for BinaryOpAlt {
    fn parse(input: ParseStream) -> Result<Self> {
        let token_match = input.parse()?;
        _ = input.parse::<syn::Token![=>]>()?;
        let ast_constructor = input.parse()?;
        Ok(Self {
            token_match,
            ast_constructor,
        })
    }
}
```

The syn crate smartly parses the token stream into the data structures based on the expected types (Token, Expr, Ident,
or BinaryOpAlt).

The final step is to generate the appropriate code from these data structures using the quote crate, which lets you
write Rust code templates that generate token streams. This is done by implementing the ToTokens trait:

```rust
impl quote::ToTokens for BinaryOp {
    fn to_tokens(
        &self,
        tokens: &mut proc_macro2::TokenStream,
    ) {
        let binary_op_alts = &self.binary_op_alts;
        let parse_exp = &self.parse_exp;
        let stream = generate_token_stream(parse_exp, binary_op_alts);
        tokens.extend(stream);
    }
}

fn generate_token_stream(
    parse_exp: &Expr,
    alts: &[BinaryOpAlt],
) -> proc_macro2::TokenStream {
    let whiles = alts.iter().map(|alt| {
        let token_match = &alt.token_match;
        let ast_constructor = &alt.ast_constructor;
        quote::quote! {
            while Token::#token_match == self.lexer.current() {
               self.lexer.next();
               vec.push(#parse_exp);
            }
            if vec.len() > 1 {
                vec = vec![QueryExprIR::#ast_constructor(vec)];
            }
        }
    });
    let tokens = alts.iter().map(|alt| {
        let token_match = &alt.token_match;
        quote::quote! {
            Token::#token_match
        }
    });

    quote::quote! {
        let mut vec = Vec::new();
        vec.push(#parse_exp);
        loop {
            #(#whiles)*
            if ![#(#tokens,)*].contains(&self.lexer.current()) {
                return Ok(vec.pop().unwrap());
            }
        }
    }
}

```

In generate_token_stream, we first generate the collection of while loops for each operator, then place them inside a
loop using the repetition syntax `#(#whiles)*`. And that's it!

You can find the full code [here](https://github.com/FalkorDB/falkordb-rs-next-gen/tree/main/falkordb-macro)







