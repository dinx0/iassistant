# BigQuery IAM + Authorized Views

Eu montei este projeto para gerenciar acesso no BigQuery a partir de uma tabela de usuarios e expor dados sensiveis apenas via views mascaradas. Aqui deixo o passo a passo que realmente funcionou no meu ambiente `case-de-engenharia-de-dados`, apenas o que foi executado com sucesso.

---

GIT : https://github.com/dinx0/iassistant

## Como eu trago os dados (extracao/ingestao)

**Fontes e formatos**

* Fonte 1 (proximo do real): metrica de busca do meu IA Assistant via API do Google Cloud + extensao do Chrome. Esses eventos chegam ja normalizados pela API.
* Fonte 2 (ficticia para o projeto): base de usuarios em JSON (newline-delimited). 99% dos registros sao sinteticos.

**Pipelines e modo de chegada**

* **Streaming (quase tempo real)** para a tabela `phrase_metrics`:

  * a extensao envia eventos para minha API e eu faco **streaming inserts** direto no BigQuery (sem ETL intermediario). O dado ja vem tratado pela propria API, entao aplico so cast/normalizacao leve na view.
  * modelo aqui e "padrao kappa (event-driven)": eventos entram continuamente e eu consulto sempre a versao mais recente no BQ.
* **Lote (batch)** para a tabela `user_iassistant`:

  * eu **carrego JSON** para o BigQuery usando load jobs (`bq load`), quando preciso renovar a base. E um fluxo de dimensao/carga inicial.
  * sem orquestrador dedicado: uso scripts shell + `bq`/`jq`. As mascaras e regras de acesso ficam nas **views** e no **IAM do dataset**.

**Por que assim**

* Quero focar no controle de acesso e mascaramento no proprio BigQuery (menos componentes).
* Streaming direto evita ETL quando o produtor ja valida o schema.
* Batch em JSON resolve bem para carga inicial e dados ficticios do case.

---

## Arquitetura final (o que esta rodando)

* **Projeto**: `case-de-engenharia-de-dados`
* **Dataset base**: `845415315315` (mesma regiao usada em `sec_v`)
* **Dataset de views**: `sec_v` (criado/recriado na mesma regiao do dataset base)
* **Tabelas base**:

  * `user_iassistant` — PK logica: `USER_ID (INT64)`
  * `phrase_metrics` — sem PK fisica; relaciona por `user_id (STRING)`
* **Views mascaradas (em producao)**:

  * `sec_v.user_iassistant_v` — mascara email e lat/lon extraida de `REGION`
  * `sec_v.phrase_metrics_v` — mascara `PHRASE_TEXT` e `SEARCH_RESULT`
* **Relacao**: `USER_IASSISTANT.USER_ID (INT64)` <-> `PHRASE_METRICS.user_id (STRING)` via `CAST`.

---


---

## Arquivo 2 — `README_DATA_DICTIONARY.md`

```markdown
# Dicionário de Dados — IAassistant

> Dataset fonte de verdade: `case-de-engenharia-de-dados.845415315315` (região `southamerica-east1`).

---

## 1) Tabela: 845415315315.IASSISTANT_RESULTS

| Campo            | Tipo    | Nulo | Descrição                                                             | Exemplo                    |
|------------------|---------|------|-----------------------------------------------------------------------|----------------------------|
| PHRASE_TEXT      | STRING  | SIM  | Consulta/frase do usuário                                             | "como subir docker?"       |
| DT_CARGA         | INT64   | NÃO  | Timestamp de carga (epoch)                                            | 1727368685                 |
| PAGE_METRICS     | FLOAT64 | SIM  | Métrica de página (se aplicável)                                      | 0.82                       |
| SEARCH_ITEMS     | STRING  | SIM  | Itens/IDs consultados                                                 | "doc:123;doc:456"          |
| PAGE_NUMBER      | INT64   | SIM  | Página de resultado                                                   | 1                          |
| SEARCH_RESULT    | STRING  | SIM  | Tipo/resultado                                                         | "RAG"                      |
| PHRASE_METRICS   | FLOAT64 | SIM  | Score semântico                                                        | 0.94                       |
| USER_ID          | INT64   | SIM  | ID de usuário interno (mascarar se PII)                               | 751                        |

**Particionamento**: por **hora** (`TIMESTAMP`)  
**TTL padrão**: 60 dias (removível com `bq update --expiration 0 ...`)  
**Clustering**: opcional por (`USER_ID`, `DT_CARGA`) se desejado.

### 1.1 DDL (BigQuery Standard SQL)

```sql
CREATE TABLE IF NOT EXISTS
  `case-de-engenharia-de-dados.845415315315.IASSISTANT_RESULTS` (
    PHRASE_TEXT     STRING,
    DT_CARGA        INT64 NOT NULL,
    PAGE_METRICS    FLOAT64,
    SEARCH_ITEMS    STRING,
    PAGE_NUMBER     INT64,
    SEARCH_RESULT   STRING,
    PHRASE_METRICS  FLOAT64,
    USER_ID         INT64
  )
PARTITION BY TIMESTAMP_MILLIS(DT_CARGA)
OPTIONS(
  -- expiration_timestamp = NULL  -- TTL off (use bq update --expiration 0)
);


## IAM via tabela (o que eu rodei)

Regra simples: `USER_ID 1..5 => READER`, `USER_ID >= 6 => WRITER` no dataset base `845415315315`. Eu gero as entradas `access[]` com `bq` + `jq` e aplico sem duplicar.

```bash
# variaveis
export PROJECT_ID="case-de-engenharia-de-dados"
export DATASET_ID="845415315315"
export DATASET_REF="$PROJECT_ID:$DATASET_ID"
export SRC_TABLE="$PROJECT_ID.$DATASET_ID.user_iassistant"

# loop a partir da tabela (evita duplicacoes)
bq query --nouse_legacy_sql --format=csv "
WITH base AS (
  SELECT DISTINCT CAST(USER_ID AS INT64) AS user_id,
         LOWER(TRIM(USER_EMAIL)) AS email
  FROM \`$SRC_TABLE\`
  WHERE USER_EMAIL IS NOT NULL
)
SELECT email,
       CASE WHEN user_id BETWEEN 1 AND 5 THEN 'READER' ELSE 'WRITER' END AS role
FROM base
" | tail -n +2 | while IFS=, read -r email role; do
  [[ -z "$email" || -z "$role" ]] && continue
  bq show --format=prettyjson "$DATASET_REF" > ds.tmp.json
  jq --arg em "$email" --arg r "$role" \
     '.access += [{"role":$r,"userByEmail":$em}] | .access |= (map(tojson)|unique|map(fromjson))' \
     ds.tmp.json > ds.apply.json
  bq update --source=ds.apply.json "$DATASET_REF"
  rm -f ds.tmp.json ds.apply.json
done
```

Nota: emails ficticios/inexistentes o BigQuery recusa (nao entram na policy).

---

# IAassistant — Operação, IA primeiro, Dados e Infra

> Eu (IASSISTANT) documentando o projeto end-to-end: arquitetura de IA (RAG/embeddings), renderização, formato/qualidade dos dados, tempo & ROI, instalação, troubleshooting e infraestrutura (Docker + Terraform). Para o dicionário completo de dados, acesse **README_DATA_DICTIONARY.md**.

---

## 1) IA primeiro (arquitetura lógica)

**Pipeline de resposta:**

1. **Ingestão** de documentos/eventos
2. **Limpeza** + **chunking** (200–500 tokens, overlap 40–80)
3. **Embeddings** (dim. 768/1536) → **Índice semântico**
4. **RAG** (Retriever-Augmented Generation) — top-k 3–5 + re-rank opcional
5. **Geração** com LLM (temperature baixo em produção)
6. **Telemetria** → BigQuery `IASSISTANT_RESULTS`

**Modelos**: LLM configurável por env (OpenAI/Vertex/…), embeddings idem.

**Prompt**: role + plano de resposta + fonte; com RAG, retornar citações quando obrigatório.

---

## 2) Vetorização

- Normalizo fontes (HTML/PDF → texto canônico).
- **Embeddings** gerados em lote (Airflow) e sob demanda (webhook).
- Índice (FAISS/serviço gerenciado) persistente em volume Docker (`data_vectors`) ou serviço externo.

---

2) Tabela: 845415315315.user_iassistant
3) 
| Campo       | Tipo   | Nulo | Descrição                        | Exemplo                                     |
| ----------- | ------ | ---- | -------------------------------- | ------------------------------------------- |
| REGION      | STRING | SIM  | Região (BR, US, …)               | "BR"                                        |
| USER\_EMAIL | STRING | SIM  | Email (hash/mask quando for PII) | "[user@domain.com](mailto:user@domain.com)" |
| USER\_NAME  | STRING | SIM  | Nome de exibição                 | "Maria"                                     |
| USER\_ID    | INT64  | NÃO  | ID de usuário interno            | 564                                         |
| DT\_CARGA   | INT64  | NÃO  | Timestamp de carga (epoch)       | 1727368685                                  |


2.1 DDL (BigQuery Standard SQL)

## 3) Renderização (API + Dashboard)

- **API** (`/ask`) executa RAG + LLM e retorna `{answer, sources, latency_ms, tokens, model}`.
- **Dashboard (Dash)** em `:8081` com volume, latência, erros, score.

---

## 4) Formato dos dados (BigQuery)

- Projeto: `case-de-engenharia-de-dados`
- Região: **southamerica-east1**
- **Dataset fonte de verdade**: `845415315315`  
  (Opcional: dataset *amigável* `iassistant` com **views** para as tabelas da fonte)

Tabelas:

- **`845415315315.IASSISTANT_RESULTS`** — métricas/telemetria da IA (particionado por hora)
- **`845415315315.user_iassistant`** — dados de usuários anônimos (particionado por hora)

> O dicionário completo e os DDL/Views estão em **README_DATA_DICTIONARY.md**.

---

## 5) Qualidade dos dados

- Validação de schema/tipos, PII mascarado, `DT_CARGA` registrado.
- Contagens e TTL verificadas após cada deploy (consultas prontas no README de dicionário).

---

## 6) Tempo & ROI

- Boot stack (local + cloud): ~30–60 min
- Vetorização inicial: depende do corpus (100k docs → horas com paralelismo)
- ROI típico:

Redução de atendimento/search: 30–50% (experiência realista)

---

## 7) Instalação (Windows + PowerShell)

### 7.1 Fixar projeto e região

