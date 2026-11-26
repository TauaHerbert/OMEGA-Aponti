# Diagnóstico do Banco de Dados — GastroCore ERP v0.3.0

*Responsável:* Integrante 01 (Banco de Dados e Diagnóstico SQL)
*Branch:* sprint-1/da-N/database-diagnostic
*Fonte analisada:* schema.sql v0.3.0 (PostgreSQL / Supabase)

---

## 1. Instruções de Execução Local

O schema.sql utiliza políticas de Row Level Security (RLS) voltadas aos roles anon e authenticated do Supabase. Em uma instância local de PostgreSQL (pgAdmin ou Docker), execute o preparo abaixo antes de rodar o schema:

sql
CREATE DATABASE gastrocore_lab;
-- Conectar ao banco gastrocore_lab:
CREATE ROLE anon NOLOGIN;
CREATE ROLE authenticated NOLOGIN;


Em seguida, execute na ordem:
1. schema.sql (criação das 13 tabelas, constraints, políticas RLS e carga seed).
2. sql/schema_diagnostic.sql (auditoria estrutural e contagem de linhas).
3. sql/orphan_keys.sql (execução da regra *DQ-05*).

---

## 2. Inventário das 13 Tabelas

| # | Tabela | PK | FKs Formais | FKs Implícitas (sem constraint) | Seed Inicial |
|---|--------|----|-------------|---------------------------------|--------------|
| 1 | kitchen_units | id | Nenhuma | Nenhuma | 3 linhas |
| 2 | ingredients | id | unit_id -> kitchen_units(id) | Nenhuma | 0 linhas |
| 3 | technical_sheets | id | Nenhuma | ingredients (JSONB -> provável ingredients.id)* | 0 linhas |
| 4 | daily_menu_plans | id | unit_id -> kitchen_units(id) | 8 colunas *_ids (TEXT[]) -> technical_sheets(id) | 0 linhas |
| 5 | waste_records | id | Nenhuma | unit_id -> kitchen_units(id) | 0 linhas |
| 6 | hot_box_deliveries | id | unit_id -> kitchen_units(id) | Nenhuma | 0 linhas |
| 7 | proposal_quotes | id | unit_id -> kitchen_units(id) | Nenhuma | 0 linhas |
| 8 | tenant_configs | id | Nenhuma | Nenhuma | 1 linha |
| 9 | app_users | id | unit_id -> kitchen_units(id) | Nenhuma | 7 linhas |
| 10 | temperature_logs | id | unit_id -> kitchen_units(id) | Nenhuma | 0 linhas |
| 11 | stock_movements | id | Nenhuma | ingredient_id -> ingredients(id) | 0 linhas |
| 12 | kds_dish_status | id | unit_id -> kitchen_units(id) | recipe_id -> technical_sheets(id) | 0 linhas |
| 13 | system_heartbeat | id | Nenhuma | Nenhuma | 0 linhas |

\*Nota: a estrutura interna do JSONB technical_sheets.ingredients e ingredients.batches precisa ser confirmada pelo Integrante 02 via gastrocore_schema.json e CSVs.

---

## 3. Mapa de Relacionamentos (Integridade Referencial)

### 3.1 FKs Formais (protegidas pelo PostgreSQL)
* ingredients.unit_id -> kitchen_units.id (ON DELETE SET NULL)
* daily_menu_plans.unit_id -> kitchen_units.id (ON DELETE SET NULL)
* hot_box_deliveries.unit_id -> kitchen_units.id (ON DELETE SET NULL)
* proposal_quotes.unit_id -> kitchen_units.id (ON DELETE SET NULL)
* app_users.unit_id -> kitchen_units.id (ON DELETE SET NULL) — user-admin e user-financeiro possuem unit_id = NULL no seed (válido para perfis globais).
* temperature_logs.unit_id -> kitchen_units.id (ON DELETE SET NULL)
* kds_dish_status.unit_id -> kitchen_units.id (ON DELETE CASCADE) — única tabela com exclusão em cascata.

