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

## Metadados de ingestão:

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
- Implementar validação de qualidade (ex: quantidade_vendida > 0)
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