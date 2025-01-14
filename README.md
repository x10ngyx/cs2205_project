## 测试

```
make
./main test/sample_src01.jtl
```
在 `result.txt` 中查看抽象语法树与宏展开后的抽象语法树。

## 词法分析与语法分析

- `lang.l lang.y lang.h lang.c`

- WhileDB + 结构化宏 + 函数过程调用的语法

```
E :=    N | V | -E | E + E | E - E | E * E | E / E | E % E |
        E < E | E <= E | E == E | E != E | E >= E | E > E | 
        E && E | E || E | !E | *E
        malloc(E) | read_int() | read_char() |
        V (E, E, ..., E) | V ()

C :=    var V |
        write_int(E) |
        write_char(E) |
        E = E |
        C; C |
        if (E) then { C } else { C } |
        while (E) do { C } |
        V (E, E, ..., E) | V () |
        return E

D :=    define_expr V (V, V, ..., V) { E } | define_expr V () { E } |
        define_cmd V (V, V, ..., V) { C } | define_cmd V () { C } |
        func V (V, V, ..., V) { C } | func V () { C } |
        proc V (V, V, ..., V) { C } | proc V () { C } |
        D ; D

P :=    C |
        D . C // 使用"."而不是";"以避免语法分析冲突

```



- 新增的终结符(token)与非终结符
```
TM_DEF_EXPR TM_DEF_CMD
TM_DEF_FUNC TM_DEF_PROC
TM_RET
TM_COMMA TM_DOT

NT_PROG
NT_DEF
NT_VAR_LIST
NT_EXPR_LIST

```

- 新增的上下文无关语法
```
NT_VAR_LIST ->  TM_IDENT |
                TM_IDENT TM_COMMA NT_VAR_LIST

NT_EXPR_LIST -> NT_EXPR |
                NT_EXPR TM_COMMA NT_EXPR_LIST

NT_DEF ->       TM_DEF_EXPR TM_IDENT TM_LEFT_PAREN NT_VAR_LIST TM_RIGHT_PAREN TM_LEFT_BRACE NT_EXPR TM_RIGHT_BRACE |
                TM_DEF_EXPR TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN TM_LEFT_BRACE NT_EXPR TM_RIGHT_BRACE |
                TM_DEF_CMD TM_IDENT TM_LEFT_PAREN NT_VAR_LIST TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                TM_DEF_CMD TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                TM_DEF_FUNC TM_IDENT TM_LEFT_PAREN NT_VAR_LIST TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                TM_DEF_FUNC TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                TM_DEF_PROC TM_IDENT TM_LEFT_PAREN NT_VAR_LIST TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                TM_DEF_PROC TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN TM_LEFT_BRACE NT_CMD TM_RIGHT_BRACE |
                NT_DEF TM_SEMICOL NT_DEF

NT_EXPR ->      TM_IDENT TM_LEFT_PAREN NT_EXPR_LIST TM_RIGHT_PAREN |
                TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN

NT_CMD ->       TM_IDENT TM_LEFT_PAREN NT_EXPR_LIST TM_RIGHT_PAREN |
                TM_IDENT TM_LEFT_PAREN TM_RIGHT_PAREN |
                TM_RET NT_EXPR

NT_PROG ->      NT_CMD |
                NT_DEF TM_DOT NT_CMD

NT_WHOLE ->     NT_PROG

        
```

- 新增的抽象语法树结构

```
varlist :       SINGLE_VAR | MULTI_VAR
exprlist :      SINGLE_EXPR | MULTI_EXPR
def :           FUNC | FUNC_NO_ARGS | PROC | PROC_NO_ARGS |
                EXPR | EXPR_NO_ARGS | CMD | CMD_NO_ARGS | SEQ_DEF
expr :          CALL_E | CALL_E_NO_ARGS
cmd :           CALL_C | CALL_C_NO_ARGS | RET
prog :          PROG_WITHOUT_DEF | PROG_WITH_DEF
```


## 宏展开

- `unfold.h unfold.c`

- 对抽象语法树进行如下处理：
1. 展开宏调用。检查 `expr` 中 ` CALL_E / CALL_E_NO_ARGS ` 和 `cmd` 中 `CALL_C / CALL_C_NO_ARGS` 结构，若是宏调用，则替换为相应定义。
2. 去除宏定义。删除 `def` 中的 `FUNC / FUNC_NO_ARGS / PROC / PROC_NO_ARGS` 结构；


- 规定宏调用只能出现在该宏的定义之后，函数/过程调用只能出现在该函数/过程的定义之后或定义体内 (参考 `test/sample_src04.jtl`)。

  我们会检查以下几种错误，并给出错误信息 `Error: definition not found` (参考 `test/sample_src05.jtl`)：

1. 在一个宏的定义之前或定义体内调用该宏；
2. 在一个函数/过程的定义之前调用该函数/过程；
3. 未定义的调用，即调用一个没有定义的符号； 

- 如果发现 `CALL_E / CALL_E_NO_ARGS` 处调用了语句宏，或在 `CALL_C / CALL_C_NO_ARGS` 处调用了表达式宏，会给出错误信息 `Error: call type error` (参考 `test/sample_src06.jtl`)。

- 简单的程序示例见 `test/sample_src01.jtl / test/sample_src02.jtl` ，较为复杂的嵌套调用见 `test/sample_src03.jtl` 。

