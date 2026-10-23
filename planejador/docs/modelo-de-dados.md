# Modelo de dados — convenções

Este documento é a referência das regras que os schemas (`../schemas`) não conseguem
expressar sozinhos. Vale para o motor (C), o parametrizador (B), os importadores (A) e
o planejador (D).

## 1. Unidades e sistema de coordenadas

- **Milímetro** em todo o modelo de dados e no motor. Ângulos em graus.
- glTF trabalha em metros: a conversão (÷1000) acontece **só na borda** (render e
  importação). O motor nunca vê metro.
- Sistema do módulo, idêntico ao do glTF/Three.js (Y para cima, destro), então não há
  troca de eixos em lugar nenhum:

```
        Y (altura, A)
        │
        │      tampo
        │   ┌───────────┐
        │   │           │      origem = canto inferior-esquerdo-TRASEIRO
        │   │   vão     │      (encostado na parede)
        │   │           │
        │   └───────────┘
        │      base
        └──────────────────── X (largura, L)
       ╱
      ╱  Z (profundidade, P): da parede (z = 0) para a frente (z = P)
```

- Faces semânticas: `esq` (−X), `dir` (+X), `baixo` (−Y), `cima` (+Y), `tras` (−Z, parede),
  `frente` (+Z, observador).
- `P` é a profundidade do **corpo** (caixa). Frentes sobrepostas ficam à frente de `z = P`.
  Validar contra o gabarito Promob na etapa 2 (ver README, pendências).

## 2. Peça de chapa

Uma peça é uma caixa retangular. Ela não é desenhada: é **calculada**.

| Campo | Significado |
|---|---|
| `eixos` | 2 letras. 1ª = eixo do módulo ao longo do **comprimento** (direção do veio); 2ª = eixo da **largura**; a espessura fica no eixo restante. |
| `comprimento`, `largura` | Medidas **finais** (acabadas, com fita). |
| `espessura` | Padrão: espessura do material do acabamento (`esp.<acabamento>`). |
| `posicao` | Canto mínimo (menor x, y, z) no sistema do módulo. |

Orientações usuais: lateral `YZ`, base/tampo/prateleira `XZ`, costa `XY`,
**porta com veio vertical `YX`**. O veio sai do próprio modelo: trocar `XY` por `YX`
gira o veio sem mexer em fórmula.

### Bordas locais (lista de corte)

```
          C2  (borda em largura = máx)
     ┌──────────────────────────┐
  L1 │  → comprimento (veio)    │ L2
     │  ↑ largura               │
     └──────────────────────────┘
          C1  (borda em largura = 0)
```

- `C1`/`C2`: bordas paralelas ao comprimento (medem o comprimento).
- `L1`/`L2`: bordas paralelas à largura (medem a largura).
- `F1`/`F2`: faces grandes, no sentido +/− do eixo da espessura.

**Fitas são escritas por face semântica** (`"fitas": ["frente"]`) e o motor converte para
C1/C2/L1/L2. Quem cadastra pensa no móvel ("fita na frente"); a produção recebe a borda
local. Pedir fita numa face que não é borda da peça (ex.: `esq` numa lateral, que é a face
grande) é erro de carga.

### Medida final × medida de corte

`regrasFita.descontarNoCorte = true` → corte = final − espessura da fita em cada borda
fitada, só para fitas com espessura ≥ `descontarAPartirDe`. Depois aplica-se
`arredondamento`. A lista de corte sai sempre com as duas medidas.

## 3. Expressões

Toda medida pode ser número ou fórmula em texto.

- **Operadores**: `+ - * / %`, parênteses, `< <= > >= == !=`, `&& || !`, ternário
  `c ? a : b`.
- **Texto literal** entre aspas simples: `fab.construcao.rodape.tipo == 'pes'`.
- **Funções**: `min max abs arred piso teto limitar se tab` (manifesto em
  `packages/engine/src/regras/biblioteca.json`). `tab('dobradicasPorAltura', 1800)` consulta
  as tabelas do padrão de fábrica.
- **Nomes visíveis**:

| Nome | Origem |
|---|---|
| `L`, `A`, `P`, `nGavetas`... | Parâmetros do módulo |
| `Li`, `Ai`, `e`... | Variáveis derivadas do módulo |
| `esp.<acabamento>` | Espessura do material atribuído ao acabamento (`esp.estrutura`) |
| `fab.<caminho>` | Padrão **efetivo** (já com linha, categoria e módulo aplicados) |
| `vao.x vao.y vao.z vao.L vao.A vao.P` | Dentro de regras de vão |
| `i`, `n` | Índice (0..n−1) e total, em peças com `repetir` |

- **Avaliação**: parser próprio (sem `eval`), grafo de dependências entre variáveis,
  ciclo = erro de carga. `?:` e `se()` são **preguiçosos**: um caminho `fab.*` inexistente
  no ramo não escolhido não é erro (permite `temPes ? fab.construcao.rodape.altura : 0`).
- **Texto com fórmula**: `"BAL2P-{L}x{A}x{P}"` interpola expressões entre chaves.

## 4. Herança

Cadeia aplicada pelo motor, cada nível como **JSON Merge Patch (RFC 7386)**:

```
padrão de fábrica
  ← linha.sobrescrever
  ← categoria.sobrescrever
  ← módulo.sobrescrever            (após resolver 'estende')
  ← instância no projeto.sobrescrever
= padrão efetivo  (exposto como fab.*)
```

Regras que fazem isso funcionar:

1. **Tudo que pode ser sobrescrito é objeto indexado por id, nunca lista.** O merge de
   listas substitui a lista inteira; com objetos, uma linha troca uma folga, um módulo remove
   uma peça herdada (`"rodape": null`), outro adiciona uma regra. Por isso `pecas`, `vaos`,
   `restricoes`, `regras`, `ferragens`, `materiais`, `papeis` são mapas. Listas só onde o
   conjunto é atômico (`partes` de uma divisão, `linhas` de uma tabela, `usinagens` de uma
   peça).
2. **`estende`**: o módulo filho é um patch sobre o módulo base. `abstrato` **não é
   herdado** (um módulo concreto que estende uma caixa abstrata é concreto).
3. **Padrão de parâmetro**: módulo (com `estende`) > `parametrosPadrao` da categoria >
   da linha. A categoria preenche lacuna, não atropela o módulo.
4. **Validação em dois tempos**: cada arquivo é validado no formato parcial; o resultado do
   merge é validado contra o schema completo (padrão efetivo e módulo efetivo).
5. **Espessura nunca é digitada**: vem do material do acabamento. A linha "Essencial" troca
   `acabamentos.estrutura` para MDF 15 e todas as fórmulas que usam `esp.estrutura` se
   ajustam.
6. **Reuso entre linhas**: geometria mora em módulos abstratos; o módulo concreto de cada
   linha é um arquivo fino (`estende` + `categoria`). A fábrica não duplica fórmula.

## 5. Vãos

Vão = volume livre (luz interna) com uma caixa `{x, y, z, L, A, P}` e as peças que o
delimitam (`bordas`). Um vão ou é **dividido** ou recebe **regras** (ou os dois: um vão
com prateleiras e portas).

- `dividir.eixo`: direção da divisão; `partes` vão da coordenada menor para a maior
  (em `y`, de baixo para cima).
- `tamanho`: expressão em mm, `"40%"` ou `"*"`/`"2*"` (restante, por peso).
- `separador`: peça criada entre as partes (travessa, divisória) — vira automaticamente a
  `borda` dos sub-vãos.
- `referencia: "frente"`: `tamanho` passa a ser a **altura da frente acabada**. O motor
  posiciona o separador no eixo da fresta entre frentes. É assim que a modulação funciona
  de verdade: a grade de frentes manda, a caixa obedece.
- Sobreposição de frente por borda: peça **externa** (lateral, base, tampo) →
  `construcao.frentes.sobreposicaoExterna × espessura`; **separador** interno →
  `sobreposicaoInterna × espessura`.

## 6. Regras

Uma regra é uma função TypeScript pura + um manifesto (`biblioteca.json`).

```ts
interface Regra<Args> {
  manifesto: ManifestoRegra;
  executar(ctx: ContextoRegra, args: Args): Contribuicao;
}
// ctx: fab (padrão efetivo), esp, avaliar(expr), chamar(idRegra, args),
//      e conforme 'aplicaEm': o módulo, o vão (caixa + bordas) ou a peça.
// Contribuicao: { pecas?, ferragens?, usinagens?, componentes3d?, alertas?, retorno? }
```

