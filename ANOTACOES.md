# Anotações — analisador léxico para `while`

Este arquivo documenta uma gramática de sete regras para reconhecer um comando `while` com bloco. Ele reúne as decisões tomadas, a gramática, o alfabeto, as expressões regulares, os AFNs, sua união, o AFD, a minimização e o scanner manual em C.

O roteiro segue a ordem de uma resolução manual: primeiro identifiquei os terminais da gramática; depois escrevi cada ER e montei seu AFN; reuni os AFNs com um novo inicial; calculei os subconjuntos do AFD; refinei as partições até estabilizarem; por último, transcrevi a tabela mínima para C.

> **Origem das ERs:** as categorias e expressões regulares partiram da definição lexical de C− adotada pelo grupo. Elas foram selecionadas para o recorte de sete regras e convertidas para a notação usada por Louden e pelo Livro do Dragão: `|` para união, justaposição para concatenação, `*` para fechamento de Kleene e parênteses para agrupamento.

## 1. Decisões de escopo

O projeto implementa um `while` com condição relacional, bloco e uma atribuição. O recorte possui **sete regras**, **11 padrões léxicos** e **24 classes de caracteres**.

O scanner reconhece os dez tokens usados pela gramática e descarta `BRANCO`. Comentários não pertencem ao recorte; `/` e `*` produzem erro léxico quando aparecem na entrada.

## 2. Gramática implementada

```bnf
1. programa      → cmd-iteracao
2. cmd-iteracao  → while ( condicao ) cmd-composto
3. cmd-composto  → { cmd-atrib }
4. condicao      → fator relop fator
5. relop         → < | <= | > | >= | == | !=
6. cmd-atrib     → ID = fator ;
7. fator         → ID | NUM
```

Exemplo pertencente à gramática:

```c
while (x < 10) {
    x = 10;
}
```

A sequência de tokens, depois de descartar os brancos, é:

```text
WHILE ABRE_PAR ID RELOP NUM FECHA_PAR ABRE_CHAVE
ID ATRIB NUM PONTO_VIRGULA FECHA_CHAVE
```

Esta é uma gramática **especializada e reescrita** a partir de C−. Ela não é uma cópia literal de sete linhas da BNF completa. Os não terminais foram fechados sobre o único formato de programa escolhido.

## 3. Categorias léxicas necessárias

O scanner produz dez categorias:

| Categoria | Lexemas ou função |
|---|---|
| `WHILE` | palavra reservada `while` |
| `ID` | uma ou mais letras ASCII |
| `NUM` | um ou mais dígitos decimais |
| `ATRIB` | `=` |
| `RELOP` | `<`, `<=`, `>`, `>=`, `==` ou `!=` |
| `PONTO_VIRGULA` | `;` |
| `ABRE_PAR` | `(` |
| `FECHA_PAR` | `)` |
| `ABRE_CHAVE` | `{` |
| `FECHA_CHAVE` | `}` |

Uma categoria adicional é reconhecida e descartada:

| Categoria | Função |
|---|---|
| `BRANCO` | espaço, tabulação, nova linha ou retorno de carro |

Portanto, as sete regras exigem **11 padrões léxicos e 11 AFNs individuais**. A quantidade de regras da BNF e a quantidade de tokens não precisam ser iguais. Comentários não pertencem ao recorte.

## 4. Alfabeto dos autômatos

O universo de entrada é ASCII de 7 bits, códigos 0 a 127. Cada byte é convertido em exatamente uma das 24 classes antes de consultar uma transição.

