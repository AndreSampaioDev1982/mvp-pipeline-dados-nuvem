# MVP: Construção de um Pipeline de Dados na Nuvem

Este repositório contém a entrega do Produto Mínimo Viável (MVP) de Engenharia de Dados. O objetivo deste projeto é construir, do zero, um pipeline de dados funcional e persistido na nuvem utilizando a plataforma **Databricks (Free Edition)** com processamento em **SQL puro**.

---

## 1. Contexto de Negócios e Perguntas (Etapa 2. e 4.1)

### Contexto dos Dados Brutos
*Descreva aqui brevemente o problema de negócio que você quer resolver. Exemplo:*
O objetivo deste projeto é analisar o comportamento das corridas de táxi na cidade de Nova York para identificar padrões de faturamento e preferências dos usuários. A base de dados utilizada provém dos conjuntos de dados públicos da Databricks (`samples.nyctaxi.trips`), contendo mais de 10 milhões de registros reais de viagens.

### Perguntas de Negócio Formulações
Para guiar este MVP, foram definidas as seguintes perguntas:
1. **Pergunta 1:** Qual o horário do dia com a maior média de gorjetas?
2. **Pergunta 2:** Quais tipos de pagamento geram os maiores valores totais por corrida?
3. **Pergunta 3:** *[Insira uma terceira pergunta de sua preferência ou use as duas do script anterior]*

### Licença dos Dados
Os dados utilizados são públicos e disponibilizados pela própria plataforma Databricks para fins educacionais e de demonstração.

---

## 2. Carga dos Dados (Etapa 4.2)

### Explicação da Ingestão
Como os dados brutos de mais de 10 milhões de linhas já residem no ambiente de armazenamento distribuído interno da Databricks, o processo de carga inicial (Camada Bronze) consistiu em ler esse diretório público e persistir as informações no formato nativo Delta Lake dentro do nosso próprio Schema de trabalho.

*   **Script de Carga:** O código utilizado para esta etapa pode ser encontrado no arquivo `[insira_o_nome_do_seu_arquivo.sql]` deste repositório.
*   **Evidência de Volumetria (+10 Milhões de Linhas):**
    > *Substitua esta linha pelo print do resultado da sua query `SELECT COUNT(*)` na tabela bronze.*
    > ![Contagem de Linhas da Tabela Bronze](link_ou_caminho_da_sua_imagem_aqui.png)

---

## 3. Modelagem e Catálogo de Dados (Etapa 4.3)

### Arquitetura Adotada
O projeto foi modelado seguindo os princípios de um Data Lakehouse através da **Arquitetura Medalhão** (dividida em camadas progressivas de refino: Bronze, Silver e Gold) combinada com um modelo analítico simplificado de **Tabela Fato e Dimensões**.

### Catálogo de Dados (Unity Catalog)

#### Tabela: `fato_corridas`
*Descrição: Armazena o registro granular de cada viagem realizada, limpa e pronta para cruzamentos analíticos.*
*   `id_corrida` (INTEGER): Identificador único da corrida. (Chave Primária).
*   `id_fretador` (STRING): Código da empresa ou prestador de serviço de transporte.
*   `hora_day` (INTEGER): Hora do dia em que a corrida ocorreu (0 a 23). (Chave Estrangeira para dim_tempo).
*   `distancia_milhas` (DOUBLE): Distância percorrida em milhas. Domínio: Valores maiores que zero.
*   `valor_tarifa` (DOUBLE): Valor base cobrado pela corrida. Domínio: Valores maiores que zero.
*   `valor_gorjeta` (DOUBLE): Valor da gorjeta deixada pelo passageiro.
*   `valor_total` (DOUBLE): Custo total da corrida incluindo taxas e gorjetas. Domínio: Valores maiores que zero.
*   `tipo_pagamento` (STRING): Forma de pagamento utilizada (ex: Credit Card, Cash).