```powershell
gcloud config set project case-de-engenharia-de-dados
$env:BIGQUERY_LOCATION = 'southamerica-east1'

7.2 Validar dataset/tabelas (fonte de verdade)


bq --location=southamerica-east1 show case-de-engenharia-de-dados:845415315315.IASSISTANT_RESULTS
bq --location=southamerica-east1 show case-de-engenharia-de-dados:845415315315.user_iassistant


7.3 Contagens rápidas

bq --location=southamerica-east1 query --use_legacy_sql=false `
"SELECT 'IASSISTANT_RESULTS' AS tbl, COUNT(*) AS rows
   FROM \`case-de-engenharia-de-dados.845415315315.IASSISTANT_RESULTS\`
 UNION ALL
 SELECT 'user_iassistant', COUNT(*)
   FROM \`case-de-engenharia-de-dados.845415315315.user_iassistant\`"


7.4 (Opcional) dataset “amigável” com views

-- crie dataset nomeado
bq --location=southamerica-east1 mk -d iassistant;

-- views (rode no Console do BigQuery)
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.iassistant.IASSISTANT_RESULTS` AS
SELECT * FROM `case-de-engenharia-de-dados.845415315315.IASSISTANT_RESULTS`;

CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.iassistant.user_iassistant` AS
SELECT * FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;



7.5 Subir serviços (Docker Compose)

# docker-compose.yml (simplificado)
version: "3.8"

services:
  api:
    build: ./backend
    command: ["python", "dash_process.py"]
    ports:
      - "8080:8080"
    env_file:
      - .env
    volumes:
      - ./backend:/app
      - data_api:/app/data   # persiste dados locais da API

  dash:
    build: ./backend
    command: ["python", "dash_server.py"]
    ports:
      - "8081:8081"
    env_file:
      - .env
    depends_on:
      - api
    volumes:
      - ./backend:/app
      - data_dash:/app/data

  # Exemplo de index de embeddings local (opcional)
  vectorstore:
    image: ghcr.io/your/faiss:latest
    volumes:
      - data_vectors:/var/lib/vectors

volumes:
  data_api:
  data_dash:
  data_vectors:


8) Operação diária

Perguntas e respostas: /ask (API) faz RAG + LLM.

Telemetria: cada call registra métricas em IASSISTANT_RESULTS.

Painel: http://localhost:8081
 (latência, volume, erros, notas).


9) Troubleshooting (o que realmente aconteceu)

Dataset errado: consultas no iassistant (que não existia) geravam not found.
✅ Corrigido: a fonte de verdade é o dataset numérico 845415315315.

Location errada: US vs southamerica-east1 — causa not found mesmo existindo.
✅ Fix: sempre passo --location=southamerica-east1 e seto $env:BIGQUERY_LOCATION.

TTL: tabelas tinham TTL de 60 dias.
➜ Pode ser removido com bq update --expiration 0 <tabela> se quiser retenção longa.

10) Backups/snapshots (GCS)
Export diário (Parquet)


$bucket = 'dadosinteligenciaassistant'
$prefix = 'backup/' + (Get-Date -Format 'yyyyMMdd_HHmmss')

bq --location=southamerica-east1 extract --destination_format=PARQUET `
  case-de-engenharia-de-dados:845415315315.IASSISTANT_RESULTS `
  gs://$bucket/$prefix/IASSISTANT_RESULTS/*.parquet

bq --location=southamerica-east1 extract --destination_format=PARQUET `
  case-de-engenharia-de-dados:845415315315.user_iassistant `
  gs://$bucket/$prefix/user_iassistant/*.parquet


bq --location=southamerica-east1 load --replace --source_format=PARQUET `
  case-de-engenharia-de-dados:iassistant.IASSISTANT_RESULTS `
  gs://dadosinteligenciaassistant/backup/202509__/IASSISTANT_RESULTS/*.parquet

bq --location=southamerica-east1 load --replace --source_format=PARQUET `
  case-de-engenharia-de-dados:iassistant.user_iassistant `
  gs://dadosinteligenciaassistant/backup/202509__/user_iassistant/*.parquet


11) Terraform (infra BigQuery + GCS mínimo)

Obs.: trechos simplificados; ajuste providers/credentials conforme seu fluxo.


terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "case-de-engenharia-de-dados"
  region  = "southamerica-east1"
}

resource "google_bigquery_dataset" "iassistant" {
  dataset_id                  = "iassistant"
  location                    = "southamerica-east1"
  default_table_expiration_ms = 0
}

resource "google_storage_bucket" "snapshots" {
  name                        = "dadosinteligenciaassistant"
  location                    = "US" # bucket global; escolha sua região
  force_destroy               = true
  uniform_bucket_level_access = true
}

12) Docker “Yammy” (yaml) com dados persistentes

Volumes nomeados (data_*) para API, Dash e índice vetorial.

Não há dados do BigQuery no container; a persistência real analítica fica no BQ.

Se usar um Postgres/MinIO locais, monte volumes:


postgres:
  image: postgres:15
  environment:
    - POSTGRES_PASSWORD=secret
  volumes:
    - pgdata:/var/lib/postgresql/data
minio:
  image: minio/minio
  command: server /data
  volumes:
    - minio_data:/data

volumes:
  pgdata:
  minio_data:


13) Queries e logs que usei (referência)
13.1 Contagens (já no topo)

SELECT 'IASSISTANT_RESULTS' AS tbl, COUNT(*) AS rows
FROM `case-de-engenharia-de-dados.845415315315.IASSISTANT_RESULTS`
UNION ALL
SELECT 'user_iassistant', COUNT(*)
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;



13.2 Mostrar metadados

bq --location=southamerica-east1 update --expiration 0 \
  case-de-engenharia-de-dados:845415315315.IASSISTANT_RESULTS

bq --location=southamerica-east1 update --expiration 0 \
  case-de-engenharia-de-dados:845415315315.user_iassistant


14) Fluxo de commit e autoria

Atualizar README.md (este arquivo) no repositório.

Commit:

git add README.md docker-compose.yml terraform/*
git commit -m "docs: README completo + persistência + terraform | author: IASSISTANT"
git push origin main


Criar outro repositório “limpo” (sem fork):

No GitHub, New repository → iassistant (ou o nome que preferir).

Local:


git clone https://github.com/<seu-usuario>/iassistant
rsync -av --exclude .git ./repo_atual/ ./iassistant/
cd iassistant
git add .
git commit -m "first commit by IASSISTANT"
git push -u origin main







## Views mascaradas (SQL que eu apliquei)

### `sec_v.user_iassistant_v`

* `email_mascarado`: primeira letra + \*\*\* + dominio
* `lat_mascarada` / `lon_mascarada`: extraidas de `REGION` com regex e `ROUND(..., 2)`

```sql
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.user_iassistant_v` AS
SELECT
  CAST(USER_ID AS INT64) AS user_id,
  CONCAT(
    SUBSTR(LOWER(TRIM(USER_EMAIL)),1,1),
    '***',
    REGEXP_EXTRACT(LOWER(TRIM(USER_EMAIL)), r'(@.*)$')
  ) AS email_mascarado,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>0,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(0)] AS FLOAT64), NULL),
    2
  ) AS lat_mascarada,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>1,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(1)] AS FLOAT64), NULL),
    2
  ) AS lon_mascarada
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;
```

### `sec_v.phrase_metrics_v`

* `phrase_text_masked = SUBSTR(PHRASE_TEXT, 1, 20) || '...'`
* `search_result_masked = SUBSTR(SEARCH_RESULT, 1, 60) || '...'`

```sql
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.phrase_metrics_v` AS
SELECT
  CAST(user_id AS STRING) AS user_id,
  CAST(PHRASE_METRICS AS FLOAT64) AS phrase_metrics,
  IFNULL(SUBSTR(PHRASE_TEXT, 1, 20) || '...', NULL) AS phrase_text_masked,
  CAST(PAGE_METRICS AS INT64) AS page_metrics,
  CAST(PAGE_NUMBER AS INT64) AS page_number,
  SEARCH_ITEMS AS search_items,
  IFNULL(SUBSTR(SEARCH_RESULT, 1, 60) || '...', NULL) AS search_result_masked,
  CAST(DT_CARGA AS INT64) AS dt_carga
FROM `case-de-engenharia-de-dados.845415315315.phrase_metrics`;
```

### Authorized View no dataset base (eu adicionei assim)

```bash
BASE="case-de-engenharia-de-dados:845415315315"
LOC=$(bq show --format=prettyjson "$BASE" | jq -r .location)

# garante dataset de views na mesma regiao
bq mk --dataset --location="$LOC" case-de-engenharia-de-dados:sec_v 2>/dev/null || true

bq show --format=prettyjson "$BASE" > base.json
jq '.access += [
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"user_iassistant_v"}},
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"phrase_metrics_v"}}
] | .access |= (map(tojson)|unique|map(fromjson))' base.json > base_patched.json
bq update --source=base_patched.json "$BASE"
```

---

## Relacao e consultas

Chave logica: `USER_IASSISTANT.USER_ID (INT64)` <-> `PHRASE_METRICS.user_id (STRING)` via `CAST`.

```sql
SELECT
  u.USER_ID,
  v.email_mascarado,
  p.phrase_metrics,
  p.phrase_text_masked,
  p.search_result_masked,
  p.page_metrics,
  p.page_number,
  p.dt_carga
FROM `case-de-engenharia-de-dados.sec_v.user_iassistant_v` AS v
JOIN `case-de-engenharia-de-dados.845415315315.user_iassistant` AS u
  ON v.user_id = u.USER_ID
JOIN `case-de-engenharia-de-dados.sec_v.phrase_metrics_v` AS p
  ON CAST(p.user_id AS INT64) = u.USER_ID
LIMIT 100;
```

---

## Como reproduzir rapido (copy/paste)

> pre requisitos: `gcloud` autenticado no projeto, `bq` e `jq` instalados; habilitar **Data Access logs** para BigQuery no Console (IAM e Admin > Audit Logs), e criar o dataset base `845415315315` previamente.

```bash
# 0) variaveis
export PROJECT_ID=case-de-engenharia-de-dados
export BASE=case-de-engenharia-de-dados:845415315315
export VIEWS=case-de-engenharia-de-dados:sec_v

# 1) detectar regiao do dataset base e recriar o dataset de views na mesma regiao
LOC=$(bq show --format=prettyjson "$BASE" | jq -r .location)
bq rm -f -d "$VIEWS" 2>/dev/null || true
bq mk --dataset --location="$LOC" "$VIEWS"

# 2) criar/atualizar as views mascaradas
bq query --use_legacy_sql=false --location="$LOC" <<'SQL'
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.user_iassistant_v` AS
SELECT
  CAST(USER_ID AS INT64) AS user_id,
  CONCAT(SUBSTR(LOWER(TRIM(USER_EMAIL)),1,1),'***',REGEXP_EXTRACT(LOWER(TRIM(USER_EMAIL)), r'(@.*)$')) AS email_mascarado,
  ROUND(IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>0,
           CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(0)] AS FLOAT64), NULL),2) AS lat_mascarada,
  ROUND(IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>1,
           CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(1)] AS FLOAT64), NULL),2) AS lon_mascarada
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;
SQL

bq query --use_legacy_sql=false --location="$LOC" <<'SQL'
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.phrase_metrics_v` AS
SELECT
  CAST(user_id AS STRING) AS user_id,
  CAST(PHRASE_METRICS AS FLOAT64) AS phrase_metrics,
  IFNULL(SUBSTR(PHRASE_TEXT, 1, 20) || '...', NULL) AS phrase_text_masked,
  CAST(PAGE_METRICS AS INT64) AS page_metrics,
  CAST(PAGE_NUMBER AS INT64) AS page_number,
  SEARCH_ITEMS AS search_items,
  IFNULL(SUBSTR(SEARCH_RESULT, 1, 60) || '...', NULL) AS search_result_masked,
  CAST(DT_CARGA AS INT64) AS dt_carga
