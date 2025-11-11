# TSL EBNF Specification

> Agreement:
> - All terminal symbols are indicated by double quotation marks `""`;
> - `(*... *)` is a note;
> - `[X]` indicates optional, and `{X}` indicates zero or multiple repetitions;
> - Uppercase identifiers are non-terminal symbols.

---

## Top-Level Structure

```ebnf
TSLProgram = { NodeDecl } ;
```

## Node Declaration

```ebnf
NodeDecl = "node", IDENT, "{",
             [MetaData],
             [DataSection],
             [CodeSection],
           "}" ;
```

### MetaData

```ebnf
MetaData = "meta_data", "{", { MetaDataBlock }, "}" ;
MetaDataBlock = ("cptm" | "rntm"), "{", { MetaItem }, "}" ;
MetaItem = /* reserved for tooling, doc, layout hints, etc. */ ;
```

### Data Section

```ebnf
DataSection = "data", "{", { DataBlock }, "}" ;

DataBlock = ("imme" | "futu"), "{", { FieldBlock }, "}" ;

FieldBlock = ("ance" | "publ" | "priv"), "{", { FieldDecl }, "}" ;

FieldDecl = TypeSpec, [ArraySize], IDENT, ";" ;

ArraySize = "[", INTEGER, "]" ;
```

### Code Section

```ebnf
CodeSection = "code", "{",
                [InstructBlock],
                [AnceBlock],
                [PublBlock],
              "}" ;

InstructBlock = "instruct", "{", { InstDecl }, "}" ;
AnceBlock     = "ance",     "{", { FnDecl }, "}" ;
PublBlock     = "publ",     "{", { FnDecl }, "}" ;
PrivBlock     = "publ",     "{", { FnDecl }, "}" ;
```

#### Instruction Declarations

```ebnf
InstructDecl = OpCode, OperTarg, OperGoal, ";" ;

OpName = IDENT ;

OperTarg | OperGoal = "(", Operand, [{ " ", Operand }], ")" ;

Operand = IDENT | "(" IDENT ")" ;
```

#### Function Declaration

```ebnf
FnDecl = "fn", IDENT,
         "(", [ParamList], ")", "=>", "(", [RetList], ")",
         ( "{" FnBody "}" | ";") ;

ParamList = Param, [{ ",", Param }] ;
RetList   = Param, [{ ",", Param }] ;

Param = "(", TypeSpec, ")", IDENT ;

TypeSpec = IDENT ;

FnBody = { LangBlock } ;

LangBlock = "'", LangTag, "'", "{", LangCode, "}" ;

LangTag = "inst_scri" | "c" | "cpp" | "rust" | IDENT ;

LangCode = { ANY_CHAR_EXCEPT_CURLY_BRACE_NESTED } ;
```

> *The content of LangCode must be lowered by a registered frontend plugin into a sequence of TSL instructions before being executed by TSLVM or EDSOS that only reference symbols declared in the enclosing function’s parameter and return lists.*

## Lexical Elements

```ebnf
IDENT       = LETTER, { LETTER | DIGIT | "_" } ;
INTEGER     = DIGIT, { DIGIT } ;
LETTER      = "a".."z" | "A".."Z" ;
DIGIT       = "0".."9" ;

(* For LangCode, lexer should treat '{...}' as nested blocks *)
ANY_CHAR_EXCEPT_CURLY_BRACE_NESTED = 
    (* Implementation-defined: typically raw string with brace balancing *)
```