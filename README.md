# MVP — Pipeline de Dados em Nuvem para Decisão de Expansão de Data Center

**Disciplina:** Sistemas de Suporte à Decisão | **Professor:** André Luiz Marques Serrano
**Aluna:** Isadora Messenberg Guimarães Macêdo Rebello | **Matrícula:** 231013387
**Departamento:** Engenharia de Produção — Universidade de Brasília

---

## 1. Objetivo

Uma empresa de armazenamento de dados em nuvem precisa decidir a configuração de uma **nova unidade (data center) de aproximadamente 5.000 TB úteis**, minimizando risco operacional e custo total ao longo de 3 anos. Este MVP constrói um pipeline de dados em nuvem que consome dados reais de confiabilidade de discos rígidos, modela essas informações em um Data Warehouse, e entrega análises descritiva, preditiva e prescritiva para apoiar a decisão.

### Perguntas de negócio

1. **Qual a taxa anualizada de falha (AFR) de cada modelo de HD?** — base para o ranking de confiabilidade.
2. **Quais sensores SMART mais antecipam falhas?** — define o plano de monitoramento.
3. **Existe efeito do fabricante na probabilidade de falha?** — orienta negociação com fornecedores.
4. **Faixas de capacidade (pequena, média, grande, enterprise) têm perfis de falha diferentes?** — orienta escolha de densidade física.
5. **Qual modelo minimiza o Custo Total de Propriedade (TCO) em 3 anos para 5.000 TB úteis?** — recomendação final.

---

## 2. Coleta dos dados