### 3.2 FKs Implícitas (risco real de chaves órfãs — cobertas no DQ-05)
* waste_records.unit_id -> kitchen_units.id (declarada como TEXT NOT NULL sem REFERENCES).
* stock_movements.ingredient_id -> ingredients.id (declarada como TEXT NOT NULL sem REFERENCES).
* kds_dish_status.recipe_id -> technical_sheets.id (declarada como TEXT NOT NULL sem REFERENCES).
* daily_menu_plans (principais_ids, guarnicoes_ids, bases_ids, saladas_ids, molhos_ids, sobremesas_ids, bebidas_ids, dietas_ids) -> technical_sheets.id (colunas TEXT[], sem suporte nativo a FK no PostgreSQL).

---

## 4. Achados Técnicos e Riscos de Qualidade no Schema

1. *Ausência de FKs em tabelas transacionais críticas:* waste_records, stock_movements e kds_dish_status permitem inserção de IDs inexistentes sem bloqueio do SGBD.
2. *Datas e horários armazenados como TEXT:* colunas como ingredients.expiration_date, daily_menu_plans.date_str, waste_records.timestamp, proposal_quotes.event_date, temperature_logs.time_str, stock_movements.timestamp e kds_dish_status.date_str são TEXT. Isso exige parsing defensivo (errors="coerce") nas validações em Python.
3. *Colunas duplicadas em technical_sheets:* existem tanto allergens_summary TEXT[] quanto allergens TEXT[]. Confirmar com o Integrante 02 qual é o campo oficial.
4. *Sensibilidade de maiúsculas/minúsculas em unidades:* ingredients.unit aceita tanto 'l' quanto 'L' no CHECK, o que pode gerar duplicidade analítica se não for normalizado.
5. *Segurança (modo piloto):* todas as 13 tabelas possuem política RLS permissiva (USING (true) WITH CHECK (true)) para anon e authenticated, e app_users.password_hash contém senhas em texto plano no seed.

---

## 5. Mapeamento de Campos para o Squad

### Para Integrante 02 (Dicionário e Datasets)
* Conferir divergências entre os nomes físicos das 13 tabelas acima e os CSVs/JSON oficiais (especialmente estrutura interna de ingredients.batches e technical_sheets.ingredients).

### Para Integrante 03 (DQ-01: PB >= PL | DQ-02: FC >= 1)
* *Atenção:* o schema.sql *não possui* colunas explícitas de Peso Bruto (PB), Peso Líquido (PL) ou Fator de Correção (FC) nas tabelas relacionais.
* A tabela technical_sheets possui cooking_factor (Fator de Cocção, não Fator de Correção) e um campo ingredients JSONB.
* As colunas reais de PB, PL e FC devem vir do dataset gastrocore_processamento_proteinas.csv (confirmar nomes físicos com o Integrante 02).

### Para Integrante 04 (DQ-03: Vencimento | DQ-04: Estoque)
* Tabela fonte no banco: ingredients
  * Estoque atual: current_stock (NUMERIC NOT NULL DEFAULT 0)
  * Estoque mínimo: min_stock (NUMERIC NOT NULL DEFAULT 0)
  * Data de validade: expiration_date (TEXT, aceita NULL — tratar formatos inválidos e valores nulos)
  * Lotes adicionais: batches (JSONB, default '[]')

### Para Integrante 05 (Integração — DQ-05)
* Executar sql/orphan_keys.sql via psycopg contra o banco PostgreSQL/Supabase.
* *Contrato de retorno garantido pela query:*
  rule_id, source, record_id, field, actual_value, expected_value, message, status
* *Resultado esperado no seed oficial:* 0 linhas (zero chaves órfãs).
* *Validação de funcionamento:* testado com 4 registros órfãos controlados (waste_records, stock_movements, kds_dish_status e daily_menu_plans.principais_ids), todos detectados corretamente, seguido de limpeza e retorno a 0 ocorrências.