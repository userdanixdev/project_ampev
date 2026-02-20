# project_ampev

## 📊 Projeto AMPEV – Pipeline de Ingestão com Azure Databricks

### 🧭 Executive Summary:

Este projeto implementa um pipeline de ingestão incremental utilizando Azure Databricks, Unity Catalog e Delta Lake, seguindo a arquitetura Medallion (Bronze → Silver → Gold).

A camada Bronze já está totalmente operacional, com:

- Ingestão incremental via Auto Loader (cloudFiles)
- Governança com Unity Catalog
- Controle de schema explícito
- Metadados de ingestão
- Execução automatizada via Jobs
- Versionamento com GitHub (Databricks Repos)

### 🏗️ Arquitetura:
```
Volumes (Landing Zone)
        ↓
Auto Loader (cloudFiles)
        ↓
Delta Tables (Bronze Layer - Unity Catalog)
        ↓
(Silver Layer - futura etapa)
```

### 📁 Estrutura do Volume:

Volume utilizado:

> /Volumes/ampev/bronze/landings

**Estrutura organizada:**

```
landings/
│
├── estabelecimentos/
├── pedidos/
│
├── samples/
│   ├── estabelecimentos/
│   └── pedidos/
│
├── _schemas/
│   ├── estabelecimentos/
│   └── pedidos/
│
└── _checkpoints/
    ├── estabelecimentos/
    └── pedidos/
```

### 📦 Dados Ingeridos
 ### 📄 estabelecimentos.csv

| Campo             | Tipo   |
| ----------------- | ------ |
| Local             | String |
| Email             | String |
| EstabelecimentoID | Long   |
| Telefone          | String |


### 📄 pedidos.csv:

| Campo              | Tipo                                |
| ------------------ | ----------------------------------- |
| PedidoID           | Long                                |
| EstabelecimentoID  | Long                                |
| Produto            | String                              |
| quantidade_vendida | Long                                |
| Preco_Unitario     | Double                              |
| data_venda         | String (futura conversão para Date) |



### ⚙️ Tecnologias Utilizadas:

- Azure Databricks
- Unity Catalog
- Delta Lake
- Auto Loader (cloudFiles)
- GitHub (via Databricks Repos)
- Jobs (Workflows)

### 🔄 Estratégia de Ingestão:

> Auto Loader separado por entidade:

- Cada entidade possui: 
        - Pasta exclusiva
        - SchemaLocation exclusivo
        - Checkpoint exclusivo
        - Tabela Delta exclusiva

```Isso evita mistura de dados e garante isolamento.```

> Schema congelado (bootstrap controlado):

Os schemas foram inferidos a partir de arquivos sample e posteriormente congelados usando StructType(...), garantindo:

- Controle de tipos
- Estabilidade do pipeline
- Evitar inferência automática incorreta
- Permitir validação futura de mudanças de layout

### Metadados de ingestão explícito:

Durante o writeStream são adicionadas colunas técnicas:

- ```_ingest_ts``` → timestamp da ingestão
- ```_source_file``` → caminho do arquivo original (via _metadata.file_path)

### 🚀 Execução do Pipeline:

> Pipeline executado via Databricks Job agendado.

```
Configuração recomendada:
Trigger: Scheduled
Frequência: a cada 10 minutos
Trigger type: availableNow=True
```

> Fluxo:

- Arquivo é colocado na pasta landing
- Job executa
- Auto Loader processa apenas novos arquivos
- Dados são gravados na Bronze
- Job encerra

### 🔐 Governança (Unity Catalog)

As tabelas são criadas em:

 - ampev.bronze.estabelecimentos
 - ampev.bronze.pedidos

### 🔍 Auditoria do Pipeline:

Foi implementado script de auditoria automática para validação de:

- Existência da tabela
- Quantidade de registros
- Presença de metadados
- Histórico Delta
- Existência de checkpoint


### 🧪 Controle de Execução:

O pipeline não inicia se a pasta estiver vazia:

> has_files(path)

Isso evita:

- Erros de inferência
- Criação de checkpoint vazio
- Execuções desnecessárias

## 🔄 Controle de Versionamento:

Integração via Databricks Repos:

> Repos/<usuario>/<repositorio>


Fluxo:

- Clone do repositório GitHub
- Commit & Push via UI do Databricks


### 📌 Boas Práticas Aplicadas:

- Separação por entidade
- Schema explícito
- Uso de checkpoints dedicados
- Uso de _metadata.file_path
- Execução via Job
- Estrutura padronizada de diretórios
- Auditoria automatizada

### 📈 Próximos Passos (Roadmap):

