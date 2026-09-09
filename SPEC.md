# SPEC — Gestão Plus (MVP)
> Documento único e autocontido para desenvolvimento assistido por agente de IA.
> Leia INTEIRO antes de escrever qualquer código. Em conflito, prevalece esta SPEC.

---

## 1. VISÃO GERAL

Sistema interno de requisição e controle de insumos (papel, toner, mouses/teclados)
para uma empresa pequena. Elimina perdas por: itens sem controle, toner vencido parado
em estoque e solicitações duplicadas.

**Regra de ouro:** não existe backend próprio. TODA regra de negócio e permissão
vive no Supabase (RLS + RPC + triggers). O JavaScript apenas chama o Supabase.

---

## 2. RESTRIÇÕES INVIOLÁVEIS (validar em toda decisão)

1. **Custo ≤ economia:** stack mínima, sem servidor, sem framework JS, sem build.
2. **Timebox:** estimativa total 104h. Não adicionar features; cortar se estourar.
3. **Simplicidade:** requisitar item em ≤ 3 telas / ≤ 2 min; aprovador decide em 1 clique.
4. **Segurança no banco:** nenhuma operação sensível confia no JS. RLS/RPC bloqueiam tudo,
   inclusive chamadas feitas manualmente via console do navegador ou API direta.

---

## 3. STACK (exata, não substituir)

| Camada | Tecnologia |
|---|---|
| Frontend | HTML5 + CSS3 + JavaScript vanilla (ES modules, sem build) |
| UI | Pico.css via CDN (classless, leve) |
| Backend/BD/Auth | Supabase (Postgres 15+, Auth, RLS, Edge Functions) |
| Login | Magic link por e-mail (sem senha) |
| Hospedagem | GitHub Pages (site estático) |
| Agendamento | Supabase Scheduled Functions (cron diário) |
| Repo | GitHub, deploy automático via Pages |

Variáveis públicas no frontend: `SUPABASE_URL` e `SUPABASE_ANON_KEY` (exposição é segura
PORQUE o RLS protege os dados — nunca colocar `service_role` key no frontend).

---

## 4. PAPÉIS DE USUÁRIO

| Papel | Permissões |
|---|---|
| REQUISITANTE | Ver catálogo; criar solicitações; ver próprias solicitações |
| APROVADOR | Tudo de REQUISITANTE + decidir solicitações de sua equipe |
| ALMOXARIFE | Ver fila de aprovadas pendentes; registrar baixa; ajustar saldo; ver alertas |
| ADMIN | Tudo acima + CRUD de itens, baseline, relatório de economia, gerenciar papéis |

Um usuário tem exatamente um papel (coluna `papel` em `perfis`).
O primeiro usuário cadastrado vira ADMIN manualmente via SQL.

---

## 5. MODELO DE DADOS (SQL completo — executar no SQL Editor do Supabase)

