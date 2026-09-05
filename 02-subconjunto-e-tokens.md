# 2. Subconjunto selecionado + Expressões Regulares dos tokens

## 2.1 Subconjunto das regras de produção

Recorte escolhido: **declaração de variáveis inteiras, atribuição, laço `while`,
bloco composto e expressões aritméticas/relacionais.** Todas as regras abaixo
são regras do C- original (indicadas na coluna da direita), apenas restritas.

```bnf
 R1.  programa            → decl-lista                          (C- 1)
 R2.  decl-lista          → decl-lista decl | decl              (C- 2)
 R3.  decl                → var-decl | cmd                      (C- 3, 13)
 R4.  var-decl            → int ID ;                            (C- 4, 5)
 R5.  cmd                 → cmd-atrib | cmd-iteracao | cmd-composto
 R6.  cmd-atrib           → ID = expressao ;                    (C- 14, 18, 19)
 R7.  cmd-iteracao        → while ( expressao ) cmd             (C- 16)
 R8.  cmd-composto        → { decl-lista }                      (C- 10)
 R9.  expressao           → expressao-simples                   (C- 18)
R10.  expressao-simples   → exp-aditiva relop exp-aditiva
                          | exp-aditiva                         (C- 20)
R11.  relop               → < | <= | > | >= | == | !=           (C- 21)
R12.  exp-aditiva         → exp-aditiva addop termo | termo      (C- 22)
R13.  addop               → + | -                               (C- 23)
R14.  termo               → termo mulop fator | fator            (C- 24)
R15.  mulop               → * | /                               (C- 25)
R16.  fator               → ( expressao ) | ID | NUM             (C- 26)
```


## 2.2 Tokens do subconjunto e suas Expressões Regulares

A notação de ER usada abaixo (concatenação, alternância `|`, fecho `*` e `+`, classes
`[ ]`) é a de Louden.
As ERs para identificadores, números, palavras reservadas e comentários seguem os
padrões apresentados por Louden

Definições auxiliares:

```
letra  = a|b|...|z|A|B|...|Z
digito = 0|1|...|9
```

| # | Token | Expressão Regular | Exemplos |
|---|---|---|---|
| T1 | `INT` | `int` | `int` |
| T2 | `WHILE` | `while` | `while` |
| T3 | `ID` | `letra letra*` (= `letra⁺`) | `soma`, `x`, `contador` |
| T4 | `NUM` | `digito digito*` (= `digito⁺`) | `0`, `42`, `100` |
| T5 | `ATRIB` | `=` | `=` |
| T6 | `RELOP` | `< \| <= \| > \| >= \| == \| !=` | `<=`, `!=` |
| T7 | `ADDOP` | `+ \| -` | `+`, `-` |
| T8 | `MULOP` | `* \| /` | `*`, `/` |
| T9 | `PONTO_VIRGULA` | `;` | `;` |
| T10 | `ABRE_PAR` | `(` | `(` |
| T11 | `FECHA_PAR` | `)` | `)` |
| T12 | `ABRE_CHAVE` | `{` | `{` |
| T13 | `FECHA_CHAVE` | `}` | `}` |
| T14 | `BRANCO` (descartado) | `( \| \t \| \n \| \r)⁺` | — |
| T15 | `COMENTARIO` (descartado) | `/\* ( [^*] \| \*⁺[^*/] )* \*⁺ /` | `/* oi */` |


## 2.3 Alfabeto e classes de caracteres

Para não escrever uma tabela de 128 colunas, os caracteres são agrupados em
**24 classes de equivalência** (caracteres que levam sempre ao mesmo estado). Essa
compressão do alfabeto é a mesma técnica que Cooper & Torczon ao
tratar do tamanho das tabelas de um scanner dirigido por tabela:

| Classe | Caracteres | Por que é separada |
|---|---|---|
| `i` `n` `t` `w` `h` `l` `e` | esses 7 caracteres | formam `int` e `while` |
| `L` | demais letras | só podem formar `ID` |
| `D` | `0-9` | `NUM` |
| `=` `<` `>` `!` | — | `ATRIB` e `RELOP` |
| `+` `-` | — | `ADDOP` |
| `*` `/` | — | `MULOP` e comentário |
| `;` `(` `)` `{` `}` | — | tokens de 1 caractere |
| `ws` | espaço `\t` `\n` `\r` | branco |
| `out` | qualquer outro | erro léxico |
