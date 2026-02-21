## project_ampev

## 📊 Projeto AMPEV – Pipeline de Ingestão com Azure Databricks

### 🧭 Executive Summary:

Este projeto implementa um pipeline de ingestão incremental utilizando Azure Databricks, Unity Catalog e Delta Lake, seguindo a arquitetura Medallion (Bronze → Silver → Gold).

#### Contexto de Negócio (AMPEV — Fabricantes de Bebidas)

A AMPEV atua como fabricante e distribuidora de bebidas, atendendo uma rede de estabelecimentos clientes (varejo, atacado, bares, mercados e distribuidores regionais). No dia a dia, a gestão comercial precisa acompanhar demanda, desempenho de portfólio e concentração de receita para tomar decisões rápidas sobre produção, logística, negociação e estratégia de vendas.

Dentro desse cenário, as áreas de Vendas, Comercial e Planejamento fazem perguntas recorrentes para entender:

Quem são os principais clientes (para priorização de atendimento, condições comerciais e retenção);

Quais produtos têm maior giro (para planejamento de estoque/produção e reposição);

Quais produtos geram mais receita (para decisões de preço, mix, campanhas e margem).

Essas análises precisam ser respondidas com base em dados confiáveis e padronizados, evitando divergências entre relatórios. Por isso, na arquitetura do lakehouse, a camada GOLD concentra as visões analíticas prontas para consumo (BI/relatórios), unificando os dados tratados na Silver em um modelo orientado ao negócio.

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

### 🥇 Camada GOLD — AMPEV Lakehouse
#### 📌 Visão Geral

A camada GOLD representa o nível analítico do Lakehouse da AMPEV – Fabricantes de Bebidas.

Enquanto:

- Bronze → Ingestão bruta (Auto Loader)

- Silver → Dados tratados, tipados e com controle histórico (SCD Type 2)

- Gold → Dados prontos para consumo de negócio (BI / relatórios / dashboards)

> A GOLD consolida as informações através de joins entre Fato e Dimensão, aplicando regras de negócio e agregações para responder perguntas estratégicas do gerente comercial.

### 🎯 Objetivo de Negócio

A camada GOLD foi construída para responder às seguintes perguntas estratégicas:

- Quais são as 5 empresas que mais compraram nossos produtos?
- Quais são os 5 produtos mais vendidos?
- Quais são os 5 produtos que mais geraram faturamento?

Essas respostas suportam decisões relacionadas a planejamento de produção, estratégia comercial, gestão de clientes-chave, otimização de mix de produtos, negociação e retenção de grandes compradores.

### 🏗 Arquitetura da GOLD

A camada GOLD é construída a partir das tabelas SCD2 da Silver:

> silver.fato_pedidos_scd2

> silver.dim_estabelecimentos_scd2

**Somente os registros vigentes (is_current = TRUE) são considerados nas análises atuais.**

### 🔄 View Base Analítica
> ampev.gold.vw_vendas_atual

Responsável por:

- Realizar o JOIN entre fato e dimensão
- Calcular métricas (ex: valor_total)
- Entregar dataset limpo para agregações

**Estrutura lógica:**
```
FROM silver.fato_pedidos_scd2 p
LEFT JOIN silver.dim_estabelecimentos_scd2 e
  ON p.EstabelecimentoID = e.EstabelecimentoID
 AND e.is_current = TRUE
WHERE p.is_current = TRUE
```
> Métrica derivada:
- valor_total = quantidade_vendida * Preco_Unitario

### 📊 Views Analíticas (Top 5)
#### 🏆 Top 5 Empresas que Mais Compraram

Baseado em:

- Soma de unidades vendidas
- Soma de faturamento total

Entrega:

- Identificação dos principais clientes
- Apoio à estratégia comercial

### 🥤Top 5 Produtos Mais Vendidos

Baseado em:

Som>  de quantidade_vendida

Entrega:

- Identificação de produtos com maior giro
- Suporte ao planejamento de estoque

#### 💰 Top 5 Produtos com Maior Faturamento