```sql
-- Extensão obrigatória
create extension if not exists "pgcrypto";

-- Perfis e papéis
create table perfis (
  id uuid primary key references auth.users(id) on delete cascade,
  nome text not null,
  email text not null unique,
  papel text not null check (papel in ('REQUISITANTE','APROVADOR','ALMOXARIFE','ADMIN')),
  created_at timestamptz not null default now()
);

-- Catálogo
create table itens (
  id uuid primary key default gen_random_uuid(),
  nome text not null,
  unidade text not null default 'un',          -- un, resma, cx...
  saldo integer not null default 0 check (saldo >= 0),
  estoque_min integer not null default 0,
  valor_unitario numeric(10,2) not null default 0,
  validade date,                                -- null = não perecível
  ativo boolean not null default true,
  created_at timestamptz not null default now()
);

-- Solicitações
create table solicitacoes (
  id uuid primary key default gen_random_uuid(),
  item_id uuid not null references itens(id),
  solicitante_id uuid not null references perfis(id),
  aprovador_id uuid references perfis(id),      -- quem decide
  qtd integer not null check (qtd > 0),
  justificativa text not null,
  status text not null default 'PENDENTE'
    check (status in ('PENDENTE','APROVADA','RECUSADA','ENTREGUE')),
  motivo_recusa text,                           -- obrigatório se RECUSADA
  duplicidade_alerta boolean not null default false, -- sinalizado ao aprovador
  created_at timestamptz not null default now(),
  decidida_em timestamptz,
  entregue_em timestamptz
);

-- Baixas (entregas)
create table baixas (
  id uuid primary key default gen_random_uuid(),
  item_id uuid not null references itens(id),
  solicitacao_id uuid references solicitacoes(id),  -- null = ajuste
  qtd integer not null check (qtd > 0),
  responsavel_id uuid not null references perfis(id), -- quem deu baixa (almoxarife)
  retirante text not null,                       -- quem retirou o material
  data_hora timestamptz not null default now()
);

-- Ajustes manuais de saldo
create table ajustes (
  id uuid primary key default gen_random_uuid(),
  item_id uuid not null references itens(id),
  delta integer not null,                        -- positivo ou negativo
  motivo text not null,
  responsavel_id uuid not null references perfis(id),
  data_hora timestamptz not null default now()
);

-- Auditoria (somente leitura, escrita por trigger)
create table auditoria (
  id bigint generated always as identity primary key,
  usuario_id uuid,
  acao text not null,                            -- INSERT/UPDATE + tabela
  tabela text not null,
  registro_id text,
  dados_json jsonb,
  data_hora timestamptz not null default now()
);

-- Baseline mensal de gastos (cadastrado pelo ADMIN antes do go-live)
create table baseline (
  id uuid primary key default gen_random_uuid(),
  mes date not null,                             -- primeiro dia do mês
  categoria text not null,                       -- papel, toner, perifericos...
  valor numeric(12,2) not null,
  unique (mes, categoria)
);
```

### 5.1 Triggers de auditoria

```sql
create or replace function audit_trigger() returns trigger as $$
begin
  insert into auditoria (usuario_id, acao, tabela, registro_id, dados_json)
  values (auth.uid(), TG_OP, TG_TABLE_NAME,
          coalesce((to_jsonb(new) ->> 'id'), (to_jsonb(old) ->> 'id')),
          to_jsonb(new));
  return coalesce(new, old);
end; $$ language plpgsql security definer;

create trigger trg_solicitacoes after insert or update on solicitacoes
  for each row execute function audit_trigger();
create trigger trg_baixas after insert on baixas
  for each row execute function audit_trigger();
create trigger trg_ajustes after insert on ajustes
  for each row execute function audit_trigger();
create trigger trg_itens after update on itens
  for each row execute function audit_trigger();
```

### 5.2 RPCs (funções security definer — única porta de escrita sensível)

```sql
-- Helper: papel do usuário atual
create or replace function meu_papel() returns text as $$
  select papel from perfis where id = auth.uid();
$$ language sql stable security definer;

-- Decisão do aprovador (transacional)
create or replace function decidir_solicitacao(
  p_solicitacao_id uuid, p_aprovar boolean, p_motivo text default null
) returns void as $$
declare v_solic record;
begin
  if meu_papel() not in ('APROVADOR','ADMIN') then
    raise exception 'Apenas aprovadores podem decidir'; end if;

  select * into v_solic from solicitacoes
   where id = p_solicitacao_id and status = 'PENDENTE' for update;
  if not found then raise exception 'Solicitação inválida ou já decidida'; end if;

  if not p_aprovar and (p_motivo is null or btrim(p_motivo) = '') then
    raise exception 'Motivo obrigatório na recusa'; end if;

  update solicitacoes set
    status = case when p_aprovar then 'APROVADA' else 'RECUSADA' end,
    motivo_recusa = case when p_aprovar then null else p_motivo end,
    aprovador_id = auth.uid(),
    decidida_em = now()
  where id = p_solicitacao_id;
end; $$ language plpgsql security definer;

-- Baixa na entrega: valida, decrementa saldo, grava baixa em uma transação
create or replace function registrar_baixa(
  p_solicitacao_id uuid, p_retirante text
) returns void as $$
declare v_solic record;
begin
  if meu_papel() not in ('ALMOXARIFE','ADMIN') then
    raise exception 'Apenas almoxarife pode dar baixa'; end if;
  if btrim(p_retirante) = '' then raise exception 'Retirante obrigatório'; end if;

  select * into v_solic from solicitacoes
   where id = p_solicitacao_id and status = 'APROVADA' for update;
  if not found then raise exception 'Solicitação não está aprovada/pendente de entrega'; end if;

  update itens set saldo = saldo - v_solic.qtd where id = v_solic.item_id;

  insert into baixas (item_id, solicitacao_id, qtd, responsavel_id, retirante)
  values (v_solic.item_id, v_solic.id, v_solic.qtd, auth.uid(), p_retirante);

  update solicitacoes set status = 'ENTREGUE', entregue_em = now()
   where id = v_solic.id;
end; $$ language plpgsql security definer;

-- Ajuste manual de saldo (avaria, doação, inventário)
create or replace function ajustar_saldo(
  p_item_id uuid, p_delta integer, p_motivo text
) returns void as $$
begin
  if meu_papel() not in ('ALMOXARIFE','ADMIN') then
    raise exception 'Apenas almoxarife pode ajustar saldo'; end if;
  if btrim(p_motivo) = '' then raise exception 'Motivo obrigatório'; end if;

  update itens set saldo = saldo + p_delta
   where id = p_item_id and saldo + p_delta >= 0;
  if not found then raise exception 'Saldo não pode ficar negativo'; end if;

  insert into ajustes (item_id, delta, motivo, responsavel_id)
  values (p_item_id, p_delta, p_motivo, auth.uid());
end; $$ language plpgsql security definer;
```