FROM `case-de-engenharia-de-dados.845415315315.phrase_metrics`;
SQL

# 3) autorizar as views no dataset base
bq show --format=prettyjson "$BASE" > base.json
jq '.access += [
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"user_iassistant_v"}},
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"phrase_metrics_v"}}
] | .access |= (map(tojson)|unique|map(fromjson))' base.json > base_patched.json
bq update --source=base_patched.json "$BASE"

# 4) criar SA de leitura e testar impersonation
SA=bq-viewer-demo
EMAIL="$SA@$PROJECT_ID.iam.gserviceaccount.com"
gcloud iam service-accounts create "$SA" 2>/dev/null || true
# papel de job no projeto (criar consultas)
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$EMAIL" --role="roles/bigquery.jobUser" >/dev/null
# acesso somente no dataset de views
bq update --dataset_access add:serviceAccount:$EMAIL:READER "$VIEWS"
# garantir que NAO tem acesso na base
bq update --dataset_access remove:serviceAccount:$EMAIL "$BASE" 2>/dev/null || true
# testar como SA
MEU=$(gcloud config get-value account)
gcloud iam service-accounts add-iam-policy-binding "$EMAIL" \
  --member="user:$MEU" --role="roles/iam.serviceAccountTokenCreator" >/dev/null

gcloud config set auth/impersonate_service_account "$EMAIL" >/dev/null
# deve falhar na base
bq query --nouse_legacy_sql 'SELECT COUNT(1) FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`' || true
# deve funcionar na view
bq query --nouse_legacy_sql 'SELECT * FROM `case-de-engenharia-de-dados.sec_v.user_iassistant_v` LIMIT 5'
# desfazer impersonation
gcloud config unset auth/impersonate_service_account >/dev/null
```

> se o seu projeto estiver em outro local, ajuste `PROJECT_ID`, datasets e os nomes das views.

---

## Validacao (como eu testei)

* Criei a SA `bq-viewer-demo@case-de-engenharia-de-dados.iam.gserviceaccount.com` com:

  * `READER` **so** em `sec_v`
  * `roles/bigquery.jobUser` no **projeto** (para criar jobs)
* Removi qualquer acesso dessa SA no dataset base
* Resultado: na base deu **Access Denied** (como esperado) e nas views funcionou com os dados mascarados

---

## Observabilidade e logs (o que eu configurei e validei)

Eu queria saber se alguem leu a tabela **base** direto (bypass da view). Entao primeiro validei o filtro no formato novo `BigQueryAuditMetadata`, depois criei a metrica. Importante: habilitar **Data Access logs** de BigQuery no projeto.

### Pre check (CLI)

```bash
gcloud logging read \
  'logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
   AND resource.type="bigquery_dataset" \
   AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
   AND protoPayload.metadata.tableDataRead.reason:* \
   AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"' \
  --limit=5 \
  --format='value(timestamp, protoPayload.metadata.tableDataRead.reason, protoPayload.resourceName)'
```

### Criacao da metrica

```bash
gcloud logging metrics create read_base_dataset_metric \
  --description="Leituras na tabela base (bypass da view)" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataRead.reason:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

### Status (deu certo no meu projeto)

* Habilitei **Data Access logs** para BigQuery (e BigQuery Storage API) no projeto.
* Os comandos `gcloud logging read` retornaram eventos de leitura da tabela base, validando o filtro.
* Ao criar a metrica recebi `subject of a conflict` porque ela **ja existia**; a metrica `read_base_dataset_metric` ja existia e permanece ativa.

### Regras extras de logging/alerta (recomendado)

Abaixo deixo mais regras que eu apliquei/validei para ampliar a cobertura de auditoria e justificar auditoria:

#### 1) Quem referenciou a tabela base em consultas (mesmo via JOIN)

**Por que**: pega consultas que citam a base; util para diferenciar "via view" de "via tabela".

```bash
gcloud logging metrics create query_ref_base_table_metric \
  --description="Queries que referenciam a tabela base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_project" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.jobCompletedEvent.job.jobStatistics.referencedTables:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

#### 2) Leituras que bateram na **view** (conforme esperado)

**Por que**: monitora o caminho certo de consumo (autorizado/mascarado).

```bash
gcloud logging metrics create read_via_view_metric \
  --description="Leituras na authorized view" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataRead.reason:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/sec_v/tables/user_iassistant_v"'
```

#### 3) DML/ingest na tabela base

**Por que**: rastrear escrita e volume.

```bash
gcloud logging metrics create base_table_dml_metric \
  --description="Mudancas de dados na tabela base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataChange:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

#### 4) Mudancas de schema/permissao no dataset base

**Por que**: detectar alteracoes de schema e IAM fora do pipeline controlado.

```bash
gcloud logging metrics create base_schema_or_policy_change_metric \
  --description="Alteracoes de schema/IAM no dataset base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND (protoPayload.metadata.tableMetadataChange:* OR protoPayload.metadata.datasetChange:*) \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315"'
```

#### 5) Alertas em cima das metricas (ex.: leu a base)

**Por que**: acionar time quando houver bypass da view.

```bash
CHANNEL="projects/case-de-engenharia-de-dados/notificationChannels/XXXXXXXXXXXX"  # substitua
POLICY_BODY=$(cat <<'JSON'
{
  "displayName": "Alerta: leitura direta na base",
  "combiner": "OR",
  "conditions": [
    {
      "displayName": "read_base_dataset_metric > 0",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/read_base_dataset_metric\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0,
        "duration": "300s"
      }
    }
  ],
  "notificationChannels": ["CHANNEL_PLACEHOLDER"]
}
JSON
)
# gcloud alpha monitoring policies create --policy "$POLICY_BODY" | sed "s/CHANNEL_PLACEHOLDER/$CHANNEL/"
```

*(eu deixei o create comentado para voce so habilitar quando ja tiver o canal de notificacao pronto)*

#### 6) Retencao e analytics dos logs

**Por que**: manter trilha por mais tempo e permitir SQL em logs.

```bash
# opcional: bucket dedicado
gcloud logging buckets create bq-security --location=global

# opcional: sink dos data_access para um dataset BigQuery
bq --location=US mk -d --description "Audit logs do projeto" audit_logs 2>/dev/null || true
SVC=$(gcloud logging sinks create sink-bq-audit \
  bigquery.googleapis.com/projects/case-de-engenharia-de-dados/datasets/audit_logs \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access"' \
  --format='value(writerIdentity)')
# conceder acesso no dataset para o SA do sink
bq update --dataset_access add:group:projectReaders audit_logs 2>/dev/null || true
# (se preferir, conceda ao writerIdentity retornado acima: roles/bigquery.dataEditor)
```

#### 7) Falhas de permissao (PERMISSION\_DENIED)

**Por que**: captura tentativas fora do fluxo (ex.: usuario/SA tentando ler base sem permissao). Ajuda a evidenciar bypass bloqueado.

```bash
gcloud logging metrics create bq_permission_denied_metric \
  --description="Tentativas PERMISSION_DENIED no BigQuery" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Factivity" \
                AND protoPayload.serviceName="bigquery.googleapis.com" \
                AND protoPayload.status.code=7'
```

> Observacao: para focar na sua base, complemente com `AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315"`.

#### 8) Consulta pesada (custo)

**Por que**: sinaliza consultas com alto `totalBilledBytes` (custo/risco). Bom para controlar SLAs e budget.

```bash
gcloud logging metrics create heavy_query_metric \
  --description="Queries com totalBilledBytes >= 1GB" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_project" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.jobCompletedEvent.eventName="job_completed" \
                AND protoPayload.metadata.jobCompletedEvent.job.jobStatistics.totalBilledBytes>=1000000000'
```

Opcional: alerta simples para quando ocorrer (substitua o canal):

```bash
CHANNEL="projects/case-de-engenharia-de-dados/notificationChannels/XXXXXXXXXXXX"
POLICY=$(cat <<'JSON'
{
  "displayName": "Alerta: consulta pesada (>=1GB)",
  "combiner": "OR",
  "conditions": [
    {
      "displayName": "heavy_query_metric > 0",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/heavy_query_metric\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 0,
        "duration": "300s"
      }
    }
  ],
  "notificationChannels": ["CHANNEL_PLACEHOLDER"]
}
JSON
)
# gcloud alpha monitoring policies create --policy "$POLICY" | sed "s/CHANNEL_PLACEHOLDER/$CHANNEL/"
```

**Resumo do "por que"**

* Base vs View: separar metricas ajuda a auditar desvio de consumo.
* DML e schema/IAM: garantem trilha de mudanca e responsabilizacao.
* Alerting: fecha o loop com acao (pager/email) quando algo anormal acontecer.
* Retencao/analytics: permite forense com SQL em cima de logs.

---

## Seguranca e privacidade (em uso)

* **Authorized Views**: consumidores so leem as views `sec_v.*`, onde aplico mascaras de email e lat/lon e redacao de texto.
* **IAM enxuto**: a service account de leitura tem permissao apenas no `sec_v` e papel `bigquery.jobUser` no projeto para executar jobs; sem acesso ao dataset base.
* **Data Access Logs**: habilitados para auditar `tableDataRead`, DML, mudancas de schema e IAM.
* **Sensitive Data Protection (DLP)**: ativei a inspecao de dados no projeto para identificar padroes sensiveis (email, possiveis localizacoes). Uso como visibilidade/monitoramento, sem bloqueio automatico.

### Nao aplicado por escopo/limitacao do projeto

* **Policy Tags / Data Catalog**: criacao de DataPolicy exige organizacao; meu projeto e standalone, entao deixei como melhoria futura.
* **Row-level Security (RLS)**: tentativa de `CREATE ROW ACCESS POLICY` falhou por grupo inexistente; mantive o modelo com authorized view, que resolve a necessidade funcional.

---

## Estado atual (bq show do dataset base)

Comando que usei:

```bash
bq show case-de-engenharia-de-dados:845415315315
```

Resumo do que apareceu no meu ambiente (conferido via terminal):