Baseado em:

> Soma de valor_total

Entrega:

- Identificação dos produtos com maior impacto financeiro
- Apoio a decisões de precificação

#### 🧠 Decisão Técnica Importante
```
Uso de registros vigentes (is_current = TRUE)
Como a Silver utiliza SCD Type 2, existem múltiplas versões históricas.
A GOLD utiliza apenas `is_current = TRUE`
Isso garante que as análises representem o estado atual do negócio.
Caso seja necessário análise histórica, pode-se remover esse filtro e trabalhar com start_date e end_date.
```

### ⚙️ Boas Práticas Aplicadas

- Separação clara entre camada histórica (Silver) e analítica (Gold)
- Métricas calculadas na camada de consumo
- Views reutilizáveis para BI

**Estrutura pronta para integração com:**
- Power BI
- Databricks SQL Dashboard
- Jobs agendados

### 🚀 Resultado

A camada GOLD transforma dados transacionais em insights estratégicos, permitindo que a AMPEV tenha:

- Visão clara dos principais clientes
- Controle sobre produtos de maior giro
- Monitoramento do faturamento por produto
- Base sólida para tomada de decisão

### 📊 Dashboard Executivo — Databricks SQL

> O Dashboard Executivo foi desenvolvido no Databricks SQL com o objetivo de transformar os dados consolidados na camada GOLD em insights visuais para tomada de decisão gerencial.

Ele responde diretamente às principais perguntas do gerente da AMPEV:

- Quais são as 5 empresas que mais compraram?
- Quais são os 5 produtos mais vendidos?
- Quais são os 5 produtos que mais geraram faturamento?

### 🏗 Arquitetura do Dashboard

Fonte de dados:
```
ampev.gold.vw_vendas_atual
ampev.gold.top5_empresas_mais_compraram
ampev.gold.top5_produtos_mais_vendidos
ampev.gold.top5_produtos_maior_faturamento
```

Fluxo de dados:

> Bronze → Silver (SCD2) → Gold (Views Analíticas) → Dashboard

### 📈 Visualizações Implementadas
- Top 5 Empresas por Faturamento
- Tipo: Bar Chart
- Métrica: ``total_faturamento``
- Dimensão: estabelecimento

> Objetivo: Identificar principais clientes da AMPEV

- Top 5 Produtos Mais Vendidos
- Tipo: Bar Chart
- Métrica: ``total_unidades`` 
- Dimensão: produto

Ob> jetivo: Identificar produtos com maior giro

- Top 5 Produtos por Faturamento:

- Tipo: Bar Chart
- Métrica: total_faturamento
- Dimensão: produto

> Objetivo: Identificar produtos com maior impacto financeiro

- **KPIs (Cards Executivos)**
```
Total de Faturamento Geral
Total de Unidades Vendidas
Total de Pedidos
```
> Esses indicadores permitem visão rápida da performance comercial.

### 🔄 Atualização dos Dados
```
O dashboard está conectado diretamente às views da camada GOLD.
Sempre que a camada Bronze recebe novos dados, a Silver aplica transformação e SCD2 e a Gold recalcula as views
O Dashboard reflete automaticamente os dados atualizados.
```

Pode ser configurado:

Refresh manual
Refresh agendado
Integração com Databricks Jobs

### 📦 Versionamento no GitHub

O repositório inclui:

```
/dashboards/ampev_dashboard.json
/docs/images/dashboard_top5_empresas.png
/docs/images/dashboard_produtos.png
```

> JSON exportado do Databricks SQL Dashboard
> Screenshots das visualizações
> Documentação técnica

```Isso garante reprodutibilidade e governança do projeto.```

### 🚀 Valor Estratégico:

O Dashboard Executivo transforma dados transacionais em uma visão clara dos principais clientes, controle sobre os produtos mais estratégicos e monitoramento contínuo de faturamento. Base confiável para decisões comerciais

Ele representa a camada final de entrega do projeto Lakehouse da AMPEV, conectando engenharia de dados à estratégia de negócio.


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