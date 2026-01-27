# Resumo: Arquitetura de Banco de Dados — Guia de Soluções Sebrae

Resumo para implementação no **Supabase** (PostgreSQL). A aplicação hoje usa a tabela `solucoes` e, no futuro, pode incluir `gestores` e `configuracoes`.

---

## 1. Visão geral

| Entidade        | Uso atual na aplicação              | Persistência |
|-----------------|--------------------------------------|--------------|
| **solucoes**    | Cadastro, consulta, importação, exclusão | Supabase     |
| **gestores**    | Login (credenciais fixas no JS)      | *(opcional)* |
| **configuracoes** | Tema, títulos (via elementSdk)     | *(opcional)* |

---

## 2. Tabela `solucoes` (obrigatória)

Armazena as soluções/assessorias exibidas na consulta e no gerenciamento.

### 2.1 Definição SQL

```sql
CREATE TABLE solucoes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  codigo            TEXT,
  momento_empresarial   TEXT,
  crescimento_empresarial TEXT,
  porte             TEXT NOT NULL,
  setor             TEXT NOT NULL,
  assessoria_nome   TEXT NOT NULL,
  assessoria_descricao TEXT,
  segmento          TEXT,
  area_conhecimento TEXT,
  temas             TEXT,
  carga_horaria     TEXT,
  instrumento       TEXT,
  fornecedor        TEXT,
  proxima_data      TEXT,
  valor             NUMERIC(12,2) DEFAULT 0,
  ordem             INT NOT NULL DEFAULT 1,
  criado_em         TIMESTAMPTZ DEFAULT now()
);

-- Índices para filtros da consulta (momento, porte, setor)
CREATE INDEX idx_solucoes_momento_empresarial ON solucoes(momento_empresarial) WHERE momento_empresarial IS NOT NULL AND momento_empresarial != '';
CREATE INDEX idx_solucoes_crescimento_empresarial ON solucoes(crescimento_empresarial) WHERE crescimento_empresarial IS NOT NULL AND crescimento_empresarial != '';
CREATE INDEX idx_solucoes_porte ON solucoes(porte);
CREATE INDEX idx_solucoes_setor ON solucoes(setor);
CREATE INDEX idx_solucoes_ordem ON solucoes(ordem);
```

### 2.2 Colunas e regras de negócio

| Coluna                  | Tipo           | Obrigatório | Descrição |
|-------------------------|----------------|-------------|-----------|
| `id`                    | UUID           | sim (PK)    | Chave primária, gerada pelo banco. |
| `codigo`                | TEXT           | não         | Ex.: 6174. |
| `momento_empresarial`   | TEXT           | não         | **Um** dos: `Gerencie`, `Planeje`, `Organize`, `Lidere`, `Potencialize`. Preenchido quando o tipo é "empresarial". |
| `crescimento_empresarial` | TEXT        | não         | **Um** dos: `Marketing e Vendas`, `Finanças`, `Planejamento`, `Pessoas`, `Operações`. Preenchido quando o tipo é "crescimento". |
| `porte`                 | TEXT           | sim         | `MEI`, `ME` ou `EPP`. |
| `setor`                 | TEXT           | sim         | `Comércio`, `Indústria` ou `Serviços`. |
| `assessoria_nome`       | TEXT           | sim         | Título/nome da solução. |
| `assessoria_descricao`  | TEXT           | não         | Descrição e benefícios. |
| `segmento`             | TEXT           | não         | Ex.: Vestuário, Jóias. |
| `area_conhecimento`     | TEXT           | não         | Ex.: Gestão da Produção. |
| `temas`                 | TEXT           | não         | Temas relacionados. |
| `carga_horaria`        | TEXT           | não         | Ex.: 14. |
| `instrumento`           | TEXT          | não         | Ex.: Cursos Presenciais. |
| `fornecedor`            | TEXT          | não         | Ex.: SENAI. |
| `proxima_data`          | TEXT          | não         | Data da próxima turma/edição (ex.: 15/03/2026 ou A definir). |
| `valor`                 | NUMERIC(12,2) | não         | 0 = gratuito. Valor em R$. |
| `ordem`                 | INT           | sim         | Ordem de exibição (1, 2, …). Usado no `ORDER BY` dos resultados. |
| `criado_em`             | TIMESTAMPTZ   | não         | Data/hora de criação. |

