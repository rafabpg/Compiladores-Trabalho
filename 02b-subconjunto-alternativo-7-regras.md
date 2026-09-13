# 2b. Subconjunto alternativo (7 regras) + Expressões Regulares dos tokens

> **Este arquivo é uma proposta alternativa a `02-subconjunto-e-tokens.md`, não uma
> substituição.** Ele existe pra dar ao grupo uma segunda opção de recorte, mais
> enxuta, antes da decisão final de qual subconjunto seguir pro trabalho manual
> (AFN → AFD → AFD mínimo). Nenhum dos dois arquivos foi apagado ou alterado.

## Por que esta opção existe

O recorte de `02-subconjunto-e-tokens.md` (16 regras, com `while` e `RELOP`) tem dois
problemas discutidos com o grupo:

1. **Semântico**: se o corpo do laço não tiver expressão aritmética (como no rascunho
   inicial de `ANOTACOES.md`, que usa só `fator → ID | NUM`), o `while` nunca consegue
   alterar a própria condição — vira, na prática, um `if`, não um laço de verdade.
2. **Custo do automato**: `RELOP` (`< <= > >= == !=`) sozinho respondia por **35% dos
   estados do AFN** no recorte anterior (22 de 62 estados) — o token mais caro de
   longe, por causa da ambiguidade de 1 ou 2 caracteres (`<` pode virar `<=`, `=` pode
   virar `==`, etc.).

Esta proposta remove `while`/`if`/`RELOP` do recorte e foca só em **declaração de
variável + atribuição + expressão aritmética com soma/subtração** — o suficiente pra
gerar código MIPS de verdade (declarar espaço, somar/subtrair registradores, atribuir)
sem nenhum dos dois problemas acima.

## 2b.1 Subconjunto de regras de produção

Igual ao `02`, cada regra cita entre parênteses a regra correspondente do C- completo
(`01-bnf-c-minus.md`), pra manter rastreabilidade.

```bnf
R1. programa   → decl-lista                                    (C- 1)
R2. decl-lista → decl-lista decl | decl                        (C- 2)
R3. decl       → var-decl | cmd                                (C- 3, 13 — simplificado)
R4. var-decl   → int ID ;                                      (C- 4, 5 — restrito a int)
R5. cmd        → ID = expr ;                                   (C- 14, 18 — só atribuição)
R6. expr       → expr addop fator | fator                      (C- 22, 26 — sem mult/parênteses)
R7. addop      → + | -                                         (C- 23)
```

`fator → ID | NUM` não conta como regra separada aqui — está embutido na regra R6 pra
manter o total em 7 (é o mesmo `fator` do C- original, regra 26, só que restrito a
`ID | NUM`, sem `( expressao )` nem chamada de função).

> **Simplificação deliberada (R3):** no C- completo, um comando nunca aparece sozinho
> no topo do programa — ele só existe dentro do corpo de uma função (`fun-declaracao`
> → `composto-decl`). Reproduzir essa estrutura completa (função + parâmetros + bloco +
> listas locais, C- regras 1,2,3,6,7,10,11,12...) consome sozinha mais de 9 regras, antes
> de qualquer comando de verdade. Pra caber em 7 regras com algum comando executável,
> assumimos que `decl-lista` representa o corpo de um único `main` implícito — os
> comandos soltos no topo do programa **substituem** a necessidade de escrever
> `void main ( void ) { ... }` por extenso. Essa é a mesma simplificação que
> `02-subconjunto-e-tokens.md` já usa em sua regra R3.

### Exemplo de programa válido

```c
int x;
int y;
x = 1;
y = x + 2;
y = y - x;
```

Sequência de tokens de `y = x + 2;`:

```
ID(y)  ATRIB(=)  ID(x)  ADDOP(+)  NUM(2)  PONTO_VIRGULA(;)
```

Não existe nenhuma condição, laço ou comparação — todo programa deste subconjunto é
uma sequência linear de declarações e atribuições, sempre termina e nunca tem
ambiguidade de "o laço não itera".

## 2b.2 Tokens do subconjunto e suas Expressões Regulares

Definições auxiliares (iguais a `02-subconjunto-e-tokens.md`):

```
letra  = a|b|...|z|A|B|...|Z
digito = 0|1|...|9
```

| # | Token | Expressão Regular | Exemplos |
|---|---|---|---|
| T1 | `INT` | `int` | `int` |
| T2 | `ID` | `letra letra*` (= `letra⁺`) | `x`, `soma`, `contador` |
| T3 | `NUM` | `digito digito*` (= `digito⁺`) | `0`, `1`, `42` |
| T4 | `ATRIB` | `=` | `=` |
| T5 | `ADDOP` | `+ \| -` | `+`, `-` |
| T6 | `PONTO_VIRGULA` | `;` | `;` |
| T7 | `BRANCO` (descartado) | `( \| \t \| \n \| \r)⁺` | — |

**7 tokens** (6 emitidos + 1 descartado) contra os 15 do recorte de 16 regras — sem
`RELOP`, sem `WHILE`, sem parênteses/chaves/vírgula (não há bloco composto nem chamada
de função neste recorte).

## 2b.3 Alfabeto e classes de caracteres

Como não há mais palavra reservada de 5 letras (`while`) nem operadores de 1-ou-2
caracteres (`RELOP`), o alfabeto fica bem menor que as 24 classes do recorte anterior:

| Classe | Caracteres | Por que é separada |
|---|---|---|
| `i` `n` `t` | esses 3 caracteres | formam a palavra reservada `int` |
| `L` | demais letras `a-z`, `A-Z` | só podem formar `ID` |
| `D` | `0-9` | `NUM` |
| `=` | — | `ATRIB` (símbolo único, sem ambiguidade com `==`) |
| `+` | — | `ADDOP` |
| `-` | — | `ADDOP` |
| `;` | — | `PONTO_VIRGULA` |
| `ws` | espaço, `\t`, `\n`, `\r` | `BRANCO` |
| `out` | qualquer outro código | erro léxico |

**9 classes**, menos da metade das 24 usadas no recorte com `while`+`RELOP`. Nenhuma
classe tem ambiguidade de "pode virar um token de 2 caracteres" — todo símbolo aqui já
decide o token sozinho, então não deve haver o tipo de explosão de estados que o
`RELOP` causava (ver `ANOTACOES.md` §7-9 pra comparação).

## 2b.4 Estimativa de tamanho do automato (antes de fazer o AFN de verdade)

Baseado nos números já calculados em `ANOTACOES.md` para tokens equivalentes:

| Token | Estados no AFN (estimado) |
|---|---:|
| `INT` (3 letras) | ~6 |
| `ID` | ~6 |
| `NUM` | ~6 |
| `ATRIB` | 2 |
| `ADDOP` (`+`, `-`) | ~4 |
| `PONTO_VIRGULA` | 2 |
| `BRANCO` | ~6 |
| **Total (AFN unido, estimado)** | **~32**, contra os 63 do recorte com `while` |

Depois de determinizar e minimizar, a expectativa é um AFD mínimo na faixa de
**8 a 12 estados** (contra os 19 do recorte anterior) — sem o conflito letra-a-letra de
`while` vs `ID` e sem a ambiguidade de lookahead do `RELOP`, que eram os dois maiores
multiplicadores de estado. Os números exatos só ficam definitivos depois de fazer o
AFN → AFD → AFD mínimo de verdade (etapa do Integrante 2, `docs/03`), caso o grupo
decida seguir por esta opção.