- Implementar camada Silver 
- Normalização de tipos (converter data_venda para DateType)
- Criar branch strategy (dev/main)

### 🎯 Status Atual:

- ✅ Volume criado
- ✅ Estrutura organizada
- ✅ Auto Loader configurado
- ✅ Schema congelado
- ✅ WriteStream configurado
- ✅ Job configurado
- ✅ Integração com Git funcionando

Pipeline Bronze operacional.

### 📦 Camada Silver — Implementação SCD Type 2

### Objetivo

Implementar controle de histórico (Slowly Changing Dimension Type 2) nas tabelas da camada Silver:

- `ampev.silver.dim_estabelecimentos_scd2`
- `ampev.silver.fato_pedidos_scd2`

A solução garante:

- Histórico completo de alterações
- Rastreabilidade Bronze → Silver
- Controle temporal
- Comparação eficiente via hash
- Compatibilidade com modelagem dimensional (Gold)

---

### 🏗 Arquitetura Geral

```
Bronze (raw ingest)
        ↓
Staging (padronização + deduplicação + hash)
        ↓
Silver SCD2 (Delta Lake)
```
---

### 🏢 Tabela Dimensão — Estabelecimentos (SCD2)

### Tabela:

```sql
ampev.silver.dim_estabelecimentos_scd2
```

### 🔑 Chave de Negócio:

```
EstabelecimentoID
```

### 🧱 Estrutura

```sql
CREATE TABLE ampev.silver.dim_estabelecimentos_scd2 (
  surrogate_key BIGINT GENERATED ALWAYS AS IDENTITY,

  EstabelecimentoID BIGINT,
  Local STRING,
  Email STRING,
  Telefone STRING,

  start_date DATE,
  end_date DATE,
  is_current BOOLEAN,

  _attr_hash STRING,
  _bronze_ingest_ts TIMESTAMP,
  _bronze_source_file STRING,
  _silver_ts TIMESTAMP
)
USING DELTA;
```

---

### 🔄 Lógica SCD2 — Dimensão

### 🆕 Novo Estabelecimento
```
Quando '_is_new' = TRUE, significa que o 'EstabelecimentoID' ainda não existe na dimensão (silver.dim_estabelecimentos_scd2).
Ou seja, estamos lidando com um registro totalmente novo, e não com uma atualização.
```

- `_is_new = TRUE`
- Insere nova linha:
  - `start_date = current_date()` -> Data em que o registro passa a ser válido
  - `end_date = 9999-12-31` ->  -> Data futura simbólica indicando que o registro está ativo
  - `is_current = TRUE` -> Indica que esta é a versão atual do estabelecimento

> No caso de um novo estabelecimento não há histórico anterior. Ele já nasce como a versão vigente. Futuras alterações gerarão novas > > > versões, preservando essa original.  

---

### 🔁 Alteração de Atributo

