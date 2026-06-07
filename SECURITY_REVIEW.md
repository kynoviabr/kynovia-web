# Revisao de Arquitetura e Seguranca

Data: 2026-06-07

## Escopo

- Website Kynovia: `www.kynovia.com.br`
- Pagina de modelo: `/modelos/riot-burger/`
- Aplicacao VitrineWeb: `vitrineweb.kynovia.com.br`

## Resumo executivo

O website Kynovia e as paginas de modelos sao majoritariamente estaticos, sem backend neste repositorio. Isso reduz a superficie de ataque. O principal ponto de atencao e a aplicacao VitrineWeb, que possui login, onboarding, portal de cliente/admin e integracao com Supabase.

## Acoes aplicadas neste repositorio

- Adicionados headers de seguranca no `vercel.json`:
  - `Content-Security-Policy`
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
  - `X-Frame-Options: SAMEORIGIN`
  - `Permissions-Policy`

## Observacoes sobre o website Kynovia

- HTTPS e redirecionamento HTTP -> HTTPS estao funcionando na Vercel.
- O site utiliza scripts e estilos inline. Por isso, a CSP atual permite `'unsafe-inline'` para evitar quebra das paginas.
- Uma CSP mais forte exigiria mover JS e CSS inline para arquivos separados e remover handlers inline como `onclick` e `onerror`.
- As paginas usam imagens externas, principalmente Unsplash e Pexels. A CSP foi configurada para permitir esses dominios.

## Observacoes sobre VitrineWeb

O dominio `vitrineweb.kynovia.com.br` publica uma aplicacao client-side com rotas como:

- `/login.html`
- `/portal-admin.html`
- `/portal-cliente.html`
- `/onboarding.html`
- `/resultado.html`

O bundle publico referencia Supabase. Isso e normal quando se usa Supabase no frontend, mas exige Row Level Security bem configurado.

## Checklist recomendado para VitrineWeb

Prioridade alta:

- Confirmar que RLS esta habilitado em todas as tabelas com dados de clientes, pedidos, onboarding, diagnosticos e usuarios.
- Confirmar que nenhuma policy permite `select`, `insert`, `update` ou `delete` amplo usando apenas a chave `anon`.
- Validar que o portal admin checa perfil/permissao no banco, nao apenas no frontend.
- Revisar Storage buckets do Supabase para garantir que arquivos privados nao estejam publicos.
- Confirmar que nao existe service role key exposta em bundle, HTML, logs ou variaveis publicas.

Prioridade media:

- Separar a landing publica do VitrineWeb do portal administrativo em projetos/subdominios distintos.
- Adicionar headers de seguranca tambem no projeto VitrineWeb.
- Aplicar rate limit/captcha em fluxos publicos de cadastro, login, recuperacao de senha e diagnostico.
- Revisar politicas de CORS para evitar origem ampla quando nao for necessario.

## Proximo passo tecnico

Auditar diretamente o projeto Supabase da VitrineWeb:

```sql
select schemaname, tablename, rowsecurity
from pg_tables
where schemaname = 'public'
order by tablename;

select schemaname, tablename, policyname, cmd, roles, qual, with_check
from pg_policies
where schemaname = 'public'
order by tablename, policyname;
```

