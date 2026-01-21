---
layout: "post"
title:  "Introducing ungrammar"
date:   "2050-01-21 14:23:35 +0530"
categories: rust
--- 

This post introduces ungrammars: a new formalism for describing concrete syntax trees. The ideas behind ungrammar are simple, and are more valuable than a specific implementation. Nonetheless, an implementation is available here: https://github.com/rust-analyzer/ungrammar

At a glance, ungrammar looks a lot like EBNF notation:

```
Module = 
    Attr* Visibility?
    `mod` Name
    (ItemList | ';')
```

The two differ at a fundamental level though:

> EBNF specifies a language - a set of strings
> Ungrammar describes concrete syntax tree - a set of datatypes (or a set of trees, if you will).

That's why it is called ungrammar!

## Motivation

So, what exactly does "describing syntax tree" mean and why is it useful? When writing an IDE, one of the core data structure is the concrete syntax tree. It is a full-fidelity tree which represents the original code in details, including parenthesis, comments and whitespace. CSTs are used for initial analysis of the language. They are also a vocabulary type for refactors. Although the ultimate result of a refactor is a text diff, tree modification is a more convenient internal representation.

At the lowest level, the CST is typically unityped: there's some Node superclass, which has a collection of Node children and an optional Node parent. On top of this raw layer, a more AST-like API is provided: Struct has a .name() and a list of .fields(), etc. This typed API is huge! For rust-analyzer, it is comprised of more than 130 types! And it is also more detailed than a typical AST: Struct also has .l_curly() and .r_curly().

What's worse, this API changes a lot, especially at the beginning. You may start with nesting `.fields()` directly under the Struct, but then introduce a StructFields node for everything between the curly braces to share the code with enum variants.

In short, writing this by hand sucks :-> Ungrammar is a notation to concisely describe the structure of the syntax tree, which can be used by a code generator to build an API in the target language. If you've heard about ASDL, ungrammar is ASDL for concrete syntax trees. For rust-analyzer's case, that means taking the following input:

```
Module = 
    Attr* Visibility?
    'mod' Name
    (ItemList | ';')
```

And generating the following output:

```rust
impl ast::AttrsOwner for Module {}
impl ast::VisibilityOwner for Module {}
impl ast::NameOwner for Module {}
impl Module {
    pub fn mod_token(&self) -> Option<SyntaxToken> { ... }
    pub fn item_list(&self) -> Option<ItemList> { ... }
    pub fn semicolon_token(&self) -> Option<SyntaxToken> { ... }
}
```

In typical parser generators, something similar can be achieved by generating both the parser and the syntax tree from the same grammar. This works to some extent, but has an inherent problem that the shape of the tree you want for the programmatic API, and the shape of the grammar you need to implement the parser are often different. "Technical" transformations like left-recursion elimination don't affect the language described by the grammar, but completely change the shape of the parse tree. In contrast, ungrammar focuses solely on the second task, which radically reduces the complexity of the grammar. In rust-analyzer, it is paired with a hand-written parser.

Treated as an ordinary (context free) grammar, ungrammar describes a superset of the language. For example, the programmatic API it might be convenient to treat commas in comma-separate lists as a part of the list element (rust-analyzer doesn't do this yet, but it should). This leads to the following ungrammar, which obviously doesn't treat commas precisely:

```
FieldList = 
    '{' Field* '}'
Field:
    Name ':' Type ','?
```

Similarly, ungrammar defines binary and unary expressions, but doesn't specify their relative precedence and associativity.

An interesting side-effect is that the resulting grammars turn out to be pretty human readable. For example, a full production ready Rust grammar takes around 600 short lines. This might be a good fit for reference documentation!.

## Nuts and Bolts

Now that we've answered the "why" question, let's look at how ungrammar works.

Like grammars, ungrammar operate with a set of terminals and non-terminals. Terminals are atomic indivisible tokens, like keyword `fn` or a semicolon: `;`. Non-terminals are composite internal nodes consisting of other nodes and tokens.

Tokens (terminals) are spelled using quotes: '+', 'fn', 'ident', 'int_number'. Tokens are defined outside of an ungrammar, and don't need to be declared to use them. By convention, keywords and punctuation are represented using themselves, other tokens use lower_snake_case. Because ungrammar describes trees, it uses parser tokens rather than lexer tokens. What this means is that context sensitive keywords like default are recognised as separate tokens('default'). The same goes for composite tokens like '<<'.

Nodes(non-terminals) are defined within the grammar by associating node name and a rule. The ungrammar itself is a set of node definitions. By convention, nodes are named using UpperCamelCase. Each node must be defined exactly once. Rules are regular expressions over the set of tokens and nodes.

Here's ungrammar which decribes ungrammar syntax:

```
Grammar = 
    Node*
Node = 
    name: 'ident' '=' Rule

Rule = 
    'ident'                     // Alphabetic identifier
|    'token_ident'               // Single quoted string
|    Rule*                       // Concatenation
|    Rule('|' Rule)*             // Alteration
|    Rule '?'                    // Zero or one repetition
|    Rule '*'                    // Kleene star
|    '(' Rule ')'                // Grouping
|    label: 'ident' ':' Rule     // Labeled rule
```
The only unusual thing are optional labels. By default, the names in the generated code are derived automatically from the type, but a label can be used as an override, or if there's an ambiguity:

```
Expr = 
    literal
|   lhs:Expr op:('+' | '-' | '*' | '/') rhs:Expr
```

By convention, ungrammar is indented with two spaces, leading | is not indented.

Ungrammar doesn't specify any particular way to lower rules to syntax node definitions. It's up to the generator to pattern-match rules to target language contructs: java would use inheritance, Rust enums and Typescript - union types. The generator can accept only a subset of all possible rules. An example of restriction might be: "Alteration (|) is only allowed at the top level. Alternatives must be other nodes". With this restriction, an alternative can be lowered to an interface definition with a number of subclasses.

The ungrammar crate provides a Rust API for parsing ungrammars, use it if you code generator is implemented in Rust. Alternatively, ungrammar2json binary converts ungrammar syntax into equivalent JSON. FOr an example of generator, take a look at gen_syntax in rust-analyzer.

## Designing ungrammar

The concluding section briefly mentions lessons learned.

The Node and Token terminology is inherited from rowan, rust-analyzer's syntax library. A better choice would be Tree and Token, as nodes contain other nodes and are trees.

Always single quoting terminals is a nice concrete syntax for grammars. Some parser generators I've worked with required only some terminals to be quoted, which, without knowing the rules by heart, reduced readability. Similarly, spelling PLUS instead of '+' is not very readable.

"Recursive regular expressions" feels like a convenient syntax for CFGs. Not restricting right-hand-side to be a flat list of alternatives, using () for grouping and allowing basic conveniences like `*` and `?` subjectively makes the resulting quite readable. The catch is that one needs union types and anonymous records to faithfully lower arbitraty regex-represented rule. Placing restrictions into the specific generator, rather then the base language, feels like a better division responsibility.

By quoting terminals, using punctuation (: = () | * ?) for syntax and completely avoiding keywords, ungrammar avoids clashes between names of productions and the syntax of ungrammar  itself.
