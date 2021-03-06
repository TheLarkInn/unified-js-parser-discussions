# Parser Features

The following is a list of all possible parser features, along with commonly agreed terminology for them.

- [Node location](#node-location)
- [Token collection](#token-collection)
- [Comment collection](#comment-collection)
- [Comment attachment](#comment-attachment)
- [Language versioning](#language-versioning)
- [Event delegation](#event-delegation)
- [Exported parser class](#exported-parser-class)
- [Pure tokenization mode](#pure-tokenization-mode)

## Node Location

Node location refers to the addition of properties to each syntax node to denote the location of the node in the source text.

### Location object: Variant A

Implemented by: SpiderMonkey parser, Esprima, Acorn, Flow parser.

The `loc` property is an object with two properties, `start` and `end`.
Each property represents a position, a object of a `line` number (one-indexed) and a `column` number (zero-indexed):

If the node contains no information about the source location, `loc` is null.

```
interface Node {
    type: string;
    loc: SourceLocation | null;
}
interface SourceLocation {
    start: Position;
    end: Position;
}
interface Position {
    line: number; // >= 1
    column: number; // >= 0
}
```

Example:
```js
> parser.parse('x = 2', options);
Program {
  type: 'Program',
  body:
   [ ExpressionStatement {
       type: 'ExpressionStatement',
       expression: [Object],
       loc: [Object] } ],
  loc: { start: { line: 1, column: 0 }, end: { line: 1, column: 5 } } }
```

### Location object: Variant B

Implemented by: Traceur parser.

```
interface Node {
    type: string;
    location: Location;
}
interface Location {
    start: Position;
    end: Position;
}
interface Position {
    offset: number;
    line: number;
    column: number;
}
```

### Zero-based character range

Implemented by: Esprima, Acorn, Flow parser.

The `range` property is a two-element array that holds the start and end position (exclusive).
Each array is an integer number where 0 points to the beginning of the source text.

```
interface Node {
    type: string;
    range: Range;
}
```

Example:
```js
> parser.parse('x = 2', options);
Program {
  type: 'Program',
  body: 
   [ ExpressionStatement {
       type: 'ExpressionStatement',
       expression: [Object],
       range: [Object] } ],
  range: [ 0, 5 ] }
```

### Start and end location

Implemented by: Acorn, Babylon, TypeScript parser.

The `start` and `end` properties are two integer numbers representing the location of the node in the source text.
Each number is a zero-based index, 0 points to the beginning of the source text

```
interface Node {
    type: string;
    start: number;
    end: number;
}
```

Example:
```js
> parser.parse('x = 2')
Node {
  type: 'Program',
  start: 0,
  end: 5,
  body: 
   [ Node {
       type: 'ExpressionStatement',
       start: 0,
       end: 5,
       expression: [Object] } ],
  sourceType: 'script' }
```

**Note**: TypeScript parser produces the pair `pos` and `end`.

### Start and end token

Implemented by: UglifyJS2 parser.

The `start` and `end` properties are two objects representing the first token and the last token of the node.
Each token contains the information of its location in the form of zero-based indexes (`pos` and `endpos`), one-based line numbers (`line` and `endline`), and zero-based column numbers (`col` and `endcol`).

```
interface Node {
    type: string;
    start: Token;
    end: Token;
}
interface Token {
    type: string;
    pos: number;
    col: number;
    line: number;
    endpos: number;
    endcol: number;
    endline: number;
}
```

Example:
```js
> require('uglify-js').parse('x = 2')
AST_Node {
  end: 
   AST_Token {
     raw: '2',
     file: null,
     comments_before: [],
     nlb: false,
     endpos: 5,
     endcol: 5,
     endline: 1,
     pos: 4,
     col: 4,
     line: 1,
     value: 2,
     type: 'num' },
  start: 
   AST_Token {
     raw: undefined,
     file: null,
     comments_before: [],
     nlb: false,
     endpos: 1,
     endcol: 1,
     endline: 1,
     pos: 0,
     col: 0,
     line: 1,
     value: 'x',
     type: 'name' },
body: [ AST_Node { end: [Object], start: [Object], body: [Object] } ] }
```

### Location mapping

Implemented by: Shift parser.

The `locations` property is a WeakMap that maps every node with its location information.

```
interface SyntaxTree {
    tree: Script | Module;
    locations: WeakMap<Node, Location>
}
interface Location {
    line: number;
    column: number;
    offset: number;
}
```

Example:
```js
> var ast = shift.parseScriptWithLocation('x = 2')
{ tree: Script { type: 'Script', directives: [], statements: [ [Object] ] },
  locations: WeakMap {} }
> ast.locations.get(ast.tree)
{ start: { line: 1, column: 0, offset: 0 },
  end: { line: 1, column: 5, offset: 5 } }
```

## Token Collection

Implemented by: Esprima, Babylon.

Token collection refers to the parser's ability to collect every token discovered during its internal lexical analysis and store them in an array.

Example:
```js
> var result = require('esprima').parse('const answer = 42', { tokens: true });
> result.tokens
[ { type: 'Keyword', value: 'const' },
  { type: 'Identifier', value: 'answer' },
  { type: 'Punctuator', value: '=' },
  { type: 'Numeric', value: '42' } ]
```

## Comment Collection

Implemented by: Esprima, Traceur parser, Babylon, Flow parser, TypeScript parser.

Comment collection refers to the parser's ability to gather all single-line and multiple-line comments in an array.

Example:
```js
> var ast = require('babylon').parse('42 // answer')
> ast.comments
[ { type: 'CommentLine',
    value: ' answer',
    start: 3,
    end: 12,
    loc: SourceLocation { start: [Object], end: [Object] } } ]
```

## Comment Attachment

Implemented by: Esprima, Babylon, UglifyJS2, TypeScript parser.

Comment attachment refers to the parser's ability to attach every comment to the closest syntax node. Since a node can precede or follow a comment, it has to refer to both, usually denoted as `leadingComments` and `trailingComments` arrays, respectively.

**Warning**: Currently, there is no unified standard algorithm for comment attachment. Each parser has a different approach to determine the anchor node where a comment is attached to.

Example:
```js
> var ast = require('esprima').parse('42 // answer', { attachComment: true })
> ast.body[0].expression
Literal {
  type: 'Literal',
  value: 42,
  raw: '42',
  trailingComments: [ { type: 'Line', value: ' answer', range: [Object] } ] }
```

References:
* [TypeScript syntax trivia](https://github.com/Microsoft/TypeScript/wiki/Architectural-Overview#trivia) and comment attachment
* Esprima comment attachment implementation: [comment-handler.ts](https://github.com/jquery/esprima/blob/master/src/comment-handler.ts)

**Note**: In UglifyJS2, the comment is attached in the `comments_before` array, a property of a `start` or an `end` token.

## Language Versioning

Implemented by: Acorn

Language versioning refers to the parser's ability to distinguish the syntax of a number of different ECMAScript editions.

Example:
```js
> require('acorn').parse('x={y,z}', { ecmaVersion: 5 })
SyntaxError: Unexpected token (1:4)

> require('acorn').parse('x={y,z}', { ecmaVersion: 6 })
Node {
  type: 'Program',
  start: 0,
  end: 7,
  body: 
   [ Node {
       type: 'ExpressionStatement',
       start: 0,
       end: 7,
       expression: [Object] } ],
  sourceType: 'script' }
```

## Event Delegation

Implemented by: Acorn, Esprima.

Event delegation refers to the parser's ability to invoke a callback when a certain event is fired.

**Note**: There is no single standard as to the types of event which can be bound to a callback function.

Example with Acorn:
```javascript
> function showToken(t) { console.log(t.type.label, ':', t.value) }
> require('acorn').parse('answer = 42', { onToken: showToken });
name : answer
= : =
num : 42
eof : undefined
```

Example with Esprima:
```javascript
> require('esprima').parse('42', {}, function(n) { console.log(n) })
Literal { type: 'Literal', value: 42, raw: '42' }
ExpressionStatement {
  type: 'ExpressionStatement',
  expression: Literal { type: 'Literal', value: 42, raw: '42' } }
Program {
  type: 'Program',
  body: [ ExpressionStatement { type: 'ExpressionStatement', expression: [Object] } ],
  sourceType: 'script' }
Program {
  type: 'Program',
  body: [ ExpressionStatement { type: 'ExpressionStatement', expression: [Object] } ],
  sourceType: 'script' }
```

## Exported Parser Class

To be written.

## Pure Tokenization Mode

Implemented by: Esprima, Acorn.

In this mode, the parser utilizes its (internal) tokenizer to perform a lexical analysis on the source text without running the complete parsing process. The result is a list of tokens. The source text does not need to represent a syntactically valid program.

Example:
```js
> require('esprima').tokenize('const answer = 42')
[ { type: 'Keyword', value: 'const' },
  { type: 'Identifier', value: 'answer' },
  { type: 'Punctuator', value: '=' },
  { type: 'Numeric', value: '42' } ]
```

**Note**: Without a parser, it is challenging to [distinguish](http://stackoverflow.com/questions/38846887/distinguishing-regexes-from-a-slash-division-in-a-javascript-files-with-node-j/) a forward slash (`/`) from being a division operator (e.g. `1/2`) or as a part of a regular expression (e.g. `/[0-9]+/g`). With only a pure tokenizer, this is possible by implementing the algorithm described in [Sweeten your JavaScript: hygienic macros for ES5](https://users.soe.ucsc.edu/~cormac/papers/dls14a.pdf) (DLS '14).
