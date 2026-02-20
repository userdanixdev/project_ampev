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

## 📦 Dados Ingeridos
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



## ⚙️ Tecnologias Utilizadas:

- Azure Databricks
- Unity Catalog
- Delta Lake
- Auto Loader (cloudFiles)
- GitHub (via Databricks Repos)
- Jobs (Workflows)

## 🔄 Estratégia de Ingestão:

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

## Metadados de ingestão explícito:

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

## 🔐 Governança (Unity Catalog)

As tabelas são criadas em:

 - ampev.bronze.estabelecimentos
 - ampev.bronze.pedidos

## 🔍 Auditoria do Pipeline:

Foi implementado script de auditoria automática para validação de:

- Existência da tabela
- Quantidade de registros
- Presença de metadados
- Histórico Delta
- Existência de checkpoint


## 🧪 Controle de Execução:

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


## 📌 Boas Práticas Aplicadas:

- Separação por entidade
- Schema explícito
- Uso de checkpoints dedicados
- Uso de _metadata.file_path
- Execução via Job
- Estrutura padronizada de diretórios
- Auditoria automatizada

## 📈 Próximos Passos (Roadmap):

- Implementar camada Silver (join pedidos ↔ estabelecimentos)
- Normalização de tipos (converter data_venda para DateType)
- Adicionar monitoramento via alertas
- Criar branch strategy (dev/main)

## 🎯 Status Atual:

- ✅ Volume criado
- ✅ Estrutura organizada
- ✅ Auto Loader configurado
- ✅ Schema congelado
- ✅ WriteStream configurado
- ✅ Job configurado
- ✅ Integração com Git funcionando

Pipeline Bronze operacional.

# 📦 Camada Silver — Fato Pedidos (SCD Type 2)

## 📌 Objetivo

Construir a tabela `ampev.silver.fato_pedidos_scd2` aplicando:

- Padronização de schema
- Tipagem correta
- Deduplicação por ingestão
- Controle de histórico via SCD Type 2
- Chave composta (`PedidoID`, `EstabelecimentoID`)
- Hash para detecção de mudanças

---

# 🏗 Arquitetura

```
Bronze (raw)
    ↓
Staging (deduplicação + padronização + hash)
    ↓
SCD2 (Delta Lake)
```

---

# 🥉 Fonte Bronze

Tabela origem:

```sql
ampev.bronze.pedidos
```

Contém:
- Dados crus
- Metadados de ingestão (`_ingest_ts`, `_source_file`)

---

# 🧹 1. Staging (Padronização e Deduplicação)

### ✔ Tipagem aplicada

| Coluna | Tipo Final |
|---------|------------|
| PedidoID | BIGINT |
| EstabelecimentoID | BIGINT |
| Produto | STRING |
| quantidade_vendida | BIGINT |
| Preco_Unitario | DOUBLE |
| data_venda | DATE |

---

### ✔ Deduplicação

Aplicado `row_number()` com janela:

```python
Window.partitionBy("PedidoID", "EstabelecimentoID")
      .orderBy(F.col("_ingest_ts").desc())
```

Mantém apenas:

```
_rn = 1 → versão mais recente por chave composta
```

---

# 🔐 2. Hash de Atributos

Criado `_attr_hash` com:

```python
F.sha2(
    F.concat_ws("||",
        Produto,
        quantidade_vendida,
        Preco_Unitario,
        data_venda
    ),
    256
)
```

### 🎯 Objetivo

Detectar alterações em qualquer atributo do pedido sem comparar coluna por coluna.

---

# 🏛 3. Estrutura da Tabela SCD2

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

# 🔄 4. Lógica SCD Type 2

## 📌 Chave Composta

A chave de negócio é:

```
(PedidoID, EstabelecimentoID)
```

---

## 🆕 Novos Registros

Quando não existe versão atual:

```
_is_new = TRUE
```

→ Insere nova linha como:

- `start_date = current_date()`
- `end_date = 9999-12-31`
- `is_current = TRUE`

---

## 🔁 Registros Alterados

Quando `_attr_hash` é diferente:

1. Fecha versão atual:
   - `end_date = current_date() - 1`
   - `is_current = FALSE`

2. Insere nova versão com novos atributos.

---

# 📊 Exemplo de Histórico

| PedidoID | EstabelecimentoID | Produto | start_date | end_date | is_current |
|-----------|------------------|----------|------------|----------|------------|
| 1 | 1 | Cerveja | 2026-02-01 | 2026-02-10 | FALSE |
| 1 | 1 | Cerveja Premium | 2026-02-11 | 9999-12-31 | TRUE |

---

# 🔎 Consulta de Histórico

```sql
SELECT *
FROM ampev.silver.fato_pedidos_scd2
WHERE PedidoID = 1
  AND EstabelecimentoID = 1
ORDER BY start_date;
```

---

# 🚀 Benefícios da Implementação

- Histórico completo de alterações
- Rastreabilidade (bronze → silver)
- Performance otimizada (comparação por hash)
- Controle via `is_current`
- Compatível com camadas Gold e Modelagem Dimensional

---

# ⚠ Considerações Técnicas

### 1️⃣ Data Sentinela

`9999-12-31` é utilizada como padrão para registros vigentes.

### 2️⃣ Execução Diária

A lógica assume carga diária.  
Para cargas intradiárias recomenda-se uso de `TIMESTAMP`.

### 3️⃣ Performance

A comparação é feita apenas contra registros `is_current = TRUE`.

---

# 🧠 Conclusão

A implementação do SCD Type 2 na tabela fato permite:

- Manter histórico completo
- Evitar sobrescrita de dados
- Garantir integridade temporal
- Suportar análises históricas

---

**Autor:** Projeto AMPEV  
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