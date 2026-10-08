# Fase 1 — Plano de estudo (de ponta a ponta, local)

- **Data:** 2026-10-08
- **Autor:** Pablo Pires
- **Papel do revisor:** ensinar e revisar — o autor escreve todo o código.

## Objetivo da Fase 1

Rodar, na máquina local e de graça, o pipeline inteiro que responde a tese:

> *"Quanto dinheiro parado desde 2019 perdeu em poder de compra, vs. se tivesse sido investido?"*

Entregável final: um **dashboard** com um número chocante + um **README** forte.
Nada de Databricks ainda (isso é Fase 2).

## Como vamos trabalhar

- Programar e enviar o resultado ao final de cada passo (commit + link, ou colar o código).
- Revisão focada em: correção de conceito, boas práticas e o "porquê" — nunca reescrever.
- Regra de ouro: entender antes de digitar. Cada passo tem "o que estudar" antes de codar.

---

## Passo 0 — Setup

**O que estudar:**
- A diferença de papel entre **PySpark** (processamento distribuído/em lote),
  **DuckDB** (OLAP embutido, zero servidor), **dbt** (transformação declarativa +
  testes + documentação) e **Airflow** (orquestração/scheduling).
- Por que os 4 juntos, e quem chama quem.

**O que construir:**
- `venv` + dependências (`pyspark`, `duckdb`, `dbt-duckdb`, `pyarrow`, `requests`).
- Airflow: docker-compose (mais real) ou `pip install apache-airflow` (mais leve).

**Armadilhas:** Spark local conflita com Java; DuckDB e PyArrow não precisam de servidor.

**Critério de revisão:** explicar, em 1 frase cada, o que cada ferramenta faz e por que coexistem.

---

## Passo 1 — Ingestão

**O que estudar:**
- API do **Banco Central (SGS)** e do **Yahoo Finance**.
- O que é **Bronze** (dado cru, imutável, particionado) e por que guardar o raw.

**O que construir:**
- Script que baixa: **IPCA** (mensal), **Selic** (diária/anual), **câmbio USD**,
  e preço de 1–2 benchmarks (ex.: BOVA11, IVVB11).
- Grava em `data/bronze/` como **Parquet particionado**.

**Armadilhas:**
- IPCA publicado com **lag** (~15 dias) — como lidar com o último mês ausente.
- **Idempotência**: rodar 2x não pode duplicar.
- Selic **anual vs. diária** (códigos SGS diferentes). Fator diário:
  `(1 + taxa_aa)^(1/252)` (dias úteis).

**Critério de revisão:** re-baixar dá o mesmo resultado; datas corretas; saber o que cada código SGS significa.

---

## Passo 2 — Bronze → Silver

**O que estudar:**
- O porquê do **Silver**: limpar, tipar, deduplicar, padronizar (moeda, fuso, tipos).
- SparkSession **local** (`.master("local[*]")`) e leitura/escrita de Parquet.

**O que construir:**
- PySpark que lê bronze e grava `data/silver/` com **schemas explícitos**.

**Armadilhas:** não usar Spark "só pra usar Spark" — ter consciência do papel de cada ferramenta.

**Critério de revisão:** schema explícito, dedup funcional, justificar por que Spark (e não DuckDB) aqui.

---

## Passo 3 — dbt

**O que estudar (mais importante da Fase 1):**
- `dbt init`, `profiles.yml` (adapter **duckdb**), `sources`, `seeds`.
- Camadas: **staging** (1:1 com silver) → **intermediate** → **marts** (métrica final).
- **Tests**: `unique`, `not_null`, `accepted_values`.
- **Jinja** básico (`ref`, `source`).

**O que construir (a lógica da tese — pensar, não copiar):**
1. **Índice IPCA** — acumular o IPCA mensal num índice (ex.: 2019 = 100).
2. **Rentabilidade nominal** de cada ativo desde uma data-base.
3. **Rentabilidade real** = nominal deflacionado pelo índice IPCA.
4. **Rentabilidade pós-imposto** — tabela regressiva de IR (22,5% → 20% → 17,5% → 15%, >720 dias).
   Para horizonte longo, simplificar em 15% e **justificar**.
5. **Custo da inação** = (valor investido real/pós-IR) − (valor parado real), em reais de hoje.

**Armadilhas:**
- IPCA lag — a inflação do último mês pode faltar; decidir explicitamente.
- **Dias úteis vs. corridos** na acumulação da Selic.
- Deflacionar corretamente: `valor_real = valor_nominal / (índice_hoje/índice_base)`.

**Critério de revisão:** `dbt test` verde, camadas separadas, explicar a fórmula de trás pra frente.

---

## Passo 4 — Orquestração

**O que estudar:**
- Conceito de **DAG**, `schedule`, `depends_on`, **backfill**.
- Airflow **só orquestra** (não processa dados).

**O que construir:**
- DAG diária: `ingest → spark(silver) → dbt run → export gold → dashboard data`.

**Armadilhas:** Airflow não deve conter lógica de negócio — ele chama os scripts. Cada task idempotente.

**Critério de revisão:** DAG sem estado compartilhado; saber o que é backfill e por que importa.

---

## Passo 5 — Dashboard

**O que estudar:** Streamlit básico + query no DuckDB sobre a gold exportada.

**O que construir:** tela com **o número chocante** + comparação nominal × real × pós-IR.

**Critério de revisão:** o número vem da gold (não hard-coded); mensagem clara em 5 segundos.

---

## Passo 6 — README / documentação

**O que estudar:** o README já tem a base — agora refletir o que de fato funciona.

**O que construir:** arquitetura (ASCII), como rodar do zero, exemplo do número, e os "porquês" das decisões.

---

## Rubrica de revisão

Em cada passo: (1) conceito certo, (2) boas práticas, (3) justificativa das escolhas.
Se algo estiver errado, apontar o problema e fazer a pergunta — o autor corrige.

## Ordem e tempo

Ordem rígida (cada passo depende do anterior). Ritmo sugerido: Passo 0–1 no primeiro
dia; Passo 2–3 no segundo; 4–6 depois. Não pular o Passo 3 — é o que diferencia
Analytics Engineer de "quem roda script".

## Decisões abertas

- [ ] Incluir cripto (CoinGecko) na Fase 1 ou deixar para depois.
- [ ] Dashboard: Streamlit Cloud vs. Looker Studio vs. DuckDB WASM + GitHub Pages.
- [ ] Baseline do "parado": 0% nominal (colchão) vs. poupança (TR + 0,5% a.a.).