### 5.3 Views auxiliares

```sql
-- Alerta de validade (60 e 30 dias) com saldo > 0
create or replace view alertas_validade as
select id, nome, saldo, validade,
       case when validade <= current_date + interval '30 days' then 'CRITICO'
            else 'ATENCAO' end as nivel
  from itens
 where validade is not null
   and saldo > 0
   and validade <= current_date + interval '60 days';

-- Itens abaixo do estoque mínimo
create or replace view alertas_estoque_min as
select id, nome, saldo, estoque_min
  from itens where ativo and saldo < estoque_min;

-- Economia do mês corrente vs baseline
create or replace view economia_mes as
select to_char(date_trunc('month', now()), 'YYYY-MM') as mes,
       coalesce((select sum(b.valor) from baseline b
                  where date_trunc('month', b.mes) = date_trunc('month', now())), 0) as baseline_mes,
       coalesce((select sum(bx.qtd * i.valor_unitario)
                   from baixas bx join itens i on i.id = bx.item_id
                  where date_trunc('month', bx.data_hora) = date_trunc('month', now())), 0) as consumo_mes,
       coalesce((select count(*) from solicitacoes
                  where date_trunc('month', created_at) = date_trunc('month', now())), 0) as total_solicitacoes,
       coalesce((select count(*) from solicitacoes
                  where duplicidade_alerta
                    and date_trunc('month', created_at) = date_trunc('month', now())), 0) as alertas_duplicidade;
```

### 5.4 RLS (ativar em TODAS as tabelas)

```sql
alter table perfis enable row level security;
alter table itens enable row level security;
alter table solicitacoes enable row level security;
alter table baixas enable row level security;
alter table ajustes enable row level security;
alter table auditoria enable row level security;
alter table baseline enable row level security;

-- perfis: todos autenticados leem (necessário para nomes); escrita só ADMIN
create policy p_perfis_select on perfis for select to authenticated using (true);
create policy p_perfis_admin on perfis for all to authenticated
  using (meu_papel() = 'ADMIN') with check (meu_papel() = 'ADMIN');

-- itens: leitura autenticados; escrita ADMIN
create policy p_itens_select on itens for select to authenticated using (true);
create policy p_itens_admin on itens for all to authenticated
  using (meu_papel() = 'ADMIN') with check (meu_papel() = 'ADMIN');

-- solicitacoes: insert por qualquer autenticado (RPC valida regras de negócio
-- na decisão); leitura: próprias OU quem decide (APROVADOR/ALMOXARIFE/ADMIN) lê pendentes.
-- Update NUNCA direto — só via decidir_solicitacao/registrar_baixa (security definer).
create policy p_sol_select_propria on solicitacoes for select to authenticated
  using (solicitante_id = auth.uid() or meu_papel() in ('APROVADOR','ALMOXARIFE','ADMIN'));
create policy p_sol_insert on solicitacoes for insert to authenticated
  with check (solicitante_id = auth.uid() and status = 'PENDENTE');

-- baixas/ajustes/auditoria: só leitura (escrita ocorre dentro das RPCs/transações)
create policy p_baixas_read on baixas for select to authenticated using (true);
create policy p_ajustes_read on ajustes for select to authenticated using (true);
create policy p_audit_read on auditoria for select to authenticated
  using (meu_papel() in ('ADMIN','ALMOXARIFE'));

-- baseline: só ADMIN
create policy p_baseline_admin on baseline for all to authenticated
  using (meu_papel() = 'ADMIN') with check (meu_papel() = 'ADMIN');
```