| Item | Valor |
|---|---|
| Fonte | Backblaze Hard Drive Data (Q1 2017) |
| Via | [Kaggle — backblaze/hard-drive-test-data](https://www.kaggle.com/datasets/backblaze/hard-drive-test-data) |
| Licença | Backblaze Open Source License (uso irrestrito, exige atribuição) |
| Data de coleta | 2026-11-08 |
| Volume original | 14 arquivos CSV, 301 MB brutos |
| Formato original | CSV com 96 colunas (metadata + atributos SMART) |

---

## 3. Modelagem — Esquema Estrela

```
                 ┌──────────────┐          ┌──────────────┐
                 │  dim_tempo   │          │  dim_modelo  │
                 │  (14 dias)   │          │ (55 modelos) │
                 │              │          │              │
                 │ id_tempo PK  │          │ id_modelo PK │
                 │ date         │          │ model        │
                 │ dia, mes, ano│          │ capacity_TB  │
                 │ dia_semana   │          │ faixa_cap.   │
                 └──────┬───────┘          │ fabricante   │
                        │                  └──────┬───────┘
                        │                         │
                        └─────┐           ┌───────┘
                              ▼           ▼
                     ┌───────────────────────────────┐
                     │      fato_hd_diario           │
                     │     (1.031.501 registros)     │
                     │                               │
                     │  id_tempo (FK)                │
                     │  id_modelo (FK)               │
                     │  serial_number                │
                     │  falhou (métrica binária)     │
                     │  7 atributos SMART            │
                     └───────────────────────────────┘
```

**Grão da tabela-fato:** 1 HD × 1 dia. **Formato físico:** Parquet com compressão Snappy (70% menor que CSV).

---

## 4. Carga — Pipeline ETL

O notebook implementa um pipeline de três etapas explicitamente documentado:

- **Extract** — leitura seletiva dos 14 CSVs, mantendo apenas as colunas úteis (4 base + 7 SMART).
- **Transform** — tipagem de datas, derivação de `capacity_TB`, discretização em `faixa_capacidade`, extração de `fabricante` do nome, construção das dimensões, renomeação dos atributos SMART para nomes de negócio.
- **Load** — gravação em Parquet/CSV no Google Drive montado como persistência em nuvem. Caminho alternativo para Google Cloud Storage comentado no notebook.

---

## 5. Análise de Qualidade dos Dados

Avaliação nas 6 dimensões clássicas:

| Dimensão | Resultado | Observação |
|---|---|---|
| **Completude** | 95% média | 5 SMART com 100%; 2 com ~62% (NaN estrutural reportado pelo fabricante) |
| **Unicidade** | 0 duplicatas | Chave (id_tempo, serial_number) é 100% única |
| **Consistência** | OK | Alvo `falhou` ∈ {0,1}; HDs com falha não reaparecem no período |
| **Conformidade** | 0,02% fora | 211 observações de temperatura fora de [15, 60]°C — provável sensor defeituoso, mantido mas sinalizado |
| **Acurácia** | OK | Horas ligadas plausíveis (mediana ≈ 1,7 anos, máximo ≈ 7,5 anos) |
| **Atualidade** | OK | 14 dias contínuos, sem lacunas |

---

## 6. Principais resultados

### P1 — AFR por modelo
AFR varia de 0% (HGST HMS5C4040ALE640) a 2,77% (HGST HMS5C4040BLE640) entre os modelos com base amostral robusta. Intervalos de confiança 95% calculados via aproximação normal.

### P2 — Sinais SMART mais informativos
Hierarquia por importância Random Forest: **horas ligadas (39%) > temperatura (21%) > setores pendentes (16%) > erros não corrigíveis (10%) > setores incorrigíveis (10%)**. AUC-ROC: 0,55 (RF) e 0,65 (regressão logística). AUC modesto é esperado dada a amostra curta — relevante é a hierarquia dos sinais, consistente com literatura.

### P3 — Efeito do fabricante (inconclusivo)
Fabricantes com base relevante convergem para AFR 1,22–1,57%. Toshiba aparece com 15,56%, mas sobre apenas 4.691 HD-dias — estatisticamente inconclusivo.

### P4 — Efeito da faixa de capacidade
| Faixa | HD-dias | AFR |
|---|---|---|
| ENTERPRISE (≥7 TB) | 122.710 | 0,30% |
| GRANDE (4–7 TB) | 793.180 | 1,70% |
| MÉDIA (1–4 TB) | 93.749 | 1,95% |
| PEQUENA (<1 TB) | 21.862 | 3,34% |

AFR da faixa pequena é **11× maior** que enterprise.

### P5 — TCO em 3 anos (5.000 TB úteis)
| Modelo | AFR | TCO 3a (US$) | TCO/TB |
|---|---:|---:|---:|
| HGST HMS5C4040ALE640 | 0,00% | 187.500 | 37,50 |
| ST8000DM002 | 0,30% | 274.540 | 54,91 |
| ST4000DM000 | 1,80% | 1.234.920 | 246,98 |
| HGST HMS5C4040BLE640 | 2,77% | 1.800.030 | 360,01 |
| WDC WD30EFRX | 2,36% | 2.019.450 | 403,89 |

**Diferença entre melhor e pior cenário: US$ 1,83 milhão** para a mesma capacidade. Em modelos com AFR > 2%, o custo de downtime esperado supera o próprio CAPEX.

---

## 7. Conclusões

1. **Dados operacionais internos são a alavanca principal** para decisões de infraestrutura. Sem histórico de falhas, qualquer escolha de fornecedor é aposta.
2. **A recomendação final não depende do menor preço de aquisição**, e sim do menor TCO por TB útil no horizonte de planejamento. CAPEX baixo é anulado por OPEX alto quando o AFR é ruim.
3. **Manutenção preventiva a cada 6 meses** é uma resposta razoável; a evolução natural é disparar substituições individualmente a partir do modelo preditivo.
4. **Como Sistema de Suporte à Decisão, este MVP atende três critérios essenciais:** transparente (fórmulas explícitas), parametrizável (o decisor ajusta cenários no notebook) e acionável (entrega números concretos — modelo, quantidade, custo, cronograma).

---

## 8. Autoavaliação

**Quais perguntas foram respondidas e quais não?**
Todas as 5 perguntas receberam resposta técnica. A P3 (efeito do fabricante) é a mais frágil: a base da Toshiba é insuficiente para conclusão estatística — reportado como inconclusivo.

**Limitações dos dados que condicionaram os resultados?**
Horizonte de 14 dias desconsidera sazonalidade e efeitos de longo prazo da idade. 45 falhas totais fragmentam demais o treino preditivo. Parâmetros econômicos (preços, custo de downtime) são premissas de mercado, não negociações reais.

**Decisões técnicas que tomaria de outra forma?**
Usar dataset estendido (Q1–Q4 2017). Implementar análise de sobrevivência (Kaplan-Meier) para capturar o efeito da idade. Negociar custos reais antes do exercício prescritivo.

**Extensões para transformar em solução de uso contínuo?**
ETL automatizado em Databricks Jobs. Dashboard em Looker Studio sobre BigQuery. Sistema de alertas por HD individual. Retreinamento mensal do modelo.

---

## 9. Estrutura do repositório

```
├── README.md                                 # este arquivo
├── LICENSE                                   # MIT License para o código
├── notebooks/
│   └── MVP_v2.ipynb                          # pipeline completo
├── scripts/
│   └── etl.py                                # mesmo pipeline em .py para execução batch
├── catalogo/
│   ├── catalogo_dados.csv                    # catálogo formal
│   └── relatorio_qualidade.json              # métricas de qualidade
├── warehouse/
│   ├── fato_hd_diario.parquet                # saída do ETL
│   ├── dim_modelo.parquet
│   ├── dim_tempo.parquet
│   └── afr_por_modelo.csv                    # análises derivadas
└── evidencias/
    ├── 01_colab_execucao.png                 # print da execução no Colab
    ├── 02_drive_warehouse.png                # print dos arquivos persistidos no Drive
    └── 03_resumo_executivo.png               # print do dashboard final
```

---

## 10. Reprodução

1. Baixar o `archive.zip` do [Kaggle](https://www.kaggle.com/datasets/backblaze/hard-drive-test-data)
2. Abrir o notebook `notebooks/MVP_v2.ipynb` no Google Colab
3. Executar as células em ordem — na célula de upload, enviar o `archive.zip`
4. Autorizar a montagem do Google Drive para persistência
5. Ajustar parâmetros do cenário prescritivo (Seção 6) conforme o contexto

**Fonte dos dados:** [Backblaze Hard Drive Data — Kaggle](https://www.kaggle.com/datasets/backblaze/hard-drive-test-data)