* **Authorized Views**: `case-de-engenharia-de-dados:sec_v.user_iassistant_v` listado (conforme esperado).
* **Members (recorte)**: varios emails validos, o grupo implicito `projectReaders` e a SA `bq-viewer-demo@case-de-engenharia-de-dados.iam.gserviceaccount.com`.
* \*\*Schema da tabela \*\*\`\`:

  * `REGION`: STRING
  * `USER_EMAIL`: STRING
  * `USER_NAME`: STRING
  * `USER_ID`: INTEGER
  * `DT_CARGA`: INTEGER
* **Particionamento**: por `HOUR`, com `expirationMs=5184000000` (\~60 dias).
* **Total rows**: `1000`.
* **Total logical bytes**: \~`88 KB`.

Isso confirma que:

1. o dataset base esta com a authorized view ativa,
2. a tabela esta com schema/particionamento conforme esperado,
3. as entradas de IAM estao no lugar (incluindo a SA de viewer que usei para testar acesso somente nas views mascaradas).

---

## Como subir este README no GitHub (Cloud Shell)

Abaixo deixo o que eu usei para colocar este README no repo `dinx0/iassistant`.

### Pre requisitos

* ter o repo no GitHub: `https://github.com/dinx0/iassistant`
* estar logado no Cloud Shell

### 0) identidade do git (uma vez so)

```bash
git config --global user.name "Dih Oliver"
git config --global user.email "dih.oliver08@gmail.com"
```

### 1) se o repo ja estiver clonado

```bash
cd ~/iassistant          # entre no diretorio do repo
nano README.md           # cole/edite o conteudo deste arquivo
# salve (Ctrl+O, Enter) e saia (Ctrl+X)

git add README.md
git commit -m "docs: adiciona README do projeto (BigQuery + views + logging)"
```

### 2) se ainda nao tiver clonado

```bash
git clone https://github.com/dinx0/iassistant.git
cd iassistant
nano README.md
# salve e saia

git add README.md
git commit -m "docs: adiciona README do projeto (BigQuery + views + logging)"
```

### 3) autenticar e fazer o push (recomendado: GitHub CLI)

```bash
sudo apt-get update && sudo apt-get install -y gh
gh auth login -w        # GitHub.com -> HTTPS -> abrir no navegador e confirmar

git branch -M main      # garante que a branch e main
git push -u origin main
```

### Alternativas de autenticacao

* **HTTPS com token (PAT)**: gere um token com escopo `repo` em *Settings -> Developer settings -> Personal access tokens*. No `git push`, use `dinx0` como username e cole o token no campo de password.
* **SSH**: gere uma chave `ed25519` (`ssh-keygen -t ed25519 -C "dih.oliver08@gmail.com"`), adicione a publica em *Settings -> SSH and GPG keys*, e troque a URL remota:

  ```bash
  git remote set-url origin git@github.com:dinx0/iassistant.git
  git push -u origin main
  ```

### Dica: limpar credencial errada em cache (se pedir username esquisito)

```bash
git config --global --unset credential.helper 2>/dev/null || true
git credential reject <<EOF
protocol=https
host=github.com
EOF
```

### Conferencia

```bash
git log --oneline -n 3
git remote -v
```

# Case de Engenharia de Dados (BigQuery) — passo a passo final

Eu construí este case inteiro no projeto **`case-de-engenharia-de-dados`** para mostrar como:

* carregar bases (lote e streaming)
* publicar **authorized views** com **mascara de dados**
* controlar acesso somente pelas views
* auditar leitura direta na base via **Data Access Logs** + metricas
* monitorar atividade do **GitHub** via dataset `github_activity`

Tudo aqui é o que eu realmente executei e funcionou. Sem firula.

> **Regiao**: meu dataset base fica na regiao `southamerica-east1`. O dataset `github_activity` (do Marketplace) é sempre em **US**, entao as consultas nele precisam de `--location=US`.

---

## Visao geral (diagramas)

### Arquitetura
```mermaid
flowchart LR
  subgraph GCP[Projeto: case-de-engenharia-de-dados]
    subgraph BQ[BigQuery]
      direction TB
      BASE[(Dataset base
845415315315)]
      VIEWS[(Dataset views
sec_v)]
      BASE -->|Authorized View| V1[view: sec_v.user_iassistant_v]
      BASE -->|Authorized View| V2[view: sec_v.phrase_metrics_v]
    end
    LOGS[Cloud Logging
(Data Access logs)]
  end

  EXT1[Chrome Ext + API
(IASSISTANT)] -->|streaming| BASE
  CSV[NDJSON/CSV
usuarios] -->|load job| BASE

  LOGS -.audita leitura base.-> BASE
  LOGS -.audita leitura view.-> V1
```

### Modelo (ER simples)
```mermaid
erDiagram
  USER_IASSISTANT {
    INT64 USER_ID PK
    STRING USER_NAME
    STRING USER_EMAIL
    STRING REGION "lat,lon"
    INT64 DT_CARGA
  }
  PHRASE_METRICS {
    STRING user_id FK
    FLOAT64 PHRASE_METRICS
    STRING PHRASE_TEXT
    INT64 PAGE_METRICS
    INT64 PAGE_NUMBER
    STRING SEARCH_ITEMS
    STRING SEARCH_RESULT
    INT64 DT_CARGA
  }
  USER_IASSISTANT ||--o{ PHRASE_METRICS : "user_id = CAST(USER_ID AS STRING)"
```

### Mascaras aplicadas
```mermaid
flowchart TD
  A[USER_EMAIL] -->|REGEXP_REPLACE| AM[email_mascarado
"f***@dominio"]
  R[REGION "lat,lon"] -->|regex + round(2)| LAT[lat_mascarada]
  R -->|regex + round(2)| LON[lon_mascarada]
  T[PHRASE_TEXT] -->|SUBSTR(1,20)||> three dots| TM[phrase_text_masked]
  S[SEARCH_RESULT] -->|SUBSTR(1,60)||> three dots| SM[search_result_masked]
```

---

## 1) Datasets e tabelas

**Projeto**: `case-de-engenharia-de-dados`

**Datasets**
- Base: `845415315315` (regiao `southamerica-east1`)
- Views: `sec_v` (mesma regiao do base)

**Tabelas base**
- `845415315315.user_iassistant`  
  PK logica: `USER_ID (INT64)`
- `845415315315.phrase_metrics`  
  Relaciona em `user_id (STRING)` com `USER_ID` via `CAST`.

**Como garanti o dataset de views na MESMA regiao do base**
```bash
BASE="case-de-engenharia-de-dados:845415315315"
LOC=$(bq show --format=prettyjson "$BASE" | jq -r .location)
bq rm -f -d case-de-engenharia-de-dados:sec_v 2>/dev/null || true
bq mk --dataset --location="$LOC" case-de-engenharia-de-dados:sec_v
```

---

## 2) Views mascaradas (DDL usado)

### `sec_v.user_iassistant_v`
```sql
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.user_iassistant_v` AS
SELECT
  CAST(USER_ID AS INT64) AS user_id,
  CONCAT(
    SUBSTR(LOWER(TRIM(USER_EMAIL)),1,1),
    '***',
    REGEXP_EXTRACT(LOWER(TRIM(USER_EMAIL)), r'(@.*)$')
  ) AS email_mascarado,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>0,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(0)] AS FLOAT64), NULL),
    2
  ) AS lat_mascarada,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>1,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(1)] AS FLOAT64), NULL),
    2
  ) AS lon_mascarada
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;
```

### `sec_v.phrase_metrics_v`
```sql
CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.phrase_metrics_v` AS
SELECT
  CAST(user_id AS STRING) AS user_id,
  CAST(PHRASE_METRICS AS FLOAT64) AS phrase_metrics,
  IFNULL(SUBSTR(PHRASE_TEXT, 1, 20) || '...', NULL) AS phrase_text_masked,
  CAST(PAGE_METRICS AS INT64) AS page_metrics,
  CAST(PAGE_NUMBER AS INT64) AS page_number,
  SEARCH_ITEMS AS search_items,
  IFNULL(SUBSTR(SEARCH_RESULT, 1, 60) || '...', NULL) AS search_result_masked,
  CAST(DT_CARGA AS INT64) AS dt_carga
FROM `case-de-engenharia-de-dados.845415315315.phrase_metrics`;
```

### Autorizar as views no dataset base
```bash
BASE="case-de-engenharia-de-dados:845415315315"

bq show --format=prettyjson "$BASE" > base.json
jq '.access += [
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"user_iassistant_v"}},
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"phrase_metrics_v"}}
] | .access |= (map(tojson)|unique|map(fromjson))' base.json > base_patched.json
bq update --source=base_patched.json "$BASE"
```

---

## 3) IAM dinamico por tabela (o que eu executei)

Regra: `USER_ID 1..5 => READER`, `>= 6 => WRITER` no dataset base.

```bash
PROJECT_ID="case-de-engenharia-de-dados"
DATASET_ID="845415315315"
DATASET_REF="$PROJECT_ID:$DATASET_ID"
SRC_TABLE="$PROJECT_ID.$DATASET_ID.user_iassistant"

bq query --nouse_legacy_sql --format=csv "
WITH base AS (
  SELECT DISTINCT CAST(USER_ID AS INT64) AS user_id,
         LOWER(TRIM(USER_EMAIL)) AS email
  FROM \`$SRC_TABLE\`
  WHERE USER_EMAIL IS NOT NULL
)
SELECT email,
       CASE WHEN user_id BETWEEN 1 AND 5 THEN 'READER' ELSE 'WRITER' END AS role
FROM base
" | tail -n +2 | while IFS=, read -r email role; do
  [[ -z "$email" || -z "$role" ]] && continue
  bq show --format=prettyjson "$DATASET_REF" > ds.tmp.json
  jq --arg em "$email" --arg r "$role" \
     '.access += [{"role":$r,"userByEmail":$em}] | .access |= (map(tojson)|unique|map(fromjson))' \
     ds.tmp.json > ds.apply.json
  bq update --source=ds.apply.json "$DATASET_REF"
  rm -f ds.tmp.json ds.apply.json
done
```
> Emails invalidos o BigQuery recusa; a lista final ficou so com emails validos.

---

## 4) Teste de seguranca (impersonation)

Criei uma SA **somente leitora de views** e testei com impersonation.

```bash
PROJECT_ID=case-de-engenharia-de-dados
VIEWS=case-de-engenharia-de-dados:sec_v
BASE=case-de-engenharia-de-dados:845415315315

SA=bq-viewer-demo
EMAIL="$SA@$PROJECT_ID.iam.gserviceaccount.com"

gcloud iam service-accounts create "$SA" 2>/dev/null || true
# pode criar jobs
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$EMAIL" --role="roles/bigquery.jobUser"
# leitura so no dataset de views
bq update --dataset_access add:serviceAccount:$EMAIL:READER "$VIEWS"
# garante que NAO tem acesso ao base
bq update --dataset_access remove:serviceAccount:$EMAIL "$BASE" 2>/dev/null || true

# permite eu impersonar
MEU=$(gcloud config get-value account)
gcloud iam service-accounts add-iam-policy-binding "$EMAIL" \
  --member="user:$MEU" --role="roles/iam.serviceAccountTokenCreator"

gcloud config set auth/impersonate_service_account "$EMAIL"
# deve falhar
bq query --nouse_legacy_sql 'SELECT COUNT(*) FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`' || true
# deve funcionar
bq query --nouse_legacy_sql 'SELECT * FROM `case-de-engenharia-de-dados.sec_v.user_iassistant_v` LIMIT 5'

