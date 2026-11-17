# ar-saude-es | Engenharia de Dados de Qualidade do Ar e Saúde Pública no Espírito Santo

Pipeline de Engenharia e Análise de Dados que integra as séries históricas de monitoramento da qualidade do ar do [Instituto Estadual de Meio Ambiente e Recursos Hídricos (IEMA)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica) com variáveis meteorológicas do projeto irmão [chuva-es](https://github.com/renanricardosilva/chuva-es) e registros de internações hospitalares por doenças respiratórias do Sistema Único de Saúde (SIH/DataSUS) no Espírito Santo.

## Contexto e Problema de Negócio

A gestão da qualidade do ar e a vigilância em saúde ambiental exigem a integração de bases governamentais heterogêneas, com diferentes frequências temporais, estruturas de arquivos e regras de validação normativa:

1. **[Rede Automática de Monitoramento da Qualidade do Ar (RAMQAr - IEMA)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica)**: Monitoramento horário contínuo de gases ($\text{CO}$, $\text{SO}_2$, $\text{NO}_x$, $\text{O}_3$, $\text{HC}$), material particulado ($\text{MP}_{2,5}$, $\text{MP}_{10}$, $\text{PTS}$) e parâmetros meteorológicos nas redes da Região da Grande Vitória (RGV, 10 estações), Região Sul (Anchieta e Guarapari, 6 estações operadas pela Samarco) e Região Norte (Linhares). Os dados brutos são disponibilizados em planilhas de formato largo (*wide*) com cabeçalho hierárquico de 8 linhas e colunas pareadas de medição e `Flag` de controle operacional.
2. **[Rede Manual de Partícula Sedimentável (RMPS - IEMA)](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/manual_grafico)**: Monitoramento gravimétrico oficial regulamentado (normas ASTM D1739-98 e ABNT NBR 12065:1991) com exposição de 30 dias em jarros coletores ($\text{g/m}^2/\text{30 dias}$), complementado recentemente por sensores automáticos de resolução horária voltados à gestão operacional e identificação de flutuações intradiárias e semanais.
3. **Morbidade Hospitalar (SIH/DataSUS)**: Registros de Autorizações de Internação Hospitalar (AIH) por doenças do aparelho respiratório (Capítulo X da CID-10) nos municípios capixabas, permitindo avaliar correlações espaço-temporais e defasagens (*lags*) entre episódios críticos de poluição atmosférica e desfechos de saúde pública.

## Arquitetura de Dados (Medallion Architecture)

O projeto adota a **Arquitetura Medalhão** (*Bronze*, *Silver* e *Gold*) para garantir rastreabilidade, reprodutibilidade e separação clara entre o dado bruto governamental e as tabelas analíticas consumidas em BI e modelos estatísticos:

| Camada | Diretório / Destino | Formato | Responsabilidade Técnica |
| :--- | :--- | :--- | :--- |
| **Bronze (Raw)** | `data/bronze/` | `.xlsx`, `.csv`, `.txt`, `.dbc` | Preservação imutável dos arquivos originais baixados do [Portal da RAMQAr](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica), da [Página da RMPS](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/manual_grafico) e do DataSUS (`rgv_2024.xlsx`, `rmps_rgv_consolidado.xlsx`). |
| **Silver (Cleaned & Enriched)** | `data/silver/` | `.parquet` + PostgreSQL/PostGIS | Desmembramento do cabeçalho de 8 linhas da RAMQAr, conversão de *wide* para *long* (tidy data), expurgo de `Flags` inválidas (`IF`, `IP`, `ID`, `IU`, `IC`, `IT`), padronização de unidades e georreferenciamento das estações (`SIRGAS 2000` / `EPSG:4674`). |
| **Gold (Analytical / Curated)** | `data/gold/` | `.parquet` + Views SQL/PostGIS | Séries agregadas diárias e mensais, cálculo do Índice de Qualidade do Ar (IQAr), cruzamento multi-granularidade (RMPS mensal oficial vs. RMPS horária automática) e matriz integrada Ar + Meteorologia + Internações SUS por município e buffer de influência das estações. |

## Estrutura do Repositório

```text
ar-saude-es/
├── data/
│   ├── bronze/
│   │   ├── datasus/                # Arquivos brutos de morbidade hospitalar (SIH/SUS)
│   │   ├── ramqar/                 # Planilhas brutas da rede automática (ex: rgv_2024.xlsx)
│   │   └── rmps/                   # Planilhas consolidadas de poeira sedimentável (ex: rmps_rgv_consolidado.xlsx)
│   ├── silver/                     # Dados normalizados, validados e tipados em formato Parquet
│   └── gold/                       # Tabelas analíticas agregadas para consumo em Power BI e modelos
├── docs/
│   ├── decisoes/                   # Architecture Decision Records (ADRs)
│   └── retrospectivas/             # Relatórios de revisão e retrospectiva de cada Sprint
├── notebooks/                      # Análises exploratórias (EDA) e prototipação
├── sql/                            # Scripts DDL/DML para PostgreSQL + extensão espacial PostGIS
├── src/
│   ├── ingestion/                  # Leitores de arquivos brutos (Excel multi-header, CSV/TXT, DataSUS)
│   ├── transform/                  # Limpeza de flags, pivotagem wide-to-long e regras de negócio
│   └── database/                   # Conectores SQLAlchemy e carga em banco de dados
├── tests/                          # Testes automatizados de qualidade de dados e transformação (Pytest)
├── .gitignore
├── requirements.txt
└── README.md
```

## Governança Ágil (Scrum Solo)

Seguindo o modelo consolidado no repositório [chuva-es](https://github.com/renanricardosilva/chuva-es), o desenvolvimento é gerido via **GitHub Projects** e **GitHub Issues** com Sprints de 1 semana e estimativa de esforço em *Story Points* (escala Fibonacci):

| Sprint | Foco Principal | Entregáveis Chave | Status |
| :--- | :--- | :--- | :--- |
| **Sprint 1** | Fundação, Governança e Ingestão Bronze/Silver (IEMA) | Estrutura Medalhão, ADR-001, parser Python do cabeçalho de 8 linhas da RAMQAr (`rgv_2024.xlsx`) e normalizador das 4 abas da RMPS (`rmps_rgv_consolidado.xlsx`). | Em andamento |
| **Sprint 2** | Modelagem Relacional e Geoespacial (PostgreSQL/PostGIS) | Schema dimensional e espacial das estações (RGV, Sul e Norte), integração com dados meteorológicos e automação de ingestão *zero-cost* para arquivos `CSV`/`TXT`. | Planejado |
| **Sprint 3** | Integração Saúde Pública (DataSUS) e Camada Gold | Extração do SIH/SUS (CID-10 Capítulo X), agregação espaço-temporal e views analíticas para cruzamento de poluentes críticos ($\text{MP}_{10}$, $\text{MP}_{2,5}$, $\text{SO}_2$, $\text{O}_3$, PS) com desfechos respiratórios. | Planejado |
| **Sprint 4** | Visualização Executiva e Storytelling para Gestão Pública | Painel interativo no Power BI, mapas de dispersão/buffers no QGIS e documentação executiva de resultados. | Planejado |

## Como Executar Localmente

1. Clone o repositório e acesse a pasta do projeto:
   ```powershell
   git clone https://github.com/renanricardosilva/ar-saude-es.git
   cd ar-saude-es
   ```
2. Crie e ative o ambiente virtual Python:
   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   ```
3. Baixe as planilhas públicas na página da [RAMQAr - IEMA](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica) e no painel da [RMPS - IEMA](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/manual_grafico), alocando-as em `data/bronze/ramqar/rgv_2024.xlsx` e `data/bronze/rmps/rmps_rgv_consolidado.xlsx`.

## Autor

**Renan Ricardo Silva**
Engenheiro Agrônomo (ESALQ/USP), graduando em Tecnologia da Informação (SENAC) e gestor público na área de Gestão e Qualidade Ambiental no Espírito Santo.
- GitHub: [github.com/renanricardosilva](https://github.com/renanricardosilva)