### 2.3 Domínios de valores (para validação/CHECK)

- **porte:** `MEI`, `ME`, `EPP`
- **setor:** `Comércio`, `Indústria`, `Serviços`
- **momento_empresarial:** `Gerencie`, `Planeje`, `Organize`, `Lidere`, `Potencialize`
- **crescimento_empresarial:** `Marketing e Vendas`, `Finanças`, `Planejamento`, `Pessoas`, `Operações`

*(Opcional: criar `CHECK` ou tabelas auxiliares para garantir só esses valores.)*

### 2.4 Uso na aplicação

- **Consulta:**  
  Filtro por `momento_empresarial` **ou** `crescimento_empresarial` (conforme o tipo de momento), `porte` e `setor`. Ordenação por `ordem`; limitação a 2 resultados.
- **Gerenciamento:**  
  `SELECT *`, `INSERT`, `DELETE` por `id`. A importação CSV/Excel monta registros nesse formato e chama o mesmo `INSERT`.

---

## 3. Row Level Security (RLS) — `solucoes`

Recomendação para uso com a **anon key** do Supabase:

- **SELECT:** liberado para todos (consulta é pública).
- **INSERT / DELETE:** apenas para usuários autenticados (ou anon, se a estratégia de segurança permitir).

Exemplo de políticas (ajuste `auth.role()` conforme Supabase Auth ou `service_role`):

```sql
ALTER TABLE solucoes ENABLE ROW LEVEL SECURITY;

-- Leitura pública (consulta)
CREATE POLICY "Permitir leitura pública de solucoes"
  ON solucoes FOR SELECT
  USING (true);

-- Inserir: somente autenticados (ex.: gestores)
CREATE POLICY "Permitir insert para gestores"
  ON solucoes FOR INSERT
  WITH CHECK (auth.role() = 'authenticated');
-- Se ainda não usar Supabase Auth, use WITH CHECK (true) temporariamente.

-- Deletar: somente autenticados
CREATE POLICY "Permitir delete para gestores"
  ON solucoes FOR DELETE
  USING (auth.role() = 'authenticated');
```

Se o gerenciamento for feito ainda **sem** Supabase Auth (somente front e anon key), pode-se, **só em ambiente controlado**, usar `USING (true)` e `WITH CHECK (true)` para INSERT/DELETE, e migrar para Auth depois.

---

## 4. Tabela `gestores` (opcional)

Hoje o login usa credenciais fixas no JS (`GESTOR_CREDENTIALS`) e `localStorage` para `gestor_autenticado`. Para centralizar no banco:

### 4.1 Modelo sugerido

```sql
CREATE TABLE gestores (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  usuario    TEXT NOT NULL UNIQUE,
  senha_hash TEXT NOT NULL,
  ativo      BOOLEAN DEFAULT true,
  criado_em  TIMESTAMPTZ DEFAULT now()
);
```

- **Alternativa recomendada:** usar **Supabase Auth** (`auth.users`) em vez de tabela própria de senha. A tabela `gestores` pode então guardar apenas dados extras (ex.: `user_id` referenciando `auth.users.id`).

### 4.2 Quando implementar

- Se mantiver login somente no front: não é necessário.
- Se for usar Supabase Auth: criar `gestores` (ou só `auth.users`) e trocar o fluxo de login no JS para `signInWithPassword` (ou similar) e usar `auth.role() = 'authenticated'` nas políticas de `solucoes`.

---

## 5. Tabela `configuracoes` (opcional)

A aplicação usa `defaultConfig` e `onConfigChange` (ex.: `elementSdk`). Para persistir no banco:

```sql
CREATE TABLE configuracoes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chave             TEXT NOT NULL UNIQUE,
  valor             JSONB NOT NULL,
  atualizado_em      TIMESTAMPTZ DEFAULT now()
);
```

Exemplo de chave: `app_guia` com `valor`:

```json
{
  "titulo_principal": "Guia de Soluções Sebrae",
  "subtitulo": "Encontre a solução ideal para o diagnóstico",
  "cor_principal": "#005CA9",
  "cor_secundaria": "#00A859",
  "cor_fundo": "#f0f7ff",
  "cor_texto": "#333333",
  "fonte_familia": "Montserrat",
  "tamanho_fonte": 16
}
```

- **Políticas:** `SELECT` público; `UPDATE` apenas para gestores/authenticated, se quiser edição pelo painel.

---

## 6. Ordem de implementação sugerida

1. **Fase 1 (atual)**  
   - Criar `solucoes` com o `CREATE TABLE` e índices da seção 2.  
   - Habilitar RLS e políticas de SELECT (público) e INSERT/DELETE (conforme sua estratégia: anon temporário ou Auth).

2. **Fase 2 (evolução)**  
   - Adotar **Supabase Auth** para gestores.  
   - (Opcional) Tabela `gestores` ou uso direto de `auth.users`.  
   - Ajustar políticas de `solucoes` para `auth.role() = 'authenticated'`.

3. **Fase 3 (opcional)**  
   - Tabela `configuracoes` para tema e textos.  
   - Ajustar o front para carregar e salvar `configuracoes` em vez de (ou em conjunto com) `defaultConfig`.

---

## 7. Diagrama simplificado

```
┌─────────────────────────────────────────────────────────────┐
│                      solucoes (obrigatória)                 │
├─────────────────────────────────────────────────────────────┤
│ id (PK, UUID)     assessoria_nome (*)   valor (0=gratuito)   │
│ codigo            assessoria_descricao ordem                │
│ momento_empresarial  segmento          criado_em            │
│ crescimento_empresarial  area_conhecimento, temas           │
│ porte (*)         carga_horaria, instrumento, fornecedor    │
│ setor (*)                                                   │
└─────────────────────────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐
│  gestores        │     │  configuracoes   │
│  (opcional)      │     │  (opcional)      │
├──────────────────┤     ├──────────────────┤
│ id, usuario,     │     │ id, chave,       │
│ senha_hash, ativo │     │ valor (JSONB)    │
└──────────────────┘     └──────────────────┘
```

---

## 8. Respostas da aplicação ao banco

| Ação                | Tabela     | Operação SQL (Supabase)                    |
|---------------------|-----------|---------------------------------------------|
| Carregar soluções   | `solucoes`| `SELECT * FROM solucoes`                    |
| Cadastrar solução   | `solucoes`| `INSERT` (sem `id`; `id` e `criado_em` via default) |
| Excluir solução     | `solucoes`| `DELETE FROM solucoes WHERE id = $id`       |
| Importar CSV/Excel  | `solucoes`| Vários `INSERT` (um por linha válida)       |

Não há `UPDATE` de `solucoes` na aplicação hoje; as alterações são feitas por exclusão e novo cadastro. Se no futuro houver edição, incluir política RLS para `UPDATE` em `solucoes`.

---

## 9. Checklist de implementação (Supabase)

- [ ] Criar tabela `solucoes` (SQL da seção 2.1).  
- [ ] Criar índices em `momento_empresarial`, `crescimento_empresarial`, `porte`, `setor`, `ordem`.  
- [ ] `ALTER TABLE solucoes ENABLE ROW LEVEL SECURITY`.  
- [ ] Política `SELECT` com `USING (true)`.  
- [ ] Políticas `INSERT` e `DELETE` para anon (temporário) ou `authenticated` (com Auth).  
- [ ] Preencher `SUPABASE_URL` e `SUPABASE_ANON_KEY` no `GuiaSebrae.html`.  
- [ ] (Opcional) Supabase Auth + `gestores` e políticas por `authenticated`.  
- [ ] (Opcional) Tabela `configuracoes` e integração no front.
