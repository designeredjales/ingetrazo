# Planejador 3D paramétrico de móveis — proposta de arquitetura

Alternativa web e simplificada ao Promob. Este diretório contém, por enquanto, **apenas o
contrato**: estrutura de pastas, schemas JSON e um catálogo de exemplo validado. Nenhuma
linha do motor foi escrita: a ideia é aprovar o modelo de dados antes de codar.

> Local provisório: este diretório está dentro de um fork do IngeTrazo (GPL-3.0).
> Ver "Pendências" antes de escrever código.

## Tese

O diferencial não é o 3D, é a **fidelidade de produção**: o mesmo JSON de módulo que
desenha o móvel na tela gera a lista de corte, as fitas, a furação e as ferragens que a
fábrica executa. Por isso a ordem é **C → B → A → D** e o primeiro critério de viabilidade
é bater com o XML de produção do Promob.

## Stack

Vite + React + TypeScript · Three.js via React Three Fiber + drei · Zustand ·
glTF como formato 3D interno · JSON próprio para módulos, validado por JSON Schema
(Ajv) · Vitest.

## Estrutura de pastas

```
planejador/
├── README.md
├── package.json                  workspaces npm (engine, promob, importers, web)   [etapa 1]
├── tsconfig.base.json                                                              [etapa 1]
├── docs/
│   ├── modelo-de-dados.md        convenções: eixos, peças, fórmulas, herança, vãos, regras
│   └── adr/                      uma decisão de arquitetura por arquivo
├── schemas/                      CONTRATO — fonte da verdade (JSON Schema 2020-12)
│   ├── comum.schema.json         expressão, faces, eixos, usinagem, invocação de regra
│   ├── padrao-fabrica.schema.json
│   ├── no-hierarquia.schema.json linha e categoria
│   ├── modulo.schema.json
│   └── regra.schema.json         manifesto da biblioteca de regras
├── catalogo/                     DADOS — o que a fábrica edita e o parametrizador grava
│   ├── fabricas/fab-exemplo.json
│   ├── linhas/cozinha-essencial.json
│   ├── categorias/cozinha-balcoes.json
│   ├── modulos/
│   │   ├── _base/caixa-balcao.json      módulo abstrato (a caixa, escrita uma vez)
│   │   ├── balcao-2-portas.json         fórmulas puras + regra de portas
│   │   ├── gaveteiro.json               regra de gavetas com n e divisão parametrizados
│   │   └── balcao-gaveta-2-portas.json  vão dividido pela grade de frentes
│   └── modelos3d/                glb de puxadores, pés, acessórios
├── packages/
│   ├── engine/                   (C) TypeScript PURO — lint proíbe importar three/react
│   │   ├── src/
│   │   │   ├── tipos/            gerados dos schemas (json-schema-to-typescript)
│   │   │   ├── expr/             tokenizer, parser, avaliador, grafo de dependências
│   │   │   ├── catalogo/         carga, validação, herança (merge patch), 'estende'
│   │   │   ├── montagem/         parâmetros → variáveis → peças → vãos → regras
│   │   │   ├── regras/           biblioteca nativa + biblioteca.json (manifestos)
│   │   │   ├── producao/         medida de corte, fitas C1/C2/L1/L2, ferragens, usinagens
│   │   │   └── index.ts          gerarModulo(), listaCorte()
│   │   └── test/
│   │       ├── unidade/
│   │       └── gabarito-promob/<caso>/{entrada.json, promob.xml, mapa.json}
│   ├── promob/                   parsers de XML Promob (produção e entidades)
│   ├── importers/                (A) DXF/GLB/OBJ/DAE/FBX → glTF + rascunho de grupos; STEP depois
│   └── web/                      Vite + React + R3F + drei + Zustand
│       └── src/
│           ├── estado/           stores Zustand (catálogo, projeto, seleção)
│           ├── visualizador/     <Peca3D>, <Modulo3D>: render do ResultadoModulo
│           ├── parametrizador/   (B)
│           ├── planejador/       (D) ambiente, paredes, snap, colisão
│           └── workers/          motor e importadores fora da thread principal
└── tools/                        CLI: validar-catalogo, gerar-tipos, comparar-promob
```

Hoje existem: `docs/modelo-de-dados.md`, os 5 schemas, o catálogo de exemplo e o
manifesto `packages/engine/src/regras/biblioteca.json`.

## Decisões de arquitetura