```
Quando o '_attr_hash' é diferente, significa que algum atributo relevante do estabelecimento foi alterado (ex: nome, cidade, categoria, etc.). Nesse caso, não sobrescrevemos o registro antigo. Aplicamos a estratégia de SCD Type 2, preservando o histórico.
Se `_attr_hash` for diferente fecha a versão atual:  - `end_date = current_date() - 1` e  - `is_current se torna 'FALSE'
e insere nova versão com dados atualizados.
```
---

### 📊 Exemplo Histórico

| EstabelecimentoID | Email | start_date | end_date | is_current |
|------------------|--------|------------|----------|------------|
| 1 | antigo@email.com | 2026-02-01 | 2026-02-10 | FALSE |
| 1 | novo@email.com   | 2026-02-11 | 9999-12-31 | TRUE |

---

### 🧾 Tabela Fato — Pedidos (SCD2)

### Tabela

```sql
ampev.silver.fato_pedidos_scd2
```

### 🔑 Chave de Negócio (Composta)

```
(PedidoID, EstabelecimentoID)
```

---

### 🧱 Estrutura:

```sql
CREATE TABLE ampev.silver.fato_pedidos_scd2 (
  surrogate_key BIGINT GENERATED ALWAYS AS IDENTITY,

  PedidoID BIGINT,
  EstabelecimentoID BIGINT,
  Produto STRING,
  quantidade_vendida BIGINT,
  Preco_Unitario DOUBLE,
  data_venda DATE,

  start_date DATE,
  end_date DATE,
  is_current BOOLEAN,

  _attr_hash STRING,
  _bronze_ingest_ts TIMESTAMP,
  _bronze_source_file STRING,
  _silver_ts TIMESTAMP
)
USING DELTA;
```
---
### 🧹 Staging

```A etapa de staging serve para deixar a fonte “pronta” antes de aplicar regras de negócio (ex: SCD2). Aqui a ideia é padronizar, limpar e criar colunas técnicas que facilitem o processamento.
```

- `Tipagem correta` -> Garanta que cada coluna esteja no tipo esperado (ex.: PedidoID/EstabelecimentoID como numérico, quantidade como inteiro/decimal, etc.). Isso evita erro de join, comparação, agregação e escrita.
- Conversão `data_venda` → DATE
- Deduplicação via `row_number()` com chave composta -> Como podem existir múltiplas versões do mesmo registro na Bronze (reprocessamento/ingestões), você mantém apenas a versão mais recente.
- Hash dos atributos -> Crie um hash (ex.: _attr_hash) com os atributos de negócio relevantes (excluindo colunas técnicas) para detectar mudanças de forma simples. Se o hash não mudou → sem alteração real nos atributos. Se o hash mudou → houve alteração → dispara SCD2 (fechar versão antiga e criar nova)

---

## 🔄 Lógica SCD2 — Fato

### 🆕 Novo Pedido

Um pedido é considerado novo quando a chave do registro não existe entre os registros vigentes (is_current = TRUE) da sua tabela fato SCD2 (ex.: silver.fato_pedidos_scd2). Então é inserido como versão vigente.

---

### 🔁 Pedido Alterado

> Quando `_attr_hash` for diferente para um registro novo. Ocorer a fecha automática da versão atual e insere nova versão

---

## 📊 Exemplo Histórico

| PedidoID | EstabelecimentoID | Produto | start_date | end_date | is_current |
|----------|------------------|----------|------------|----------|------------|
| 1 | 1 | Cerveja | 2026-02-01 | 2026-02-10 | FALSE |
| 1 | 1 | Cerveja Premium | 2026-02-11 | 9999-12-31 | TRUE |

---

## 🔍 Consultas de Histórico

### Estabelecimentos

```sql
SELECT *
FROM ampev.silver.dim_estabelecimentos_scd2
WHERE EstabelecimentoID = 1
ORDER BY start_date;
```

### Pedidos

```sql
SELECT *
FROM ampev.silver.fato_pedidos_scd2
WHERE PedidoID = 1
  AND EstabelecimentoID = 1
ORDER BY start_date;
```
---
## ⚙️ Decisões Técnicas:

### 🔐 Uso de Hash
- Evita comparação coluna a coluna. Em vez de comparar cada atributo individualmente (nome, cidade, categoria, valor, etc.), geramos um hash (ex.: _attr_hash) a partir da concatenação dos atributos relevantes.

**Como funciona na prática?**
- Selecionamos apenas os atributos de negócio, concatenamos os valores (normalizados) e Aplicamos uma função de hash (ex.: sha2)
- Dessa forma, armazenamos como '_attr_hash' em coluna. Se o hash mudou então houve alteração real nos dados → aplica SCD2
- Se o hash é igual então nenhuma mudança relevante → não cria nova versão

#### Data Sentinela
`9999-12-31` representa registros vigentes.

#### ✔ is_current
Facilita filtros e melhora performance.

#### ✔ Delta Lake
Permite:
- MERGE
- UPDATE
- Time Travel
- Controle transacional

### Versionamento via branch: DEV

A branch dev representa o ambiente de desenvolvimento e testes do projeto.
O propósito da `Branch dev` é desenvolver novas features (ex: nova lógica SCD2), testar transformações (Bronze → Silver → Gold)
ajustar notebooks e Jobs do Databricks, validar performance e regras de qualidade. Ela funciona como um ambiente seguro, onde mudanças podem ser feitas sem impactar a produção.
---

### 🚀 Benefícios

- Histórico completo
- Auditoria Bronze → Silver
- Performance otimizada
- Compatível com modelagem dimensional
- Pronto para camada Gold

---

### 📌 Próximo Passo

Camada Gold:
- Fato consolidado
- Join com dimensão SCD2
- Métricas agregadas
- Performance otimizada

---

**Projeto:** AMPEV  
**Camada:** Silver  
**Padrão:** Delta Lake + SCD Type 2  

📈 Roadmap Estratégico
🔹 Próxima Fase – Silver Layer

Conversão data_venda para DateType

Join entre pedidos e estabelecimentos

Tratamento de dados inválidos

Deduplicação

🔹 Fase Gold

KPIs

Métricas agregadas

Camada para BI / Power BI

🔹 Melhorias Futuras

Alertas automáticos

Testes de qualidade de dados

CI/CD com branches dev/main

Monitoramento de SLA