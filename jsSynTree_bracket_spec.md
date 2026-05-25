# jsSynTree Labelled Bracket Notation Specification

This document provides the formal specification for the labelled bracket notation format used in **jsSynTree**. It describes the grammar, parser rules, exporter behaviour, character escaping, and visual customisations supported by this format.

---

## 1. Syntax Overview

The bracket notation in jsSynTree is an extension of standard labelled bracket notation (commonly used in generative linguistics to represent phrase structure trees). It maps text-based hierarchical structures directly to nodes, dominance relations, styles, and feature matrices on the visual canvas.

### Core Structure
The notation represents a hierarchical tree of nodes. A node may contain:
- A text label (with optional formatting and unique ID reference).
- An optional shape style (e.g. circles or boxes).
- An optional feature matrix (a list of semicolon-separated features).
- Zero or more nested child nodes.

---

## 2. Formal Grammar (EBNF)

The formal grammar for the parser is defined below in Extended Backus-Naur Form (EBNF):

```ebnf
tree             ::= node ;

node             ::= open-bracket, whitespace?, label, style?, features?, children*, close-bracket
                   | bare-word ;

open-bracket     ::= "[" | "(" ;
close-bracket    ::= "]" | ")" ;

label            ::= label-char+, ( "_" , id )? ;

style            ::= "|" , ( "box" | "circle" ) ;

features         ::= "{" , feature , ( ";" , feature )* , "}" ;

children         ::= whitespace+ , node ;

bare-word        ::= word-char+ ;

label-char       ::= (* Any character except unescaped '[', ']', '(', ')', '{', '}', '|', or whitespace *) ;
word-char        ::= (* Any character except '[', ']', '(', ')', or whitespace *) ;
feature-char     ::= (* Any character except unescaped ';' or '}' *) ;
id               ::= [A-Za-z0-9]+ ;
whitespace       ::= ? Unicode whitespace character ? ;
```

---

## 3. Lexical and Parser Behaviour

The `BracketParser` parses the input string character-by-character using the following rules:

### 3.1 Bracket Normalisation
On import, the parser first normalises all parentheses. Square brackets `[` / `]` and round parentheses `(` / `)` are treated as completely interchangeable. All instances of `(` are replaced with `[` and all instances of `)` are replaced with `]` prior to parsing.
On export, the canonical square brackets `[` and `]` are always used.

### 3.2 Comments
Lines starting with the `%` character (ignoring any leading whitespace) are treated as comments and are stripped before parsing begins.
```text
% This is a comment and will be ignored
[TP [NP John] [VP left]]
```

### 3.3 Trailing Input
The parser expects a single root-level node representing the main tree. If the input contains trailing characters (including additional root-level bracketed groups) after the first complete root node has been parsed, the trailing input is ignored, and a warning is logged in the developer console.

### 3.4 Bare Words (Leaf Nodes)
A bare word is a sequence of characters not enclosed in brackets and containing no whitespace. It is parsed as a leaf node with:
- The label set to the word text (with backslash escapes resolved).
- Style set to `'none'`.
- An empty feature matrix.

For example, `[NP John]` parses `John` as a bare word child of `NP`. This is equivalent to writing `[NP [John]]`.

### 3.5 Node Labels and ID Suffixes
A node label begins immediately after an open bracket and optional whitespace. It is read until the parser encounters an unescaped delimiter: whitespace, `|`, `{`, `[`, or `]`.

#### ID Suffix Hint (`_id`)
If a label ends with an underscore followed by an alphanumeric sequence (e.g. `_A` or `_1`), this sequence is treated as a node ID hint.
- **Regular Expression**: `/^(.+)_([A-Za-z0-9]+)$/`
- **Behaviour**: The suffix (including the underscore) is stripped from the visual label, and the captured ID is retained internally as `_bracketId`.
- **Purpose**: While the import engine currently disregards this hint (as movement arrows and decorations are stored in JSON saves and cannot be fully reconstructed from bracket notation alone), the exporter automatically generates these suffixes to indicate nodes that are targets of movement arrows or decorations.

---

## 4. Visual Modifiers

### 4.1 Node Shape Styles
A node can be styled visually by appending `|box` or `|circle` directly to the label (without intervening whitespace):
- `|box`: Renders the node label inside a rectangular border.
- `|circle`: Renders the node label inside a circular border.
- If no style is specified (or an unsupported style name is provided), it defaults to `none` (plain text with no enclosing border).