| Ordem | Classe | Membros | Uso |
|---:|---|---|---|
| 1 | `i` | letra `i` | palavra reservada e `ID` |
| 2 | `n` | letra `n` | `ID` |
| 3 | `t` | letra `t` | `ID` |
| 4 | `w` | letra `w` | palavra reservada e `ID` |
| 5 | `h` | letra `h` | palavra reservada e `ID` |
| 6 | `l` | letra `l` | palavra reservada e `ID` |
| 7 | `e` | letra `e` | palavra reservada e `ID` |
| 8 | `L` | demais letras `a–z` e `A–Z` | `ID` |
| 9 | `D` | `0–9` | `NUM` |
| 10 | `=` | igual | `ATRIB` e `RELOP` |
| 11 | `<` | menor | `RELOP` |
| 12 | `>` | maior | `RELOP` |
| 13 | `!` | exclamação | início possível de `!=` |
| 14 | `+` | mais | não inicia token neste recorte |
| 15 | `-` | menos | não inicia token neste recorte |
| 16 | `*` | asterisco | não inicia token neste recorte |
| 17 | `/` | barra | não inicia token neste recorte |
| 18 | `;` | ponto e vírgula | `PONTO_VIRGULA` |
| 19 | `(` | abre parêntese | `ABRE_PAR` |
| 20 | `)` | fecha parêntese | `FECHA_PAR` |
| 21 | `{` | abre chave | `ABRE_CHAVE` |
| 22 | `}` | fecha chave | `FECHA_CHAVE` |
| 23 | `ws` | espaço, `\t`, `\n`, `\r` | `BRANCO` |
| 24 | `out` | demais códigos ASCII | erro léxico |

As classes formam uma partição: não há sobreposição e os 128 códigos ASCII aparecem uma única vez. `EOF` não pertence ao alfabeto. Acentos, `ç`, emojis e qualquer byte acima de 127 ficam fora do universo adotado.

As classes `+`, `-`, `*` e `/` fazem parte da partição adotada, mas não iniciam tokens deste recorte e levam ao estado morto.

## 5. Expressões regulares

As ERs abaixo seguem a notação dos livros. `ID`, `NUM` e `BRANCO` aparecem sem as abreviações `letra`, `digito` e `ws`.

### 5.1 Palavra reservada

```text
WHILE → "w" "h" "i" "l" "e"
```

### 5.2 Identificador

A especificação lexical adotada define `ID` como uma ou mais letras ASCII. Dígitos e `_` não pertencem a um identificador.

```text
ID → ("a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" |
      "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" |
      "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" |
      "y" | "z" | "A" | "B" | "C" | "D" | "E" | "F" |
      "G" | "H" | "I" | "J" | "K" | "L" | "M" | "N" |
      "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" |
      "W" | "X" | "Y" | "Z")
     ("a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" |
      "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" |
      "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" |
      "y" | "z" | "A" | "B" | "C" | "D" | "E" | "F" |
      "G" | "H" | "I" | "J" | "K" | "L" | "M" | "N" |
      "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" |
      "W" | "X" | "Y" | "Z")*
```

Consequência: `x1` é tokenizado como `ID(x)` seguido de `NUM(1)`. Isso não é um erro léxico, embora possa causar erro sintático conforme a posição.

### 5.3 Número

```text
NUM → ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9")
      ("0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9")*
```

### 5.4 Atribuição e operadores relacionais

```text
ATRIB → "="

RELOP → "<" | "<" "=" | ">" | ">" "=" | "=" "=" | "!" "="
```

O maior lexema diferencia `=` de `==`: o primeiro é `ATRIB`; o segundo é `RELOP`.

### 5.5 Delimitadores

```text
PONTO_VIRGULA → ";"
ABRE_PAR      → "("
FECHA_PAR     → ")"
ABRE_CHAVE    → "{"
FECHA_CHAVE   → "}"
```

### 5.6 Brancos

```text
BRANCO → (" " | "\t" | "\n" | "\r")
         (" " | "\t" | "\n" | "\r")*
```


## 6. Construção dos AFNs individuais

Os AFNs foram construídos antes da união e sem minimização. Cada um possui numeração local, um inicial e um único final.

Foram preservadas as operações da construção de Thompson:

- um literal usa dois estados e uma transição consumidora;
- a concatenação liga fragmentos por ε;
- a união cria uma nova entrada e uma nova saída, conectadas aos ramos por ε;
- o fechamento `*` cria caminhos ε para entrar, repetir, ignorar e terminar;
- `ID`, `NUM` e `BRANCO` consomem classes completas, sem criar um ramo para cada caractere individual.

| AFN | Estados | Inicial | Final | Transições |
|---|---:|---|---|---:|
| `WHILE` | 10 | `q0` | `q9` | 9 |
| `ID` | 6 | `q0` | `q5` | 7 |
| `NUM` | 6 | `q0` | `q5` | 7 |
| `ATRIB` | 2 | `q0` | `q1` | 1 |
| `RELOP` | 22 | `q0` | `q21` | 26 |
| `PONTO_VIRGULA` | 2 | `q0` | `q1` | 1 |
| `ABRE_PAR` | 2 | `q0` | `q1` | 1 |
| `FECHA_PAR` | 2 | `q0` | `q1` | 1 |
| `ABRE_CHAVE` | 2 | `q0` | `q1` | 1 |
| `FECHA_CHAVE` | 2 | `q0` | `q1` | 1 |
| `BRANCO` | 6 | `q0` | `q5` | 7 |
| **Total** | **62** | — | — | **62** |

Depois de desenhar cada AFN, salvei sua figura e sua tabela de transições. Na revisão, comparei as transições e o reconhecimento esperado por ε-fecho; as **192.282 verificações** usadas nessa conferência não encontraram divergências.

`WHILE` também pertence à linguagem de `ID`. Os dois finais foram mantidos nesta etapa; a prioridade só é aplicada depois da determinização.

## 7. União dos AFNs

Os 11 conjuntos locais foram copiados sem fundir estados. Eles foram renumerados globalmente e receberam um novo inicial `S0`.

| Componente | Estados globais | Entrada pela união | Final rotulado |
|---|---|---|---|
| `WHILE` | `q1–q10` | `S0 —ε→ q1` | `q10` |
| `ID` | `q11–q16` | `S0 —ε→ q11` | `q16` |
| `NUM` | `q17–q22` | `S0 —ε→ q17` | `q22` |
| `ATRIB` | `q23–q24` | `S0 —ε→ q23` | `q24` |
| `RELOP` | `q25–q46` | `S0 —ε→ q25` | `q46` |
| `PONTO_VIRGULA` | `q47–q48` | `S0 —ε→ q47` | `q48` |
| `ABRE_PAR` | `q49–q50` | `S0 —ε→ q49` | `q50` |
| `FECHA_PAR` | `q51–q52` | `S0 —ε→ q51` | `q52` |
| `ABRE_CHAVE` | `q53–q54` | `S0 —ε→ q53` | `q54` |
| `FECHA_CHAVE` | `q55–q56` | `S0 —ε→ q55` | `q56` |
| `BRANCO` | `q57–q62` | `S0 —ε→ q57` | `q62` |

Resultado da união:

- 62 estados originais mais `S0`: **63 estados**;
- **11 novas transições ε**, uma para cada componente;
- **11 finais rotulados**, sem criar um final comum;
- nenhuma transição interna foi removida ou alterada;
- nenhuma determinização ou minimização foi realizada nesta etapa;
- **16.847 verificações** da união passaram sem divergência.

## 8. Conversão do AFN unido para AFD

Foi usada a construção de subconjuntos. Para cada subconjunto `T` e cada uma das 24 classes `a`, calculou-se:

```text
move(T, a) = estados alcançados consumindo a
E(move(T, a)) = fecho-ε do resultado
```

Cada conjunto novo recebeu o próximo nome `Dk`; um conjunto já conhecido reutilizou seu estado. O conjunto vazio foi preservado como estado morto.

### 8.1 Fecho inicial

```text
D0 = E({S0})
   = {S0, q1, q11, q17, q23, q25, q26, q28, q32,
      q34, q38, q42, q47, q49, q51, q53, q55, q57}
```