| # | Decisão | Por quê |
|---|---|---|
| 1 | **JSON Schema é a fonte da verdade**; tipos TS são gerados | O formato é escrito por gente, pelo parametrizador e, no futuro, por IA. Um contrato neutro serve aos três. |
| 2 | **Motor puro**: síncrono, determinístico, mm, sem I/O, sem three/react | Testável peça a peça contra o Promob; roda em Web Worker, servidor ou CLI. |
| 3 | **Chapa é gerada, nunca esticada de malha** | Malha esticada não gera lista de corte confiável. Importado serve para inferir peças e para o que não é chapa. |
| 4 | **Objetos indexados por id + JSON Merge Patch** em toda a hierarquia | Sobrescrever uma folga, remover uma peça (`null`) ou adicionar uma regra sem copiar o resto. |
| 5 | **Espessura vem do material do acabamento** | Trocar MDF 18 por 15 numa linha recalcula tudo sem tocar em fórmula. |
| 6 | **Código = comportamento; padrão de fábrica = números e tabelas** | A fábrica ajusta folga e tabela de dobradiças sem programador. |
| 7 | **Regra declara o que lê** (`le`) e o motor confere na carga | Erro aparece no cadastro, não na produção. |
| 8 | **Eixos iguais ao glTF** (Y para cima, +Z para a frente, origem traseira) | Zero conversão entre motor, render e arquivos. |
| 9 | **Módulo versionado (semver); projeto fixa a versão** | Atualizar a biblioteca não altera pedido vendido. |
| 10 | **Português como linguagem ubíqua** (JSON e TS) | Um só vocabulário entre marceneiro, cadastrador e código. |

## Plano do protótipo de viabilidade

| Etapa | Bloco | Entrega | Prazo-alvo |
|---|---|---|---|
| 0 | — | Contrato: schemas + catálogo de exemplo validado | **feito** |
| 1 | C | Workspace, tipos gerados, parser de expressões, herança, peças por fórmula, fitas, medida de corte, lista de corte; testes dos 3 módulos. Visualizador de **depuração** (R3F, sliders de L/A/P) | 3 dias |
| 2 | C | Regras v0 (portas, gavetas, prateleiras, dobradiças, corrediça, puxador, canal de costa) + comparador com o XML do Promob | 2 dias |
| — | — | **Gate 1: fidelidade** | — |
| 3 | B | Importar GLB → explodir nós → atribuir papel → inferir eixos e medidas → propor fórmulas → salvar JSON | 3 dias |
| 4 | A | DXF 2D (camadas e blocos), OBJ/DAE/FBX pelos loaders do Three.js; spike de STEP (opencascade.js em worker) | 2 dias |
| 5 | D | Ambiente, paredes, arrastar/alinhar/colisão, edição em tempo real, screenshot + profundidade | depois do gate 1 |

## Critérios de viabilidade (go / no-go)

| Gate | Métrica | Meta |
|---|---|---|
| G1 Fidelidade | Peças batendo com o XML do Promob (3 módulos × 3 medidas) | ≥ 95 % das peças em ±0,5 mm; 100 % em quantidade e fitas |
| G2 Velocidade | Tempo do motor por módulo | < 5 ms (100 módulos editáveis em tempo real) |
| G3 Cobertura | Módulos de uma linha real (amostra de 30) expressáveis com as regras v0 | ≥ 80 % |
| G4 Cadastro | Tempo para parametrizar um módulo novo, sem programar | < 20 min |

## Pendências

1. **Repositório.** Este diretório está num fork do IngeTrazo, licenciado em GPL-3.0. Se o
   planejador for produto comercial, recomenda-se repositório próprio antes da etapa 1.
   O código Python do IngeTrazo (ex.: `core/pieces.py`, `core/parts.py`, que separam peças
   e medem a "caixa de corte" de malhas importadas) serve de referência de algoritmo; copiar
   o código traria a GPL junto.
2. **Gabarito.** XML de produção do Promob de 3 módulos (balcão 2 portas, gaveteiro,
   balcão com gaveta e portas) em 3 medidas cada, mais o padrão real da fábrica
   (folgas, furação, tabela de dobradiças).
3. **Convenções a confirmar com o gabarito**: `P` inclui ou não a frente; desconto de fita
   no corte; arredondamento.

Os valores de `catalogo/fabricas/fab-exemplo.json` são de referência e precisam ser
calibrados com a fábrica real.