gcloud config unset auth/impersonate_service_account
```

---

## 5) Observabilidade e controle (logs + metricas)

Primeiro **habilitei Data Access logs** de BigQuery (Console > IAM e Admin > Audit Logs > BigQuery > marcar Data Read/Write).

### Validar leitura direta na base (CLI)
```bash
gcloud logging read \
  'logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
   AND resource.type="bigquery_dataset" \
   AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
   AND protoPayload.metadata.tableDataRead.reason:* \
   AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"' \
  --limit=5 \
  --format='value(timestamp, protoPayload.metadata.tableDataRead.reason, protoPayload.resourceName)'
```

### Metricas que eu criei

**1) Leitura direta na base**
```bash
gcloud logging metrics create read_base_dataset_metric \
  --description="Leituras na tabela base (bypass da view)" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataRead.reason:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

**2) Consultas que referenciam a tabela base**
```bash
gcloud logging metrics create query_ref_base_table_metric \
  --description="Queries que referenciam a tabela base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_project" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.jobCompletedEvent.job.jobStatistics.referencedTables:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

**3) Leituras via authorized view**
```bash
gcloud logging metrics create read_via_view_metric \
  --description="Leituras na authorized view" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataRead.reason:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/sec_v/tables/user_iassistant_v"'
```

**4) DML na base**
```bash
gcloud logging metrics create base_table_dml_metric \
  --description="Mudancas de dados na tabela base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND protoPayload.metadata.tableDataChange:* \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'
```

**5) Mudanca de schema/IAM**
```bash
gcloud logging metrics create base_schema_or_policy_change_metric \
  --description="Alteracoes de schema/IAM no dataset base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
                AND resource.type="bigquery_dataset" \
                AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
                AND (protoPayload.metadata.tableMetadataChange:* OR protoPayload.metadata.datasetChange:*) \
                AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315"'
```

> Em producao: configure alertas no Cloud Monitoring apontando para essas metricas (email/pager).

---

## 6) Qualidade de dados (checks simples que eu rodei)

**Comparar contagem base vs view**
```sql
-- deve bater
SELECT COUNT(*) FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;
SELECT COUNT(*) FROM `case-de-engenharia-de-dados.sec_v.user_iassistant_v`;
```

**Verificar mascara de email**
```sql
SELECT USER_EMAIL, email_mascarado
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant` u
JOIN `case-de-engenharia-de-dados.sec_v.user_iassistant_v` v
ON v.user_id = u.USER_ID
LIMIT 20;
```

**Validar parse de latitude/longitude**
```sql
SELECT REGION, lat_mascarada, lon_mascarada
FROM `case-de-engenharia-de-dados.sec_v.user_iassistant_v`
WHERE (lat_mascarada IS NULL OR lon_mascarada IS NULL)
LIMIT 20;
```

**Campos mascarados na phrase_metrics_v**
```sql
SELECT phrase_text_masked, search_result_masked
FROM `case-de-engenharia-de-dados.sec_v.phrase_metrics_v`
LIMIT 20;
```

---

## 7) Monitor de atividade do GitHub (GitHub Activity Data)

> Este dataset sempre fica em **US**. Se nao existir, crie o dataset manualmente:  
> `bq --location=US mk -d case-de-engenharia-de-dados:github_activity`

**Eventos por tipo (30 dias)**
```sql
SELECT type, COUNT(*) AS total
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant'
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY type
ORDER BY total DESC;
```

**Commits por dia e autor (PushEvent)**
```sql
SELECT
  DATE(created_at) AS dia,
  actor.login AS pusher,
  COUNT(1) AS commits
FROM `case-de-engenharia-de-dados.github_activity.events`,
UNNEST(payload.commits) c
WHERE repo.name = 'dinx0/iassistant'
  AND type = 'PushEvent'
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY dia, pusher
ORDER BY dia DESC, commits DESC;
```

**Ultimos 20 commits (mensagem/autor/sha)**
```sql
SELECT
  c.sha,
  c.author.name  AS author_name,
  c.author.email AS author_email,
  c.message,
  created_at
FROM `case-de-engenharia-de-dados.github_activity.events`,
UNNEST(payload.commits) c
WHERE repo.name = 'dinx0/iassistant'
  AND type = 'PushEvent'
ORDER BY created_at DESC
LIMIT 20;
```

**PRs abertos/mergeados/fechados (90 dias)**
```sql
SELECT
  COUNTIF(payload.action = 'opened')                                             AS prs_abertos,
  COUNTIF(payload.action = 'closed' AND payload.pull_request.merged)             AS prs_mergeados,
  COUNTIF(payload.action = 'closed' AND NOT payload.pull_request.merged)         AS prs_fechados_sem_merge
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant'
  AND type = 'PullRequestEvent'
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY);
```

**Lead time de PR (criado -> merge)**
```sql
SELECT
  payload.pull_request.number AS pr_number,
  payload.pull_request.title  AS title,
  TIMESTAMP_DIFF(payload.pull_request.merged_at,
                 payload.pull_request.created_at, HOUR) AS lead_time_horas
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant'
  AND type = 'PullRequestEvent'
  AND payload.action = 'closed'
  AND payload.pull_request.merged
ORDER BY lead_time_horas DESC
LIMIT 20;
```

**Issues por acao**
```sql
SELECT payload.action, COUNT(*) AS total
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant'
  AND type = 'IssuesEvent'
GROUP BY payload.action
ORDER BY total DESC;
```

**Stars e forks**
```sql
-- stars
SELECT DATE(created_at) AS dia, COUNT(*) AS stars
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant' AND type = 'WatchEvent'
GROUP BY dia ORDER BY dia DESC;

-- forks
SELECT DATE(created_at) AS dia, COUNT(*) AS forks
FROM `case-de-engenharia-de-dados.github_activity.events`
WHERE repo.name = 'dinx0/iassistant' AND type = 'ForkEvent'
GROUP BY dia ORDER BY dia DESC;
```

Dica: ao usar `bq` nessas consultas, inclua `--location=US`.

---

## 8) Exportar e versionar infra do BigQuery no repo

**Export (ambiente -> git)**
```bash
PROJECT=case-de-engenharia-de-dados
BASE_DS=845415315315
VIEWS_DS=sec_v
mkdir -p sql/views infra/bq schema data/samples
bq show --format=prettyjson $PROJECT:$VIEWS_DS.user_iassistant_v | jq -r .view.query > sql/views/user_iassistant_v.sql
bq show --format=prettyjson $PROJECT:$VIEWS_DS.phrase_metrics_v  | jq -r .view.query > sql/views/phrase_metrics_v.sql
bq show --format=prettyjson $PROJECT:$BASE_DS | jq '{access}' > infra/bq/dataset_access.json
bq show --schema --format=prettyjson $PROJECT:$BASE_DS.user_iassistant > schema/user_iassistant.schema.json
bq show --schema --format=prettyjson $PROJECT:$BASE_DS.phrase_metrics > schema/phrase_metrics.schema.json
```

**Apply (git -> ambiente)**
```bash
PROJECT=case-de-engenharia-de-dados
BASE_DS=845415315315
VIEWS_DS=sec_v
BASE="$PROJECT:$BASE_DS"; VIEWS="$PROJECT:$VIEWS_DS"
LOC=$(bq show --format=prettyjson "$BASE" | jq -r .location)
bq mk --dataset --location="$LOC" "$VIEWS" 2>/dev/null || true
jq '{access}' infra/bq/dataset_access.json > /tmp/ds_access.json
bq show --format=prettyjson "$BASE" | jq '.access = (input.access)' /tmp/ds_access.json > /tmp/base_apply.json
bq update --source=/tmp/base_apply.json "$BASE"
bq query --use_legacy_sql=false --location="$LOC" < sql/views/user_iassistant_v.sql
bq query --use_legacy_sql=false --location="$LOC" < sql/views/phrase_metrics_v.sql
```

**Scripts utilitarios**
- `scripts/export_bq.sh` e `scripts/apply_bq.sh` (mesmas linhas acima em shell, prontos para automatizar).

---

## 9) Publicar/atualizar este README no GitHub (Cloud Shell)

```bash
# identidades do git
git config --global user.name "Dih Oliver"
git config --global user.email "dih.oliver08@gmail.com"

# clonar (se ainda nao tiver)
git clone https://github.com/dinx0/iassistant.git
cd iassistant

# editar README.md
nano README.md

git add README.md
git commit -m "docs: atualiza README (BigQuery + views + logs + github activity)"

# autenticar e enviar (via GitHub CLI)
sudo apt-get update && sudo apt-get install -y gh
gh auth login -w

git branch -M main
git push -u origin main
```

Se pedir usuario/senha via HTTPS: use **Personal Access Token** como senha, ou troque o remoto para **SSH**.

---

## 10) Politicas adicionais e limites do case

* Ativei **Sensitive Data Protection (DLP)** no projeto para inspecao (sem bloqueios automaticos).
* **Policy Tags/Data Policies**: deixei como melhoria futura (exige org). Mantive **authorized view** como mecanismo principal de mascaramento.
* **Row-level Security**: tentativas com grupos nao existentes falharam; mantive apenas authorized view.

---

### Conclusao

O case entrega:
- mascaramento de PII na borda de consumo (views)
- isolamento de acesso (dataset `sec_v` vs base)
- auditoria de bypass (metricas + logs)
- monitor de atividade do repo GitHub com queries prontas
- scripts para exportar/aplicar infra no BigQuery e versionar no git

Qualquer ajuste futuro: adicionar Policy Tags, RLS por grupo real e CI/CD via Actions/Workload Identity para aplicar as mudancas de `sql/` e `infra/` automaticamente.


# IAssistant

**IAssistant** é uma extensão Chrome desenvolvida para auxiliar na análise de resultados de buscas por meio de tokenização. A extensão calcula percentuais de similaridade entre a consulta realizada e os resultados obtidos, permitindo a visualização de métricas e gráficos que podem ser integrados a um serviço em nuvem (como o BigQuery).

## Funcionalidades

- **Tokenização & Análise de Similaridade:**  
  A extensão utiliza tokenização para comparar a pesquisa realizada com os resultados obtidos, calculando percentuais de similaridade.

- **Interface Dual – Popup e Side Panel:**  
  Utiliza um arquivo `index.html` para exibir a interface principal, disponível tanto como popup quanto como painel lateral (side panel) no Chrome.

- **Comandos Rápidos e Omnibox:**  
  - **Atalho:** Pressione `Ctrl+B` (ou `Command+B` no Mac) para executar a ação principal.  
  - **Omnibox:** Digite `assistente` na barra de endereços para interagir com a extensão.

- **Visualização de Gráficos e Métricas:**  
  Ao executar os comandos `show graphics` ou `show metrics`, a extensão realiza requisições para exibir gráficos e métricas detalhadas. (_Nota: Caso os endpoints ainda estejam configurados para `localhost`, ocorrerão erros de conexão em ambientes de produção._)