O fecho inicial possui **18 posições**: o novo inicial e as entradas alcançadas pelas ligações ε internas.

### 8.2 Resultado da determinização

- 25 subconjuntos não vazios alcançáveis;
- um estado morto `D25 = ∅`;
- **26 estados**, de `D0` a `D25`;
- tabela completa de `26 × 24`;
- **624 cálculos** individuais de `move` e fecho-ε;
- todos os estados alcançáveis e todos os subconjuntos distintos;
- 18.050 cadeias comparadas entre AFN e AFD, sem divergência;
- nenhuma minimização aplicada nesta etapa.

### 8.3 Estados por resultado léxico

| Resultado | Estados do AFD |
|---|---|
| não final | `D0`, `D7` |
| `WHILE` | `D24` |
| `ID` | `D1`, `D2`, `D14`, `D15`, `D22`, `D23` |
| `NUM` | `D3`, `D16` |
| `ATRIB` | `D4` |
| `RELOP` | `D5`, `D6`, `D17`, `D18`, `D19`, `D20` |
| `PONTO_VIRGULA` | `D8` |
| `ABRE_PAR` | `D9` |
| `FECHA_PAR` | `D10` |
| `ABRE_CHAVE` | `D11` |
| `FECHA_CHAVE` | `D12` |
| `BRANCO` | `D13`, `D21` |
| morto | `D25` |

Há 23 estados finais antes da minimização. `D13` e `D21` são finais, mas `BRANCO` é descartado pelo scanner.

### 8.4 Conflito entre palavra reservada e identificador

```text
D24 = {q10, q14, q15, q16}
```

Esse subconjunto contém o final `q10=WHILE` e o final `q16=ID`. A ordem de prioridade escolhe `WHILE`. A prioridade resolve apenas empates de mesmo comprimento; `whilex` continua sendo um único `ID` pela regra do maior lexema.

## 9. Minimização do AFD

Foi usado refinamento de partições pelo método de Moore. A partição inicial separou os não finais e cada categoria léxica, pois estados que produzem tokens diferentes não podem ser equivalentes.

### 9.1 Partição inicial

```text
B0  não final       = {D0,D7,D25}
B1  WHILE           = {D24}
B2  ID              = {D1,D2,D14,D15,D22,D23}
B3  NUM             = {D3,D16}
B4  ATRIB           = {D4}
B5  RELOP           = {D5,D6,D17,D18,D19,D20}
B6  PONTO_VIRGULA   = {D8}
B7  ABRE_PAR        = {D9}
B8  FECHA_PAR       = {D10}
B9  ABRE_CHAVE      = {D11}
B10 FECHA_CHAVE     = {D12}
B11 BRANCO          = {D13,D21}
```

### 9.2 Refinamentos

Para cada estado foi registrada uma assinatura com os 24 blocos de destino. Estados do mesmo bloco com assinaturas diferentes foram separados.

| Rodada | Blocos antes | Blocos depois | Divisões principais |
|---:|---:|---:|---|
| 1 | 12 | 16 | não finais em 3 grupos; `ID` em 2; `RELOP` em 2 |
| 2 | 16 | 17 | `D22` separado dos demais estados de `ID` |
| 3 | 17 | 18 | `D15` separado de `{D1,D2,D14}` |
| 4 | 18 | 19 | `D2` separado de `{D1,D14}` |
| 5 | 19 | 19 | nenhuma divisão; partição estável |

Sequência das quantidades de blocos:

```text
12 → 16 → 17 → 18 → 19 → estável
```

O arquivo `assinaturas-refinamento.csv` contém **130 linhas de cálculo**: 26 estados em cada uma das cinco rodadas, com as 24 classes explícitas.

### 9.3 Estados do AFD mínimo

Cada bloco estável tornou-se um estado `Mk`. `M0` contém o inicial antigo e o morto foi colocado por último.