**IMPORTANTE:** conceder uso das RPCs e views:
```sql
grant execute on function decidir_solicitacao(uuid, boolean, text) to authenticated;
grant execute on function registrar_baixa(uuid, text) to authenticated;
grant execute on function ajustar_saldo(uuid, integer, text) to authenticated;
grant select on alertas_validade, alertas_estoque_min, economia_mes to authenticated;
```

---

## 6. DUPLICIDADE (REQ-104)

Na criação da solicitação (frontend, antes do insert), verificar:
existe outra solicitação do mesmo `solicitante_id` + `item_id` com status 'APROVADA'
nos últimos 30 dias? Se sim, setar `duplicidade_alerta = true` no insert.
A tela do aprovador destaca essas solicitações com selo "POSSÍVEL DUPLICIDADE".
(Detecção é informativa — não bloqueia.)

---

## 7. E-MAILS (Edge Functions)

1. **`notify-aprovador`** — disparada por trigger insert em `solicitacoes`:
   envia ao aprovador (definir destinatário: por ora, coluna `aprovador_id` = gestor
   padrão cadastrado em `perfis`; se vazio, enviar para todos os APROVADORes)
   link `https://<user>.github.io/gestao-plus/aprovacao.html?solicitacao=<id>`.
2. **`notify-solicitante`** — após decisão (chamada no final de `decidir_solicitacao`
   via `net.http_post` para a função): resultado + motivo se recusada.
3. **`daily-alerts`** (Scheduled, cron diário 07:00 America/Sao_Paulo) — consulta
   `alertas_validade` e `alertas_estoque_min`; se houver itens, e-mail resumo ao ALMOXARIFE.

Enviar com `resend` ou SMTP configurado no Supabase. Assuntos prefixados "[Gestão Plus]".

---

## 8. PÁGINAS (5 arquivos HTML, CSS/JS inline ou em `/assets`)

### 8.1 `index.html` — Catálogo + Requisição (REQUISITANTE)
- Grid de cards de itens ativos: nome, unidade, saldo destacado, valor unitário.
- Saldo = 0 → card cinza, badge "INDISPONÍVEL", sem botão.
- Botão "Solicitar" → modal com quantidade (máx = saldo) + justificativa obrigatória.
- Aba "Minhas solicitações": lista próprias com status (cores: PENDENTE âmbar,
  APROVADA verde, RECUSADA vermelho + motivo, ENTREGUE azul).
- Header: nome do usuário, papel, botão sair.

### 8.2 `aprovacao.html` — Decisão em 1 clique (APROVADOR)
- Aberta via link do e-mail com `?solicitacao=<id>`.
- Se não autenticado → magic link (após login, retorna à página).
- Mostra: item, qtd, solicitante, justificativa, data, e selo vermelho
  "POSSÍVEL DUPLICIDADE — já aprovado nos últimos 30 dias" quando `duplicidade_alerta`.
- Dois botões grandes: ✅ Aprovar / ❌ Recusar (recusa abre campo de motivo obrigatório).
- Após decisão: tela de confirmação "Decisão registrada". Nada mais.

### 8.3 `almoxarifado.html` — Entregas e alertas (ALMOXARIFE)
- Seção 1 "Aguardando entrega": solicitações APROVADAS ordenadas por data.
  Cada card: item, qtd, solicitante, botão "Confirmar entrega" → campo retirante
  (obrigatório) → chama `registrar_baixa`.
- Seção 2 "Alertas": cards de `alertas_validade` (CRITICO vermelho / ATENCAO âmbar,
  com dias restantes) e `alertas_estoque_min` (azul, "repor").
- Seção 3 "Ajuste de saldo": selecionar item, delta (+/−), motivo obrigatório
  → chama `ajustar_saldo`.

