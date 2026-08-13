# Desafio Técnico - Engenharia de Dados

Resolução dos quatro desafios SQL do teste, sobre a base de um marketplace B2B de
alimentos (80 mil pedidos, 214 mil itens).

Todo o trabalho está em **[`desafio_engenharia_dados.ipynb`](desafio_engenharia_dados.ipynb)**:
as queries comentadas, as validações e a análise de negócio junto de cada resultado. O
notebook já está com as saídas salvas, então dá para ler tudo direto no GitHub sem
executar nada.

## Como rodar

**Google Colab:** suba os 6 CSVs da pasta `dados/` para `/content`, descomente a célula
`!pip install -q pandasql` e execute tudo (Ambiente de execução, Executar tudo).

**Local:**

```bash
pip install pandas pandasql matplotlib jupyter
jupyter notebook desafio_engenharia_dados.ipynb
```

O notebook procura os CSVs sozinho em `dados/`, `../dados`, `/content` e
`/content/dados`, então não precisa ajustar caminho. Para executar sem abrir a
interface:

```bash
jupyter nbconvert --to notebook --execute --inplace desafio_engenharia_dados.ipynb
```

Roda inteiro em menos de um minuto.

## Estrutura

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

Stack: `pandas` + `pandasql` (SQLite por cima dos DataFrames), seguindo o notebook
modelo do enunciado.

## O caminho da análise

Antes dos desafios, o notebook faz uma auditoria da base. O que encontrei ali muda a
forma correta de escrever praticamente todas as queries:

- **A base é um snapshot que termina em 29/11/2024.** Janela de "últimos N meses"
  ancorada em `DATE('now')` devolve zero linhas; tudo é ancorado na data máxima dos
  dados (com o ponto de troca para produção comentado).
- **Há 2 pedidos com `id` duplicado** (cara de reingestão). Todas as queries deduplicam
  com `ROW_NUMBER()`, ficando com a versão mais recente.
- **Integridade referencial ok**: nenhuma chave órfã, nenhum pedido sem item.
- **Status de pedido e de pagamento são 100% consistentes**, então filtrar por
  `orders.status` basta para faturamento (verifiquei antes de descartar o join com
  `payments`).
- **`orders.total_value` já é líquido de desconto.** Serve de fonte confiável para
  faturamento e GMV, e é exatamente o campo errado para o desafio dos descontos, que
  pede o valor bruto.

## Resultados em uma linha cada

| Desafio | Resposta | O que apareceu no caminho |
|---|---|---|
| 1 - Faturamento mensal | R$ 715,6 mi em 12 meses (45.191 pedidos) | O relatório sem filtro de status inflava o número em **32,8%**; e "tirar cancelados" não é o mesmo que "só completed/delivered": há R$ 94 mi em `processing` no meio |
| 2 - Crescimento de GMV | Top 10 com líder a **+69,35%** (Q3 vs Q2/2024) | O "trimestre atual" da base é parcial; compará-lo zeraria todo crescimento positivo e inverteria o ranking |
| 3 - Descontos abusivos | **901 pedidos**, todos de **8 dos 120 sellers** | Os descontos têm duas populações separadas por um vazio (nada entre 15% e 35%); 4 dos 8 sellers estão no top 10 do desafio 2, ou seja, o "crescimento" deles é desconto virando GMV |
| 4 - Produto nunca protagonista | **Zero produtos, e zero é a resposta certa** | `unit_price` muda a cada venda (267,7 preços distintos por produto): sem identidade de preço, o produto descrito no enunciado não pode existir nesta base |

As decisões de modelagem (por que competência e não caixa, por que `RANK()` e não
`ROW_NUMBER()`, qual denominador usar no desconto, como detectar o último trimestre
completo sem cravar data) estão discutidas no notebook, junto das queries de validação
que sustentam cada escolha.
