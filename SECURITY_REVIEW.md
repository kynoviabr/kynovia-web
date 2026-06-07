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

## Auditoria local encontrada para VitrineWeb

Foi localizado em `/Users/dempas/Downloads/auditoria-seguranca-vitrineweb.md` um relatorio especifico para o projeto Supabase `utnezxtsvxhxwthwkrlf`, datado de 2026-05-08. Ele indica que algumas correcoes de frontend ja haviam sido aplicadas, mas que ainda havia correcoes criticas pendentes no banco.

Principais achados desse relatorio:

- `vitrineweb_pedidos` permitia `UPDATE` amplo para cliente, criando risco de privilege escalation por alteracao de campos administrativos como etapa, valor, plano, status e contrato assinado.
- A autorizacao dependia de comparacao por e-mail (`jwt.email = pedido.email`), o que abre risco de email squatting/account takeover para pedidos ainda nao vinculados a um `auth.uid()`.
- Havia XSS armazenado em telas administrativas por uso de `innerHTML` sem escape; o relatorio indica que isso foi corrigido em `portal-admin`, `portal-cliente` e `contrato`.
- `vitrineweb_contratos` nao tinha garantia de contrato unico por `pedido_id`, permitindo assinaturas duplicadas/conflitantes.
- A RPC `save_vitrineweb_onboarding` usava `SECURITY DEFINER` e tambem dependia do modelo antigo de ownership por e-mail.
- O bucket `vitrineweb-assets` era publico; isso pode ser aceitavel para imagens comerciais, mas nao para documentos sensiveis.

Ponto de cautela: o SQL de hardening encontrado no relatorio local contem uma secao incompleta para `save_vitrineweb_onboarding`, com instrucao para reutilizar o corpo vigente da RPC. Portanto, esse SQL nao deve ser executado integralmente sem antes recuperar a definicao atual da funcao no Supabase.

## Checklist recomendado para VitrineWeb

Prioridade alta:

- Obter acesso ao projeto Supabase `utnezxtsvxhxwthwkrlf`; ele nao apareceu no conector Supabase desta sessao.
- Confirmar que RLS esta habilitado em todas as tabelas com dados de clientes, pedidos, onboarding, diagnosticos e usuarios.
- Substituir ownership por e-mail por ownership por `auth.uid()`/`auth_user_id`, mantendo e-mail apenas como etapa de claim com `email_verified = true`.
- Bloquear `UPDATE` de cliente em campos administrativos de `vitrineweb_pedidos`.
- Adicionar `UNIQUE(pedido_id)` em `vitrineweb_contratos` e normalizar dados do contrato a partir do pedido oficial.
- Recuperar a definicao atual de `save_vitrineweb_onboarding` antes de alterar a RPC.
- Revisar Storage buckets do Supabase para garantir que arquivos privados nao estejam publicos e que uploads sejam restritos ao dono do pedido ou admin.
- Confirmar que nao existe service role key exposta em bundle, HTML, logs ou variaveis publicas.

Prioridade media:

- Separar a landing publica do VitrineWeb do portal administrativo em projetos/subdominios distintos.
- Adicionar headers de seguranca tambem no projeto VitrineWeb.
- Aplicar rate limit/captcha em fluxos publicos de cadastro, login, recuperacao de senha e diagnostico.
- Revisar politicas de CORS para evitar origem ampla quando nao for necessario.

## Proximo passo tecnico

Auditar diretamente o projeto Supabase da VitrineWeb. O projeto esperado e `utnezxtsvxhxwthwkrlf`, mas ele nao apareceu entre os projetos Supabase conectados nesta sessao.

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

Tambem recuperar a definicao das funcoes criticas:

```sql
select
  n.nspname as schema,
  p.proname as function_name,
  pg_get_functiondef(p.oid) as definition
from pg_proc p
join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public'
  and p.proname in ('is_admin', 'save_vitrineweb_onboarding', 'claim_vitrineweb_pedido');
```
