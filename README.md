# Guia de Soluções Sebrae

Aplicação web para consulta e gerenciamento de soluções e assessorias do Sebrae.

## Estrutura do Projeto

- `GuiaSebrae.html` - Aplicação principal (HTML único com JavaScript embutido)
- `ARQUITETURA-BANCO-DADOS-GUIA-SEBRAE.md` - Documentação da arquitetura do banco de dados
- `CONFIGURACAO-SUPABASE.md` - Guia de configuração do Supabase

## Tecnologias Utilizadas

- HTML5
- Tailwind CSS (via CDN)
- JavaScript (Vanilla)
- Supabase (PostgreSQL)
- SheetJS (para importação de Excel)

## Configuração

Consulte o arquivo `CONFIGURACAO-SUPABASE.md` para instruções detalhadas de configuração do Supabase.

## Funcionalidades

- Consulta de soluções por momento empresarial, porte e setor
- Gerenciamento de soluções (cadastro, exclusão, importação)
- Interface responsiva e moderna
- Integração com Supabase para persistência de dados

## Deploy

Este projeto é um arquivo HTML estático e não requer build. Arquivos de configuração estão incluídos para:

- **Vercel**: Use `vercel.json` (já configurado)
- **Netlify**: Use `netlify.toml` (já configurado)
- **GitHub Pages**: Configure para servir o arquivo `GuiaSebrae.html`

### Importante

O projeto **NÃO** é Angular e não requer `ng build`. Se a plataforma de deploy tentar executar `ng build`, verifique:
1. Se os arquivos `vercel.json` ou `netlify.toml` estão no repositório
2. Se o `package.json` está configurado corretamente
3. Desabilite a detecção automática de framework na plataforma de deploy
