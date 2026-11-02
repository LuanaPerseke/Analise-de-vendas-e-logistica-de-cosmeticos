# Análise Geral e Logística de Cosméticos

Uma solução de Business Intelligence e Engenharia de Dados desenvolvida para analisar vendas, receita, custos, fornecedores, logística e qualidade em uma operação de produtos de cuidados pessoais. O projeto utiliza SQL Server para importar e tratar os dados de origem, estruturar a tabela fato e as dimensões e preparar o modelo para análise no Power BI. O dashboard é dividido em **duas páginas interativas**, conectadas por botões de navegação com setas laterais.

## Sumário

- [Visão Geral do Projeto](#visão-geral-do-projeto)  
- [Arquitetura de Dados](#arquitetura-de-dados)  
- [Tecnologias Utilizadas](#tecnologias-utilizadas)  
- [Tratamento de Dados e Modelagem SQL](#tratamento-de-dados-e-modelagem-sql)  
  1. [Criação da tabela de origem e carga com BULK INSERT](#1-criação-da-tabela-de-origem-e-carga-com-bulk-insert)  
  2. [Tratamento dos dados e criação da fato](#2-tratamento-dos-dados-e-criação-da-fato)  
  3. [Criação de identificadores para dimensões](#3-criação-de-identificadores-para-dimensões)  
- [Dashboard Power BI](#dashboard-power-bi)  
  1. [Visão Geral Financeira & Vendas](#página-1-visão-geral-financeira--vendas)  
  2. [Eficiência Logística & Qualidade](#página-2-eficiência-logística--qualidade)  
- [Como Executar o Projeto](#como-executar-o-projeto)  
- [Desenvolvedora](#desenvolvedora)

---

## Visão Geral do Projeto

O objetivo do projeto é transformar dados de uma cadeia de suprimentos de produtos de cuidados pessoais em indicadores úteis para acompanhar o desempenho comercial, financeiro, operacional e de qualidade.

O dashboard permite explorar, entre outros aspectos:

- **Desempenho financeiro e vendas:** receita, custo total, lucro, preço médio e quantidade de produtos vendidos.  
- **Perfil dos clientes:** comparação dos resultados por grupos demográficos e tipo de produto.  
- **Fornecedores e produção:** comparação entre fornecedores, volume produzido e prazos médios.  
- **Logística:** modos de transporte utilizados, tempo médio de envio e prazo médio de entrega.  
- **Qualidade:** taxa média de defeitos e distribuição dos resultados de inspeção.

### Indicadores apresentados

Indicador                      Valor exibido no dashboard

---

Receita total 577,60 mil

Custo total 52,92 mil

Lucro total 524,68 mil

Preço médio 49,46

Total de produtos vendidos 46,10 mil

Média de defeitos 2,28%

Prazo médio de entrega 17

Prazo médio de fabricação 15

Total produzido 56,78 mil&nbsp;

Os valores acima foram transcritos dos cartões do dashboard. A unidade de tempo dos prazos não está explicitada nas capturas e, por isso, não foi presumida nesta documentação.

---

## Arquitetura de Dados

```
[skincare_oficial.csv]
        |
        | BULK INSERT (SQL Server)
        v
[BASE_SKINCARE]  -- dados brutos importados
        |
        | SELECT, CASE, ROUND e seleção/padronização dos campos
        v
[FATO_COSMETICOS] -- tabela fato tratada
        |
        | criação e preenchimento de chaves de relacionamento
        +-----------------------------+
        |                             |
        v                             v
[DIM_FORNECEDORES]              [DIM_LOGISTICA]
 ID_FORNECEDOR                   ID_LOGISTICA
 FORNECEDOR                      TRANSPORTADORA
 LOCALIZACAO_FORNECEDOR          MODO_TRANSPORTE
                                 ROTAS

[DIM_PRODUTO] relacionada pelo SKU
        |
        v
[Modelo de dados no Power BI]
        |
        +--> Página 1: Visão Geral Financeira & Vendas
        |
        +--> Página 2: Eficiência Logística & Qualidade
```

&nbsp;

**Observação sobre o modelo:** durante a configuração do modelo no Power BI, algumas dimensões não estavam se relacionando corretamente com a tabela fato. Para resolver isso, foram criados identificadores para as dimensões que precisavam de uma chave explícita. Na dimensão de produto, o campo `SKU` já permitia o relacionamento e foi mantido como identificador. Nos scripts fornecidos, `ID_FORNECEDOR` é formado pela combinação de fornecedor e localização, enquanto `ID_LOGISTICA` combina transportadora, modo de transporte e rota.

---

## Tecnologias Utilizadas

- **SQL Server:** criação da tabela de origem, importação por `BULK INSERT`, transformação de campos com `CASE`, arredondamento numérico com `ROUND`, criação da tabela fato com `SELECT INTO` e geração de identificadores para dimensões.  
- **Power BI:** dashboard interativo com cartões de KPI, filtros, gráficos comparativos, gráfico de dispersão, treemap, gráfico de pizza e tabelas detalhadas.  
- **CSV:** arquivo de origem com os dados operacionais da cadeia de suprimentos.  
- **GitHub:** publicação do código SQL, documentação e imagens do projeto.

---

## Tratamento de Dados e Modelagem SQL

O fluxo de preparação começa com a carga do CSV para a tabela `BASE_SKINCARE`. Em seguida, os campos são selecionados e renomeados para português, categorias textuais são traduzidas e os valores monetários e taxas selecionados são arredondados com `ROUND`. O resultado é gravado em `FATO_COSMETICOS`.

### 1\. Criação da tabela de origem e carga com BULK INSERT

O script cria a estrutura de entrada `BASE_SKINCARE` e importa os dados do arquivo CSV com `BULK INSERT`, ignorando a primeira linha (cabeçalho) e utilizando UTF-8 (`CODEPAGE = '65001'`).

```sql
CREATE TABLE BASE_SKINCARE
(
    Product_type VARCHAR(50),
    SKU VARCHAR(50),
    Price FLOAT,
    Availability INT,
    Number_of_products_sold INT,
    Revenue_generated FLOAT,
    Customer_demographics VARCHAR(100),
    Stock_levels INT,
    Lead_times INT,
    Order_quantities INT,
    Shipping_times INT,
    Shipping_carriers VARCHAR(50),
    Shipping_costs FLOAT,
    Supplier_name VARCHAR(50),
    Location VARCHAR(100),
    Lead_time INT,
    Production_volumes INT,
    Manufacturing_lead_time INT,
    Manufacturing_costs FLOAT,
    Inspection_results VARCHAR(50),
    Defect_rates FLOAT,
    Transportation_modes VARCHAR(50),
    Routes VARCHAR(50),
    Costs FLOAT
);

BULK INSERT BASE_SKINCARE
FROM 'C:\Users\luana\Downloads\Projetos\skincare_oficial.csv'
WITH
(
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '0x0a',
    CODEPAGE = '65001'
);
```

&nbsp;

**Atenção:** o caminho do CSV acima é o caminho local utilizado no ambiente de desenvolvimento. Para executar o projeto em outro computador, atualize o caminho do arquivo e verifique se a conta do serviço do SQL Server tem permissão de leitura sobre ele.

### 2\. Tratamento dos dados e criação da fato

A transformação cria `FATO_COSMETICOS` a partir da tabela bruta. O script traduz os tipos de produto, as categorias demográficas, os resultados de inspeção e os modos de transporte para português. Também aplica `ROUND(..., 2)` aos campos numéricos selecionados.

```sql
SELECT
    SKU,
    CASE Product_type
        WHEN 'haircare' THEN 'Cuidados Capilares'
        WHEN 'skincare' THEN 'Cuidados com a Pele'
        WHEN 'cosmetics' THEN 'Cosméticos'
    END AS TIPO_PROFUTO,
    ROUND(Price, 2) AS PRECO,
    Availability AS DISPONIBILIDADE,
    Number_of_products_sold AS QNT_VENDAS,
    ROUND(Revenue_generated, 2) AS RECEITA_GERADA,
    CASE Customer_demographics
        WHEN 'Non-binary' THEN 'Não-binário'
        WHEN 'Female' THEN 'Feminino'
        WHEN 'Male' THEN 'Masculino'
        ELSE 'Não Informado'
    END AS DEMOGRAFIA_CLINTE,
    Stock_levels AS NIVEL_ESTOQUE,
    Lead_times AS TEMPO_ESPERA,
    Order_quantities AS QNT_PEDIDOS,
    Shipping_times AS TEMPO_ENVIO,
    Shipping_carriers AS TRANSPORTADORA,
    ROUND(Shipping_costs, 2) AS CUSTO_ENVIO,
    Supplier_name AS FORNECEDOR,
    Location AS LOCALIZACAO_FORNECEDOR,
    Lead_time AS PRAZO_ENTREGA,
    Production_volumes AS VOLUME_PRODUCAO,
    Manufacturing_lead_time AS PRAZO_FABRACACAO,
    ROUND(Manufacturing_costs, 2) AS CUSTO_FABRACACAO,
    CASE Inspection_results
        WHEN 'Pending' THEN 'Pendente'
        WHEN 'Fail' THEN 'Reprovado'
        WHEN 'Pass' THEN 'Aprovado'
    END AS RESULTADO_INSPECAO,
    ROUND(Defect_rates, 2) AS TAXA_DEFEITO,
    CASE Transportation_modes
        WHEN 'Road' THEN 'Rodoviária'
        WHEN 'Air' THEN 'Aéreo'
        WHEN 'Sea' THEN 'Marítimo'
        WHEN 'Rail' THEN 'Ferroviario'
    END AS MODO_TRANSPORTE,
    Routes AS ROTAS,
    ROUND(Costs, 2) AS CUSTO_TOTAL
INTO FATO_COSMETICOS
FROM BASE_SKINCARE;
```

&nbsp;

### 3\. Criação de identificadores para dimensões

Ao validar os relacionamentos no Power BI, foi necessário adicionar identificadores à tabela fato e recriar as dimensões correspondentes. A chave do fornecedor combina o nome do fornecedor e sua localização, para distinguir combinações diferentes. A chave de logística combina transportadora, modo de transporte e rota.

```sql
-- Identificador de fornecedor na tabela fato
ALTER TABLE FATO_COSMETICOS
ADD ID_FORNECEDOR VARCHAR(100);

UPDATE FATO_COSMETICOS
SET ID_FORNECEDOR = CONCAT(FORNECEDOR, '_', LOCALIZACAO_FORNECEDOR);

DROP TABLE DIM_FORNECEDORES;

SELECT DISTINCT
    CONCAT(FORNECEDOR, '_', LOCALIZACAO_FORNECEDOR) AS ID_FORNECEDOR,
    FORNECEDOR,
    LOCALIZACAO_FORNECEDOR
INTO DIM_FORNECEDORES
FROM FATO_COSMETICOS;


-- Identificador de logística na tabela fato
ALTER TABLE FATO_COSMETICOS
ADD ID_LOGISTICA VARCHAR(100);

UPDATE FATO_COSMETICOS
SET ID_LOGISTICA = CONCAT(TRANSPORTADORA, '_', MODO_TRANSPORTE, '_', ROTAS);

DROP TABLE DIM_LOGISTICA;

SELECT DISTINCT
    CONCAT(TRANSPORTADORA, '_', MODO_TRANSPORTE, '_', ROTAS) AS ID_LOGISTICA,
    TRANSPORTADORA,
    MODO_TRANSPORTE,
    ROTAS
INTO DIM_LOGISTICA
FROM FATO_COSMETICOS;
```

&nbsp;

#### Observações sobre os scripts

- `DIM_FORNECEDORES` é formada por combinações distintas de fornecedor e localização.  
- `DIM_LOGISTICA` é formada por combinações distintas de transportadora, modo de transporte e rota.  
- `SKU` é usado para relacionar a dimensão de produto à tabela fato, conforme a modelagem descrita.  
- Os scripts fornecidos documentam explicitamente a criação de `ID_FORNECEDOR` e `ID_LOGISTICA`. As demais dimensões e medidas do Power BI não foram incluídas nos arquivos SQL recebidos, então não são reproduzidas aqui.  
- Os comandos `DROP TABLE` removem as dimensões existentes antes de recriá-las. Execute os scripts com cuidado e mantenha cópias de segurança se estiver trabalhando com dados importantes.

---

## Dashboard Power BI

O dashboard possui duas páginas com identidade visual consistente em tons de rosa e fundo claro. Os botões de seta no cabeçalho permitem navegar entre as páginas.

### Página 1: Visão Geral Financeira & Vendas

<img width="1322" height="742" alt="Dashboard_Cosmeticos_pg1" src="https://github.com/user-attachments/assets/7be74986-b5a8-42b1-b7d8-2331c19735a7" />

Esta página reúne os principais indicadores comerciais e financeiros, além de permitir segmentar os resultados por tipo de produto e perfil do cliente.

- **Filtros laterais:** tipo de produto \--- Cosméticos, Cuidados Capilares e Cuidados com a Pele \--- e tipo de cliente \--- Feminino, Masculino, Não Informado e Não-binário.  
- **Cartões de KPI:** receita total (577,60 mil), custo total (52,92 mil), lucro total (524,68 mil), preço médio (49,46) e total de produtos vendidos (46,10 mil).  
- **Gráfico de colunas agrupadas:** comparação de receita, custo e lucro por tipo de produto.  
- **Barras horizontais:** preferência de produto por perfil de cliente.  
- **Gráfico de dispersão:** relação entre custo total e lucro gerado, com cores por tipo de produto.  
- **Tabela detalhada:** SKU, produto, vendas, receita total, custo total e lucro.

### Página 2: Eficiência Logística & Qualidade

<img width="1325" height="742" alt="Dashboard_Cosmeticos_pg2" src="https://github.com/user-attachments/assets/d2ba5f37-f1f3-4bab-b8c9-e3e319252c65" />

Esta página concentra indicadores operacionais, comparação de fornecedores e acompanhamento da qualidade e do transporte.

- **Filtros:** fornecedor e modo de transporte.  
- **Cartões de KPI:** média de defeito (2,28%), prazo médio de entrega (17), prazo médio de fabricação (15) e total produzido (56,78 mil).  
- **Treemap:** distribuição dos modos de transporte mais usados, com contagens por modalidade.  
- **Gráfico de pizza:** distribuição dos resultados de inspeção \--- Pendente, Reprovado e Aprovado.  
- **Gráfico de colunas agrupadas:** comparação entre média de envio e custo médio de envio por modo de transporte.  
- **Tabela detalhada:** fornecedor, localização, pedidos, média de entrega, transporte e média de envio.  
- **Barras horizontais:** comparação entre média de fabricação e média de entrega por fornecedor.

---

## Como Executar o Projeto

1. **Preparar os dados**  
   - Disponibilize o arquivo `skincare_oficial.csv` em um caminho acessível ao SQL Server.  
   - Atualize o caminho usado no comando `BULK INSERT` para corresponder ao seu ambiente.  
2. **SQL Server**  
   - Execute a criação da tabela `BASE_SKINCARE` e a carga com `BULK INSERT`.  
   - Execute o script de tratamento para criar `FATO_COSMETICOS`.  
   - Execute os comandos de criação das chaves e dimensões na ordem apresentada. Se as dimensões já existirem, confirme que pode recriá-las antes de executar os comandos `DROP TABLE`.  
3. **Power BI**  
   - Abra o arquivo `.pbix` do projeto no Power BI Desktop.  
   - Configure a conexão com o banco SQL Server.  
   - Confira os relacionamentos entre `FATO_COSMETICOS` e as dimensões, utilizando `SKU`, `ID_FORNECEDOR` e `ID_LOGISTICA` conforme o modelo.  
   - Verifique as medidas e interações dos filtros e teste a navegação entre as duas páginas pelas setas do cabeçalho.

---

## Desenvolvedora

**Desenvolvido por Luana Perseke**

Conecte-se comigo no [LinkedIn](https://www.linkedin.com/in/luana-perseke-84211b179/) | [GitHub](https://github.com/LuanaPerseke)
