# ADR 001: Adoção da Arquitetura Medalhão e Estratégia de Normalização das Redes RAMQAr e RMPS

- **Status**: Aceito
- **Data**: 16/11/2026
- **Autor**: Renan Ricardo Silva
- **Contexto do Projeto**: `ar-saude-es` (Engenharia de Dados aplicada à Qualidade do Ar e Saúde Pública no Espírito Santo)

## 1. Contexto e Problema Técnico

O projeto `ar-saude-es` precisa integrar séries históricas de monitoramento atmosférico do [Instituto Estadual de Meio Ambiente e Recursos Hídricos (IEMA)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica) com registros hospitalares do DataSUS e dados meteorológicos do projeto [chuva-es](https://github.com/renanricardosilva/chuva-es). A inspeção técnica dos arquivos brutos revelou desafios estruturais importantes:

1. **Estrutura Não-Relacional da RAMQAr (`data/bronze/ramqar/rgv_2024.xlsx`)**:
   - Os arquivos anuais publicados no portal de [Monitoramento Automático da Qualidade do Ar (RAMQAr)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica) possuem **281 colunas** e um **cabeçalho hierárquico de 8 linhas** (Linha 2: Estação; Linha 3: Categoria; Linha 4: Subcategoria; Linha 5: Parâmetro; Linha 6: Frequência e altura do sensor; Linha 7: Modo; Linha 8: Unidade de medida ou indicador de `Flag`).
   - A matriz de sensores é esparsa: nem todas as 10 estações da Região da Grande Vitória (`EMQAr - RGV1` a `RGV10`), 6 da Região Sul ou da Região Norte medem os mesmos poluentes.
   - Cada coluna de medição (`Valor [µg/m³]`, `Valor [ppm]`, etc.) é acompanhada por uma coluna adjacente de `Flag` operacional. Medições com flags como `'IF'` (Invalidação por Falha), `'IP'`, `'ID'`, `'IU'`, `'IC'` (Calibração) ou `'IT'` contêm ruídos severos de leitura (ex.: `-5519.75`) que distorcem médias e agregações caso não sejam filtrados antes da análise.
2. **Dupla Granularidade Temporal da RMPS (`data/bronze/rmps/rmps_rgv_consolidado.xlsx`)**:
   - Na [Rede Manual de Partícula Sedimentável (RMPS)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/manual_grafico), o monitoramento oficial regulamentado (ASTM D1739-98 e ABNT NBR 12065:1991) possui resolução **mensal** ($\text{g/m}^2/\text{30 dias}$), distribuído em uma planilha com 4 abas (`Série Histórica` com 7 linhas de cabeçalho de metadados, `Ocorrências` operacionais, `Localização das Estações` com coordenadas geográficas e `Legenda de Cores`).
   - Paralelamente, o Estado passou a contar com medições automáticas horárias de poeira sedimentável (não regulamentadas, voltadas à gestão operacional para análise de flutuações diárias e semanais) e recebimento periódico de arquivos planos (`CSV`/`TXT`).
3. **Restrição Orçamentária**:
   - Toda a arquitetura de armazenamento, processamento e orquestração deve operar com **custo zero** (*zero-cost data engineering*), mantendo reprodutibilidade integral.

## 2. Decisão Arquitetural

Decidiu-se estruturar o pipeline de dados sob a **Arquitetura Medalhão (Medallion Architecture)** em três camadas locais e em banco relacional/espacial (**PostgreSQL + PostGIS**), combinada com armazenamento colunar em **Apache Parquet**:

| Camada | Regra de Persistência | Transformações Aplicadas |
| :--- | :--- | :--- |
| **Bronze (`data/bronze/`)** | Arquivos brutos originais (`rgv_2024.xlsx`, `rmps_rgv_consolidado.xlsx`, arquivos `CSV`/`TXT` e `.dbc`), ignorados no Git via `.gitignore` para não inflar o repositório. | Nenhuma alteração no conteúdo original; preservação fiel para auditoria e reprocessamento. |
| **Silver (`data/silver/`)** | Tabelas normalizadas no formato *long* (*tidy data*) salvas em `.parquet` e carregadas no PostgreSQL/PostGIS. | 1. **Parser de Cabeçalho Multi-Nível (RAMQAr)**: Extração programática dos metadados das linhas 1 a 8 e desacoplamento dos pares `(Valor, Flag)`.<br>2. **Expurgo de Flags Inválidas**: Preservação dos valores brutos em coluna de auditoria (`valor_bruto`) e conversão para `NULL` na coluna analítica (`valor_validado`) sempre que a `Flag` indicar falha ou calibração (`IF`, `IP`, `ID`, `IU`, `IC`, `IT`).<br>3. **Unificação da RMPS**: Cruzamento da aba `Série Histórica` com a aba `Ocorrências` e cadastro geográfico da aba `Localização das Estações`. |
| **Gold (`data/gold/`)** | Tabelas analíticas agregadas e views espaciais para consumo no Power BI e QGIS. | Agregações diárias e mensais, separação explícita entre indicadores normativos (RMPS mensal oficial e médias móveis regulamentadas da RAMQAr) e indicadores operacionais (RMPS automática horária), além do *join* espaço-temporal com internações respiratórias do DataSUS. |

## 3. Consequências

### Pontos Positivos
- **Blindagem contra Erros de Agregação**: O tratamento explícito das colunas `Flag` na transição *Bronze -> Silver* impede que valores negativos de falha de sensor contaminem os cálculos na camada *Gold* e nos painéis de BI.
- **Flexibilidade para Novas Estações e Regiões**: Como os dados são convertidos do formato *wide* (281 colunas fixas) para o formato *long* (`timestamp`, `id_estacao`, `parametro`, `unidade`, `valor_validado`, `flag`), a inclusão dos arquivos da Região Sul, Região Norte ou de novas estações automáticas de poeira sedimentável não exige alteração no schema do banco de dados.
- **Eficiência e Custo Zero**: A conversão das planilhas `.xlsx` para `.parquet` reduz o volume de armazenamento em mais de 85% e acelera leituras em Python/Pandas, viabilizando automação gratuita via **GitHub Actions**.

### Pontos de Atenção (Trade-offs)
- Exige manutenção de testes unitários (`pytest`) para validar se eventuais mudanças de layout nas planilhas exportadas pelo IEMA continuam respeitando a estrutura de 8 linhas de cabeçalho na RAMQAr e 7 linhas na RMPS.