| Estado mínimo | Estados antigos | Resultado |
|---|---|---|
| `M0` | `{D0}` | não final/inicial |
| `M1` | `{D1,D14}` | `ID` |
| `M2` | `{D2}` | `ID` |
| `M3` | `{D3,D16}` | `NUM` |
| `M4` | `{D4}` | `ATRIB` |
| `M5` | `{D5,D6}` | `RELOP` |
| `M6` | `{D7}` | não final |
| `M7` | `{D8}` | `PONTO_VIRGULA` |
| `M8` | `{D9}` | `ABRE_PAR` |
| `M9` | `{D10}` | `FECHA_PAR` |
| `M10` | `{D11}` | `ABRE_CHAVE` |
| `M11` | `{D12}` | `FECHA_CHAVE` |
| `M12` | `{D13,D21}` | `BRANCO` |
| `M13` | `{D15}` | `ID` |
| `M14` | `{D17,D18,D19,D20}` | `RELOP` |
| `M15` | `{D22}` | `ID` |
| `M16` | `{D23}` | `ID` |
| `M17` | `{D24}` | `WHILE` |
| `M18` | `{D25}` | morto |

Resultado:

- AFD completo: 26 estados;
- AFD mínimo: **19 estados**;
- redução de 7 estados;
- inicial: `M0`;
- morto: `M18`, com laço nas 24 classes;
- tabela mínima: `19 × 24 = 456` transições;
- as 24 classes foram preservadas.

### 9.4 Verificação da minimalidade

Foram produzidas testemunhas de distinção para todos os pares:

```text
C(19,2) = 19 × 18 / 2 = 171 pares
```

Cada par possui uma palavra de classes que leva a resultados léxicos diferentes. A partição estável mostra que nenhum bloco restante pode ser dividido; as 171 testemunhas mostram que nenhum par de estados mínimos pode ser unido.

Também foram verificadas 14.425 palavras de classes de comprimento até três, confirmando a equivalência entre o AFD de 26 estados e seu quociente mínimo.

## 10. Scanner manual em C

O scanner foi implementado usando diretamente a tabela do AFD mínimo. Flex não é usado nesta parte.

Estrutura principal:

```text
parte1-scanner-manual/
├── src/
│   ├── dfa.h
│   ├── dfa.c
│   ├── dfa_table.inc
│   ├── scanner.h
│   ├── scanner.c
│   └── main.c
├── testes/validos/
├── testes/invalidos/
├── saidas/validos/
├── saidas/invalidos/
├── gerar_tabela_c.py
├── testar.py
├── Makefile
└── README.md
```

Transcrevi as 456 transições e as categorias finais para `dfa_table.inc`, conferindo cada linha com a tabela mínima. A lógica de leitura e reconhecimento foi escrita em `scanner.c`.

### 10.1 Algoritmo do scanner

1. Começar em `M0` na posição atual da entrada.
2. Classificar cada byte em uma das 24 classes.
3. Consultar a tabela mínima e avançar enquanto o destino não for `M18`.
4. Guardar o último estado final e a posição correspondente.
5. Ao encontrar o morto ou o fim da entrada, retornar ao último final.
6. Emitir o maior lexema aceito.
7. Descartar `BRANCO`, atualizando linha e coluna.
8. Se não houver final anterior, informar o byte como erro e avançar uma posição.

Comentários não são reconhecidos. `/` e `*` chegam ao estado morto a partir de `M0` e são informados como `ERRO_CARACTERE`.

Essa estratégia implementa a regra do **maior lexema**. A categoria armazenada no estado final implementa a prioridade entre `WHILE` e `ID`.

### 10.2 Saída e códigos de retorno

Cada token emitido mostra linha, coluna, categoria e lexema. Brancos aparecem apenas na contagem de descartados.

```text
LINHA  COLUNA  TOKEN                              LEXEMA
1      1       WHILE                              "while"
1      7       ABRE_PAR                           "("
1      8       ID                                 "x"
```

