# realreturns — Design

- **Data:** 2026-10-08
- **Autor:** Pablo Pires
- **Status:** validado (brainstorm) — pronto para implementação (Fase 1)

## 1. Objetivo

Portfólio público que me diferencie no mercado de **Data Engineering** e
**Analytics Engineering** (foco em vagas internacionais). Não é "mais um
dashboard de ação/cripto": é um projeto com opinião, dados reais e uma
arquitetura que cobre o espectro completo — de infra declarativa (Spark/DAB)
até modelagem de negócio (dbt) — com um gancho forte o suficiente para gerar
buzz.

## 2. Público e mensagem

- **Recrutadores de DE:** queremos mostrar arquitetura de medallion, PySpark,
  Databricks Asset Bundles e uma abstração declarativa própria ("cookbook").
- **Recrutadores de Analytics:** queremos mostrar dbt, modelagem dimensional,
  testes, documentação e visão de negócio.
- **Mensagem central:** "a verdade em BRL" — rentabilidade **nominal vs real
  (IPCA) vs pós-imposto** — e o **custo da inação** (quanto dinheiro parado
  perde em poder de compra).

## 3. A tese (o gancho)

Todo mundo tem dinheiro parado. Deixá-lo parado não é neutro — é uma perda
real, em reais, corrigida pela inflação e pelo imposto. O projeto prova isso
com dados oficiais e produz um número chocante e compartilhável.

## 4. Arquitetura

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

- **Orquestração:** Airflow (local via docker-compose), DAG diária:
  `ingest → bronze → silver → dbt run → serve`.
- **Modelagem:** dbt com **dois adapters** — `duckdb` (dev, custo zero) e
  `databricks` (`dbt-spark`, prod/estudo). Mesmo SQL, zero mudança.

## 5. Fontes de dados

| Fonte | Dados |
|---|---|
| Banco Central (SGS) | Selic, IPCA, câmbio (USD/EUR) |
| Yahoo Finance | preços históricos/atuais (BOVA11, IVVB11, benchmarks) |
| CoinGecko | BTC/ETH (opcional) |

## 6. Entregáveis

1. **Pipeline financeiro** — o produto de dados (tese "custo da inação").
2. **"Cookbook" próprio** — builder declarativo (YAML → DAB), versão minha de
   um padrão que uso na empresa, escrito do zero.
3. **Projeto dbt dual-adapter** — a modelagem (duckdb + databricks).

## 7. O "cookbook" (builder declarativo)

Layout:

```
cookbook/{job}/
├── definition.yml      → nome, cron, cluster, tags, alertas
├── recipe/*.yaml       → tabela medallion → task (notebook compartilhado)
├── task/*.yaml         → notebook seu → task customizada (depends_on)
└── (fase 2: view/, metric_view/, alerts/)
```

O builder (Python) lê, valida e gera `resources/{job}.yml` (nunca editado à
mão). `make build_pipelines` dispara.

Decisões de design (já visando o framework completo):
- **Pydantic** para validação de schema dos YAMLs.
- **Jinja2** para gerar os recursos.
- `depends_on` no schema desde o início.

> Importante: implementação **própria**, do zero, apenas inspirada no conceito
> (sem copiar código interno).

## 8. Estrutura do repositório

```
realreturns/
├── cookbook/          # mini-framework (builder + jobs)
├── src/notebooks/     # PySpark (silver etc.)
├── dbt/               # projeto dbt (profiles: duckdb + databricks)
├── dags/              # Airflow
├── dashboard/         # Streamlit (deploy grátis)
├── resources/         # gerado pelo builder (gitignored)
├── README.md
└── docs/plans/        # design doc
```

## 9. Roadmap

- **Fase 1 (1–2 semanas)** — pipeline completo **local**: Airflow + Spark
  local + DuckDB + dbt + dashboard "custo da inação". README forte.
- **Fase 2** — builder do cookbook gerando DAB + profile `dbt-spark` +
  deploy no Databricks (community/empresa).
- **Fase 3** — framework completo (views, alerts, validação) + design doc
  público (post LinkedIn/Medium).

## 10. Stack

Python · PySpark · Databricks (DAB) · dbt (duckdb + dbt-spark) · DuckDB ·
Airflow · Streamlit · Pydantic · Jinja2.

## 11. Decisões abertas

- [ ] E-mail do git local (trocar o corporativo pelo pessoal no repo público).
- [ ] Incluir lado cripto (CoinGecko) na Fase 1 ou deixar para depois.
- [ ] Dashboard: Streamlit Cloud vs. Looker Studio vs. DuckDB WASM + GitHub Pages.
