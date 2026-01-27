# Configuração do Supabase — Guia Sebrae

Passo a passo para conectar o **Guia de Soluções Sebrae** ao Supabase e usar o banco em nuvem em vez do armazenamento local.

---

## 1. Onde pegar a URL e a chave (Project URL e anon key)

1. Acesse [supabase.com](https://supabase.com) e faça login.
2. Abra seu **projeto** (ou crie um novo).
3. No menu lateral, vá em **Project Settings** (ícone de engrenagem).
4. Clique em **API** no submenu.
5. Copie:
   - **Project URL** → será o valor de `SUPABASE_URL`
   - **anon public** (em "Project API keys") → será o valor de `SUPABASE_ANON_KEY`

> A chave **anon public** é segura para uso no navegador; o controle de acesso é feito pelas políticas RLS no banco.

---

## 2. Criar a tabela `solucoes` no Supabase

1. No painel do Supabase, abra **SQL Editor**.
2. Crie uma nova query e execute o SQL abaixo.

### 2.1 Se a tabela ainda **não existe**

```sql
CREATE TABLE solucoes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  codigo            TEXT,
  momento_empresarial   TEXT,
  crescimento_empresarial TEXT,
  porte             TEXT NOT NULL,
  setor             TEXT NOT NULL,
  assessoria_nome   TEXT NOT NULL,
  assessoria_descricao   TEXT,
  segmento          TEXT,
  area_conhecimento TEXT,
  temas             TEXT,
  carga_horaria     TEXT,
  instrumento       TEXT,
  fornecedor        TEXT,
  proxima_data     TEXT,
  valor             NUMERIC(12,2) DEFAULT 0,
  ordem             INT NOT NULL DEFAULT 1,
  criado_em         TIMESTAMPTZ DEFAULT now()
);

-- Índices (recomendado)
CREATE INDEX idx_solucoes_momento_empresarial ON solucoes(momento_empresarial) WHERE momento_empresarial IS NOT NULL AND momento_empresarial != '';
CREATE INDEX idx_solucoes_crescimento_empresarial ON solucoes(crescimento_empresarial) WHERE crescimento_empresarial IS NOT NULL AND crescimento_empresarial != '';
CREATE INDEX idx_solucoes_porte ON solucoes(porte);
CREATE INDEX idx_solucoes_setor ON solucoes(setor);
CREATE INDEX idx_solucoes_ordem ON solucoes(ordem);
```

### 2.2 Se a tabela **já existe** e falta a coluna `proxima_data`

```sql
ALTER TABLE solucoes ADD COLUMN IF NOT EXISTS proxima_data TEXT;
```

---

## 3. Row Level Security (RLS) e políticas

O Supabase usa RLS. Para o Guia funcionar **sem** Supabase Auth (apenas com a anon key), é preciso liberar as operações.

### 3.1 Habilitar RLS

```sql
ALTER TABLE solucoes ENABLE ROW LEVEL SECURITY;
```

### 3.2 Políticas para uso só com anon key (ambiente controlado)

Se você **ainda não** usa login do Supabase (Auth), use políticas que permitem tudo na tabela `solucoes`:

```sql
-- Leitura para todos (consulta pública)
CREATE POLICY "Leitura publica solucoes"
  ON solucoes FOR SELECT
  USING (true);

-- Inserir (cadastro/importação) – anon
CREATE POLICY "Insert anon solucoes"
  ON solucoes FOR INSERT
  WITH CHECK (true);

-- Deletar – anon
CREATE POLICY "Delete anon solucoes"
  ON solucoes FOR DELETE
  USING (true);
```

> **Atenção:** `WITH CHECK (true)` e `USING (true)` deixam qualquer pessoa com a anon key inserir e apagar. Use só em ambiente interno/controlado. Quando adotar Supabase Auth para gestores, troque para `auth.role() = 'authenticated'`.

### 3.3 (Opcional) Se der conflito de política

Se aparecer erro ao criar políticas (ex.: nome duplicado), apague as antigas:

```sql
DROP POLICY IF EXISTS "Leitura publica solucoes" ON solucoes;
DROP POLICY IF EXISTS "Insert anon solucoes" ON solucoes;
DROP POLICY IF EXISTS "Delete anon solucoes" ON solucoes;
```

Depois execute novamente os `CREATE POLICY` da seção 3.2.

---

## 4. Onde colar no `GuiaSebrae.html`

No início do bloco de script, procure as linhas:

```javascript
const SUPABASE_URL = '';   // ex: https://seu-projeto.supabase.co
const SUPABASE_ANON_KEY = ''; // ex: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Substitua pelos valores do seu projeto:

```javascript
const SUPABASE_URL = 'https://XXXXX.supabase.co';
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.XXXX...';
```

- **Project URL** → `SUPABASE_URL`
- **anon public** → `SUPABASE_ANON_KEY`

Salve o arquivo.

---

## 5. Abrir o HTML por HTTP (não por `file://`)

O `import()` do Supabase e as requisições à API precisam rodar em um contexto **http** ou **https**. Abrir o `GuiaSebrae.html` direto do disco (`file:///...`) pode causar:

- Erro de CORS
- Falha ao carregar `@supabase/supabase-js`

**Como resolver:**

1. **Servidor local (recomendado)**  
   - Na pasta em que está o `GuiaSebrae.html`, suba um servidor, por exemplo:
     - **Node:** `npx serve .` ou `npx http-server .`
     - **Python 3:** `python -m http.server 8000`
   - Acesse: `http://localhost:3000/GuiaSebrae.html` (ou a porta que o comando indicar).

2. **Hospedagem**  
   - Publique o HTML em um site (ex.: GitHub Pages, Netlify, servidor do Sebrae) e abra via `https://...`.

3. **VS Code / Cursor**  
   - Use a extensão "Live Server" e abra o arquivo com "Open with Live Server"; a URL será algo como `http://127.0.0.1:5500/GuiaSebrae.html`.

---

## 6. Como saber se está conectado

- Na **sidebar**, no rodapé, aparece o indicador de banco:
  - **Supabase** (bolinha verde): leitura/escrita indo para o Supabase.
  - **Local** (bolinha cinza): usando `localStorage` (URL/KEY vazios, erro de rede, RLS, tabela inexistente, etc.).

- Se URL e KEY estiverem preenchidos e ainda assim aparecer **Local**, deve abrir o **Console** do navegador (F12 → Console) e ver a mensagem de erro (ex.: `Supabase: erro ao carregar`, `falha ao carregar client`). Isso ajuda a saber se o problema é:
  - Tabela `solucoes` não criada
  - RLS bloqueando
  - CORS (geralmente ao usar `file://`)
  - URL ou KEY incorretos

---

## 7. Checklist rápido

- [ ] Projeto criado no Supabase  
- [ ] **Project URL** e **anon public** copiados  
- [ ] Tabela `solucoes` criada (ou `proxima_data` adicionada se já existia)  
- [ ] RLS habilitado e políticas `SELECT` / `INSERT` / `DELETE` criadas  
- [ ] `SUPABASE_URL` e `SUPABASE_ANON_KEY` preenchidos no `GuiaSebrae.html`  
- [ ] Página aberta via **http** ou **https** (não `file://`)  
- [ ] Indicador na sidebar em **Supabase** (verde)

---

## 8. Referência da arquitetura

Consulte também o arquivo **ARQUITETURA-BANCO-DADOS-GUIA-SEBRAE.md** para o desenho completo do banco, índices, RLS com Auth e tabelas opcionais (`gestores`, `configuracoes`).
