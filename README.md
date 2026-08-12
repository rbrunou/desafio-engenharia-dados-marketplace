# Desafio Técnico — Engenharia de Dados

Resolução do teste técnico de engenharia de dados sobre a base de um marketplace B2B de
distribuição de alimentos.

Toda a análise está em **[`desafio_engenharia_dados.ipynb`](desafio_engenharia_dados.ipynb)**,
com as queries comentadas e as decisões de modelagem documentadas ao lado de cada resultado.

### Índice

- [Como rodar](#como-rodar)
- [Auditoria da base](#auditoria-da-base) — os 5 achados que condicionam todas as queries
- [Desafio 1 — Faturamento bruto mensal](#desafio-1--faturamento-bruto-mensal-últimos-12-meses)
- [Desafio 2 — Crescimento de GMV por seller](#desafio-2--ranking-de-crescimento-de-gmv-por-seller)
- [Desafio 3 — Descontos abusivos](#desafio-3--pedidos-com-desconto-abusivo)
- [Desafio 4 — Produtos nunca protagonistas](#desafio-4--produtos-que-nunca-são-o-item-de-maior-valor)
- [Síntese](#síntese-o-que-a-base-revela)

### Resumo dos resultados

| Desafio | Resultado | Achado principal |
|---|---|---|
| 1 | R$ 715,6 mi em 12 meses, 45.191 pedidos | O relatório sem filtro de status inflava o faturamento em **32,8%** |
| 2 | Top 10 sellers, líder com **+69,35%** | Usar o trimestre parcial zeraria todo crescimento positivo |
| 3 | **901 pedidos**, de apenas **8 de 120 sellers** | 4 desses 8 estão no top 10 de crescimento — a suspeita de fraude se confirma |
| 4 | **Zero produtos** — e zero é a resposta certa | `unit_price` é sorteado por linha, não é atributo do produto |

---

## Como rodar

### Google Colab (recomendado)

1. Abra o notebook no Colab.
2. Faça upload dos 6 CSVs da pasta `dados/` para `/content`.
3. Descomente `!pip install -q pandasql` na primeira célula de código.
4. `Ambiente de execução → Executar tudo`.

### Local

```bash
pip install pandas pandasql matplotlib jupyter
jupyter notebook desafio_engenharia_dados.ipynb
```

O notebook localiza os CSVs automaticamente em `dados/`, `../dados`, `/content` ou
`/content/dados` — não é preciso ajustar caminhos.

Para executar sem abrir a interface:

```bash
jupyter nbconvert --to notebook --execute --inplace desafio_engenharia_dados.ipynb
```

Tempo de execução completo: **~10 segundos**.

### Estrutura

```
.
├── README.md
├── .gitignore
├── desafio_engenharia_dados.ipynb   # resolução completa
└── dados/                           # CSVs de origem
    ├── buyers.csv        (3.001 linhas)
    ├── order_items.csv   (214.768 linhas)
    ├── orders.csv        (80.002 linhas)
    ├── payments.csv      (80.000 linhas)
    ├── products.csv      (800 linhas)
    └── sellers.csv       (120 linhas)
```

**Stack:** `pandas` + `pandasql` (SQLite como motor SQL sobre os DataFrames), conforme o
notebook modelo fornecido no enunciado.

---

## Auditoria da base

O notebook começa por uma auditoria da base, porque cada achado abaixo muda a forma correta
de escrever as queries seguintes.

| # | Achado | Impacto |
|---|---|---|
| 1 | **A base é um snapshot histórico** — o último pedido é de `2024-11-29` | Ancorar a janela de 12 meses em `DATE('now')` retorna **zero linhas**. A janela é ancorada em `MAX(created_at)` e parametrizada |
| 2 | **2 pedidos com `id` duplicado** (`79990`, `79991`), mesma chave e conteúdo levemente diferente | Assinatura de reingestão/CDC sem deduplicação. As queries deduplicam por `ROW_NUMBER()`, mantendo a versão mais recente |
| 3 | **Status de pedido e de pagamento são 100% consistentes** — `completed`/`delivered` sempre correspondem a pagamento `paid` | Filtrar por `orders.status` é suficiente; um `JOIN` com `payments` não mudaria nenhum número |
| 4 | **`orders.total_value` = soma dos itens líquida de desconto** (bate em 80.000 de 80.002 registros; as 2 exceções são as duplicatas) | Permite calcular faturamento sem varrer as 214k linhas de `order_items` |
| 5 | **Integridade referencial íntegra** — nenhuma chave órfã, nenhum pedido sem itens, nenhum `NULL` nas colunas usadas | Não é necessário tratamento defensivo de órfãos nos joins |

---

## Desafio 1 — Faturamento bruto mensal (últimos 12 meses)

> *"O time financeiro reclama que o total de faturamento mensal varia dependendo de quem
> puxa o relatório. Pedidos com status `cancelled` e `refunded` estão sendo incluídos em
> alguns relatórios."*

### Decisões de modelagem

| Ponto | Decisão | Por quê |
|---|---|---|
| Janela de 12 meses | Ancorada em `MAX(created_at)`, não em `DATE('now')` | Achado 1 — a base termina em 2024-11 |
| Recorte da janela | 12 meses-calendário fechados (2023-12 a 2024-11) | Um relatório mensal do financeiro compara meses inteiros; uma janela móvel de 365 dias produziria meses parciais nas pontas |
| Definição de "bruto" | `SUM(total_value)` = GMV do pedido | "Bruto" aqui se opõe a *líquido de cancelamentos/estornos* — exatamente o que o filtro de status resolve. O desconto comercial já está aplicado no preço praticado |
| Deduplicação | `ROW_NUMBER()` por `id`, versão mais recente | Achado 2 |
| Filtro de status | `LOWER(TRIM(status))` | Defesa contra variação de casing/espaços na origem |
| Ticket médio | `SUM(total_value) / NULLIF(COUNT(*), 0)` | `NULLIF` elimina o risco de divisão por zero |
| Data de referência | `orders.created_at` | O financeiro pede faturamento por mês do pedido (competência); `payments.paid_at` mediria regime de caixa |

### Resultado

| mês | pedidos | faturamento bruto | ticket médio |
|---|---:|---:|---:|
| 2024-11 | 3.643 | 56.710.238,63 | 15.566,91 |
| 2024-10 | 3.863 | 62.493.439,13 | 16.177,44 |
| 2024-09 | 3.779 | 60.274.462,50 | 15.949,84 |
| 2024-08 | 3.743 | 60.102.539,82 | 16.057,32 |
| 2024-07 | 3.862 | 59.769.406,95 | 15.476,28 |
| 2024-06 | 3.644 | 58.797.774,37 | 16.135,50 |
| 2024-05 | 3.848 | 60.730.315,72 | 15.782,31 |
| 2024-04 | 3.653 | 57.779.312,59 | 15.816,95 |
| 2024-03 | 3.877 | 61.525.913,64 | 15.869,46 |
| 2024-02 | 3.628 | 57.926.376,26 | 15.966,48 |
| 2024-01 | 3.789 | 59.473.253,15 | 15.696,29 |
| 2023-12 | 3.862 | 60.053.131,70 | 15.549,75 |

**Total do período: R$ 715.636.164,46 em 45.191 pedidos.**

### Tamanho do erro

| Leitura | Valor (12 meses) |
|---|---:|
| Relatório sem filtro de status | R$ 950.317.556,81 |
| **Faturamento correto** | **R$ 715.636.164,46** |
| Diferença | R$ 234.681.392,35 |
| Superestimativa | **32,8%** |

### Leitura de negócio

Um detalhe relevante para o time financeiro: **"desconsiderar cancelados" não é a mesma
coisa que "considerar apenas `completed` e `delivered`"**.

Os R$ 94 mi em pedidos `processing` não estão cancelados, mas também não são receita
reconhecida. Quem filtrasse apenas `status NOT IN ('cancelled','refunded')` ainda ficaria
com um número inflado — e é provavelmente daí que vem a divergência entre os relatórios que
circulam hoje: cada área está usando a sua própria definição de "pedido válido".

**Recomendação:** materializar essa regra numa view/tabela canônica (`fct_pedidos_faturados`)
para que ninguém precise reescrever o filtro à mão.

**Ressalva analítica:** o faturamento é notavelmente estável nos 12 meses (R$ 56,7 mi –
R$ 62,5 mi, amplitude de ~10%), sem sazonalidade nem tendência de crescimento — incomum
para um marketplace em expansão. Vale confirmar com o time se a base é uma amostra
sintética ou parcial antes de usar esses números em qualquer projeção.

---

## Desafio 2 — Ranking de crescimento de GMV por seller

> *"Ranking dos 10 sellers com maior crescimento de GMV entre o trimestre atual e o
> anterior, apenas sellers com pelo menos 50 pedidos em ambos os trimestres."*

### O problema escondido: qual é o "trimestre atual"?

A leitura ingênua usaria o trimestre da última data da base — **2024-Q4**. Mas a base
termina em **29/11/2024**: esse trimestre tem só 2 dos 3 meses (R$ 119 mi contra os
~R$ 178 mi de um trimestre cheio).

Medi o efeito de usar o trimestre parcial como referência:

| | Q4 parcial vs Q3 | Q3 vs Q2 (completos) |
|---|---:|---:|
| Sellers no corte de 50 | 43 | 118 |
| Com crescimento positivo | **0** | 70 |
| Melhor resultado | **−2,26%** | +69,35% |

**Achado 6 — comparar contra um trimestre parcial inutiliza a análise.** Nenhum seller
teria crescimento positivo: o "ranking dos 10 com maior crescimento" viraria "os 10 que
menos caíram", e a diretora comercial decidiria sobre um artefato de calendário. O corte de
50 pedidos ainda derrubaria os elegíveis de 118 para 43 — filtrando por falta de dados em
vez de por relevância comercial, o oposto da intenção do filtro.

**Decisão:** o "trimestre atual" é o **último trimestre completo** (2024-Q3), comparado com
2024-Q2. A query detecta isso sozinha — verifica se o último dia do trimestre já ocorreu na
base, sem data cravada no código.

### Demais decisões de modelagem

| Ponto | Decisão | Por quê |
|---|---|---|
| Definição de GMV | `SUM(total_value)` de `completed` + `delivered` | Mantém a definição canônica do Desafio 1; incluir cancelados premiaria quem gera pedido que não se concretiza |
| Trimestre | Derivado do 1º dia do trimestre via `printf`/`DATE` | O SQLite não tem função de trimestre nativa; o primeiro dia permite aritmética direta com `'-3 months'` |
| "50 pedidos" | Contados sobre pedidos faturados | Coerente com o GMV que está sendo medido |
| Divisão por zero | `NULLIF(gmv_anterior, 0)` | O corte de 50 já garante GMV > 0, mas a query não deve depender disso |
| Pivot dos trimestres | `SUM(CASE WHEN ...)` numa passada | Evita dois scans + `JOIN` |

### Resultado — top 10 (2024-Q3 vs 2024-Q2)

| # | seller | UF | GMV Q2 | GMV Q3 | crescimento |
|--:|---|---|---:|---:|---:|
| 1 | Oliveira & Santos Distribuidora | PA | 707.665,01 | 1.198.419,33 | **+69,35%** |
| 2 | Santos & Almeida Suprimentos | MG | 650.216,62 | 1.072.819,57 | +64,99% |
| 3 | Nascimento & Ferreira Grupo | PB | 1.064.281,20 | 1.477.862,38 | +38,86% |
| 4 | Lima & Souza Mercado | MG | 741.504,83 | 1.028.966,47 | +38,77% |
| 5 | Souza & Almeida Suprimentos | RS | 907.742,32 | 1.245.697,34 | +37,23% |
| 6 | Costa & Costa Atacado | SC | 2.419.835,47 | 3.199.599,63 | +32,22% |
| 7 | Oliveira & Rodrigues Grupo | RN | 763.277,94 | 990.224,86 | +29,73% |
| 8 | Silva & Santos Comércio | DF | 818.610,93 | 1.054.629,78 | +28,83% |
| 9 | Ferreira & Oliveira Brasil | ES | 833.586,40 | 1.053.038,74 | +26,33% |
| 10 | Santos & Silva Atacado | BA | 3.076.927,70 | 3.880.342,01 | +26,11% |

### Leitura de negócio

**O filtro de 50 pedidos quase não atua nesta base:** exclui apenas 2 dos 120 sellers, e
ambos por pouco (49 e 45 pedidos). Todos os 120 operam em todos os trimestres — não existe
aqui o cenário de seller novo/inativo que a diretora quis evitar. O filtro continua sendo a
defesa correta para dados reais, mas é importante reportar que **não é ele que está
moldando o ranking**.

**Contexto:** dos 118 elegíveis, 70 cresceram e 48 caíram, média de +2,3%. O top 10 é
performance genuinamente acima da média, não maré geral.

**Decompondo GMV em volume × ticket, aparecem três padrões que pedem ações diferentes:**

1. **Por volume** — *Santos & Almeida* (+63,5% pedidos, ticket estável). Conquista de
   demanda real; o caso mais saudável e replicável.
2. **Por ticket** — *Silva & Santos* (+26,6% ticket, +1,7% pedidos) e *Santos & Silva*
   (+22,6% ticket, +2,8% pedidos). GMV subiu quase sem pedido novo: mix, preço ou poucos
   compradores grandes? Se for concentração, é receita frágil.
3. **Nos dois eixos** — *Oliveira & Santos*, o nº 1 (+27,9% pedidos, +32,4% ticket).

**Ressalva estatística:** 8 dos 10 têm base pequena (50–91 pedidos), onde poucos pedidos
grandes movem muito o percentual. Os únicos com volume relevante são *Costa & Costa*
(176→210) e *Santos & Silva* (212→218) — em valor absoluto adicionaram mais GMV que os
demais.

**Recomendação:** acompanhar o ranking percentual ao lado do **crescimento absoluto de
GMV**. Ordenar só por percentual favorece sistematicamente o seller pequeno e esconde onde
o dinheiro realmente se moveu.

## Desafio 3 — Pedidos com desconto abusivo

> *"Encontre todos os pedidos onde o desconto total representa mais de 40% do valor bruto do
> pedido, listando o seller responsável e a data. Exclua pedidos cancelados."*

### Raciocínio

Três decisões determinam se o resultado está certo:

1. **Qual é o "valor bruto"?** É o preço de tabela **antes** do desconto —
   `SUM(qty * unit_price)`. O caminho errado seria usar `orders.total_value`, que a
   auditoria (Achado 4) mostrou já ser líquido, trocando o denominador de `bruto` por
   `bruto − desconto`.
2. **Em que nível agregar?** O desconto abusivo é do **pedido**, não do item — um item com
   50% dentro de um pedido grande pode ser promoção legítima. Somamos itens por `order_id`
   antes de comparar.
3. **Como comparar sem dividir?** `desconto > 0.40 * bruto` em vez de
   `desconto / bruto > 0.40`: elimina divisão por zero sem `NULLIF` e é mais barato numa
   varredura de 214k linhas.

**Validação do denominador:** usar `total_value` retornaria **1.655** pedidos em vez de
**901** — inflação de **84%**, com 754 pedidos legítimos enviados ao time de fraudes. Num
processo que pode suspender seller, é um erro caro.

### Resultado

**901 pedidos sinalizados**, R$ 6,07 mi de desconto sobre R$ 12,90 mi de valor bruto — e
**todos** vêm de apenas **8 dos 120 sellers**.

| seller | UF | plano | pedidos | sinalizados | taxa | desc. médio pond. |
|---|---|---|---:|---:|---:|---:|
| Oliveira & Santos Distribuidora | PA | basic | 454 | 104 | 22,91% | 24,36% |
| Santos & Almeida Suprimentos | MG | free | 468 | 104 | 22,22% | 24,64% |
| Souza & Almeida Suprimentos | RS | premium | 469 | 96 | 20,47% | 23,56% |
| Santos & Rodrigues Distribuidora | AM | basic | 426 | 87 | 20,42% | 23,53% |
| Souza & Ferreira Alimentos | MG | premium | 433 | 87 | 20,09% | 23,66% |
| Costa & Costa Atacado | SC | free | 1.276 | 254 | 19,91% | 23,37% |
| Oliveira & Costa Comércio | GO | basic | 437 | 86 | 19,68% | 23,87% |
| Costa & Oliveira Atacado | BA | basic | 449 | 83 | 18,49% | 23,08% |

O padrão é **sistemático, não incidental**: cada um sinaliza 18,5%–22,9% dos próprios
pedidos, com desconto médio ponderado de ~23,7% contra **7,5%** dos outros 112 sellers.

### Achado 7 — o desconto tem duas populações, não uma distribuição contínua

Histograma do desconto por item, em faixas de 5 pontos:

| faixa | itens | | faixa | itens |
|---|---:|---|---|---:|
| 0%–5% | 69.737 | | 35%–40% | 1.080 |
| 5%–10% | 69.649 | | 40%–45% | 1.044 |
| 10%–15% | 70.028 | | 45%–50% | 1.051 |
| **15%–35%** | **0** | | 50%–55% | 1.096 |
| | | | 55%–60% | 1.082 |

- **Regime normal:** 0%–15%, uniforme (~69,7 mil itens por faixa)
- **Vazio absoluto:** nenhum item entre 15% e 35%
- **Regime anômalo:** 35%–60%, novamente uniforme (~1,05 mil por faixa)

Não há transição gradual. Não é "alguns sellers dando desconto maior" — são **duas
políticas de precificação distintas** convivendo na plataforma.

### Achado 8 — a suspeita do time de fraudes se confirma

A hipótese tem duas partes: *"descontos abusivos"* (confirmado acima) e *"para inflar
volume artificialmente"*. Cruzando com o ranking do Desafio 2:

| # | seller | UF | crescimento | desconto abusivo |
|--:|---|---|---:|:--:|
| 1 | Oliveira & Santos Distribuidora | PA | +69,35% | **SIM** |
| 2 | Santos & Almeida Suprimentos | MG | +64,99% | **SIM** |
| 3 | Nascimento & Ferreira Grupo | PB | +38,86% | – |
| 4 | Lima & Souza Mercado | MG | +38,77% | – |
| 5 | Souza & Almeida Suprimentos | RS | +37,23% | **SIM** |
| 6 | Costa & Costa Atacado | SC | +32,22% | **SIM** |
| 7–10 | *(demais)* | | | – |

Os 8 sellers sinalizados são **6,7% da base**, mas ocupam **4 das 10 posições** do ranking
— **incluindo 1º e 2º lugar**. Cerca de **6× a presença esperada** se crescimento e desconto
fossem independentes.

O ranking que a diretora comercial usaria para premiar performance está, no topo, medindo
em boa parte **desconto agressivo convertido em GMV**.

### Ressalvas e questionamentos

**1. O limiar de 40% corta no meio da população errada.** O regime anômalo começa em
**35%**, não em 40%. Um corte em 30% (dentro do vazio entre as duas populações) capturaria
**1.564 pedidos** em vez de 901 — os 663 pedidos entre 30% e 40% têm o mesmo comportamento
dos sinalizados, mas escapam da régua atual. **Recomendo revisar o limiar para 30%.**

**2. "Excluir cancelados" merece discussão.** Segui o enunciado à risca (`status <>
'cancelled'`), removendo 91 pedidos. Mas para investigação de fraude, pedido cancelado com
55% de desconto é evidência, não ruído. Sugiro mantê-los numa visão separada.

**3. Desconto abusivo não é necessariamente fraude.** Os dados provam *padrão anômalo e
concentrado*, não intenção. Os 8 estão em planos e estados variados (`free`, `basic`,
`premium`; RS, GO, MG, PA, SC, BA, AM), o que não sugere um segmento com regra comercial
própria — mas a confirmação é do time comercial.

**4. Achado 9 — a análise de margem, que fecharia o caso, não é viável nesta base.**
O passo natural seria checar se o preço com desconto cai abaixo de `products.unit_cost`.
Testei: o markup vai de **−97,5%** a **+23.935%** (média 468%), e **19,2% dos itens já são
vendidos abaixo do custo mesmo sem desconto anômalo**. `unit_price` e `unit_cost` são
independentes entre si — qualquer conclusão sobre margem seria artefato do dado sintético.
Análise registrada como bloqueada até confirmar com o time se `unit_cost` reflete custo
real; se refletir, a plataforma tem um problema de precificação bem maior que o dos
descontos.

### Nota técnica

No SQLite o operador `||` tem precedência **maior** que `*` e `/` — `x * 5 || '...'` é lido
como `x * (5 || '...')`. A query do histograma exige parênteses explícitos; o bug é
silencioso, produz número em vez de texto sem lançar erro.

## Desafio 4 — Produtos que nunca são o item de maior valor

> *"Total de unidades vendidas > 1.000, mas que em nenhum pedido foram o item de maior valor
> unitário. Dica: window functions podem ser suas aliadas e avalie os resultados e
> viabilidade da análise."*

### Primeiro questionamento: o enunciado define "maior" de três formas

| Trecho do enunciado | Métrica implícita |
|---|---|
| *"item mais **vendido** dentro de nenhum pedido"* | maior `qty` |
| *"nunca é o item de **maior valor** num pedido"* | maior `qty * unit_price` |
| *"foram o item de maior **valor unitário**"* | maior `unit_price` |

Adotei `unit_price` (o que consta na especificação formal), mas testei as três — a escolha
não deveria ficar com quem executa a query.

### Decisão técnica: `RANK()`, não `ROW_NUMBER()`

Há **3 pedidos com empate** no item mais caro. Com `ROW_NUMBER()`, um empatado seria
arbitrariamente rebaixado e um produto poderia ser classificado como "nunca foi o mais caro"
por sorteio do motor SQL. `RANK()` dá posição 1 a todos os empatados.

### Resultado: **zero produtos** — e zero é a resposta correta

Um resultado vazio não pode ser entregue sem explicação: ele tanto significa "nada se
encaixa" quanto "a query está errada". Aqui é estrutural, por quatro razões:

**1. O filtro de "> 1.000 unidades" não exclui ninguém.** Todos os 800 produtos passam —
o menos vendido tem **5.322 unidades**, 5× o limiar.

**2. 19,89% dos pedidos têm um único item** (15.915). Neles, o item é o mais caro por
definição. Cada produto cai nessa situação ~20 vezes.

**3. Achado 10 — `unit_price` não é atributo do produto.** Cada produto pratica **267,7
preços distintos em 268 linhas**: um preço diferente a cada venda. **Nenhum** dos 800 tem
preço fixo, e todos percorrem a mesma faixa (~R$ 5 a ~R$ 500). Isso destrói a premissa: a
análise só faz sentido se existir "produto barato" e "produto caro". Aqui não existe produto
barato — existe *venda* barata.

**4. Achado 11 — a variação entre produtos é indistinguível de acaso.**

| | |
|---|---:|
| Desvio-padrão observado da taxa de "ser o mais caro" | **3,01 pp** |
| Desvio-padrão esperado se fosse puro sorteio | **2,95 pp** |

Estatisticamente idênticos: toda a variação entre os 800 produtos é ruído amostral. O
produto menos protagonista ainda lidera **26,85%** das suas aparições. E a probabilidade de
um produto nunca liderar em 268 aparições é da ordem de **10⁻⁵⁴** — não é raro, é
efetivamente impossível.

**As três definições dão zero igualmente**, inclusive descartando pedidos de item único —
a ambiguidade do enunciado não é o que explica o resultado vazio.

### Conclusão e questionamentos ao time

O produto descrito no enunciado **não existe nesta base, e não por coincidência**: ele não
*pode* existir, porque o preço é sorteado por linha em vez de ser característica do produto.

**A pergunta de negócio é legítima** — um produto de giro alto que nunca é o item principal
da compra é o clássico **item complementar**, que entra de carona no carrinho mas não motiva
o pedido. Isso muda decisões reais de precificação e campanha. A pergunta é boa; a base é
que não responde.

**O que eu levaria ao time:**

1. **Qual das três definições de "maior" interessa?** São perguntas de negócio diferentes.
2. **Por que `unit_price` varia a cada linha?** Se é ruído da geração da base, precisamos de
   preço real. Se é genuíno (preço 100% negociado por pedido, sem tabela), então a noção de
   "produto caro" não existe no negócio — e várias análises de catálogo ficam inviáveis,
   não só esta.
3. **O limiar de 1.000 unidades foi calibrado com que base?** Não exclui nenhum produto. Algo
   como o percentil 90 de unidades vendidas seria adaptativo em vez de fixo.
4. **Trocar o critério binário por um contínuo.** "Nunca ser o maior" é frágil — uma única
   aparição em pedido de item único zera o resultado. Um **índice de protagonismo**
   (participação média no valor do pedido, ou taxa de vezes que lidera) ordena o catálogo
   inteiro e sobrevive a exceções pontuais.

---

## Síntese: o que a base revela

Os quatro desafios convergem para uma leitura só. **O Desafio 1** mostrou que o faturamento
circulava 32,8% inflado por falta de uma definição canônica de "pedido válido". **O Desafio
2** mostrou que a janela de comparação, se mal ancorada, transforma artefato de calendário
em queda de 34%. **O Desafio 3** encontrou 8 sellers com política de desconto própria que
ocupam 4 das 10 posições do ranking de crescimento — ou seja, os desafios 2 e 3 medem o
mesmo fenômeno por ângulos diferentes. **O Desafio 4** mostrou que nem toda pergunta bem
formulada é respondível com os dados disponíveis.

Um tema atravessa os quatro: **a régua importa tanto quanto o cálculo**. Status de pedido,
janela temporal, denominador do desconto, definição de "maior" — em todos os casos a
decisão de modelagem mudou o resultado mais do que qualquer detalhe de implementação SQL.
É exatamente o que produz "números diferentes dependendo de quem puxa o relatório".

**Sinais de base sintética** encontrados ao longo do trabalho, que recomendo confirmar antes
de qualquer uso em produção: faturamento sem sazonalidade nem tendência (Desafio 1);
descontos em duas populações uniformes separadas por um vazio absoluto (Achado 7);
`unit_price` e `unit_cost` independentes entre si (Achado 9); e `unit_price` sorteado por
linha em vez de ser atributo do produto (Achado 10).