IAssistant/ ├── images/ │   ├── icon-16.png │   ├── icon-32.png │   ├── icon-48.png │   └── icon-128.png ├── scripts/ │   └── iaServiceWorker.js ├── index.html └── manifest.json


## Estrutura do Projeto

A estrutura da aplicação segue o padrão de uma extensão Chrome utilizando Manifest V3:


- **manifest.json:**  
  Arquivo de configurações que define:
  - Versão da extensão e do Manifest (Manifest V3).
  - Permissões necessárias: `"tabs"`, `"activeTab"`, `"sidePanel"`, `"storage"`, `"unlimitedStorage"`.
  - Configuração do **side panel** e do **popup** (ambos usando `index.html` como ponto de interface).
  - Detalhes dos ícones e comandos de atalho (ex.: `Ctrl+B` e a palavra-chave `assistente` para o omnibox).

- **index.html:**  
  Responsável pela interface gráfica do popup e do painel lateral, onde os usuários podem visualizar resultados, gráficos e métricas.

- **scripts/iaServiceWorker.js:**  
  Service worker que atua como backend interno da extensão, gerenciando eventos, processando requisições e, possivelmente, comunicando-se com APIs externas para enviar dados e obter métricas.

- **images/:**  
  Diretório contendo os ícones da extensão em várias resoluções para diferentes contextos (barra de ferramentas, painel, etc.).

## Instalação e Execução

### Instalação Local
1. **Carregar a Extensão:**
   - Abra o Chrome e acesse `chrome://extensions`.
   - Habilite o **Modo de Desenvolvedor**.
   - Clique em **"Carregar sem compactação"** e selecione a pasta raiz do projeto (`IAssistant/`).

2. **Testar a Interface:**
   - Clique no ícone da extensão para abrir o popup e verificar a interface.
   - Utilize a funcionalidade do **side panel** (disponível através da configuração no manifesto).
   - Utilize os atalhos (`Ctrl+B`/`Command+B`) e a palavra-chave `assistente` na Omnibox para testar os comandos.

### Executando Comandos Específicos
- **Show Graphics / Show Metrics:**  
  Estes comandos acionam funções que fazem requisições a endpoints responsáveis por retornar gráficos e métricas.  
  **Atenção:** Certifique-se que os endpoints estejam configurados para um domínio público (não "localhost") em produção.

## Configuração do Backend

A extensão se integra a um serviço backend para:
- Registrar os dados de tokenização.
- Processar as informações e exibir gráficos e métricas.
- Enviar os dados analisados para o Google BigQuery ou outros serviços em nuvem.

**Recomendação:**  
- Se estiver utilizando `localhost` durante o desenvolvimento, altere para a URL pública do seu servidor na versão publicada para evitar erros de conexão (_net::ERR_CONNECTION_REFUSED_).

## Resolução de Problemas

- **Erro de Conexão com `localhost`:**  
  Verifique se os endpoints configurados nos comandos `show graphics` e `show metrics` apontam para um servidor público e ativo.  
  Caso contrário, ajustados os endpoints ou utilize variáveis de ambiente para alternar entre desenvolvimento e produção.

- **Recursos Não Encontrados (ERR_FILE_NOT_FOUND):**  
  Certifique-se de que os caminhos para arquivos e recursos (como imagens, scripts e estilos) estão corretos no pacote da extensão.

- **Problemas com o Service Worker:**  
  Utilize `chrome://extensions` para inspecionar o service worker e verifique os logs para identificar possíveis erros.

## Contribuição

Contribuições para melhorias e novas funcionalidades são bem-vindas!

1. Fork o projeto.
2. Crie uma branch com a nova funcionalidade ou correção:  
   `git checkout -b feature/nome-da-feature`
3. Faça os commits com suas alterações.
4. Envie a branch para o repositório e abra um Pull Request.

## Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

# IAssistant

O **IAssistant** é uma ferramenta em Python desenvolvida para analisar conteúdos textuais de documentos (PDF, HTML, TXT, CSV) e calcular métricas de similaridade entre frases extraídas dos documentos e termos de busca informados pelo usuário. Além disso, o projeto integra uma interface interativa—por meio do Dash e Plotly—para visualizar frequências, estatísticas e gráficos que evidenciam os resultados da análise.

## Funcionalidades

- **Extração e Processamento de Conteúdo:**
  - Suporte à leitura de documentos em diversos formatos:
    - PDFs (usando o PyMuPDF, via o módulo `fitz`)
    - Arquivos HTML, TXT e CSV através de tratamento de conteúdo ou requisições HTTP
  - Tokenização do texto utilizando NLTK, com remoção de stopwords (para os idiomas inglês e português) e limpeza textual via expressões regulares.

- **Análise de Similaridade e Frequência:**
  - Cálculo de métricas de similaridade (por meio do **cosine similarity**) entre o conteúdo extraído e os termos de busca desejados.
  - Divisão do documento em partes (splits por frases) para facilitar a análise por página ou por trecho.
  - Geração de distribuições de frequência usando `FreqDist` do NLTK e construção de dataframes no Pandas, permitindo a visualização e o agrupamento dos dados.

- **Armazenamento e Manipulação de Dados:**
  - Classes dedicadas à manipulação de arquivos: salvando conteúdo em formatos HTML, TXT e PDF.
  - Integração com um sistema de gerenciamento de dados via `BaseManager` para compartilhamento de informações entre processos.

