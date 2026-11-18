# ar-saude-es

Pipeline de dados que integra o monitoramento da qualidade do ar do Espírito Santo, publicado pelo [IEMA](https://iema.es.gov.br/qualidadedoar), com as internações hospitalares do SUS, para servir de base a análises sobre poluição atmosférica e saúde respiratória e cardiovascular.

Projeto irmão do [chuva-es](https://github.com/renanricardosilva/chuva-es), com o qual compartilha o banco de dados e as tabelas de municípios e regiões.

> **Status:** em construção.
> **Já funciona:** estrutura do repositório e documentação das decisões.
> **Próximo passo:** coletor horário da API do Índice de Qualidade do Ar (IQAr).

## Pergunta que a base vai responder

Aumentos diários de material particulado (MP10 e MP2,5) na Grande Vitória estão associados a mais internações por doenças respiratórias nos dias seguintes, depois de considerar tendência, sazonalidade e clima?

A engenharia vem primeiro: as camadas de dados são desenhadas para que essa pergunta possa ser respondida com rastreabilidade. A análise estatística entra numa etapa posterior.

## Fontes de dados

Apenas dados públicos.

| Fonte | O que traz | Frequência | Formato | Como obter |
| :--- | :--- | :--- | :--- | :--- |
| **RAMQAr** (Rede Automática de Monitoramento da Qualidade do Ar) | Poluentes e meteorologia por estação, dado validado | Horária, publicada em arquivos anuais ou semestrais | `.xlsx` / `.xls` | [Página de dados do IEMA](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/automatica), por região (Grande Vitória, Sul, Norte) |
| **API do IQAr** | Índice e poluente determinante por estação, dado preliminar | Horária, só a hora corrente | JSON | `https://qualidadedoarapi.es.gov.br/api/mapa` (a mesma usada pelo [mapa do IEMA](https://iema.es.gov.br/qualidadedoar/indicedequalidadedoar)) |
| **RMPS** (Rede Manual de Poeira Sedimentável) | Taxa de deposição (g/m²/30 dias), ocorrências e coordenadas das estações | Mensal | `.xlsx` | [Página da RMPS](https://iema.es.gov.br/qualidadedoar/dadosdemonitoramento/manual_grafico) → painel da região → ícone de download |
| **SIH/SUS** (Sistema de Informações Hospitalares) | Internações com data, município de residência, idade e diagnóstico (CID-10) | Mensal | `.dbc` | DataSUS |
| **Poeira sedimentável automática** | Deposição em resolução horária, sem regulamentação | Horária | CSV/TXT | Ainda não publicada pelo IEMA. O conector será testado com dados sintéticos |

## Arquitetura

Arquitetura medalhão, com custo zero de infraestrutura:

- **Bronze:** arquivos originais, intactos, com registro de cada arquivo recebido (hash, data e número de linhas).
- **Silver:** dados no formato longo (estação, parâmetro, horário, valor), com a regra de validade aplicada e a origem de cada valor (preliminar ou validado), em Parquet.
- **Gold:** séries agregadas para análise e painéis, em PostgreSQL compartilhado com o chuva-es.

Detalhes e alternativas consideradas estão nos [ADRs](docs/decisoes/).

## Como executar

```powershell
git clone https://github.com/renanricardosilva/ar-saude-es.git
cd ar-saude-es
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Esta seção cresce a cada entrega.

## Método de trabalho

Scrumban adaptado para uma pessoa, o mesmo método do chuva-es:

- Ciclos de cerca de três dias, com uma nota curta de retrospectiva em [`docs/retrospectivas/`](docs/retrospectivas/).
- Quadro no GitHub Projects: Backlog → Pronto para fazer → Fazendo (no máximo 2 itens) → Revisão → Feito.
- Cada entrega entra por Pull Request, com commits no padrão Conventional Commits.

**Definição de Pronto:**
- O código roda do zero seguindo este README.
- Há pelo menos um teste para a regra principal da entrega.
- README e ADRs refletem o que foi feito.
- Nenhuma afirmação da documentação fica sem evidência no repositório.

## Decisões

- [ADR-001: Arquitetura medalhão e estratégia de ingestão](docs/decisoes/adr_001_arquitetura_medalhao_e_ingestao.md)

## Limitações conhecidas

- A cobertura varia muito entre estações e poluentes. Em 2024, algumas estações da Grande Vitória não têm nenhum dado válido de material particulado.
- A API do IQAr traz um único poluente por estação (o que define o índice) e dado preliminar, sujeito a correção na publicação validada.
- O formato dos arquivos da RAMQAr muda ao longo dos anos (`.xls` até 2017, páginas web em 2018 e 2019, divisão por semestre em 2025).
- O SIH cobre apenas internações pelo SUS e informa o município de residência, não a localização exata.
- Em 2020 e 2021, a pandemia distorceu as internações respiratórias.

## Autor

**Renan Ricardo Silva**
Engenheiro agrônomo (ESALQ/USP), graduando em Tecnologia em Banco de Dados (Senac) e gestor público na área de gestão e qualidade ambiental no Espírito Santo.

- GitHub: [github.com/renanricardosilva](https://github.com/renanricardosilva)
