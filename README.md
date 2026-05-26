# Dashboard Comercial — Análise de Vendas

Teste técnico para **Analista de Dados**. Dashboard desenvolvido em **Power BI** para responder às principais perguntas de negócio do time comercial, a partir de quatro bases de dados (produtos, vendas, clientes e vendedores).

> Todo o tratamento de dados foi realizado dentro do Power BI, utilizando **Power Query (M)** para limpeza/modelagem e **DAX** para os cálculos analíticos.

---

## 📁 Estrutura da entrega

| Arquivo | Descrição |
|---|---|
| `Dashboard - Thiago Amorim.pbix` | Dashboard do Power BI (modelo, transformações, medidas e visuais) |
| `Documentacao - Dashboard Comercial.docx` | Documentação técnica completa |
| `invoices.csv` / `clients.csv` / `products.csv` / `sellers.csv` | Bases originais |
| `Tema Dynamox.json` | Tema visual customizado (identidade da empresa) |

---

## 🗂️ Modelagem

Modelo em **estrela (star schema)** com `invoices` como tabela fato:

| Dimensão | Chave | Cardinalidade |
|---|---|---|
| clients | `business_partner_id` → `business_partner_code` | 1:* |
| products | `item_id` → `item_code` | 1:* |
| sellers | `seller_id` → `seller_code` | 1:* |

Direção do filtro cruzado: **única** (dimensões → fato).

---

## 🧹 Tratamento de dados (Power Query)

A base continha problemas de qualidade que foram tratados:

- **products** — padronização de classificação/categoria (erros de digitação e maiúsculas/minúsculas) em colunas `*_clean`.
- **sellers** — remoção de duplicatas (códigos 87, 78, 92), remoção de registro com `seller_id` nulo, e recuperação do nome do vendedor 74 (Camila Sousa) a partir do e-mail.
- **invoices** — conversão de `invoice_value` de centavos para reais (÷100) e padronização de tipos para garantir os relacionamentos.

---

## 📐 Medidas DAX

```dax
Faturamento Total = SUM(invoices[invoice_value]) / 100

Qtd Vendas = DISTINCTCOUNT(invoices[invoice_number])

Ticket Médio = DIVIDE([Faturamento Total], [Qtd Vendas])

Variedade de Produtos = DISTINCTCOUNT(invoices[item_code])

Clientes Ativos 12m =
VAR DataMax = MAX(invoices[release_date])
VAR DataIni = EDATE(DataMax, -12)
RETURN
CALCULATE(
    DISTINCTCOUNT(invoices[business_partner_code]),
    invoices[release_date] > DataIni && invoices[release_date] <= DataMax
)

Clientes Uma Compra =
COUNTROWS(
    FILTER(
        VALUES(invoices[business_partner_code]),
        CALCULATE(DISTINCTCOUNT(invoices[invoice_number])) = 1
    )
)

Tempo Médio entre compras =
AVERAGEX(
    FILTER(
        VALUES(invoices[business_partner_code]),
        CALCULATE(DISTINCTCOUNT(invoices[release_date])) >= 2
    ),
    VAR Datas = CALCULATETABLE(DISTINCT(invoices[release_date]))
    VAR Primeira = MINX(Datas, invoices[release_date])
    VAR Segunda = MINX(FILTER(Datas, invoices[release_date] > Primeira), invoices[release_date])
    RETURN DATEDIFF(Primeira, Segunda, DAY)
)
```

Coluna calculada para resolver o nome do vendedor com fallback para não cadastrados:

```dax
Vendedor =
VAR cod = invoices[seller_code]
VAR nome = LOOKUPVALUE(sellers[name], sellers[seller_id], cod)
RETURN
SWITCH(
    TRUE(),
    cod = 74, "Camila Sousa",
    ISBLANK(nome), "Não cadastrado",
    nome
)
```

---

## ✅ Respostas às perguntas

| # | Pergunta | Resposta |
|---|---|---|
| 1 | Produtos mais vendidos e sazonalidade | Produtos 101112 e 101110 lideram; vendas se destacam no 2º semestre, pico no **Q4** |
| 2 | Performance dos vendedores | Tabela por faturamento + variedade de produtos (Paulo Vieira, Laura Pinto e Carlos Mendes no topo) |
| 3 | Clientes nos últimos 12 meses | **400** |
| 4 | Clientes com apenas uma compra | **242** |
| 5 | Ticket médio por produto | Tabela por produto; ticket médio geral de **R$ 44,4 mil** |
| 6 | Tempo médio entre 1ª e 2ª compra | **~126 dias** |

Faturamento total: **R$ 227,8 milhões**.

---

## ⚠️ Observações de qualidade de dados

- **~37% do faturamento (R$ 85M)** está associado a códigos de vendedor **sem cadastro** em `sellers.csv` — recomenda-se completar o cadastro.
- Existem **vendas com valor zerado** (`invoice_value = 0`), filtradas na análise de ticket por produto.
- Cadastros com duplicatas, nulos e nomes ausentes foram tratados no Power Query.

---

**Autor:** Thiago Amorim
