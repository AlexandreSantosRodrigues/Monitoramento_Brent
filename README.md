# 🛢️ Pipeline de Paridade do Petróleo Brent (USD para BRL)

## 📌 O Problema
Calcular a paridade de importação do Petróleo Brent exige o cruzamento diário do preço do barril no mercado internacional (em Dólares) com a taxa de câmbio oficial do Brasil (PTAX). Fazer isto manualmente em folhas de cálculo gera três grandes problemas de engenharia:
1. **Trabalho manual repetitivo e suscetível a erro humano.**
2. **Assimetria de Calendários:** O mercado internacional (Brent) e o mercado nacional (Banco Central) possuem feriados diferentes. Cruzar dados de dias em que apenas um dos mercados operou gera paridades falsas ou distorcidas.
3. **Falta de um repositório centralizado** para consumo analítico por ferramentas de Business Intelligence (BI).

## 💡 A Solução
Desenvolvimento de uma arquitetura de dados *Serverless* (sem necessidade de gerir servidores) que executa uma pipeline ETL (Extração, Transformação e Carga) 100% automatizada. O robô extrai as cotações financeiras, limpa e alinha os dados temporais, e injeta os resultados de forma incremental num Data Warehouse na nuvem (Google BigQuery).

## 🏗️ Como os Dados São Importados (Ingestão e Arquitetura)
O fluxo de dados foi desenhado em três camadas (Extract, Transform, Load):

1. **Extração (APIs):** O script em Python consome os dados de fechamento do barril de Brent através da biblioteca `yfinance` e a cotação do Dólar (PTAX de Venda) através da API pública OData (Olinda) do Banco Central do Brasil.
2. **Transformação (Pandas):** Os dados são convertidos para o mesmo fuso horário (*tz-naive*) e padronizados para as 00:00:00. O cálculo da paridade (`Preço Brent USD * Cotação USD BRL`) é gerado dinamicamente.
3. **Carga Incremental (BigQuery):** Através de uma *Service Account* segura, o Python liga-se ao Google BigQuery e executa a query `SELECT MAX(Data)`. O script compara a última data existente no banco com os dados extraídos, inserindo (modo *Append*) apenas as linhas correspondentes a novos dias.

## 🛡️ Assertividade e Qualidade dos Dados
Para garantir que os dados não são sintéticos, duplicados ou corrompidos, a pipeline implementa duas regras rígidas de negócio:

*   **Alinhamento de Calendário (Inner Join):** A junção dos dados do Yahoo Finance e do Banco Central é feita obrigatoriamente através de um `inner join` na chave `Data`. Isto garante que o cálculo da paridade só ocorre em dias úteis partilhados por ambos os mercados, eliminando automaticamente fins de semana e feriados isolados.
*   **Idempotência na Inserção:** A lógica de carga incremental previne a duplicação de dados. O script pode ser executado múltiplas vezes ao dia; se não houver um novo fecho de mercado consolidado em ambas as fontes, a pipeline aborta a inserção para preservar a integridade do banco de dados.

## 🛠️ Stack Tecnológica
- **Python 3** (`pandas`, `requests`, `yfinance`, `google-cloud-bigquery`)
- **Google Cloud Platform (GCP)** (BigQuery, IAM)
- **Kaggle Notebooks** (Orquestração / Cron diário)
- 
## 🚀 Como Executar
1. Crie um projeto no GCP, ative a API do BigQuery e crie o dataset/tabela.
2. Gere uma chave JSON de uma Service Account com as permissões `BigQuery Data Editor` e `BigQuery User`.
3. Guarde o conteúdo do JSON no *Kaggle Secrets* com o nome `gcp_bq_json`.
4. Execute o script Python no Kaggle e configure o *Schedule* (Trigger > Frequency > Daily) para um horário noturno (ex: 23h00).
