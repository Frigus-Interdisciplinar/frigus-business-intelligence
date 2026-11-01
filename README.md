# Frigus - Business Intelligence

![Databricks](https://img.shields.io/badge/Databricks-FF3621?logo=databricks&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-00ADD8?logo=delta&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?logo=postgresql&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?logo=supabase&logoColor=white)

Pipeline de dados, KPIs e dashboard do Frigus, construídos no Databricks a partir dos dados gerados pela aplicação.

## Visão geral

O repositório reúne a entrega de Business Intelligence do projeto interdisciplinar. Os dados da aplicação (Supabase) são exportados em CSV, ingeridos no Databricks, tratados e transformados em uma base final, que alimenta o dashboard **Frigus - Transações e pagamentos**.

O pipeline roda por um job do Databricks, sem precisar executar as células uma a uma.

## Fluxo dos dados

```
CSV exportado do Supabase
        |
        v
  Volume (entrada)  ->  Raw  ->  Cleaned  ->  BI  ->  Dashboard
```

| Camada | Schema | O que contém |
|---|---|---|
| Entrada | `workspace.frigus_arquivos` (volume `entrada`) | CSVs exportados da aplicação |
| Raw | `workspace.frigus_raw` | Cópia fiel das tabelas em Delta, com a coluna `_ingested_at` |
| Cleaned | `workspace.frigus_cleaned` | Tabelas tratadas, com a coluna `_cleaned_at` |
| BI | `workspace.frigus_bi` | Base final do dashboard e KPIs diários |

## Notebooks

Os notebooks ficam em `notebooks/` e rodam nesta ordem:

| Notebook | O que faz |
|---|---|
| `BI - 00 Preparar Ambiente` | Cria os schemas (`frigus_arquivos`, `frigus_raw`, `frigus_cleaned`, `frigus_bi`) e o volume de entrada |
| `BI - 01 Ingestao e Tratamento` | Lê os CSVs, grava a camada Raw e trata os dados até a camada Cleaned |
| `BI - 02 Base Final e KPIs` | Valida a entrada e gera `transacoes_dashboard` e `kpis_transacoes_diarios` |
| `BI - 03 Preparacao Dashboard e Filtros` | Define a fonte, os filtros e as visualizações do dashboard |
| `Business KPIs - Frigus` | KPIs de negócio de usuários, grupos, planos, produtos, receitas, desperdício e conversas |

### Ingestão e tratamento

As 10 tabelas da aplicação são ingeridas: `transactions`, `plans`, `users`, `subscriptions`, `discard`, `stock_movements`, `products`, `stock_products`, `stocks` e `groups`.

Na ingestão, a tabela `transactions` recebe tipagem em `created_at` (timestamp), `amount` (decimal 10,2) e `attempts` (inteiro). Na passagem de Raw para Cleaned, cada tabela passa por:

1. Remoção de registros integralmente duplicados
2. Identificação de valores nulos por coluna
3. Remoção de espaços nas extremidades dos campos textuais
4. Inclusão da data de tratamento (`_cleaned_at`)
5. Gravação em Delta e validação da contagem de registros

Se alguma tabela falhar, o notebook interrompe a execução com erro, e o job aparece como falho.

### Base final

A tabela `workspace.frigus_bi.transacoes_dashboard` tem uma linha por transação e as colunas:

| Coluna | Descrição |
|---|---|
| `transacao_id` | Identificador da transação |
| `data_transacao` | Data de criação |
| `metodo_pagamento` | Método de pagamento (vazio vira "Nao informado") |
| `status_transacao` | Status da transação (vazio vira "Nao informado") |
| `valor_transacao` | Valor da transação |
| `tentativas` | Número de tentativas de pagamento |
| `teve_retry` | 1 quando houve mais de uma tentativa, 0 caso contrário |
| `faixa_valor` | Faixa do valor: negativo, 0 a 20, 20 a 50, 50 a 100 e a partir de 100 |

A tabela `workspace.frigus_bi.kpis_transacoes_diarios` agrega por dia, método e status: quantidade de transações, valor total, valor médio e transações com retry.

Antes de gerar a base, o notebook confere que a origem não está vazia, que não há `id`, data ou valor nulos e que não há IDs duplicados. Depois, confere que a contagem da base final é igual à da origem.

## Dashboard

**Frigus - Transações e pagamentos**, construído sobre `transacoes_dashboard`.

- **Cards:** total de transações, valor total e valor médio por transação
- **Filtros:** período (data da transação) e método de pagamento
- **Linha:** evolução mensal das transações
- **Barras:** transações por método de pagamento, por status e por faixa de valor
- **Boxplot:** distribuição dos valores por método de pagamento

Todas as visualizações usam o mesmo dataset (`Transacoes Frigus`) e respondem aos dois filtros.

A versão exportada do dashboard está em [`docs/dashboard-transacoes-pagamentos.pdf`](docs/dashboard-transacoes-pagamentos.pdf).

## KPIs de negócio

O notebook `Business KPIs - Frigus` calcula:

| Área | KPIs |
|---|---|
| Usuários e grupos | Total de usuários ativos, usuários por grupo, média de usuários por grupo, usuários por plano |
| Transações | Método de pagamento mais usado, valor médio, taxa de retry |
| Produtos e compras | Produtos mais presentes no estoque e mais colocados na lista de compras |
| Receitas | Receitas mais sugeridas, a partir dos ingredientes das listas de compras |
| Desperdício | Quantidade perdida, custo das perdas, taxa de perda e motivos de descarte |
| Conversas | Média de conversas por grupo e conversas por tipo |

Este notebook usa os schemas `frigus.data_engineering_bronze`, `frigus.data_engineering_silver` e `frigus.data_engineering_metrics`, e não os do pipeline de transações.

## Job do Databricks

O job **BI - Pipeline Dashboard Frigus** roda em compute serverless e tem duas tarefas encadeadas:

| Tarefa | Notebook |
|---|---|
| `01_ingestao_tratamento` | `BI - 01 Ingestao e Tratamento` |
| `02_base_final_kpis` | `BI - 02 Base Final e KPIs` |

A segunda tarefa só roda se a primeira terminar com sucesso. Os notebooks `00` (preparação do ambiente) e `03` (documentação do dashboard) não fazem parte do job.

A execução é manual, pelo botão **Run now**, sem agendamento. A execução registrada terminou com status `Succeeded` e produziu 7.768 transações na base final. As evidências (configuração, histórico de execução, status e resultado gerado) estão no [relatório gerencial](docs/relatorio-gerencial.pdf).

## Vídeo de demonstração

O vídeo mostra o processo de BI do Frigus no Databricks, em três partes:

1. Configuração do pipeline, com as duas etapas: ingestão e tratamento dos dados, e geração da base final com o cálculo dos KPIs.
2. Execução do job e as tabelas geradas.
3. Dashboard de transações e pagamentos, com os indicadores, os gráficos e os filtros interativos que atualizam tudo conforme a seleção.

[Assistir à demonstração do pipeline e do dashboard](https://drive.google.com/file/d/1WXP4TanvOyTe5_Z13qDQFGzY3yky_C-y/view?usp=sharing)

## Pastas

```
.
├── notebooks/
│   ├── BI - 00 Preparar Ambiente.ipynb
│   ├── BI - 01 Ingestao e Tratamento.ipynb
│   ├── BI - 02 Base Final e KPIs.ipynb
│   ├── BI - 03 Preparacao Dashboard e Filtros.ipynb
│   └── Business KPIs - Frigus.ipynb
├── docs/
│   ├── dashboard-transacoes-pagamentos.pdf
│   └── relatorio-gerencial.pdf
├── LICENSE
└── README.md
```

## Como executar

1. No Databricks, importe os notebooks de `notebooks/` para o workspace.
2. Rode `BI - 00 Preparar Ambiente` para criar os schemas e o volume.
3. Exporte as 10 tabelas do Supabase em CSV e envie os arquivos para o volume `/Volumes/workspace/frigus_arquivos/entrada`, com o nome `<tabela>.csv`.
4. Execute o job **BI - Pipeline Dashboard Frigus** com **Run now**, ou rode os notebooks `01` e `02` na ordem.
5. Abra o dashboard e use os filtros de período e método de pagamento.

## Limites

- A fonte é um CSV exportado, então o pipeline não acompanha a aplicação em tempo real.
- Cada execução sobrescreve as camadas Raw, Cleaned e BI.
- O valor das transações soma todos os status. A receita aprovada depende de confirmar quais status representam pagamento concluído.
- O tratamento remove só duplicatas integrais. Duplicatas por `id` fazem o notebook `02` parar com erro.

## Relatório gerencial

O relatório com a apresentação da base, o problema analisado, as estratégias de tratamento, os KPIs e o fluxo do pipeline está em [`docs/relatorio-gerencial.pdf`](docs/relatorio-gerencial.pdf).

## Licença

MIT. Veja o arquivo [LICENSE](LICENSE).