Códigos de retorno:

- `0`: arquivo sem erro léxico;
- `1`: um ou mais erros léxicos;
- `2`: erro de uso, abertura, leitura ou memória.

O scanner faz análise léxica. Uma sequência de tokens pode ser léxica e ainda contrariar a BNF; detectar isso exigiria um parser, que não pertence ao escopo desta parte.

## 11. Arquivos válidos, inválidos e testes

Foram criados quatro exemplos válidos:

1. `01-while-simples.cm`;
2. `02-while-com-identificadores.cm`;
3. `03-prefixo-de-palavra-reservada.cm`;
4. `04-igualdade.cm`.

Eles cobrem números, identificadores, `<`, `>=`, `!=`, `==` e o caso `whilex`, reconhecido como `ID`.

Também foram criados quatro exemplos inválidos:

1. `@`;
2. `+`, que não pertence aos operadores do recorte;
3. `!` isolado;
4. `/`, que não inicia token neste recorte.

Resultados da revisão atual:

- os quatro válidos retornaram código 0;
- os quatro inválidos retornaram código 1 e indicaram o erro esperado;
- as saídas foram salvas em `parte1-scanner-manual/saidas/`;
- a tabela mínima usada pelo scanner foi regenerada e conferida com o AFD mínimo;
- a referência independente em Python gerou novamente as saídas dos oito arquivos;
- as simulações do AFN, do AFD e do AFD mínimo concordaram nos testes exaustivos e dirigidos;
- o ambiente atual não possui compilador C nem Make, portanto a compilação e a comparação com o executável devem ser repetidas em um ambiente que forneça essas ferramentas.

## 12. Resumo numérico final

| Etapa | Resultado |
|---|---:|
| Regras da gramática | 7 |
| Categorias emitidas | 10 |
| Categorias descartadas | 1 |
| Classes do alfabeto | 24 |
| AFNs individuais | 11 |
| Estados nos AFNs individuais | 62 |
| Estados no AFN unido | 63 |
| Novas transições ε na união | 11 |
| Finais rotulados na união | 11 |
| Estados no AFD | 26 |
| Cálculos da determinização | 624 |
| Estados no AFD mínimo | 19 |
| Transições na tabela mínima | 456 |
| Rodadas de refinamento | 5 |
| Pares mínimos distinguidos | 171 |
| Arquivos válidos | 4 |
| Arquivos inválidos | 4 |

Fluxo completo:

```text
7 regras
   ↓
11 ERs
   ↓
11 AFNs / 62 estados
   ↓ união por novo S0
AFN unido / 63 estados
   ↓ construção de subconjuntos
AFD / 26 estados / 624 cálculos
   ↓ refinamento de partições
AFD mínimo / 19 estados / 456 transições
   ↓ implementação da tabela
scanner manual em C
   ↓
4 códigos válidos + 4 inválidos + saídas verificadas
```

## 13. Onde consultar cada detalhe

- `01-gramatica-e-tokens.md`: gramática e ERs.
- `afns-individuais/`: imagens e especificações dos 11 AFNs.
- `afn-uniao/`: união completa e mapeamento global.
- `afd/`: construção de subconjuntos e os 624 cálculos.
- `afd-minimo/`: partições, assinaturas, mapeamento, tabela e testemunhas.
- `figuras-automatos-limpos/`: figuras sem textos explicativos, próprias para inserir no relatório.
- `parte1-scanner-manual/`: programa C, testes e saídas.

A construção dos autômatos e o scanner manual estão documentados para esta gramática. O scanner Flex da BNF completa constitui uma etapa separada do trabalho.

## 14. Sugestão alternativa: somente atribuições

Como alternativa para reduzir ainda mais o trabalho manual, pode-se implementar somente sequências de atribuições entre identificadores:

```c
x = y;
resultado = valor;
x = y; z = x;
```