- **Código = comportamento, padrão = números.** Nenhuma folga, quantidade ou tabela vive
  no código. "Quantas dobradiças" é a tabela `dobradicasPorAltura`; "como distribuí-las" é
  a regra.
- **`le`** lista os caminhos do padrão que a regra usa. O motor confere na carga do catálogo:
  se a fábrica não definiu `folgas.gavetas.traseira`, o erro aparece no cadastro, não na
  produção.
- **Argumentos**: para tipos `medida/inteiro/numero/expressao` o texto é fórmula; para
  `opcao/ferragem/peca/faces/lista` o texto é **literal**. Ligar a um parâmetro ou fórmula:
  `{"param": "puxador"}` ou `{"expr": "L > 900 ? 'progressiva' : 'iguais'"}`.
- **Ids estáveis**: peças geradas recebem id derivado do caminho
  (`vao.interno/gavetas/gaveta[2]/frente`). Testes, gabarito Promob e ajustes manuais no
  projeto dependem disso.
- **Estado `planejada`**: o catálogo aceita a chamada e o motor ignora com aviso. Permite
  cadastrar o módulo completo antes da regra existir.

## 7. Geometria importada (glTF)

- **Chapa é sempre gerada pelo motor, nunca uma malha esticada.** Malha esticada não gera
  lista de corte confiável. Malhas importadas servem para (a) inferir peças no
  parametrizador e (b) itens que não são chapa: puxadores, pés, perfis, acessórios,
  eletros (`modelos3d`).
- Comportamento por eixo: `fixo` (tamanho original), `esticar` (até `tamanho`, com
  `protegerBordas` para ponteiras), `repetir` (`quantidade` entre `de` e `ate`, ou `passo`).
  `ancora` + `em` prendem o min/centro/max da malha numa coordenada do módulo (ancorar).
- Importadores gravam o rascunho de grupos em `node.extras.planejador`:
  `{ id, origem: "layer:LATERAIS" | "bloco:..." | "componente:...", papelSugerido, confianca }`.
  `modelos3d.*.fonte.no` aponta para `extras.planejador.id` (estável), não para o nome.

## 8. Saída do motor

`gerarModulo(moduloEfetivo, parametros, padraoEfetivo) → ResultadoModulo`, síncrono,
determinístico, sem I/O:

- `pecas[]`: id, papel, nome, material, espessura, comprimento/largura finais e de corte,
  fitas `{C1, C2, L1, L2}`, veio, transformação (posição + eixos) para o 3D, usinagens,
  origem (template ou regra).
- `ferragens[]`: id, quantidade, origem.
- `componentes3d[]`: arquivo, nó, matriz por instância.
- `alertas[]`: restrições disparadas e avisos de regra.
- `valores`: todos os parâmetros e variáveis resolvidos (a "tabela de parâmetros" do
  Inventor; essencial para depurar contra o Promob).

`listaCorte(resultados[])` agrupa por material + medidas de corte + fitas + usinagens.

## 9. Instância no projeto (bloco D, referência)

```json
{ "modulo": "cz.balcao-2-portas", "versaoModulo": "0.1.0",
  "parametros": { "L": 900, "nPrateleiras": 2 },
  "acabamentos": { "frente": "MDF18_FREIJO" },
  "sobrescrever": {}, "posicao": [0, 0, 0], "rotacaoY": 0 }
```

A instância **fixa a versão do módulo**: atualizar a biblioteca não altera projeto vendido.

## 10. Gabarito Promob

Cada caso em `packages/engine/test/gabarito-promob/<caso>/`:

- `entrada.json`: módulo + parâmetros + padrão.
- `promob.xml`: XML de produção exportado pelo Promob para o mesmo móvel.
- `mapa.json`: descrição Promob → papel (a ponte de vocabulário).

O teste compara peça a peça: quantidade, material, medidas de corte (tolerância ±0,5 mm) e
fitas, e imprime a tabela de divergências. O parser do XML é escrito a partir dos arquivos
reais da fábrica (o layout varia com a versão e o plugin de exportação), não de suposição.