### 8.4 `admin.html` — Itens, baseline e economia (ADMIN)
- CRUD de itens (modal form com todos os campos; desativar em vez de excluir).
- Tabela de baseline: mês, categoria, valor; form de inclusão.
- Painel `economia_mes`: cards Baseline do mês / Consumo do mês / Economia
  (baseline − consumo) / Total de solicitações / Alertas de duplicidade.
- Botão "Exportar CSV" (economia + baixas do mês).
- Gerenciar papéis: trocar `papel` de qualquer usuário.

### 8.5 `auth.html` — Login
- Campo de e-mail → `supabase.auth.signInWithOtp({ email, options: { emailRedirectTo: <origem> } })`.
- Mensagem: "Verifique seu e-mail para o link de acesso."
- Callback do magic link cria/atualiza perfil: `nome` derivado do e-mail
  (parte antes do @), `papel` padrão REQUISITANTE (ADMIN ajusta depois).

---

## 9. REGRAS TRANSVERSAIS

- **Idioma:** todo o UI em pt-BR.
- **Datas:** exibidas em DD/MM/AAAA (timezone America/Sao_Paulo).
- **Erros de RPC:** exibir a mensagem do Postgres ao usuário em toast vermelho.
- **Estados vazios:** toda lista sem registros mostra mensagem amigável + ação sugerida.
- **Loading:** spinner em todo fetch; botões desabilitados durante submit.
- **Acessibilidade mínima:** labels em todos os campos, contraste adequado.
- **Commits:** `T-XXX: descrição` (ex.: `T-101: catalogo com saldo e indisponiveis`).

---

## 10. ORDEM DE IMPLEMENTAÇÃO (sequencial, validar antes de avançar)

| # | Tarefa | REQ | Est. |
|---|--------|-----|------|
| 1 | Projeto Supabase + SQL seção 5 completo (tabelas, triggers, RPCs, views, RLS, grants) | — | 6h |
| 2 | Auth magic link + perfis/papéis + `auth.html` | RNF-2 | 4h |
| 3 | Repo GitHub + GitHub Pages + `supabase.js` + layout base | — | 4h |
| 4 | `index.html`: catálogo com saldo, indisponíveis, modal de solicitação | REQ-101–103 | 6h |
| 5 | Detecção de duplicidade no insert | REQ-104 | 3h |
| 6 | `aprovacao.html` + RPC `decidir_solicitacao` | REQ-106 | 6h |
| 7 | Edge Function `notify-aprovador` (trigger insert) | REQ-105 | 4h |
| 8 | Edge Function `notify-solicitante` (pós-decisão) | REQ-107 | 3h |
| 9 | `almoxarifado.html` + RPC `registrar_baixa` + fila de entregas | REQ-201 | 7h |
| 10 | RPC `ajustar_saldo` + seção de ajuste | REQ-205 | 4h |
| 11 | Triggers de auditoria (SQL) + verificação | REQ-204 | 3h |
| 12 | Views de alertas + seção de alertas no almoxarifado | REQ-202, REQ-203 | 5h |
| 13 | Scheduled Function `daily-alerts` | REQ-202 | 2h |
| 14 | `admin.html`: CRUD de itens | REQ-301 | 6h |
| 15 | Cadastro de baseline | REQ-302 | 3h |
| 16 | View `economia_mes` + painel | REQ-303 | 6h |
| 17 | Export CSV | REQ-304 | 3h |
| 18 | **Teste de segurança:** tentar violar RLS via console/API direta (deve falhar em todos os casos) | RNF-6 | 4h |
| 19 | **Teste UX:** 5 usuários reais cronometrados, requisição ≤ 2 min | RNF-1 | 3h |
| 20 | Carga inicial (itens, saldos, baseline) + comunicado | — | 2h |

**Total: ~104h**

---

## 11. DEFINITION OF DONE

- [ ] Todas as tarefas 1–20 concluídas
- [ ] Teste de segurança (18): zero ações proibidas bem-sucedidas
- [ ] Teste UX (19): mediana ≤ 2 minutos, zero travamentos
- [ ] Baseline cadastrado e painel de economia demonstrável
- [ ] Auditoria registrando inserções/atualizações corretamente
- [ ] Deploy no GitHub Pages acessível e funcional

---

## 12. FORA DE ESCOPO (não implementar, registrar como ideia futura)

Mobile/PWA, QR, integração ERP, reposição automática, múltiplos almoxarifados,
dashboard para diretoria (Admin exporta e envia), notificações push.
