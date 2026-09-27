# IMPORTANTE!!! ----------------------------------------------------------
> Detalhes, explicações e desenvolvimento esperados para as seções expostas neste README.md encontram-se no Notebook: MVP_Consorcios_Imobiliarios disponível no GitHub: [https://github.com/prchiquito83/mvp-consorcios-imobiliarios](https://github.com/prchiquito83/mvp-consorcios-imobiliarios/blob/main/MVP_Data%20Base_Consorcios_Imobiliarios.ipynb)

> Outra maneira de visualizar todos os detalhes de desenvolvimento, é efetuando o download do arquivo html e abrindo o arquivo dentro do Chrome ou Edge.
# -----------------------------------------------------------------------------

# MVP - Banco de Dados para Análise de Risco Operacional em Consórcios Imobiliários

**Aluno:** Paulo Roberto Chiquito  
**Matrícula:** 4052026000805  
**Data:** 27/09/2026  
**Ambiente:** Databricks Free Edition  
**Data-base dos dados:** julho de 2026  
**Fonte:** https://www.bcb.gov.br/estabilidadefinanceira/consorciobd
Arquivos baixados presentes na pasta 202607Consorcios.zip


## 1. Contexto de Negócios e Perguntas (Etapa 2 e 4.1)

### Problema

As administradoras de consórcios operam grupos com diferentes quantidades de cotas, prazos, valores médios dos bens e taxas de administração. Comparações baseadas apenas em valores absolutos podem favorecer administradoras ou grupos de maior porte.

Este trabalho propõe criar indicadores proporcionais que permitam comparar grupos de tamanhos diferentes. O principal indicador será a taxa de inadimplência das cotas ativas, complementada pelas quantidades de cotas excluídas e de cotas contempladas com crédito ainda pendente de utilização.

**Pergunta central:** Quais administradoras e grupos de consórcios imobiliários apresentam os maiores sinais de atenção operacional na data-base de julho de 2026?

### Perguntas de Negócio

1. Quais administradoras possuem mais grupos imobiliários?
2. Quais administradoras concentram mais cotas ativas?
3. Quais grupos apresentam maior taxa de inadimplência?
4. Qual é a taxa de inadimplência agregada por administradora?
5. A taxa de inadimplência varia entre faixas de prazo?
6. Existe associação entre taxa de administração e inadimplência?
7. Quais administradoras possuem mais cotas excluídas?
8. Quais administradoras possuem mais cotas com crédito pendente de utilização?
9. Quais grupos combinam alta inadimplência e grande quantidade de cotas excluídas?
10. Os valores calculados a partir dos grupos são compatíveis com o arquivo consolidado?

### Contexto dos Dados Brutos

Os dados utilizados são públicos e provenientes do Banco Central do Brasil (BCB), disponibilizados na página de Estabilidade Financeira - Base de Dados de Consórcios.

**Estrutura dos arquivos:**

| Arquivo | Descrição | Formato |
|---|---|---|
| 202607Segmentos_Consolidados.csv | Dados consolidados por segmento (não utilizado) | CSV (sep. ;, decimal ,) |
| 202607Bens_Imoveis_Grupos.csv | Dados por grupo de consórcio imobiliário | CSV (sep. ;, decimal ,) |
| Significado_dos_campos_e_metricas.xlsx | Dicionário de dados | Excel (XLSX) |

**Características do arquivo principal:**
- Separador de campos: ponto e vírgula
- Separador decimal: vírgula
- Nomes de colunas com acentuação
- CNPJ com zeros à esquerda (texto)
- Data-base no formato AAAAMM
- Códigos de grupo alfanuméricos
- Encoding: windows-1252
- Granularidade: uma linha por administradora, data-base, código do grupo e código do segmento

### Licença dos Dados

Os dados publicados pelo Banco Central do Brasil na Base de Dados de Consórcios são de uso público e gratuito. A política institucional de dados abertos do BCB permite a livre utilização, desde que citada a fonte.

> Fonte: Banco Central do Brasil - https://www.bcb.gov.br/estabilidadefinanceira/consorciobd - Data-base: julho de 2026.

---

## 2. Carga dos Dados (Etapa 4.2)

A carga de dados foi realizada em um notebook Python no Databricks, utilizando Spark DataFrame API para leitura dos arquivos CSV armazenados em um Unity Catalog Volume (workspace.default.arquivos_mvp). O arquivo XLSX foi mantido como metadado de referência.

**Passos da carga:**
1. Configuração dos caminhos dos arquivos no UC Volume
2. Verificação da existência dos arquivos
3. Criação do schema workspace.mvp_consorcios no Unity Catalog
4. Leitura do CSV com preservação de todas as colunas como texto

A leitura inicial como texto foi intencional. Se o Spark fizesse a inferência automática, o CNPJ poderia virar número e perder os zeros à esquerda, e os valores com vírgula decimal poderiam vir incorretos.

> Script de referência: O notebook MVP_Consorcios_Imobiliarios está disponível no GitHub: https://github.com/prchiquito83/mvp-consorcios-imobiliarios

---

## 3. Modelagem e Catálogo de Dados (Etapa 4.3)

A modelagem seguiu a arquitetura de camadas Bronze, Silver e Gold, persistidas no Unity Catalog (workspace.mvp_consorcios) no formato Delta Lake.

### Catálogo de Tabelas

| Tabela | Camada | Descrição | Granularidade |
|---|---|---|---|
| bronze_grupos_imoveis | Bronze | Dados com colunas normalizadas | Uma linha por grupo/segmento/data-base |
| silver_grupos_imoveis | Silver | Dados tratados e convertidos | Uma linha por grupo/segmento/data-base |
| gold_administradoras | Gold | Indicadores agregados por administradora | Uma linha por administradora/data-base |
| gold_grupos_risco | Gold | Dados por grupo com indicadores e classificações de risco | Uma linha por grupo/segmento/data-base |
| vw_monitoramento_risco | View | Visão de monitoramento filtrando grupos com cotas ativas | Mesma granularidade da Gold de grupos |

### Estrutura - gold_administradoras

| Coluna | Tipo | Descrição |
|---|---|---|
| cnpj_da_administradora | string | Raiz do CNPJ (8 dígitos) |
| nome_da_administradora | string | Nome da administradora |
| data_base | date | Data-base de referência |
| quantidade_grupos | long | Número de grupos distintos |
| quantidade_cotas_ativas | long | Total de cotas ativas |
| quantidade_cotas_inadimplentes | long | Total de cotas inadimplentes |
| quantidade_cotas_excluidas | long | Total de cotas excluídas |
| quantidade_credito_pendente | long | Total de cotas com crédito pendente |
| quantidade_contempladas_mes | long | Cotas contempladas no mês |
| valor_medio_bem_media_simples | double | Média simples do valor médio do bem |
| taxa_administracao_ponderada | double | Média ponderada da taxa de administração |
| taxa_inadimplencia | double | Taxa de inadimplência calculada |
| taxa_credito_pendente | double | Taxa de crédito pendente calculada |

### Estrutura - gold_grupos_risco

Inclui todas as colunas originais tratadas, mais:
- data_base_texto (string) - texto original da data-base antes da conversão
- quantidade_cotas_inadimplentes (long)
- quantidade_cotas_ativas_calculada (long)
- taxa_inadimplencia (double)
- taxa_em_dia (double)
- faixa_inadimplencia (string): Baixo, Moderado, Alto, Muito alto
- faixa_prazo (string): Até 120, 121 a 180, 181 a 240, Acima de 240 meses

> As tabelas podem ser visualizadas no Catalog Explorer do Databricks em workspace.mvp_consorcios.
>
> 
> <img width="1352" height="600" alt="image" src="https://github.com/user-attachments/assets/11ae9bff-ea94-4e91-acc5-f28490cd3d43" />


---

## 4. Pipeline de Dados (Etapa 4.4)

### Organização do Pipeline

O pipeline ETL foi organizado em um único notebook (MVP_Consorcios_Imobiliarios), seguindo a arquitetura de camadas Bronze, Silver e Gold. A escolha por um único notebook se deve ao fato de ser um MVP com uma única data-base.

### Fluxo do Pipeline (ETL)

1. **Extract** - Leitura dos arquivos CSV do UC Volume, preservando todas as colunas como texto
2. **Transform** - Padronização de nomes, limpeza de espaços, conversão de tipos, criação de indicadores
3. **Load** - Persistência em três camadas: Bronze (dados normalizados), Silver (dados tratados), Gold (indicadores agregados)

### Estrutura do Notebook

| Seção | Células | Descrição |
|---|---|---|
| Contexto e Perguntas | 1-2 | Objetivo, contexto, perguntas, dados brutos e licença |
| Carga dos Dados | 3-12 | Configuração, verificação e leitura do CSV |
| Pipeline de Dados | 13 | Visão geral do pipeline ETL |
| Inspeção do Esquema | 14-16 | Análise do esquema bruto |
| Padronização | 17-19 | Normalização de nomes de colunas |
| Limpeza e Conversão | 20-22 | Conversão de tipos, trim, CNPJ, datas |
| Criação de Indicadores | 23-24 | Cálculo de cotas ativas, inadimplência |
| Qualidade de Dados | 25-35 | Testes de completude, unicidade, validade |
| Modelagem e Persistência | 36-44 | Tabelas Bronze, Silver e Gold |
| Análise de Dados | 45-84 | Consultas SQL, estatísticas e correlações |
| Segurança e Metadados | 85-86 | Ética, privacidade, metadados e linhagem |
| Autoavaliação | 87-91 | Objetivos, dificuldades, limitações, trabalhos futuros, conclusão |

### Persistência das Tabelas

As tabelas foram persistidas no Databricks usando o formato Delta Lake no Unity Catalog (workspace.mvp_consorcios). O processo utilizou write.format(delta).mode(overwrite).saveAsTable() para cada camada.


> Script de referência: O notebook MVP_Consorcios_Imobiliarios está disponível no GitHub: https://github.com/prchiquito83/mvp-consorcios-imobiliarios

---

## 5. Qualidade de Dados (Etapa 4.5)

### Dimensões Avaliadas

Foram avaliadas: completude, unicidade, validade, consistência, conformidade e plausibilidade.

### Problemas Detectados e Soluções

| Problema | Detecção | Solução |
|---|---|---|
| Nomes de colunas com acentos e espaços | Inspeção do esquema | Função normalizar_nome_coluna() com unicodedata e regex |
| CNPJ com zeros à esquerda | Verificação de tamanho | lpad(trim(cnpj), 8, 0) |
| Valores decimais com vírgula | Inspeção dos dados | regexp_replace (vírgula por ponto) e cast(double) |
| Data-base no formato AAAAMM | Verificação de formato | to_date com concatenação de ano, mês e dia 01 |
| Espaços em branco | Inspeção dos dados | trim() em todas as colunas textuais |
| Valores zeros em colunas de cotas | Contagem de zeros | Mantidos, pois representam situações válidas |
| Duplicidade da chave de negócio | Teste de unicidade | Nenhuma duplicidade encontrada |
| Valores impossíveis | Regras de validação | Nenhuma violação encontrada |

### Transformações Realizadas

1. Padronização de nomes (remoção de acentos, espaços e caracteres especiais)
2. Limpeza textual com trim()
3. CNPJ com lpad para 8 dígitos
4. Data-base convertida de AAAAMM para date
5. Conversão decimal: vírgula para ponto e cast(double)
6. Conversão de inteiros: trim() e cast(long)
7. Criação de indicadores: cotas inadimplentes, cotas ativas calculadas, taxa de inadimplência
8. Classificação de risco: Baixo, Moderado, Alto, Muito alto
9. Classificação de prazo: Até 120, 121 a 180, 181 a 240, Acima de 240 meses

---

## 6. Análise de Dados (Etapa 4.5)

### Consultas SQL

**Consulta 1 - Visão Geral da Camada Gold:** validação do carregamento. Cada administradora possui uma linha com indicadores consolidados.

**Consulta 2 - Administradoras com Mais Grupos:** Bradesco (438), Porto Seguro (422), Itaú (263), HS (152), Santander (147)

**Consulta 3 - Grupos com Maior Inadimplência:** KSK ADM CONS LTDA. domina o topo com taxas de 87,86% a 38,32%

**Consulta 4 - Taxa Consolidada por Administradora:** KSK (72,91%), ALPHA (46,53%), BP (31,33%), Consórcio Reserva (24,53%)

**Consulta 5 - Inadimplência por Faixa de Prazo:** grupos mais longos apresentam maior inadimplência (Acima de 240 meses: 13,48% vs Até 120 meses: 6,70%)

**Consulta 6 - Inadimplência vs Volume Operacional:** as maiores taxas permanecem no topo mesmo após filtro de volume (maior ou igual a 5.000 cotas)

**Consulta 7 - Crédito Pendente:** grupos com até 40% de cotas com crédito pendente, concentrados em SICREDI e SICOOB

**Consulta 8 - Risco Combinado:** 30 grupos com faixa Muito alto e cotas excluídas, liderados por KSK e ALPHA

**Consulta 9 - Distribuição por Faixa de Risco:** Baixo: 855 (30,2%), Moderado: 1.324 (46,8%), Alto: 554 (19,5%), Muito alto: 102 (3,6%)

### Estatísticas Descritivas

- Total de grupos: 2.835
- Total de administradoras: 69
- Valor médio do bem: R$ 253.236,29 (mínimo: R$ 113,68 / máximo: R$ 1.598.745,75)
- Taxa de administração: média 19,31% (mínimo: 0% / máximo: 35%)

### Correlação Exploratória

- Taxa de administração x Inadimplência: correlação = 0,314 (positiva moderada-fraca)
- Prazo x Inadimplência: correlação = 0,190 (positiva fraca)
- Ambas as correlações são de baixa magnitude para fins preditivos
- Correlação não implica causalidade

---

## 7. Autoavaliação

### Objetivos Atingidos

- Construir uma solução mínima viável de dados para organizar, tratar, armazenar e analisar informações públicas sobre grupos de consórcios imobiliários
- Criar indicadores proporcionais que permitam comparar grupos de tamanhos diferentes
- Identificar administradoras e grupos com maior proporção de cotas inadimplentes, excluídas e créditos pendentes
- Persistir os dados em camadas (Bronze, Silver, Gold) no Databricks
- Responder às 10 perguntas de negócio formuladas
- Avaliar a qualidade dos dados em múltiplas dimensões

### Dificuldades Encontradas

1. Tratamento do encoding e separador decimal (windows-1252 e vírgula)
2. Preservação do CNPJ com zeros à esquerda
3. Definição da chave de negócio (testes de unicidade necessários)
4. Ausência de dados históricos (apenas uma data-base)
5. Classificação de risco criada para fins acadêmicos, sem base normativa
6. Correlações de baixa magnitude evidenciando natureza multifatorial
7. Limitações do Databricks Free Edition (compute e armazenamento)

### Limitações

1. A análise utiliza somente a data-base de julho de 2026
2. Não há histórico mensal para estudar tendências
3. Não há dados geográficos
4. Não há dados individuais dos consorciados
5. Não há valor individual da parcela ou saldo devedor
6. Não há motivo de exclusão ou atraso
7. Não há identificação das contemplações por sorteio ou lance
8. A classificação de risco foi criada exclusivamente para fins acadêmicos
9. Os indicadores representam associação descritiva e não causalidade
10. O valor médio do bem não deve ser tratado como saldo devedor

### Trabalhos Futuros

1. Incorporar arquivos mensais anteriores e posteriores a julho de 2026
2. Construir série temporal da inadimplência
3. Identificar entrada e encerramento de grupos ao longo do tempo
4. Adicionar descrição oficial dos códigos de índice de correção
5. Integrar dados econômicos (inflação, juros, desemprego)
6. Incorporar localização da sede ou área de atuação das administradoras
7. Criar controles automatizados de qualidade para cada nova carga
8. Desenvolver alertas para variações anormais de inadimplência
9. Avaliar modelos estatísticos ou de aprendizado de máquina após histórico suficiente
10. Investigar formalmente as diferenças entre os grupos e os valores consolidados
11. Implementar carga incremental e controle de versões
12. Criar painel executivo com filtros de administradora, prazo e faixa de risco
13. Incluir os dados provenientes do arquivo de segmento

### Conclusão

O MVP demonstrou a construção de uma solução de dados para análise de grupos de consórcios imobiliários no Databricks Free Edition. Os arquivos originais foram ingeridos em uma camada bruta, padronizados e convertidos para tipos adequados e transformados em indicadores na camada analítica.

A principal métrica desenvolvida foi a taxa de inadimplência, calculada como a proporção entre as cotas ativas inadimplentes e o total de cotas ativas consideradas. O indicador foi agregado de forma ponderada, evitando que grupos de tamanhos diferentes recebessem o mesmo peso.

A solução permitiu comparar administradoras, identificar grupos com maiores sinais de atenção, avaliar diferenças entre faixas de prazo e confrontar os resultados calculados com o arquivo consolidado.

Como evolução, recomenda-se incorporar diversas datas-base, variáveis econômicas, informações geográficas e dados adicionais. Com isso, a solução poderá evoluir de um MVP descritivo para uma plataforma de monitoramento temporal e apoio à decisão.

---

## Referências

- Material Didático da Especialização PUC-RIO - Engenharia de Dados
- Dicionário de campos e métricas dos dados de consórcios. Fonte: https://www.bcb.gov.br/estabilidadefinanceira/consorciobd
- Arquivos de segmentos consolidados e grupos de bens imóveis, data-base julho de 2026. Fonte: https://www.bcb.gov.br/estabilidadefinanceira/consorciobd