- **Dashboards e Visualização Interativa:**
  - Criação de interfaces gráficas interativas utilizando o [Dash](https://dash.plotly.com/) e [Plotly](https://plotly.com/python/), com elementos como gráficos de dispersão, barras e tabelas dinâmicas.
  - Componentes interativos (dropdowns, botões e popovers via dash-bootstrap-components) que permitem filtrar os dados e visualizar métricas detalhadas por página, palavra e frase.
  - Funções utilitárias para definir cores (baseado na intensidade dos valores) que auxiliam na representação visual dos níveis de similaridade (ex.: vermelho para baixa similaridade, verde para alta).

- **Configuração Dinâmica do Backend:**
  - Funções para obter configurações dinâmicas de endpoints e dados de busca (através de GET requests para URLs configuradas).  
    **Atenção:** Por padrão, esses endpoints utilizam `http://localhost:8080`. Em produção, é necessário atualizar esse valor para um domínio público e acessível, evitando erros de conexão.
    
# IAssistant — Case Completo de Engenharia de Dados + Extensão Chrome + Backend + BigQuery + Integração GPT

Projeto criado para demonstrar, de ponta a ponta, uma solução que captura dados de buscas e documentos via extensão Chrome, processa em um backend com Spark e Python, aplica anonimização, gera métricas e dashboards interativos, e integra com múltiplos destinos de armazenamento e análise: Delta Lake (Databricks), SQL Server on-premise e BigQuery. Além disso, conecta-se à API da OpenAI para análise de contexto e geração de insights.

---

## Objetivo

O IAssistant nasceu para integrar captura em tempo real de dados não estruturados (texto, PDFs, HTML) com processamento inteligente e visualização. É um ecossistema que combina coleta via browser, pipeline de processamento distribuído, armazenamento otimizado e análise assistida por IA, tudo conectado por APIs bem definidas.

---

## Arquitetura Geral

```mermaid
graph TD
  A[Extensão Chrome] -->|POST JSON| B[/API Flask upload_api.py]
  B -->|Processa| C[Spark - Anonimização + Validação]
  C -->|Delta Format| D[Delta Lake - Databricks]
  C --> E[SQL Server On-prem]
  C --> F[Função BigQuery - GCP]
  B --> G[Dash Server Flask/Dash]
  G --> H[Dashboards Interativos]
  G --> I[OpenAI GPT API]
```

---

## Backend — Componentes

### API de Upload (`upload_api.py`)

* Recebe JSON da extensão.
* Cria DataFrame Spark.
* Aplica máscara em campos sensíveis (`mask_email_udf`).
* Valida dados (ex.: contagem de linhas > 0).
* Grava em Delta Lake e SQL Server.

### Dash Server (`app.py`)

* Fornece endpoints REST para servir dados e configurações.
* Rota `/api/v1/document`: recebe documento/texto/PDF, persiste e aciona GPT.
* Serve arquivos estáticos e integra com dashboard interativo.

### Dash Process (`services/dash_process.py`)

* Aguarda API subir (`http://localhost:8080/configurations`).
* Inicia servidor Dash na porta `8081`.

### Integração GPT (`client.py`)

* Wrapper simples para `chat.completions` da OpenAI.
* Parametriza temperatura (criatividade), número máximo de opções e tokens.

### Função BigQuery (`bigquery_function.js`)

* Recebe requisição POST.
* Formata linha com `timestamp`, `user_id`, `search_query`, `best_answer`, `percent`.
* Insere no dataset/tabela configurados.

---

## Instalação — Backend

```bash
# Clonar repositório
cd IAssistant/backend

# Criar ambiente virtual
python -m venv .venv
source .venv/bin/activate

# Instalar dependências
pip install -r requirements.txt

# Configurar variáveis no .env
# - OPENAI_API_KEY
# - Config BigQuery (GCP)
# - Config SQL Server (ODBC)

# Subir API principal
python app.py

# Em outro terminal, iniciar Dash Process
python services/dash_process.py
```

---

## Instalação — Extensão Chrome

1. Ir em `chrome://extensions`.
2. Ativar **Modo do desenvolvedor**.
3. Clicar em **Carregar sem compactação**.
4. Selecionar pasta `IAssistant/extension`.
5. Configurar endpoint da API no `index.html`.

---

## Pipeline de Processamento

1. Extensão coleta dados (DOM, PDFs, inputs do usuário).
2. Envia via `fetch` POST para `/uploadData`.
3. Spark processa e mascara dados sensíveis.
4. Dados persistem em:

   * Delta Lake (arquitetura otimizada para queries analíticas).
   * SQL Server (uso interno/on-prem).
   * BigQuery (análises agregadas e cruzamento com outros datasets).
5. API dispara análise GPT e retorna sugestões/insights.
6. Dashboards consomem dados processados.

---

## Similaridade e Estatística

* **Tokenização e vetorização**: textos quebrados em tokens, aplicando limpeza (stopwords, stemming) para comparação.
* **Métrica de similaridade**: cálculo de similaridade semântica entre campos (cosine similarity sobre embeddings ou TF-IDF dependendo do caso).
* **Estatística descritiva**: contagem, médias, percentuais, distribuição de termos.
* **Validação**: regras simples (mínimo de linhas, campos obrigatórios) e logs estruturados.

---

## Bibliotecas Utilizadas

* **Backend**: Flask, Flask-CORS, PyODBC, Pyspark, Dash, Dash Bootstrap Components.
* **Integração GPT**: openai.
* **GCP**: @google-cloud/bigquery (Node.js).
* **Outros**: logging, json, base64, urllib.

---

## Fluxo de Inserção no BigQuery

1. Extensão ou backend envia JSON para endpoint da função.
2. Função Node.js formata linha conforme schema.
3. Executa `table.insert()` no BigQuery.
4. Registro disponível quase em tempo real para consulta via SQL.

---

## Resultado

Com essa arquitetura, conseguimos ingerir, processar, anonimizar e disponibilizar dados não estruturados de forma escalável, com integração entre ambiente on-prem e cloud, além de análise contextual via GPT e visualização imediata em dashboards.

Case de Engenharia de Dados + IAssistant (Extensão + Backend + BigQuery)

Eu montei este case unificando a extensão IAssistant com um backend analítico e um projeto BigQuery governado via authorized views. Ideia simples: capturo consulta + conteúdo (PDF/HTML/TXT/CSV), tokenizo, calculo TF-IDF + cosseno por frase/página, mostro gráficos no Dash e persiste no BigQuery (com máscaras e auditoria).

Projeto GCP: case-de-engenharia-de-dados
Região base: southamerica-east1
Dataset Marketplace github_activity: sempre em US → consultas com --location=US.

Canvas 1 — Arquitetura (fim-a-fim)
flowchart LR
  subgraph Browser[Chrome]
    EXT[Extensão IAssistant\npopup/side-panel/omnibox]
  end

  EXT -- POST /api/v1/document --> API8080[API Core (Flask)\n8080 (dash_server.py)]
  EXT -- POST /uploadData --> API5000[Upload API (Flask+Spark)\n5000 (api_uploader.py)]

  API8080 --> DASH[Dash 8081\n(dash_process.py + IAUtils.start_server())]
  API8080 -- opcional --> GPT[OpenAI client.py]

  API5000 -- insert --> BQ[(BigQuery\niaassistant.logs/pages/terms)]
  API5000 -- append --> DELTA[(Delta Lake/DBFS)]
  API5000 -- mirror --> SQL[(SQL Server on-prem)]

  subgraph GCP[Projeto: case-de-engenharia-de-dados]
    subgraph BQSEC[BigQuery Governança]
      direction TB
      BASE[(Dataset base\n845415315315)]
      VIEWS[(Dataset views\nsec_v)]
      BASE -->|Authorized View| V1[sec_v.user_iassistant_v]
      BASE -->|Authorized View| V2[sec_v.phrase_metrics_v]
    end
    LOGS[Cloud Logging\n(Data Access Logs)]
  end

  CSV[NDJSON/CSV usuarios] -->|load job| BASE
  LOGS -.audita leitura base.-> BASE
  LOGS -.audita leitura view.-> V1


Padrão de dados que usei

Kappa-like por padrão: um único caminho de eventos (extensão → APIs) atende realtime + replay.

Lambda on-demand: quando preciso de lotes massivos, rodo batch e converjo no BigQuery. Operação simples.

Canvas 2 — Pipeline analítico (tokenização/TF-IDF/cosseno)
flowchart LR
  Q[Consulta] --> N[Normalização\nlower/regex]
  D[Documento (PDF/HTML/TXT/CSV)] --> N
  N --> T[Tokenização + stopwords (NLTK)]
  T --> V[TF-IDF]
  Q --> V
  V --> S[Cosine por frase/trecho]
  S --> A[Agregação por página (PDF)]
  S --> F[FreqDist tokens]
  A --> G1[Gráfico by_page]
  F --> G2[Gráfico top_words]
  S --> G3[Tabela frases (score)]


Como calculei

Normalização: minúsculas, remoção de pontuação/ruído com regex.

Split: PDF por página (PyMuPDF) + frase (NLTK). HTML/TXT idem (frase/bloco).

TF-IDF: 
t
f
i
d
f
(
𝑡
,
𝑑
)
=
t
f
(
𝑡
,
𝑑
)
⋅
log
⁡
𝑁
𝑑
𝑓
(
𝑡
)
tfidf(t,d)=tf(t,d)⋅log
df(t)
N
	​


Cosseno: 
cos
⁡
(
𝜃
)
=
𝑞
⃗
⋅
𝑑
⃗
∥
𝑞
⃗
∥
⋅
∥
𝑑
⃗
∥
cos(θ)=
∥
q
	​

∥⋅∥
d
∥
q
	​

⋅
d
	​


page_metric_by_page: média/percentil dos scores das frases daquela página.

tokens_top: FreqDist (termo, frequência, páginas).

Thresholds (ex.: 0.15) para realce visual.

Canvas 3 — Modelo (ER) unificado
erDiagram
  LOGS {
    TIMESTAMP ts
    STRING page_url
    STRING query
    STRING lang
    STRING source_type
    STRING source_url
    STRING client_version
    STRING user_id
  }
  PAGES {
    TIMESTAMP ts
    STRING doc_id
    INT64 page_number
    FLOAT64 page_metric_by_page
  }
  TERMS {
    TIMESTAMP ts
    STRING doc_id
    STRING token
    INT64 frequency
    ARRAY<INT64> pages
  }
  USER_IASSISTANT {
    INT64 USER_ID PK
    STRING USER_NAME
    STRING USER_EMAIL
    STRING REGION "lat,lon"
    INT64  DT_CARGA
  }
  PHRASE_METRICS {
    STRING  user_id FK
    FLOAT64 PHRASE_METRICS
    STRING  PHRASE_TEXT
    INT64   PAGE_METRICS
    INT64   PAGE_NUMBER
    STRING  SEARCH_ITEMS
    STRING  SEARCH_RESULT
    INT64   DT_CARGA
  }
  USER_IASSISTANT ||--o{ PHRASE_METRICS : "user_id = CAST(USER_ID AS STRING)"

Instalação rápida

Backend (Python 3.9+)

cd IAssistant/backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt
python -c "import nltk; nltk.download('punkt'); nltk.download('stopwords')"


Crie .env (a partir do .env.example) e ajuste:

API_CORE_HOST=0.0.0.0
API_CORE_PORT=8080
DASH_HOST=0.0.0.0
DASH_PORT=8081
CORS_ALLOWED_ORIGINS=*

BQ_ENABLED=true
BQ_PROJECT_ID=case-de-engenharia-de-dados
BQ_DATASET=iaassistant
BQ_TABLE_LOGS=logs
BQ_TABLE_PAGES=pages
BQ_TABLE_TERMS=terms
GOOGLE_APPLICATION_CREDENTIALS=/abs/path/service-account.json

DELTA_ENABLED=false
DELTA_OUTPUT_PATH=dbfs:/mnt/extension_data/processed/

SQL_ENABLED=false
SQL_SERVER=YOUR_SQL_SERVER_ADDRESS
SQL_DATABASE=YOUR_DATABASE_NAME
SQL_USERNAME=YOUR_USERNAME
SQL_PASSWORD=YOUR_PASSWORD
SQL_TABLE=ExtensionData

OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini
OPENAI_AGENT_ID=
OPENAI_INSTRUCTIONS="Responda com até {0} opções claras..."


Suba os serviços (2 terminais):

python dash_server.py            # porta 8080
python services/dash_process.py  # porta 8081


Extensão (Chrome)

chrome://extensions → Modo dev → Carregar sem compactação → IAssistant/extension.

Em index.html, ajuste:

<script>
  const defaultConfig = {
    API_METRICS_URL: "http://localhost:8080/api/metrics",
    API_GRAPHICS_URL: "http://localhost:8080/graphics",
    API_UPLOAD_URL:  "http://localhost:5000/uploadData",
    API_GPT_URL:     "http://localhost:8080/gpt/complete"
  };
</script>


BigQuery (dataset analítico IAssistant)

CREATE SCHEMA IF NOT EXISTS `case-de-engenharia-de-dados.iaassistant`;

CREATE TABLE IF NOT EXISTS `case-de-engenharia-de-dados.iaassistant.logs` (
  ts TIMESTAMP, page_url STRING, query STRING, lang STRING,
  source_type STRING, source_url STRING, client_version STRING, user_id STRING
);

CREATE TABLE IF NOT EXISTS `case-de-engenharia-de-dados.iaassistant.pages` (
  ts TIMESTAMP, doc_id STRING, page_number INT64, page_metric_by_page FLOAT64
);

CREATE TABLE IF NOT EXISTS `case-de-engenharia-de-dados.iaassistant.terms` (
  ts TIMESTAMP, doc_id STRING, token STRING, frequency INT64, pages ARRAY<INT64>
);


BigQuery (case governança: base + views)

BASE="case-de-engenharia-de-dados:845415315315"
LOC=$(bq show --format=prettyjson "$BASE" | jq -r .location)
bq rm -f -d case-de-engenharia-de-dados:sec_v 2>/dev/null || true
bq mk --dataset --location="$LOC" case-de-engenharia-de-dados:sec_v


Views com máscara:

CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.user_iassistant_v` AS
SELECT
  CAST(USER_ID AS INT64) AS user_id,
  CONCAT(SUBSTR(LOWER(TRIM(USER_EMAIL)),1,1),'***',
         REGEXP_EXTRACT(LOWER(TRIM(USER_EMAIL)), r'(@.*)$')) AS email_mascarado,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>0,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(0)] AS FLOAT64), NULL), 2
  ) AS lat_mascarada,
  ROUND(
    IF(ARRAY_LENGTH(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)'))>1,
       CAST(REGEXP_EXTRACT_ALL(REGION, r'(-?[0-9]+(?:\.[0-9]+)?)')[OFFSET(1)] AS FLOAT64), NULL), 2
  ) AS lon_mascarada
FROM `case-de-engenharia-de-dados.845415315315.user_iassistant`;

CREATE OR REPLACE VIEW `case-de-engenharia-de-dados.sec_v.phrase_metrics_v` AS
SELECT
  CAST(user_id AS STRING) AS user_id,
  CAST(PHRASE_METRICS AS FLOAT64) AS phrase_metrics,
  IFNULL(SUBSTR(PHRASE_TEXT, 1, 20) || '...', NULL) AS phrase_text_masked,
  CAST(PAGE_METRICS AS INT64) AS page_metrics,
  CAST(PAGE_NUMBER AS INT64) AS page_number,
  SEARCH_ITEMS AS search_items,
  IFNULL(SUBSTR(SEARCH_RESULT, 1, 60) || '...', NULL) AS search_result_masked,
  CAST(DT_CARGA AS INT64) AS dt_carga
FROM `case-de-engenharia-de-dados.845415315315.phrase_metrics`;


Autorizar as views no dataset base:

bq show --format=prettyjson "$BASE" > base.json
jq '.access += [
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"user_iassistant_v"}},
  {"view":{"projectId":"case-de-engenharia-de-dados","datasetId":"sec_v","tableId":"phrase_metrics_v"}}
] | .access |= (map(tojson)|unique|map(fromjson))' base.json > base_patched.json
bq update --source=base_patched.json "$BASE"

Canvas 4 — Scripts (copy-paste)

Cada bloco abaixo é um canvas de script com o arquivo inteiro. Copie o que precisar.

->  backend/dash_server.py
from IAUtils import IAssistantRequests,IAssistantRuntimeData
from flask import Flask
from flask import redirect, request
from urllib import parse
from flask_cors import CORS
from client import sendQuestionGenIA
import init_dash_servers
import os, base64, urllib, time
from flask import send_file

diretorio_atual = os.path.dirname(os.path.abspath(__file__))
pasta_static = os.path.join(diretorio_atual, 'static')
os.makedirs(pasta_static, exist_ok=True)

runtime_data = IAssistantRuntimeData()
ia_download_manager = IAssistantRequests()

def start_api_server():
    app_core = Flask(__name__)
    CORS(app_core, origins='*', allow_headers='*')

    @app_core.route('/')
    def index():
        caminho_index = os.path.abspath(os.path.join(os.path.dirname(__file__), '..', 'index.html'))
        return send_file(caminho_index)

    @app_core.route('/graphics')
    def send_file_graphics():
        return redirect("http://localhost:8081/index.html")

    @app_core.route('/configurations')
    def get_configurations():
        import json
        url_tmp  = urllib.parse.quote(runtime_data.current_url)
        return json.dumps({'url': url_tmp}), 200

    @app_core.route('/search/options')
    def get_input_data():
        import json
        return json.dumps(runtime_data.current_inputs), 200

    @app_core.route('/api/v1/document', methods=['POST'])
    def create_document():
        data = request.json
        inputSearch = parse.unquote(base64.b64decode(data["inputSearchTarget"]))
        listOptionsGenIA = data['listOptionsGenIA']
        typ = data["type"]
        runtime_data.current_site_url = data['url']

        if typ=='text':
          contentDocument = parse.unquote(base64.b64decode(data["contentText"]).decode('utf-8'))
          binary_content = parse.unquote(base64.b64decode(data["BinaryContent"]).decode('utf-8'))
          runtime_data.current_url = ia_download_manager.save_text_file_binary_content(
              content=binary_content, filename='ia_html_binary_file_output', code_content_to_filename=0)
        elif typ=='pdf':
          binary_content = base64.b64decode(data['BinaryContent'])
          contentDocument = parse.unquote(base64.b64decode(data["pdf"]).decode('utf-8'))
          runtime_data.current_url = ia_download_manager.save_pdf_file_binary_content(content=binary_content)

        output = sendQuestionGenIA(inputSearch, contentDocument,
                                   number_response_options=listOptionsGenIA, creativity_degree=1.5)

        runtime_data.current_inputs = {
          'inputSearchTarget': inputSearch,
          'search_options': output,
          'tabId': data['tabId'],
          'listOptionsGenIA': listOptionsGenIA,
          'doc': contentDocument,
          'url': runtime_data.current_site_url
        }

        init_dash_servers.init_dashboard_services()
        time.sleep(15)
        return "output.html", 201

    return app_core

if __name__ == '__main__':
   app = start_api_server()
   app.run(port=8080,host='0.0.0.0', debug=True)

-> backend/services/dash_process.py
import IAUtils, time, requests
from IAUtils import start_server

for i in range(10):
    try:
        requests.get("http://localhost:8080/configurations", timeout=1)
        break
    except requests.exceptions.RequestException:
        print(f"Aguardando API (tentativa {i+1}/10)...")
        time.sleep(1)
else:
    raise RuntimeError("API nunca ficou disponível. Abortando dashboard.")

app_dash = start_server()
app_dash.run(port=8081, host="0.0.0.0")

-> backend/client.py (OpenAI opcional)
from openai import OpenAI
from dotenv import load_dotenv
import os
load_dotenv()

def sendQuestionGenIA(message,inputDocument,number_response_options,creativity_degree):
    client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
    completion = client.chat.completions.create(
        max_tokens=1200,
        model=os.getenv("OPENAI_MODEL"),
        temperature=creativity_degree,
        messages=[
          {"role": "system", "content": os.getenv("OPENAI_INSTRUCTIONS").format(number_response_options)},
          {"role": "user", "content": message}
        ]
    )
    return completion.choices[0].message.content

if __name__=="__main__":
    print("="*20,"\nLoaded IAssistant Module!!!\n","="*20)

-> backend/api_uploader.py (Spark/Delta + SQL)
from flask import Flask, request, jsonify
import logging, json, pyodbc
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, udf
from pyspark.sql.types import StringType

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger(__name__)
app = Flask(__name__)

spark = SparkSession.builder.appName("ChromeExtensionUploader").getOrCreate()

def mask_email(email):
    if email and "@" in email:
        parts = email.split("@")
        return parts[0][:2] + "*" * max(len(parts[0])-2,0) + "@" + parts[1]
    return email

mask_email_udf = udf(mask_email, StringType())

def update_on_premise_sql(data):
    try:
        connection_string = (
            "Driver={ODBC Driver 17 for SQL Server};"
            "Server=YOUR_SQL_SERVER_ADDRESS;"
            "Database=YOUR_DATABASE_NAME;"
            "UID=YOUR_USERNAME;"
            "PWD=YOUR_PASSWORD;"
        )
        cnxn = pyodbc.connect(connection_string)
        cursor = cnxn.cursor()
        cursor.execute("INSERT INTO ExtensionData (data_json) VALUES (?)", json.dumps(data))
        cnxn.commit(); cursor.close(); cnxn.close()
        logger.info("Dados inseridos no SQL on premise.")
    except Exception as e:
        logger.error("Erro ao atualizar SQL on premise: %s", str(e))
        raise e

@app.route("/uploadData", methods=["POST"])
def upload_data():
    try:
        content = request.get_json()
        logger.info("Dados recebidos: %s", content)
        df = spark.createDataFrame([content])
        if "email" in df.columns:
            df = df.withColumn("email_masked", mask_email_udf(col("email"))).drop("email")
        if df.count() < 1:
            return jsonify({"status":"error","message":"Nenhum dado recebido!"}), 400
        df.write.format("delta").mode("append").save("dbfs:/mnt/extension_data/processed/")
        update_on_premise_sql(content)
        return jsonify({"status":"success","message":"Dados recebidos e processados."}), 200
    except Exception as e:
        logger.error("Erro ao processar: %s", str(e))
        return jsonify({"status":"error","message":str(e)}), 500

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000, debug=True)

- > backend/bigquery_function.js (HTTP → BigQuery)
const { BigQuery } = require('@google-cloud/bigquery');
const bigquery = new BigQuery();

const datasetId = 'case-de-engenharia-de-dados';
const tableId = 'search_iassistent';

exports.insertSearchData = async (req, res) => {
  if (req.method !== 'POST') { res.status(405).send('Method Not Allowed'); return; }
  try {
    const body = req.body;
    if (!body.timestamp) body.timestamp = new Date().toISOString();

    const row = {
      timestamp: body.timestamp,
      user_id: body.user_id,
      search_query: body.search_query,
      best_answer: body.best_answer,
      percent: body.percent || null
    };

    await bigquery.dataset(datasetId).table(tableId).insert([row]);
    res.status(200).send('Data inserted successfully!');
  } catch (err) {
    console.error('Error inserting data: ', err);
    res.status(500).send('Error inserting data.');
  }
};

Canvas 5 — Auditoria (Data Access Logs) e IAM

Habilite Data Access Logs (Console > IAM e Admin > Audit Logs > BigQuery > Data Read/Write).

Checagens rápidas:

# leitura direta na base
gcloud logging read \
  'logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" \
   AND resource.type="bigquery_dataset" \
   AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" \
   AND protoPayload.metadata.tableDataRead.reason:* \
   AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"' \
  --limit=5 \
  --format='value(timestamp, protoPayload.metadata.tableDataRead.reason, protoPayload.resourceName)'


Métricas que criei:

# leituras na base
gcloud logging metrics create read_base_dataset_metric \
  --description="Leituras na tabela base (bypass da view)" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" AND resource.type="bigquery_dataset" AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" AND protoPayload.metadata.tableDataRead.reason:* AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'

# queries referenciando a base
gcloud logging metrics create query_ref_base_table_metric \
  --description="Queries que referenciam a tabela base" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" AND resource.type="bigquery_project" AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" AND protoPayload.metadata.jobCompletedEvent.job.jobStatistics.referencedTables:"projects/case-de-engenharia-de-dados/datasets/845415315315/tables/user_iassistant"'

# leituras via view
gcloud logging metrics create read_via_view_metric \
  --description="Leituras na authorized view" \
  --log-filter='logName="projects/case-de-engenharia-de-dados/logs/cloudaudit.googleapis.com%2Fdata_access" AND resource.type="bigquery_dataset" AND protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.BigQueryAuditMetadata" AND protoPayload.metadata.tableDataRead.reason:* AND protoPayload.resourceName:"projects/case-de-engenharia-de-dados/datasets/sec_v/tables/user_iassistant_v"'


IAM dinâmico por dataset (exemplo):

PROJECT_ID="case-de-engenharia-de-dados"
DATASET_ID="845415315315"
DATASET_REF="$PROJECT_ID:$DATASET_ID"
SRC_TABLE="$PROJECT_ID.$DATASET_ID.user_iassistant"

bq query --nouse_legacy_sql --format=csv "
WITH base AS (
  SELECT DISTINCT CAST(USER_ID AS INT64) AS user_id,
         LOWER(TRIM(USER_EMAIL)) AS email
  FROM \`$SRC_TABLE\`
  WHERE USER_EMAIL IS NOT NULL
)
SELECT email,
       CASE WHEN user_id BETWEEN 1 AND 5 THEN 'READER' ELSE 'WRITER' END AS role
FROM base
" | tail -n +2 | while IFS=, read -r email role; do
  [[ -z "$email" || -z "$role" ]] && continue
  bq show --format=prettyjson "$DATASET_REF" > ds.tmp.json
  jq --arg em "$email" --arg r "$role" \
     '.access += [{"role":$r,"userByEmail":$em}] | .access |= (map(tojson)|unique|map(fromjson))' \
     ds.tmp.json > ds.apply.json
  bq update --source=ds.apply.json "$DATASET_REF"
  rm -f ds.tmp.json ds.apply.json
done

Consultas úteis (BigQuery)
-- páginas mais “quentes” por relevância (IAssistant)
SELECT *
FROM `case-de-engenharia-de-dados.iaassistant.pages`
WHERE page_number > 2 AND page_metric_by_page < 0.25
ORDER BY ts DESC;

-- top tokens por documento (IAssistant)
SELECT doc_id, token, SUM(frequency) AS freq
FROM `case-de-engenharia-de-dados.iaassistant.terms`
GROUP BY doc_id, token
ORDER BY freq DESC
LIMIT 50;

-- auditoria de origens (IAssistant)
SELECT page_url, COUNT(*) AS hits
FROM `case-de-engenharia-de-dados.iaassistant.logs`
GROUP BY page_url
ORDER BY hits DESC
LIMIT 100;

Notas finais

Extensão manda contexto; APIs processam e persistem; Dash explica; BigQuery governa.

Máscaras nas views, Data Access Logs para auditoria, e métricas para sinalizar bypass.

OpenAI aqui é para resumo/rotulagem, não decide ranking (o ranking é matemático e auditável).