```text
[NP|circle John]      % Renders 'NP' inside a circle
[T|box {EPP}]         % Renders 'T' inside a box, with feature EPP
```

### 4.2 Feature Matrices
Features are enclosed in curly braces `{...}` following the label and optional style.
- Individual features inside the braces are separated by semicolons (`;`).
- Leading and trailing whitespace around each individual feature is trimmed.
- Empty features are ignored.
- Features are rendered within square brackets directly below the node label.
- The rendering style of the feature brackets (e.g. `'rounded'`, `'angular'`, or `'serif'`) is controlled by the application's settings and is applied globally on the canvas.

```text
[NP {uφ; +nom} [N cat]]    % Two features: "uφ" and "+nom"
```

---

## 5. Escaping Special Characters

To include characters that would otherwise act as delimiters inside labels or features, you must escape them with a backslash (`\`).

| Delimiter Character | Purpose in Syntax | Escape Sequence | Resulting Text |
| :--- | :--- | :--- | :--- |
| `\|` | Style modifier | `\|` | Literal pipe (`|`) inside a label |
| `\;` | Feature separator | `\;` | Literal semicolon (`;`) inside a feature |
| `\n` | Line break | `\n` | Line break (newline) inside a label |

### Parser Escaping Logic
1. When parsing the **label**, only `\|`, `\;`, and `\n` are recognised during tokenisation to skip delimiter checks.
2. Other escaped character sequences of the form `\c` (such as `\[` or `\]`) will **not** prevent the delimiter from terminating the label.
3. Upon successfully capturing the raw string, the parser runs a decoding function that replaces `\n` with a literal newline character, and any other sequence `\c` with `c`.

---

## 6. Exporter Behaviour

The `BracketExporter` generates a canonical bracketed string representing the canvas state using the following logic:

### 6.1 Multi-line vs Single-line Formatting
To ensure readability, the exporter dynamically chooses between single-line and multi-line formats:
- **Multi-line format** (indented with 2 spaces per depth level) is used if:
  - The maximum depth of the tree is greater than 2, **OR**
  - The total number of nodes in the tree is greater than 4.
- **Single-line format** is used for smaller trees.

### 6.2 Disconnected Nodes
Nodes that have no parent and no children (completely isolated nodes) are excluded from the main hierarchical trees. Instead, they are appended to the very end of the exported text under a `% disconnected:` comment block, with one node per line.

### 6.3 Automatic ID Suffix Generation
To preserve relationships for external processing, the exporter automatically generates sequential alphabetic ID suffixes (e.g. `_A`, `_B`, ... `_Z`, `_AA`, ...) for nodes that:
1. Are connected to a movement arrow (either as the source or destination).
2. Have their geometric centre enclosed within a decoration box.

These suffixes are appended to the node label in the exported string (e.g. `[NP_A]`).

---

## 7. Examples

### 7.1 Simple Single-Line Tree
A small tree with a depth of 2 and 4 nodes:
```text
% geometry, movement arrows, and decorations are not represented
[TP [NP John] [VP left]]
```

### 7.2 Complex Multi-Line Tree
A larger tree exported with multi-line formatting:
```text
% geometry, movement arrows, and decorations are not represented
[TP
  [NP
    [D the]
    [N cat]
  ]
  [T'
    [T sat]
    [VP
      [V t]
    ]
  ]
]
```

### 7.3 Styled Nodes and Feature Matrices
Demonstrates `circle` and `box` node styles, newline character escapes (`\n`), bold and italic markup, and a feature matrix:
```text
% geometry, movement arrows, and decorations are not represented
[TP
  [NP|circle {uφ; +nom}
    [D the]
    [N cat]
  ]
  [T'
    [T|box {EPP}]
    [VP
      [V saw]
      [NP|box {+acc}
        [D the]
        [N dog]
      ]
    ]
  ]
]
```

### 7.4 Multi-line Node Label
A node label that renders on two lines on the canvas:
```text
% geometry, movement arrows, and decorations are not represented
[DP\n*i*|circle {uφ; EPP}]
```

### 7.5 References for Movement Arrows and Decoration Boxes
Nodes `DP` and `t` are marked with ID suffixes `_A` and `_B` because they are connected by a movement arrow:
```text
% geometry, movement arrows, and decorations are not represented
[TP
  [DP_A [D the] [N cat]]
  [VP
    [V chased]
    [DP_B t]
  ]
]
```
