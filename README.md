# realreturns

> **R$ 100 mil parados desde 2019 não são R$ 100 mil — são ~R$ 62 mil de poder de compra.**
> Este projeto prova isso, com dados oficiais do Banco Central, em reais, corrigido por inflação e imposto.

`realreturns` é um portfólio de **Data Engineering** e **Analytics Engineering** com uma tese
clara: a maioria dos dashboards mostra o número bruto (nominal). Eu mostro a **verdade em BRL** —
**nominal vs. real (IPCA) vs. pós-imposto** — e calculo o **custo da inação**: quanto o dinheiro
parado perde em poder de compra ao longo do tempo.

Não é um projeto genérico. São três entregáveis que contam a mesma história:

1. **Pipeline de dados** — medallion (bronze → silver → gold) sobre APIs públicas reais.
2. **Um mini-framework declarativo** ("cookbook") que gera Databricks Asset Bundles a partir de YAML.
3. **Modelagem dbt** com dois adapters (`duckdb` local + `databricks` prod) — o mesmo SQL, zero mudança.

## Arquitetura

```
APIs públicas (BCB/SGS, Yahoo Finance, CoinGecko)
        │  ingestão (Python)
        ▼
BRONZE  — dados crus (parquet)                    → PySpark
        │  limpeza, tipos, dedup, moeda
        ▼
SILVER  — dados tipados e normalizados            → PySpark
        │  modelagem de negócio, testes
        ▼
GOLD    — real vs nominal vs pós-imposto          → dbt
        │
        ▼
Serving — DuckDB (query) → dashboard (Streamlit)
```

## Stack

`Python` · `PySpark` · `Databricks (DAB)` · `dbt (duckdb + dbt-spark)` · `DuckDB` ·
`Airflow` · `Streamlit` · `Pydantic` · `Jinja2`

## Estrutura

```
realreturns/
├── cookbook/          # mini-framework (builder + jobs)
├── src/notebooks/     # PySpark (bronze/silver)
├── dbt/               # projeto dbt (profiles: duckdb + databricks)
├── dags/              # Airflow
├── dashboard/         # Streamlit
└── resources/         # gerado pelo builder (gitignored)
```

## Roadmap

- [ ] **Fase 1** — pipeline completo local (Airflow + Spark local + DuckDB + dbt + dashboard).
- [ ] **Fase 2** — builder do cookbook gerando DAB + deploy no Databricks.
- [ ] **Fase 3** — framework completo (views, alerts, validação) + post público.