#### Tabela: `dim_tempo`
*Descrição: Tabela dimensão que permite a análise temporal das métricas agregadas por hora ou dia.*
*   `hora_dia` (INTEGER): Hora do dia (0 a 23). (Chave Primária).
*   `data_completa` (DATE): Data no formato AAAA-MM-DD.
*   `dia_semana` (INTEGER): Dia da semana mapeado numericamente.
*   `mes` (INTEGER): Mês do ano (1 a 12).

### Evidência de Persistência na Nuvem
> *Substitua esta linha pelo print da aba "Catalog" do Databricks mostrando as tabelas criadas no seu Schema.*
> ![Tabelas Persistidas no Databricks](link_da_sua_imagem_do_catalogo_aqui.png)

---

## 4. Pipeline de Dados (Etapa 4.4)

### Organização do Processo ETL
O pipeline de dados foi desenvolvido de forma modular através de scripts SQL dentro da plataforma de nuvem. O fluxo segue rigorosamente a linhagem:
1. `samples.nyctaxi.trips` (Origem Externa) ➔ `bronze_taxi_trips` (Cópia idêntica)
2. `bronze_taxi_trips` ➔ `silver_taxi_trips` (Filtros de limpeza e tipagem)
3. `silver_taxi_trips` ➔ `fato_corridas` e `dim_tempo` (Estruturação analítica Gold)

Cada etapa foi documentada no script com explicações detalhadas sobre as decisões de joins e filtros aplicados.

---

## 5. Qualidade de Dados (Etapa 4.5)

Durante a fase de exploração e refino do dado bruto para a camada **Silver**, foram aplicadas as seguintes regras de qualidade para mitigar problemas comuns de dados sujos:
*   **Completude:** Filtro para garantir que nenhuma linha possuísse o identificador `trip_id` nulo.
*   **Acurácia e Outliers:** Remoção de anomalias estatísticas e operacionais, como registros onde `trip_distance`, `fare_amount`, `total_amount` ou `passenger_count` apresentavam valores zerados ou negativos.
*   **Consistência:** Forçamento de tipo (*cast*) para garantir que campos textuais de data e hora fossem armazenados estritamente como o tipo `TIMESTAMP`.

---

## 6. Análise de Dados (Etapa 4.5)

### Respostas às Perguntas do Objetivo

#### Pergunta 1: Qual o horário do dia com a maior média de gorjetas?
*Análise Técnica:* Query SQL executada agrupando o valor médio de gorjeta por hora do dia.
*Discussão do Resultado:* (Escreva aqui a sua interpretação dos números, conectando com o negócio).
> *Substitua esta linha pelo print do gráfico/tabela gerado no Databricks para a Pergunta 1.*
> ![Gráfico Pergunta 1](link_do_grafico_1.png)

#### Pergunta 2: Quais tipos de pagamento geram os maiores valores totais por corrida?
*Análise Técnica:* Query SQL executada calculando o ticket médio e o faturamento total por modalidade de pagamento.
*Discussão do Resultado:* (Escreva aqui a sua interpretação dos números, conectando com o negócio).
> *Substitua esta linha pelo print do gráfico/tabela gerado no Databricks para a Pergunta 2.*
> ![Gráfico Pergunta 2](link_do_grafico_2.png)

---

## 7. Autoavaliação

### Atingimento dos Objetivos
*Discorra aqui sobre se você conseguiu atingir os objetivos traçados antes de iniciar o projeto. Se faltou algo, explique o porquê (lembrando que, conforme o edital, isso demonstra maturidade analítica e não desconta nota).*

### Dificuldades Encontradas
*Escreva sobre os desafios técnicos enfrentados na manipulação de uma base volumosa com mais de 10 milhões de linhas dentro do ambiente limitado do Databricks Community.*

### Trabalhos Futuros
*Sugira ideias para enriquecer este projeto no seu portfólio pessoal no futuro (ex: criar uma orquestração com Airflow, cruzar com dados de clima de NY, etc).*
