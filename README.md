# MVP: Construção de um Pipeline de Dados na Nuvem

Este repositório contém a entrega do Produto Mínimo Viável (MVP) de Engenharia de Dados. O objetivo deste projeto é construir, do zero, um pipeline de dados funcional e persistido na nuvem utilizando a plataforma **Databricks (Free Edition)** com processamento em **SQL puro**.

---

## 1. Contexto de Negócios e Perguntas (Etapa 2. e 4.1)

### Contexto dos Dados Brutos
O objetivo deste projeto é analisar o comportamento das corridas de táxi na cidade de Nova York para identificar padrões de faturamento, eficiência de rotas e preferências dos usuários. A base de dados utilizada provém dos conjuntos de dados públicos da Databricks (`/databricks-datasets/nyctaxi/tripdata/yellow/`), contendo arquivos CSV brutos históricos que totalizam **mais de 10 milhões de registros reais**.

### Perguntas de Negócio Formuladas
Para guiar este MVP, foram definidas as seguintes perguntas:
1. **Pergunta 1:** Qual o horário do dia com a maior média de gorjetas?
2. **Pergunta 2:** Quais tipos de pagamento geram os maiores valores totais por corrida?
3. **Pergunta 3:** Qual a relação entre a distância percorrida (em milhas) e o valor total cobrado?

### Licença dos Dados
Os dados utilizados são públicos e disponibilizados pela própria plataforma Databricks em seu repositório de amostras globais para fins educacionais, científicos e de demonstração.

---

## 2. Carga dos Dados (Etapa 4.2)

### Explicação da Ingestão
Os dados brutos originais estavam distribuídos em arquivos CSV sem cabeçalho explícito no sistema de arquivos distribuído do Databricks. O processo de carga inicial (**Camada Bronze**) consistiu em ler esse diretório de objetos e persistir as informações no formato nativo Delta Lake dentro do nosso próprio Schema de trabalho (`mvp_pipeline_dados`), garantindo a imutabilidade e a rastreabilidade da matéria-prima.

