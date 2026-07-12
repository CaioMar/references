Este documento resume a arquitetura de **MLOps Lean-Batch** que desenhamos. Esta estrutura foi concebida para alta escalabilidade (suportando até 200+ modelos), baixo custo operacional e máxima agilidade para experimentação, eliminando dependências pesadas como SageMaker.
# Documento de Planejamento: Plataforma de ML Lean-Batch
## 1. Visão Geral
O sistema baseia-se em uma arquitetura de **Data-Centric MLOps**, onde a engenharia de dados é separada da engenharia de features (transformação). Utilizamos o **Athena** como motor de organização de dados e o **ECS Fargate (com Polars)** como motor de computação (inferência e features complexas).
## 2. Arquitetura de Componentes

| Componente | Ferramenta | Responsabilidade |
| :--- | :--- | :--- |
| **Storage** | S3 (Parquet) | Data Lake organizado por hash_bucket. |
| **Org. de Dados** | Athena (CTAS) | Particiona e otimiza os dados brutos (Raw -> Optimized). |
| **Orquestração** | Step Functions | Gerencia o pipeline de inferência, retentativas e paralelismo. |
| **Compilador SQL** | AWS Lambda | Gera queries dinâmicas (templates) para o Athena. |
| **Engine de Features** | ECS Fargate + Polars | Leitura lazy, joins e cálculo de features (lags, médias, etc). |
| **Agendamento** | EventBridge | Gatilho para o início do processamento. |

## 3. Fluxo de Execução (O "Pipeline Simétrico")
O sistema opera em duas fases principais: **Materialização** e **Inferência**.
### A. Fase de Preparação (Athena + CTAS)
Para evitar I/O desnecessário e permitir joins eficientes:
 1. **Bucketing:** Todos os "books" de dados são processados via **Athena CTAS** para adicionar uma coluna de hash_bucket (baseada no ID do cliente).
 2. **Organização:** Os dados são salvos no S3 em pastas particionadas (ex: s3://bucket/book_name/hash_bucket=01/).
 3. **Resultado:** Dados prontos para leitura seletiva, evitando scan de arquivos desnecessários.
### B. Fase de Inferência (Fargate + Polars)
Quando um batch é disparado:
 1. **Lambda de Preparação:** Recebe o model_id e lista de books. Compila os templates SQL necessários e valida as dependências.
 2. **Step Functions:** Dispara o Fargate (Task Batch) passando os parâmetros.
 3. **Processamento no Fargate:**
   * **Lazy Loading:** O código utiliza polars.scan_parquet filtrando pelos hash_buckets específicos do batch.
   * **Feature Engineering:** Aplica transformações (lags, janelas móveis, coeficientes angulares) em memória, de forma vetorizada.
   * **Inferência:** Executa o modelo carregado em memória.
   * **Output:** Salva o resultado no S3.
## 4. Diferenciais Técnicos Estratégicos
 * **Feature Registry Dinâmico:** Em vez de materializar tabelas "Gold" para tudo, mantemos um manifesto no DynamoDB (ou simples config JSON). Isso permite que cientistas criem novas features rapidamente sem rodar pipelines de ETL complexos.
 * **Lazy Evaluation:** O uso do Polars permite que o Fargate processe datasets maiores que a memória RAM, usando streaming e processamento lazy.
 * **Hash Bucketing:** Elimina a necessidade de *shuffles* pesados durante o join, permitindo unir 30 books com alta performance.
 * **Orquestração Nativa:** O uso de *Step Functions* elimina a necessidade de gerenciar instâncias de orquestração (como Airflow/Airbyte), mantendo o custo estritamente atrelado ao uso (Pay-per-use).
## 5. Roadmap de Implementação
### Fase 1: Fundação de Dados (Data Foundation)
 * [ ] Implementar o job de CTAS via Athena para particionar os 30 books por hash_bucket.
 * [ ] Configurar a estrutura de pastas no S3 para seguir o padrão book_name/hash_bucket=N/.
### Fase 2: O Compilador (Orquestração)
 * [ ] Criar a **Lambda de Preparação** (com Jinja2) que recebe o model_id e retorna o SQL necessário.
 * [ ] Configurar o **Step Functions** com Map State para processar os books em paralelo e gerenciar os estados do Athena.
### Fase 3: Engine de Inferência (Fargate)
 * [ ] Desenvolver o container Fargate (usando Python + Polars).
 * [ ] Implementar a lógica de leitura seletiva (scan_parquet com filtro de hash).
 * [ ] Criar a camada de abstração para features autoregressivas (lag/rolling window).
### Fase 4: Integração Final
 * [ ] Configurar **EventBridge** para disparar o Step Functions.
 * [ ] Testar o fluxo completo com um batch real de clientes.
 * [ ] Implementar monitoramento via CloudWatch (logs de execução e erros do Athena).
Este plano cria uma fundação robusta que trata **features como código**, mantendo o custo de infraestrutura extremamente baixo e permitindo que você escale o número de modelos sem precisar alterar a arquitetura base.
