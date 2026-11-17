# Olist E-commerce

Repositório com projetos de dados e machine learning usando a base pública da **Olist**, um marketplace brasileiro de e-commerce. A base contém dados de pedidos, produtos, vendedores, clientes, pagamentos, avaliações e geolocalização — mais de 1 milhão de registros geolocalizados e cerca de 100 mil pedidos.

A ideia é construir uma **camada de dados compartilhada** (bronze → silver) e, a partir dela, criar **projetos independentes**, cada um com sua própria tabela gold, EDA e modelagem.

## Plataforma

Todos os projetos são desenvolvidos no **Databricks**, aproveitando suas ferramentas nativas para organizar o pipeline de ponta a ponta:

- **Unity Catalog** — governança dos dados com catálogos, schemas e tabelas Delta, controle de acesso e volumes para arquivos
- **Notebooks Databricks** — desenvolvimento interativo em Python e SQL, com execução em clusters gerenciados
- **Delta Lake** — formato de armazenamento transacional (ACID) para todas as tabelas, com suporte a merge, otimização e versionamento
- **Volumes do Unity Catalog** — armazenamento dos CSVs originais antes da ingestão para o bronze
- **SQL** — usado na criação de catálogos, schemas, volumes e tabelas, e em consultas analíticas
- **PySpark** — processamento distribuído para limpeza, transformação e feature engineering em escala
- **Databricks SQL / Dashboards** — consultas SQL e visualizações para análise exploratória e relatórios
- **Integração com BI** — conexão com ferramentas externas como Power BI para visualização e dashboards

Essa estrutura permite separar claramente as camadas de dados (bronze, silver, gold) e criar projetos independentes que consomem a mesma base limpa.

---

## Arquitetura (Medallion)

Os notebooks `00_setup`, `01_bronze` e `02_silver` rodam uma única vez e preparam a base para todos os projetos.

| Notebook | Função |
| --- | --- |
| `00_setup` | Cria o catalog `olist`, os schemas `bronze`, `silver` e `gold`, e o volume `olist.bronze.raw` no Unity Catalog |
| `01_bronze` | Lê os 9 CSVs originais do volume e salva como tabelas Delta em `olist.bronze.*` |
| `02_silver` | Limpa, tipa e deduplica cada tabela bronze, salvando em `olist.silver.*` com checagens de qualidade (duplicados, nulos, integridade referencial e validações de domínio) |

### Tabelas na silver

| Tabela | Linhas | Descrição |
| --- | --- | --- |
| `olist.silver.orders` | 99.441 | Pedidos com timestamps de compra, aprovação e entrega |
| `olist.silver.order_items` | 112.650 | Itens de cada pedido com preço e frete |
| `olist.silver.order_payments` | 103.886 | Pagamentos por pedido (tipo, parcelas, valor) |
| `olist.silver.order_reviews` | 99.224 | Avaliações dos clientes (nota de 1 a 5) |
| `olist.silver.products` | 32.951 | Produtos com categoria, dimensões e peso |
| `olist.silver.customers` | 99.441 | Clientes com CEP, cidade e estado |
| `olist.silver.sellers` | 3.095 | Vendedores com CEP, cidade e estado |
| `olist.silver.geolocation` | 1.000.163 | Coordenadas por prefixo de CEP |
| `olist.silver.category_translation` | 71 | Tradução de categorias (PT → EN) |

---

## Projetos

Cada projeto mora em sua própria pasta e tem seu próprio objetivo, tabela gold, análise exploratória e modelagem. Os notebooks variam conforme a necessidade de cada projeto.

### 📁 Projeto Pricing — Previsão de Preços

**Pasta:** [projeto pricing/](https://github.com/yuji0ka/Olist-E-commerce/tree/main/projeto%20pricing)

**Problema:** estimar o preço de produtos no marketplace a partir de suas características (categoria, peso, dimensões, vendedor, frete, variáveis de calendário).

**Abordagem:**
- Split temporal (80/20, sem embaralhamento)
- Validação cruzada com `TimeSeriesSplit` (3 folds)
- Modelos: **CatBoost** (melhor) e XGBoost, ambos com `log1p(price)`
- 4 feature sets testados: {volume vs dimensões} × {com vs sem frete}
- Tuning manual com busca aleatória (15 trials CatBoost, 12 XGBoost)

**Resultado final (CatBoost, `dims_com_frete` + log):**

| Métrica | Treino | Teste |
| --- | --- | --- |
| MAE | 30,47 | 49,38 |
| MAPE | 0,22 | 0,46 |
| RMSE | 106,47 | 147,35 |

O modelo melhora ~40% o MAE em relação ao baseline (mediana = R$ 82,10). O gap treino-teste (overfitting) é atribuído à alta cardinalidade de `product_id` (44% dos produtos do teste são desconhecidos no treino) e `seller_id`.

> Todos os detalhes — gold, EDA completa, feature engineering e modelagem — estão nos notebooks dentro da pasta [projeto pricing/](https://github.com/yuji0ka/Olist-E-commerce/tree/main/projeto%20pricing).

---

## Fonte dos dados

Base pública da Olist no Kaggle: https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce

## Ordem de execução

1. `00_setup` — cria catalog, schemas e volume
2. `01_bronze` — carrega os CSVs para o bronze
3. `02_silver` — limpa e tipa as tabelas
4. `projeto pricing/03_gold` → `EDA_pricing` → `modelagem` — pipeline do projeto pricing

> Novos projetos serão adicionados conforme forem criados, cada um com sua própria pasta e tabela gold.
