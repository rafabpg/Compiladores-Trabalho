# 1. Linguagem escolhida: **C-** (C Minus)

Linguagem de programação simplificada definida por **Kenneth C. Louden** em
*Compiler Construction: Principles and Practice*. A linguagem é apresentada em
**§1.8 — "C-Minus: A Language for a Compiler Project"** e especificada por completo
no **Apêndice A**, que traz as convenções léxicas, a BNF e programas de exemplo.
É um subconjunto próprio de C, projetado exatamente para trabalhos de compiladores:
tem BNF completa, publicada e pequena (29 regras).

A terminologia usada aqui (token, lexema, padrão) e a divisão das categorias léxicas
em palavras reservadas, símbolos especiais e "outros tokens" seguem Louden.
---

## 1.1 BNF completa do C-

```bnf
 1. programa            → declaracao-lista
 2. declaracao-lista    → declaracao-lista declaracao | declaracao
 3. declaracao          → var-declaracao | fun-declaracao
 4. var-declaracao      → tipo-especificador ID ;
                        | tipo-especificador ID [ NUM ] ;
 5. tipo-especificador  → int | void
 6. fun-declaracao      → tipo-especificador ID ( params ) composto-decl
 7. params              → param-lista | void
 8. param-lista         → param-lista , param | param
 9. param               → tipo-especificador ID | tipo-especificador ID [ ]
10. composto-decl       → { local-declaracoes statement-lista }
11. local-declaracoes   → local-declaracoes var-declaracao | vazio
12. statement-lista     → statement-lista statement | vazio
13. statement           → expressao-decl | composto-decl | selecao-decl
                        | iteracao-decl | retorno-decl
14. expressao-decl      → expressao ; | ;
15. selecao-decl        → if ( expressao ) statement
                        | if ( expressao ) statement else statement
16. iteracao-decl       → while ( expressao ) statement
17. retorno-decl        → return ; | return expressao ;
18. expressao           → var = expressao | simples-expressao
19. var                 → ID | ID [ expressao ]
20. simples-expressao   → soma-expressao relacional soma-expressao
                        | soma-expressao
21. relacional          → <= | < | > | >= | == | !=
22. soma-expressao      → soma-expressao soma termo | termo
23. soma                → + | -
24. termo               → termo mult fator | fator
25. mult                → * | /
26. fator               → ( expressao ) | var | ativacao | NUM
27. ativacao            → ID ( args )
28. args                → arg-lista | vazio
29. arg-lista           → arg-lista , expressao | expressao
```

## 1.2 Convenções léxicas do C-

| Item | Definição |
|---|---|
| Palavras reservadas | `else` `if` `int` `return` `void` `while` |
| Símbolos especiais | `+` `-` `*` `/` `<` `<=` `>` `>=` `==` `!=` `=` `;` `,` `(` `)` `[` `]` `{` `}` |
| `ID` | `letra letra*` |
| `NUM` | `digito digito*` |
| `letra` | `a..z` \| `A..Z` |
| `digito` | `0..9` |
| Brancos | espaço, tabulação e nova linha — descartados |
| Comentários | `/* ... */`, **não** aninhados — descartados |

> **Atenção:** em C- o identificador é
> `letra letra*`, ou seja **não aceita dígitos**. Por isso `x1` é lido como dois
> tokens: `ID(x)` e `NUM(1)`. 