*   **Script de Carga:** O código correspondente a esta etapa está contido no arquivo de scripts SQL deste repositório.
*   **Evidência de Volumetria (+10 Milhões de Linhas):**
 
    > ![Contagem de Linhas da Tabela Bronze]([001 - totallinhastabela.png](https://github.com/AndreSampaioDev1982/mvp-pipeline-dados-nuvem/blob/21c8d97ccfe45f1993e4a0defb6a71ac6aa6e40d/001%20-%20totallinhastabela.png))

---

## 3. Modelagem e Catálogo de Dados (Etapa 4.3)

### Arquitetura Adotada
O projeto foi modelado seguindo as melhores práticas de um Data Lakehouse através da **Arquitetura Medalhão**, dividida em camadas progressivas de refino (Bronze, Silver e Gold), combinada com um modelo analítico dimensional simplificado (**Tabela Fato e Dimensão Tempo**).

### Catálogo de Dados (Unity Catalog)

#### Tabela: `fato_corridas`
*Descrição: Armazena o registro individual de cada viagem realizada, limpa e pronta para cruzamentos analíticos.*
*   `id_fretador` (STRING): Código identificador da empresa ou prestador de serviço de transporte (Mapeado a partir de `_c0`).
*   `hora_dia` (INTEGER): Hora do dia em que a corrida ocorreu (0 a 23). Atua como chave de ligação com a tabela `dim_tempo`.
*   `data_completa` (DATE): Data no formato AAAA-MM-DD.
*   `distancia_milhas` (DOUBLE): Distância percorrida em milhas (Mapeado a partir de `_c4`). Domínio: Valores estritamente maiores que zero.
*   `valor_tarifa` (DOUBLE): Valor base cobrado pela corrida (Mapeado a partir de `_c12`). Domínio: Valores estritamente maiores que zero.
*   `valor_gorjeta` (DOUBLE): Valor da gorjeta deixada pelo passageiro (Mapeado a partir de `_c15`).
*   `valor_total` (DOUBLE): Custo total cobrado (Mapeado a partir de `_c17`). Domínio: Valores estritamente maiores que zero.
*   `tipo_pagamento` (STRING): Forma de pagamento utilizada pelo passageiro (Mapeado a partir de `_c11`).

#### Tabela: `dim_tempo`
*Descrição: Tabela dimensão que permite a segmentação temporal das métricas de faturamento e volume.*
*   `hora_dia` (INTEGER): Hora do dia (0 a 23). Atua como Chave Primária.
*   `data_completa` (DATE): Data no formato AAAA-MM-DD.
*   `dia_semana` (INTEGER): Dia da semana mapeado numericamente (1 = Domingo, 7 = Sábado).
*   `mes` (INTEGER): Mês do ano (1 a 12).

### Evidência de Persistência na Nuvem
> *[Substitua esta linha pelo print da aba "Catalog" do Databricks mostrando as tabelas criadas no seu Schema]*
> ![Tabelas Persistidas no Databricks](caminho_da_sua_imagem_do_catalogo.png)

---

## 4. Pipeline de Dados (Etapa 4.4)

### Organização do Processo ETL
O pipeline de dados foi desenvolvido de forma modular através de scripts SQL dentro da plataforma de nuvem. O fluxo segue rigorosamente a seguinte linhagem de transformação:
1. Arquivos brutos CSV externos ➔ Lidos e injetados na tabela `bronze_taxi_trips` (Formato Delta).
2. `bronze_taxi_trips` ➔ Tratada via mapeamento posicional e lógica condicional para gerar a `silver_taxi_trips`.
3. `silver_taxi_trips` ➔ Desmembrada nas estruturas dimensionais analíticas Gold: `fato_corridas` e `dim_tempo`.

Cada etapa foi documentada no script com comentários explicando as decisões técnicas e regras aplicadas.

---

## 5. Qualidade de Dados (Etapa 4.5)

Durante a fase de transição da camada Bronze para a **Silver**, foram aplicadas regras rígidas de qualidade de dados usando funções de tratamento seguro como `try_cast` para mitigar problemas estruturais dos arquivos brutos:
*   **Completude:** Filtro para garantir a presença obrigatória do registro temporal de embarque (`_c1 IS NOT NULL`).
*   **Acurácia e Outliers:** Remoção de anomalias operacionais através de filtros matemáticos. Foram descartadas corridas com distância (`_c4`), valor da tarifa (`_c12`), custo total (`_c17`) ou quantidade de passageiros (`_c3`) zerados ou negativos.
*   **Consistência:** Forçamento explícito de tipos (*casting*) convertendo campos de texto brutos diretamente em tipos estruturados como `TIMESTAMP`, `DOUBLE` e `INT`.

---

## 6. Análise de Dados (Etapa 4.5)

### Respostas às Perguntas do Objetivo

#### Pergunta 1: Qual o horário do dia com a maior média de gorjetas?
*Análise Técnica:* Query SQL executada agrupando o valor médio de gorjeta por hora do dia a partir da tabela `fato_corridas`.
*Discussão do Resultado:* *[Escreva aqui a sua interpretação dos números baseada nos resultados da query do Databricks]*.
> *[Substitua esta linha pelo print do gráfico/tabela gerado no Databricks para a Pergunta 1]*
> ![Gráfico Pergunta 1](caminho_grafico_1.png)

#### Pergunta 2: Quais tipos de pagamento geram os maiores valores totais por corrida?
*Análise Técnica:* Query SQL executada calculando o ticket médio e o faturamento acumulado por modalidade de pagamento.
*Discussão do Resultado:* *[Escreva aqui a sua interpretação dos números]*.
> *[Substitua esta linha pelo print do gráfico/tabela gerado no Databricks para a Pergunta 2]*
> ![Gráfico Pergunta 2](caminho_grafico_2.png)

#### Pergunta 3: Qual a relação entre a distância percorrida (em milhas) e o valor total cobrado?
*Análise Técnica:* Consulta analítica comparando faixas de distância com as médias de faturamento geradas.
*Discussão do Resultado:* *[Escreva aqui a sua interpretação dos números]*.
> *[Substitua esta linha pelo print do gráfico/tabela gerado no Databricks para a Pergunta 3]*
> ![Gráfico Pergunta 3](caminho_grafico_3.png)

---

## 7. Autoavaliação

### Atingimento dos Objetivos
*[Discorra aqui sobre se você conseguiu atingir os objetivos traçados antes de iniciar o projeto. Discuta o que foi possível responder com a infraestrutura montada].*

### Dificuldades Encontradas
*[Mencione o desafio de manipular colunas genéricas posicionais (_c0, _c1) vindas de arquivos sem cabeçalho estruturado e a necessidade de usar funções defensivas como try_cast para processar uma volumetria superior a 10 milhões de linhas de maneira performática no ambiente gratuito].*

### Trabalhos Futuros
*[Sugira ideias para enriquecer este projeto no seu portfólio pessoal (ex: criar uma orquestração automática com Airflow ou Databricks Workflows, cruzar com dados meteorológicos de Nova York, etc)].*