A gramática sugerida possui cinco regras numeradas:

```bnf
1. programa        → lista-comandos
2. lista-comandos  → lista-comandos comando | comando
3. comando         → atribuicao
4. atribuicao      → ID = valor ;
5. valor           → ID
```

Se cada alternativa da segunda regra for contada separadamente, existem seis produções. Nos dois modos de contagem, o recorte satisfaz a exigência de pelo menos cinco regras.

### 14.1 Tokens necessários

A menor versão funcional utiliza quatro padrões léxicos:

| Padrão | Função | Tratamento |
|---|---|---|
| `ID` | uma ou mais letras ASCII | emitir token |
| `ATRIB` | símbolo `=` | emitir token |
| `PONTO_VIRGULA` | símbolo `;` | emitir token |
| `BRANCO` | um ou mais espaços, tabulações, novas linhas ou retornos de carro | reconhecer e descartar |

As expressões regulares são as mesmas já apresentadas na seção 5 para `ID`, `ATRIB`, `PONTO_VIRGULA` e `BRANCO`. Não seriam necessários palavra reservada, números, operadores relacionais, parênteses nem chaves. Também não existiria conflito entre palavra reservada e identificador.

As 24 classes poderiam ser mantidas para permitir comparação direta com a gramática de `while`. As classes que não iniciam nenhum dos quatro padrões levariam ao estado morto.

### 14.2 Tamanho sem comentários

Esta seria a opção recomendada para obter o menor autômato e o menor trabalho manual:

| Etapa | Resultado |
|---|---:|
| Regras numeradas | 5 |
| Padrões léxicos | 4 |
| Estados nos AFNs separados | 16 |
| Estados no AFN unido | 17 |
| Estados no AFD | 8 |
| Cálculos da determinização | 192 (`8 × 24`) |
| Estados no AFD mínimo | 6 |
| Rodadas da minimização | 2, contando a rodada estável |
| Transições da tabela mínima | 144 (`6 × 24`) |

Nesta versão, `/` e `*` não formariam comentários e produziriam erro léxico.

### 14.3 Tamanho caso comentários fossem exigidos

Acrescentar o reconhecimento de comentários `/* ... */` elevaria o total para cinco padrões. O AFN desse padrão possui 26 estados.

| Etapa | Resultado |
|---|---:|
| Regras numeradas | 5 |
| Padrões léxicos | 5 |
| Estados nos AFNs separados | 42 |
| Estados no AFN unido | 43 |
| Estados no AFD | 15 |
| Cálculos da determinização | 360 (`15 × 24`) |
| Estados no AFD mínimo | 10 |
| Rodadas da minimização | 4, contando a rodada estável |
| Transições da tabela mínima | 240 (`10 × 24`) |

### 14.4 Comparação com a gramática de `while`

| Medida | `while` atual | Atribuições sem comentário | Atribuições com comentário |
|---|---:|---:|---:|
| Regras numeradas | 7 | 5 | 5 |
| Padrões léxicos | 11 | 4 | 5 |
| Estados nos AFNs separados | 62 | 16 | 42 |
| Estados no AFN unido | 63 | 17 | 43 |
| Estados no AFD | 26 | 8 | 15 |
| Cálculos da determinização | 624 | 192 | 360 |
| Estados no AFD mínimo | 19 | 6 | 10 |
| Transições da tabela mínima | 456 | 144 | 240 |

A sugestão de atribuições sem comentários reduz o AFN unido de 63 para 17 estados, o AFD de 26 para 8 estados e o AFD mínimo de 19 para 6 estados. Por isso, ela seria a escolha mais simples para fazer manualmente.

Um código como `x = ;` pode ser tokenizado sem erro léxico, embora contrarie a gramática. Esse caso exigiria um analisador sintático. Esta seção registra apenas uma alternativa; a implementação documentada nas seções anteriores continua sendo o `while` com bloco, sete regras e sem comentários.
