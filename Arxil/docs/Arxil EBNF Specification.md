# Arxil EBNF Specification

---

- [Arxil EBNF Specification](#arxil-ebnf-specification)
  - [Top-Level Structure](#top-level-structure)
  - [.arxtype](#arxtype)
  - [Node Declaration](#node-declaration)
    - [MetaData](#metadata)
    - [Data Section](#data-section)
    - [Code Section](#code-section)
      - [Instruction Declarations](#instruction-declarations)
      - [Function Declaration](#function-declaration)
  - [Lexical Elements](#lexical-elements)


---

> Agreement:
> - All terminal symbols are indicated by double quotation marks `""`;
> - `(*... *)` is a note;
> - `[X]` indicates optional, and `{X}` indicates zero or multiple repetitions;
> - Uppercase identifiers are non-terminal symbols.

---

## Top-Level Structure

```ebnf
ArxilProgram = { .arxtype }, { .arxlib }, { NodeDecl } ;
```

## .arxtype

```ebnf
ArxilTypeFile = TypeDecl ;

TypeDecl = "type", IDENT, "{",
             TypeName,
             TypeSize,
             [TypeLayout],
             [TypeOps],
             [TypeInterop],
             [TypeValidators],
             [TypeDocs],
             [TypeHints],
           "}" ;

TypeName = "name", "=", STRING, ";" ;

TypeSize = "size", "=", INTEGER, ("bits" | "bytes"), ";" ;

TypeLayout = "layout", "{", { LayoutField }, "}" ;

LayoutField = IDENT, "{",
                "offset", "=", BitExpr, ";",
                "width",  "=", BitExpr, ";",
                "type",   "=", IDENT, ";",
                ["tag", "=", STRING, ";"],   (* Optional tag, e.g., "reserved" *)
              "}" ;

BitExpr = INTEGER ;  (* In the future, it can be extended to simple arithmetic expressions, e.g., "8*4" *)

TypeOps = "ops", "{", { OpDecl }, "}" ;

OpDecl = IDENT, "{",
           [OpNative],
           [OpAsmMap],
           [OpLibcall],
           [OpIntrinsic],
           [OpPointerArith],
           [OpUnary],
           [OpMayTrap],
           [OpLoweredArxil],
         "}" ;

OpNative      = "native", "=", BOOLEAN, ";" ;
OpUnary       = "unary", "=", BOOLEAN, ";" ;
OpMayTrap     = "may_trap", "=", BOOLEAN, ";" ;

OpPointerArith = "pointer_arithmetic", "=", BOOLEAN, ";",
                 ["scale_by", "=", INTEGER, ";"] ;

OpAsmMap = "asm", "{", { AsmTarget }, "}" ;

AsmTarget = IDENT, "=", STRING, ";" ;
(* e.g., x86_64 = "addss %xmm0, %xmm1"; *)

OpLibcall = "libcall", "=", STRING, ";" ;
(* e.g., libcall = "__f32_sin"; *)

OpIntrinsic = "intrinsic", "=", BOOLEAN, ";" ;

OpLoweredArxil = "lowered", "=", InstScriBlock, ";" ;

InstScriBlock = "'inst_scri'", "{", { InstLine }, "}" ;

InstLine = IDENT, { Operand }, ";" ;

Operand = IDENT | INTEGER | "(" IDENT ")" ;
(* IDENT here refers to the field name in the layout, or temporary register *)

TypeInterop = "interop", "{",
                [InteropPointerType],
                [InteropImplicitFrom],
                [InteropImplicitTo],
                [InteropBindingCompat],
                [InteropArrayOK],
              "}" ;

InteropPointerType     = "pointer_type", "=", IDENT, ";" ;
InteropImplicitFrom    = "implicit_from", "=", TypeList, ";" ;
InteropImplicitTo      = "implicit_to",   "=", TypeList, ";" ;
InteropBindingCompat   = "binding_compatible_with", "=", TypeList, ";" ;
InteropArrayOK         = "array_element_ok", "=", BOOLEAN, ";" ;

TypeList = "(", [IDENT, { ",", IDENT }], ")" ;

TypeValidators = "validators", "{",
                   [CompileTimeChecks],
                   [RuntimeChecks],
                 "}" ;

CompileTimeChecks = "compile_time", "=", CheckList, ";" ;

RuntimeChecks = "runtime", "{", { RuntimeCheckDecl }, "}" ;

CheckList = "(", [STRING, { ",", STRING }], ")" ;

RuntimeCheckDecl = IDENT, "{",
                     "condition", "=", ConditionExpr, ";",
                     "action", "=", ActionSpec, ";",
                     ["enabled_by_default", "=", BOOLEAN, ";"],
                     ["requires", "=", CapabilityList, ";"],
                   "}" ;

ConditionExpr = STRING ;  (* e.g., "address == 0" *)
ActionSpec = STRING ;     (* e.g., "trap PTR_NULL" *)
CapabilityList = "(", [IDENT, { ",", IDENT }], ")" ;

TypeDocs = "docs", "{",
             [DocUsage],
             [DocErrors],
           "}" ;

DocUsage = "usage", "=", STRING, ";" ;
DocErrors = "errors", "{", { ErrorCodeMapping }, "}" ;

ErrorCodeMapping = IDENT, "=", STRING, ";" ;
(* e.g., PTR_DANGLING = "Dangling pointer dereference" *)

TypeHints = "hints", "{", { HintDecl }, "}" ;

HintDecl = IDENT, "=", (BOOLEAN | INTEGER | STRING), ";" ;
(* e.g., cache_line_aligned = true; simd_eligible = false; *)
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
MetaItem = (* reserved for referance of .arxtype, tooling, doc, layout hints, etc. *) ;
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
                [PrivBlock]
              "}" ;

InstructBlock = "instruct", "{", { OrdnInst | ResvInst }, "}" ;
AnceBlock     = "ance",     "{", { FnDecl }, "}" ;
PublBlock     = "publ",     "{", { FnDecl }, "}" ;
PrivBlock     = "publ",     "{", { FnDecl }, "}" ;
```

#### Instruction Declarations

```ebnf
OrdnInst = OpCode, OperTarg, OperGoal, ";" ;

OpName = IDENT ;

OperTarg | OperGoal = "(", Operand, [{ Operand }], ")" ;

Operand = IDENT | "(" IDENT ")" ;

ResvInst = 
    ExecDecl | CondDecl | CyclDecl |
    WaitDecl | SgnlDecl | YielDecl | FnshDecl | WarnDecl |
    PshDecl | PopDecl | LftDecl | MrgDecl | DtcDecl ;

ExecDecl = "exec", "(", "(", [FieldList], ")", "(", [FieldList], ")", ")", IDENT, ";" ;
CondDecl = "cond", "(", IDENT, ")", "(", "(", OrdnInst | ExecDecl, ")", "(", OrdnInst | ExecDecl, ")", ")", ";" ;
CyclDecl = "cycl", "(", IDENT, ")", "(", OrdnInst | ExecDecl, ")", ";" ;
WaitDecl = "wait", "(", IDENT, ")", "(", [IDENT], ")", ";" ;
SgnlDecl = "sgnl", "(", IDENT, ")", "(", [IDENT], ")", ";" ;
YielDecl = "yiel", "(", [IDENT], ")", ";" ;
FnshDecl = "fnsh", ";" ;
WarnDecl = "warn", "(", NodeList, ")", ";" ;

FieldList = Field, { ",", Field } ;
Field     = IDENT | "(" IDENT ")" ;
NodeList  = NodeRef, { ",", NodeRef } ;
NodeRef   = "this" | "all" | IDENT ;

PshDecl = "psh", IDENT, "(", IDENT, "(", BindingList ")", "(", BindingList ")", "(", BindingList ")", ")", ";" ;
PopDecl = "pop", IDENT, ["(", IDENT, ")"], ";" ;
LftDecl = "lft", IDENT, "(", [BindingList], ")", "(", [BindingList], ")", ";" ;
MrgDecl = "mrg", IDENT, ["(", STRING, ")"], ";" ;
DtcDecl = "dtc", IDENT, "(", IDENT, "(", [DetachingList], ")", "(", [DetachingList], ")", "(", [DetachingList], ")", ")", ["(", STRING, ")"], ";" ;

BindingList   = BindingPair , { ",", BindingPair } ;
DetachingList = DetachingPair, { ",", DetachingPair } ;

BindingPair   = "(", IDENT, "=>", IDENT, ")" ;
DetachingPair = "(", IDENT, "-->", IDENT, ")" ;
```

> *Only when "Operand" refers to a `fn` can it not wrapped in `()`.*

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

LangBlock = ["@lowered"], "'", LangTag, "'", "{", LangCode, "}" ;

LangTag = "inst_scri" | "c" | "cpp" | "rust" | IDENT ;

LangCode = { ANY_CHAR_EXCEPT_CURLY_BRACE_NESTED } ;
```

> *The content of LangCode must be lowered by a registered frontend plugin into a sequence of Arxil instructions before being executed by ArxilVM or EDSOS that only reference symbols declared in the enclosing function’s parameter and return lists. Which has been lowered gets the tag "@lowered" to notify.*

## Lexical Elements

```ebnf
IDENT       = (LETTER | "_"), { LETTER | DIGIT | "_" | "." } ;
INTEGER     = DIGIT, { DIGIT } ;
STRING      = '"', { CHAR_NO_QUOTE_OR_ESCAPE | ESCAPE_SEQ }, '"' ;
BOOLEAN     = "true" | "false" ;

CHAR_NO_QUOTE_OR_ESCAPE = ? any character except " and \ ? ;
ESCAPE_SEQ  = "\", ("n" | "t" | """ | "\" | "0") ;

LETTER      = "a".."z" | "A".."Z" ;
DIGIT       = "0".."9" ;

(* For LangCode, lexer should treat '{...}' as nested blocks *)
ANY_CHAR_EXCEPT_CURLY_BRACE_NESTED = 
    (* Implementation-defined: typically raw string with brace balancing *)
```