# Miau Life — Plano da Etapa 1 (Fase A: Planejamento)

> **Status:** aguardando aprovação. Nenhum código foi escrito.
> **Data:** 06/11/2026
> **Base:** especificação "MIAU LIFE — ETAPA 1" + decisões já tomadas (listadas na seção 0).

Este documento explica **como** o Miau Life será construído antes de qualquer código existir. Cada seção começa com uma explicação em linguagem simples (🐾) e depois traz o detalhe técnico.

---

## Sumário

0. Decisões já tomadas (base do plano)
1. Arquitetura geral
2. Estrutura de pastas
3. Modelo de dados (tabelas, campos e relacionamentos)
4. Autenticação
5. Segurança: políticas RLS
6. Exclusão segura, arquivamento e dados demo
7. Datas, horários e fusos
8. Integração das futuras áreas e módulos
9. Preparação para a Miau IA
10. Navegação, telas e identidade visual
11. Hospedagem, PWA e as duas versões (computador e iPhone)
12. Escalabilidade: possíveis problemas
13. Riscos técnicos e decisões arquiteturais
14. Como vamos testar
15. O que você vai precisar fazer manualmente
16. Pontos que preciso que você aprove

---

## 0. Decisões já tomadas (base do plano)

| # | Decisão | Como aparece neste plano |
|---|---|---|
| D1 | Horários guardados como **um único instante** (`timestamptz`), não data e hora separadas. Fuso por usuário, padrão `America/Belem`. Cada atividade guarda também o seu fuso (`event_timezone`). | Seções 3 e 7 |
| D2 | Bibliotecas extras permitidas: React Router, TanStack Query, React Hook Form, Zod, date-fns, date-fns-tz, vite-plugin-pwa, lucide-react. | Seção 1 |
| D3 | Hospedagem na **Vercel**, ligada ao GitHub. | Seção 11 |
| D4 | Dados demo marcados com `is_demo = true` e botão em Configurações para removê-los. | Seção 6 |
| D5 | Exclusão segura: arquivar é o padrão; `RESTRICT` em áreas e categorias; contagem do que será afetado antes de excluir. | Seção 6 (veja o ajuste técnico proposto em **16.1**) |
| D6 | Tabela `history_log` preenchida por triggers, somente inserção. | Seção 3.9 |
| D7 | Tabela `ai_requests` para a futura Miau IA, que só **propõe** mudanças. | Seção 9 |
| D8 | RLS em todas as tabelas, com validação de que `area_id` e `category_id` pertencem ao mesmo usuário. | Seções 3.2 e 5 |

**Nenhuma tecnologia da especificação está sendo substituída.** Onde eu sugiro algo além do que foi pedido (por exemplo, uma ferramenta de testes), isso está marcado na seção 16 para você decidir.

---

## 1. Arquitetura geral

🐾 **Em palavras simples:** o Miau Life terá duas partes.
- **A parte que você vê** (telas, botões, cores) roda no navegador do celular ou do computador. Ela fica hospedada na Vercel.
- **A parte que guarda e protege os dados** é o Supabase: ele tem o banco de dados, o sistema de login e "funções de servidor" (pequenos programas que rodam fora do celular, onde ficam as chaves secretas).

O celular nunca conversa direto com o Gemini e nunca vê chaves secretas. Ele só conversa com o Supabase, e o Supabase confere, a cada pedido, se aquele dado pertence a quem está pedindo.

```
┌──────────────────────────────────────────────────────────────┐
│  NAVEGADOR (celular, tablet, computador) — PWA               │
│  React + TypeScript + Vite + Tailwind                        │
│  • React Router (páginas)    • TanStack Query (busca/cache)  │
│  • React Hook Form + Zod     • date-fns / date-fns-tz        │
│  • lucide-react (ícones)     • vite-plugin-pwa (instalação)  │
│  Usa apenas a chave PÚBLICA do Supabase (anon/publishable)   │
└───────────────┬──────────────────────────────────────────────┘
                │ HTTPS (token de login do usuário)
┌───────────────▼──────────────────────────────────────────────┐
│  SUPABASE                                                     │
│  • Auth: cadastro, login, recuperação de senha               │
│  • PostgreSQL: tabelas + RLS (cada um só vê o que é seu)     │
│  • Funções SQL (RPC): resumo do dashboard, prévia de         │
│    exclusão, dados demo, reordenar áreas                     │
│  • Storage: fotos de perfil (pasta por usuário)              │
│  • Edge Functions (futuro): Miau IA, Google Calendar,        │
│    exclusão de conta. Guardam as chaves secretas.            │
└───────────────┬──────────────────────────────────────────────┘
                │ (somente a partir do servidor, etapas futuras)
        ┌───────▼────────┐      ┌──────────────────────┐
        │  API Gemini    │      │ Google Calendar API  │
        └────────────────┘      └──────────────────────┘

Hospedagem do site: Vercel (cada envio ao GitHub gera uma nova versão)
```

### 1.1 Princípios

1. **O banco é a fonte da verdade.** Nada importante fica guardado só no navegador. O `localStorage` não será usado para dados. A única coisa que o Supabase guarda lá é o "crachá" de login (token de sessão), que é o comportamento padrão e seguro dele.
2. **A segurança mora no banco, não na tela.** Esconder um botão não protege nada. Quem protege é o RLS (seção 5). Mesmo que alguém mexa no código do navegador, o banco recusa dados de outra pessoa.
3. **Regras importantes ficam no banco.** Validações críticas (dono do registro, área e categoria do mesmo usuário, datas coerentes) são feitas por restrições do PostgreSQL. A tela valida também (com Zod), mas só para dar mensagens amigáveis.
4. **Organização por funcionalidade.** Cada parte do app (áreas, atividades, configurações...) tem sua própria pasta. Isso permite criar módulos novos sem mexer nos antigos.
5. **Código-fonte do banco versionado.** Todos os scripts SQL ficam numerados no GitHub. Assim sempre sabemos o estado exato do banco.

### 1.2 Papel de cada biblioteca

| Biblioteca | Para que serve, em linguagem simples |
|---|---|
| React + TypeScript | Monta as telas. O TypeScript avisa erros antes de o app rodar. |
| Vite | Ferramenta que "empacota" o app para publicar. |
| Tailwind CSS (v4) | Sistema de estilos (cores, espaçamentos, cantos arredondados). |
| React Router | Controla as páginas e os endereços (ex.: `/calendario`). |
| TanStack Query | Busca dados no Supabase, guarda em memória enquanto o app está aberto e atualiza a tela depois de salvar. |
| React Hook Form + Zod | Formulários com validação ("o título é obrigatório", "a hora final deve ser depois da inicial"). |
| date-fns + date-fns-tz | Formatar datas em DD/MM/AAAA, 24h e converter fusos. |
| vite-plugin-pwa | Permite instalar o app na tela inicial. |
| lucide-react | Ícones delicados e consistentes. |
| @supabase/supabase-js | Biblioteca oficial para conversar com o Supabase (implícita no uso do Supabase). |

---

## 2. Estrutura de pastas

🐾 Pense nas pastas como gavetas etiquetadas. Cada gaveta guarda uma coisa só. A estrutura abaixo é o **planejamento**: nada disso foi criado ainda.

```
THAISAPP/
├─ ESPECIFICACAO.md            (o texto da especificação, se você quiser guardá-lo aqui)
├─ docs/
│  ├─ PLANO.md                 ← este documento
│  ├─ GUIA_SUPABASE.md         passo a passo do Supabase (Fase B)
│  ├─ GUIA_VERCEL.md           passo a passo da publicação (Fase B)
│  └─ TESTES_MANUAIS.md        roteiro de testes para você e a Thaís (Fase B)
│
├─ supabase/
│  ├─ migrations/              scripts SQL numerados, executados em ordem
│  │  ├─ 0001_base.sql                 funções auxiliares comuns
│  │  ├─ 0002_profiles.sql             perfis + criação automática no cadastro
│  │  ├─ 0003_areas_categories.sql     áreas, categorias, áreas iniciais
│  │  ├─ 0004_items_goals.sql          itens e metas
│  │  ├─ 0005_activities.sql           atividades e lembretes
│  │  ├─ 0006_history.sql              histórico e seus gatilhos
│  │  ├─ 0007_ai.sql                   estrutura da Miau IA
│  │  ├─ 0008_functions.sql            resumo do dashboard, prévia de exclusão, demo...
│  │  └─ 0009_storage.sql              pasta de fotos de perfil
│  ├─ tests/
│  │  └─ rls_isolamento.sql            teste: usuário A não enxerga dados do B
│  └─ functions/                       Edge Functions (só a estrutura; sem IA real)
│     └─ miau-ia/                      esqueleto que responde "ainda não disponível"
│
├─ public/                     ícones do app, imagem para iPhone, fontes
│
├─ src/
│  ├─ main.tsx                 ponto de partida
│  ├─ app/                     montagem geral: rotas, provedores, proteção de rotas
│  ├─ lib/                     ferramentas compartilhadas
│  │  ├─ supabase.ts           conexão com o Supabase
│  │  ├─ queryClient.ts        configuração do TanStack Query
│  │  ├─ dates.ts              ÚNICO lugar que converte datas e fusos
│  │  └─ errors.ts             traduz erros técnicos para mensagens em português
│  ├─ types/                   tipos gerados a partir do banco
│  ├─ styles/                  cores, fontes, tokens visuais
│  ├─ components/
│  │  ├─ ui/                   Botão, Cartão, Campo, Modal, Aviso (toast),
│  │  │                        Barra de progresso, Seletor de cor/ícone, Vazio, Carregando
│  │  └─ layout/               Menu lateral, Barra inferior, Folha "Mais", Cabeçalho
│  ├─ features/                uma pasta por funcionalidade
│  │  ├─ auth/                 entrar, cadastrar, recuperar e redefinir senha
│  │  ├─ dashboard/            página Início
│  │  ├─ areas/                lista, criar, editar, reordenar, arquivar, excluir
│  │  ├─ categories/
│  │  ├─ items/
│  │  ├─ activities/           compromissos e tarefas
│  │  ├─ goals/                metas semanais
│  │  ├─ calendar/             página Calendário (versão inicial: agenda em lista)
│  │  ├─ miau-ia/              página Miau IA (estrutura, sem IA ativa)
│  │  ├─ settings/             perfil, fuso, preferências, dados demo, áreas
│  │  └─ history/              leitura do histórico
│  └─ modules/                 futuros módulos especializados (vazio na Etapa 1)
│                              ex.: modules/pets, modules/fazenda, modules/financeiro
│
├─ .env.example                modelo das variáveis, SEM valores reais
├─ .gitignore                  inclui .env (nunca vai para o GitHub)
├─ vercel.json                 regra para as páginas funcionarem ao recarregar
├─ index.html, package.json, vite.config.ts, tsconfig.json
```

**Regra de dependência entre pastas:** `features/` e `modules/` podem usar `components/` e `lib/`. Um módulo não importa arquivos internos de outro módulo. Isso evita que mexer no Financeiro quebre os Pets.

---

## 3. Modelo de dados

🐾 **Em palavras simples:** o banco é um conjunto de planilhas muito bem organizadas (as "tabelas"). Cada linha tem um "dono" (`user_id`). As tabelas se ligam: uma **categoria** pertence a uma **área**; uma **atividade** pode pertencer a uma área e a uma categoria. O banco não deixa ligar sua atividade a uma área de outra pessoa.

### 3.1 Visão geral das tabelas

```
auth.users (do Supabase, não mexemos)
   │ 1:1
profiles ───────────────────────────────────────────────┐
   │ 1:N                                                 │ (todas as tabelas
areas ──1:N── categories                                 │  abaixo têm user_id
   │              │                                      │  apontando para
   ├──────────────┼──── items                            │  profiles)
   ├──────────────┼──── goals                            │
   └──────────────┴──── activities ──1:N── activity_reminders
                                                         │
history_log   (registro de alterações; sem ligação rígida com os registros)
ai_requests ──1:N── ai_proposed_changes
```

| Tabela | O que guarda | Arquivável? |
|---|---|---|
| `profiles` | Nome, foto, idioma, fuso e preferências de cada pessoa | Não (é a própria conta) |
| `areas` | As áreas editáveis (Estudos, Pets, Fazenda...) | Sim |
| `categories` | Subdivisões de cada área | Sim |
| `items` | Registros genéricos organizáveis (notas, listas, registros) | Sim |
| `goals` | Metas (semanais, na Etapa 1) | Sim |
| `activities` | Compromissos e tarefas (os 4 tipos) | Sim |
| `activity_reminders` | Lembretes de cada atividade | Vai junto com a atividade |
| `history_log` | Histórico de alterações, só inserção | Não (nunca é editado) |
| `ai_requests` | Pedidos feitos à Miau IA | Sim |
| `ai_proposed_changes` | Mudanças que a IA propõe e aguardam aprovação | Não (tem status) |

**Por que `goals` é uma tabela própria e não um "tipo de item"?** A especificação pede metas semanais com progresso no dashboard. Metas têm regras próprias (período, valor-alvo, progresso). A especificação também pede para não colocar em uma tabela genérica o que vai precisar de regras próprias. Por isso `goals` fica separada.

### 3.2 Regras comuns a todas as tabelas do usuário

| Regra | Como será feita |
|---|---|
| Identificador | `id` do tipo `uuid` gerado pelo banco. É um código aleatório que não revela quantos registros existem. |
| Dono | `user_id uuid NOT NULL`, preenchido automaticamente com o usuário logado, ligado a `profiles(id)`. |
| Datas de controle | `created_at`, `updated_at` (`timestamptz`, preenchidos pelo banco), `archived_at` (vazio = ativo). |
| `updated_at` automático | Um gatilho (trigger) atualiza a data a cada alteração. |
| Dono não pode mudar | Um gatilho impede trocar o `user_id` de um registro existente. |
| `is_demo` | `boolean NOT NULL DEFAULT false` nas tabelas de conteúdo. |
| Área e categoria do mesmo dono | **Chave estrangeira composta**: a atividade aponta para `(area_id, user_id)` da área. Se a área for de outra pessoa, o banco recusa. O mesmo vale para a categoria, que além disso precisa ser da **mesma área** (`category_id, area_id, user_id`). Isso é mais confiável que um gatilho, porque o próprio PostgreSQL garante. |
| Categoria exige área | Se `category_id` estiver preenchido, `area_id` também precisa estar. |
| Listas de opções | Campos como status e tipo usam `text` + restrição `CHECK` (lista de valores aceitos), em vez de `enum`. Motivo: é mais fácil acrescentar valores novos no futuro. |
| Tamanhos | Nomes de 1 a 80 caracteres; descrições até 5.000; JSONB até ~32 KB, para evitar abuso. |
| "Ativo ou arquivado" | Usamos **apenas** `archived_at`: vazio = ativo; preenchido = arquivado (com a data). Não haverá um segundo campo "status ativo", porque dois campos dizendo a mesma coisa podem ficar contraditórios. |

### 3.3 `profiles`

Uma linha por pessoa, criada **automaticamente** no momento do cadastro (gatilho em `auth.users`).

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid, PK | Igual ao `auth.users.id`. Apaga junto se a conta for excluída. |
| `display_name` | text | Nome de exibição (ex.: "Thaís"). |
| `avatar_path` | text, opcional | Caminho da foto no Storage (pasta do próprio usuário). |
| `locale` | text, padrão `pt-BR` | Idioma. |
| `timezone` | text, padrão `America/Belem` | Validado contra a lista oficial de fusos do PostgreSQL. |
| `week_starts_on` | smallint, padrão 0 | 0 = domingo, 1 = segunda. Usado nas metas semanais e no calendário. |
| `preferences` | jsonb, padrão `{}` | Preferências de interface (tema, cartões visíveis no Início etc.). |
| `onboarding_done_at` | timestamptz, opcional | Para saber se já passou pela tela de boas-vindas. |
| `created_at`, `updated_at` | timestamptz | |

- **Não guarda e-mail nem senha.** O e-mail é lido do próprio Supabase Auth quando necessário.
- **Sem `archived_at`:** a exceção é intencional, porque "arquivar o perfil" não faz sentido. Excluir a conta será uma função de servidor futura (seção 13).

### 3.4 `areas`

| Campo | Tipo | Observação |
|---|---|---|
| `id` | uuid, PK | |
| `user_id` | uuid | Dono. |
| `name` | text | 1 a 80 caracteres. |
| `description` | text, opcional | |
| `color` | text | Nome de uma cor da paleta (ex.: `blush`, `salvia`). Guardar o **nome** em vez do código hexadecimal permite ajustar tons e criar um modo escuro sem reescrever dados. |
| `icon` | text | Nome de um ícone lucide de uma lista selecionada (ex.: `paw-print`). |
| `sort_order` | integer | Ordem de exibição. |
| `module_key` | text, opcional | Liga a área a um módulo especializado futuro (`pets`, `fazenda`, `financeiro`...). Fica único por usuário. Explicado na seção 8. |
| `is_demo` | boolean | |
| `created_at`, `updated_at`, `archived_at` | timestamptz | |

Restrições: único `(id, user_id)` (necessário para a chave composta) e índice em `(user_id, sort_order)` para áreas ativas.

**Áreas iniciais:** no cadastro, o banco cria as 10 áreas para o usuário, como **registros reais** (`is_demo = false`), porque são o ponto de partida dela e não exemplos.

| Ordem | Nome | `module_key` | Ícone sugerido | Cor sugerida |
|---|---|---|---|---|
| 1 | Estudos & Aprendizado | `estudos` | graduation-cap | lilás |
| 2 | Trabalho & Negócios | `trabalho` | briefcase | pêssego |
| 3 | Casa & Organização | `casa` | house | creme |
| 4 | Pessoal & Autocuidado | `autocuidado` | flower | rosa blush |
| 5 | Vida Fit & Saúde | `saude` | heart-pulse | verde sálvia |
| 6 | Meus Pets | `pets` | paw-print | rosa pastel |
| 7 | Fazenda & Rural | `fazenda` | tractor | verde sálvia |
| 8 | Financeiro | `financeiro` | wallet | pêssego |
| 9 | Viagens & Experiências | `viagens` | plane | lilás |
| 10 | Projetos & Sonhos | `projetos` | sparkles | rosa blush |

Ela pode renomear, mudar cor e ícone, reordenar, arquivar e excluir (exclusão segura, seção 6). Ela também pode criar áreas novas, sem `module_key`.

**Páginas fixas** (Início, Calendário, Miau IA, Configurações) **não** ficam no banco. São rotas do app e não aparecem como áreas.

### 3.5 `categories`

| Campo | Tipo | Observação |
|---|---|---|
| `id`, `user_id` | uuid | |
| `area_id` | uuid, **obrigatório** | Chave composta `(area_id, user_id)` → `areas`. |
| `name`, `description`, `color`, `icon`, `sort_order` | | Mesmas regras das áreas. Cor e ícone opcionais (herdam da área). |
| `is_demo`, `created_at`, `updated_at`, `archived_at` | | |

Restrições: únicos `(id, user_id)` e `(id, area_id, user_id)`, usados pelas tabelas que apontam para categorias.

### 3.6 `items` (registros genéricos)

Para coisas que **não** têm hora marcada: notas, listas, ideias, registros simples.

| Campo | Tipo | Observação |
|---|---|---|
| `id`, `user_id` | uuid | |
| `area_id`, `category_id` | uuid, opcionais | Validados por chave composta (mesmo dono e categoria da mesma área). |
| `title` | text | Obrigatório. |
| `description` | text | |
| `item_type` | text | `nota`, `lista`, `ideia`, `registro`, `link` (lista ampliável). |
| `status` | text | `pendente`, `em_andamento`, `concluido`, `cancelado`. |
| `priority` | smallint 1–4 | 1 baixa, 2 média, 3 alta, 4 urgente. Número permite ordenar. |
| `starts_at` | timestamptz, opcional | Início. |
| `completed_at` | timestamptz, opcional | Conclusão. Preenchido automaticamente quando o status vira "concluído". |
| `data` | jsonb, padrão `{}` | Dados complementares **leves** (ex.: itens de uma lista de checagem). Nada que precise de busca ou regra própria. |
| `is_demo`, `created_at`, `updated_at`, `archived_at` | | |

### 3.7 `goals` (metas)

| Campo | Tipo | Observação |
|---|---|---|
| `id`, `user_id`, `area_id`, `category_id` | uuid | Área e categoria opcionais. |
| `title`, `description` | text | |
| `period` | text | `semanal` na Etapa 1 (`mensal` previsto). |
| `period_start` | **date** | Primeiro dia da semana **no calendário local** do usuário. Aqui `date` é o tipo certo: "a semana de 02/11" é um conceito de calendário, não um instante (veja a seção 7). |
| `target_value` | numeric > 0 | Ex.: 5 (treinos). |
| `current_value` | numeric ≥ 0 | Progresso atual, atualizado pela usuária. |
| `unit` | text | Ex.: "treinos", "horas", "páginas". |
| `status` | text | `ativa`, `concluida`, `nao_atingida`. |
| `is_demo`, `created_at`, `updated_at`, `archived_at` | | |

### 3.8 `activities` (eventos e atividades)

O coração do calendário. Os 4 tipos ficam na **mesma tabela**, porque todos aparecem juntos no calendário e no resumo do dia. As regras de cada tipo são garantidas por `CHECK`.

| Campo | Tipo | Observação |
|---|---|---|
| `id`, `user_id` | uuid | |
| `area_id`, `category_id` | uuid, opcionais | Chaves compostas (mesmo dono). |
| `title`, `description` | text | |
| `activity_type` | text | `compromisso`, `tarefa_prazo`, `tarefa_agendada`, `tarefa_flexivel`. |
| `starts_at` | timestamptz | Início (instante único: data + hora juntos, decisão D1). |
| `ends_at` | timestamptz | Fim. Precisa ser maior ou igual ao início. |
| `all_day` | boolean | Dia inteiro (sem hora). |
| `due_at` | timestamptz | Prazo. |
| `due_has_time` | boolean | `false` = "até o fim do dia 10/11"; `true` = "até 10/11 às 14:00". |
| `event_timezone` | text | Fuso em que a atividade foi pensada (D1). Preenchido com o fuso do perfil se vier vazio. |
| `location_name` | text | Local (ex.: "Clínica Vet Amigo"). |
| `address` | text | Endereço. |
| `modality` | text, opcional | `presencial`, `online`, `hibrido`. |
| `meeting_url` | text, opcional | Link da reunião online. |
| `travel_minutes` | integer ≥ 0 | Tempo de deslocamento. |
| `estimated_minutes` | integer ≥ 0 | Duração estimada (útil para tarefas flexíveis e para a futura IA distribuir na semana). |
| `priority` | smallint 1–4 | |
| `status` | text | `pendente`, `em_andamento`, `concluida`, `cancelada`, `adiada`. |
| `completed_at` | timestamptz | Automático ao concluir. |
| `recurrence_rule` | text, opcional | Regra de repetição no padrão internacional **RRULE** (o mesmo do Google Calendar). Ex.: "toda segunda e quarta". |
| `recurrence_until` | timestamptz, opcional | Até quando repete (facilita consultas). |
| `external_source`, `external_id`, `external_etag` | text, opcionais | Reservados para sincronizar com o Google Calendar no futuro. Único por `(user_id, external_source, external_id)`. |
| `is_demo`, `created_at`, `updated_at`, `archived_at` | | |

**Regras por tipo (garantidas pelo banco):**

| Tipo | Exige | Exemplo |
|---|---|---|
| Compromisso com presença | `starts_at` e `ends_at` | Consulta do gato às 14:00–15:00 |
| Tarefa com prazo | `due_at` | Entregar trabalho até 10/11 |
| Tarefa agendada | `starts_at` (`ends_at` opcional) | Estudar inglês dia 08/11 às 19:00 |
| Tarefa flexível | nada obrigatório | Organizar o armário "esta semana" |

**`activity_reminders`** (lembretes): `id`, `user_id`, `activity_id` (apaga junto com a atividade), `minutes_before` (ex.: 30), `channel` (`app` por enquanto), `sent_at`, `created_at`, `updated_at`.
- **Por que uma tabela separada e não JSON?** Um futuro serviço de notificações precisará perguntar ao banco "quais lembretes vencem agora?". Com tabela própria e índice, essa pergunta é rápida.
- **Na Etapa 1** os lembretes são apenas **guardados**. Ainda não haverá notificação real (seção 13).

**Recorrência na Etapa 1:** o campo existe e a regra é guardada. Exceções do tipo "pular só a ocorrência de 15/11" ficam para a etapa do Calendário unificado, com uma tabela `activity_exceptions`.

### 3.9 `history_log` (histórico)

🐾 Um "diário" automático: sempre que algo importante muda, o banco anota sozinho o que mudou, quando e quem mudou. Ninguém consegue editar ou apagar essas anotações pelo app.

| Campo | Tipo | Observação |
|---|---|---|
| `id` | bigint (sequencial) | |
| `user_id` | uuid | Dono. |
| `entity_type` | text | `area`, `category`, `item`, `goal`, `activity`. |
| `entity_id` | uuid | **Sem chave estrangeira de propósito**: o histórico sobrevive mesmo se o registro for excluído. |
| `action` | text | `criado`, `atualizado`, `concluido`, `arquivado`, `restaurado`, `excluido`. |
| `changes` | jsonb | Somente os campos que mudaram: `{campo: {antes, depois}}`. Na exclusão, guarda uma cópia do registro. |
| `actor` | text | `usuario`, `sistema`, `miau_ia`. |
| `ai_request_id` | uuid, opcional | Quando a mudança veio de uma proposta aprovada da IA. |
| `created_at` | timestamptz | |

- **Somente inserção:** não há permissão de alterar ou apagar pelo app. Só os gatilhos inserem.
- **Exceções intencionais às "regras comuns":** sem `updated_at` e sem `archived_at`, porque um registro de histórico nunca muda.
- Registros demo (`is_demo = true`) **não** geram histórico, para não poluí-lo.
- Alterações que só mexem em `updated_at` ou `sort_order` não geram linha (reordenar 10 áreas não deve criar 10 anotações).

### 3.10 Índices principais (para consultas rápidas)

| Tabela | Índice | Serve para |
|---|---|---|
| todas | `user_id` (sempre como primeira coluna) | RLS e filtros por dono |
| `areas` | `(user_id, sort_order)` só das ativas | Menu e cartões |
| `categories` | `(user_id, area_id, sort_order)` | Categorias de uma área |
| `activities` | `(user_id, starts_at)` só das não arquivadas | Calendário e "próximos compromissos" |
| `activities` | `(user_id, due_at)` só das pendentes | Atrasadas e "vencem hoje" |
| `activities`, `items` | `(user_id, area_id, status)` | Contagens por área no dashboard |
| `activity_reminders` | `(sent_at, ...)` só dos não enviados | Futuro envio de notificações |
| `history_log` | `(user_id, entity_type, entity_id, created_at)` | "Histórico deste item" |
| `ai_requests` | `(user_id, created_at)` | Lista de pedidos e limite de uso |

Índices "parciais" (só ativos ou só pendentes) ficam menores e mais rápidos.

---

## 4. Autenticação

🐾 O Supabase cuida do login. Nós só construímos as telas e as regras em volta.

### 4.1 Fluxos

| Fluxo | Como funciona |
|---|---|
| **Cadastro** | Nome, e-mail, senha (mínimo 8 caracteres) e confirmação de senha, validados com Zod. O Supabase envia um e-mail de confirmação. Ao confirmar, o banco cria automaticamente o perfil e as 10 áreas iniciais. |
| **Login** | E-mail e senha. Erros traduzidos ("E-mail ou senha incorretos", "Confirme seu e-mail antes de entrar"). |
| **Logout** | Encerra a sessão e **limpa todos os dados em memória** do TanStack Query, para que, num aparelho compartilhado, a próxima pessoa não veja nada da anterior. |
| **Recuperar senha** | Tela "Esqueci minha senha" → e-mail com link → página `/redefinir-senha` → nova senha. |
| **Proteção de rotas** | Páginas internas exigem sessão. Sem login, a pessoa vai para `/entrar` e, depois de entrar, volta para onde queria ir. Lembrete: isso é **conforto de navegação**. A proteção real é o RLS. |
| **Sessão** | O Supabase renova o token automaticamente. Ela continua logada ao reabrir o app. |
| **Primeiro acesso** | Tela de boas-vindas: confirmar nome e fuso (sugerindo o fuso detectado no aparelho, com Belém como padrão) e oferecer "Carregar exemplos" (dados demo). |

### 4.2 Criação automática do perfil (gatilho)

Quando uma conta nasce em `auth.users`, uma função do banco:
1. cria a linha em `profiles` (nome vindo do formulário de cadastro, fuso `America/Belem`);
2. cria as 10 áreas iniciais.

Essa função roda com permissão elevada, mas de forma restrita: só insere dados **daquela** conta nova, com caminho de busca fixo, para evitar truques de segurança. Se ela falhar, o cadastro falha inteiro. Isso é bom (não fica conta sem perfil), mas exige teste cuidadoso (seção 14).

### 4.3 Configurações no painel do Supabase (você fará, com meu passo a passo na Fase B)

- Provedor **E-mail** ativado, com "Confirmar e-mail" ligado.
- **Site URL** = endereço da Vercel; **Redirect URLs** = endereço da Vercel + `http://localhost:5173` (para testes).
- Senha mínima de 8 caracteres.
- Modelos de e-mail (confirmação e recuperação) traduzidos para português.

---

## 5. Segurança: políticas RLS

🐾 **RLS (Row Level Security)** é um "porteiro" dentro do banco. A cada leitura ou gravação, ele confere: "esta linha pertence a quem está pedindo?". Se não pertencer, a linha simplesmente não existe para aquela pessoa.

### 5.1 Regras gerais

1. **RLS ativado em todas as tabelas** do esquema `public`, inclusive nas tabelas futuras. Cada script cria a tabela **e** suas políticas no mesmo arquivo, para que nunca exista uma tabela sem porteiro.
2. Visitantes não logados (`anon`) não têm acesso a nenhuma tabela.
3. A comparação usa `(select auth.uid())`. Esse formato é recomendado pelo Supabase porque calcula o usuário uma vez por consulta, e não uma vez por linha (muito mais rápido).
4. Funções de servidor do banco rodam como "quem chamou" sempre que possível (`SECURITY INVOKER`), para respeitar o RLS. As poucas que precisam de permissão elevada (gatilhos de perfil e histórico) têm caminho de busca fixo e fazem só uma coisa.
5. Consultas prontas (*views*) usadas pelo dashboard são criadas com `security_invoker`, para também obedecer ao RLS.
6. A chave `service_role` (que ignora o RLS) **nunca** vai para o navegador nem para o GitHub. Só existirá dentro das Edge Functions, em etapas futuras.

### 5.2 Matriz de permissões

| Tabela | Ler | Criar | Alterar | Excluir |
|---|---|---|---|---|
| `profiles` | só o próprio | ✗ (o gatilho cria) | só o próprio | ✗ (exclusão de conta via servidor, no futuro) |
| `areas`, `categories` | só as próprias | só com `user_id` = o próprio | só as próprias, sem trocar o dono | só as próprias, **e o banco bloqueia se houver vínculos** |
| `items`, `goals`, `activities` | só os próprios | idem | idem | idem (o padrão na tela é arquivar) |
| `activity_reminders` | só os próprios | idem + atividade precisa ser do mesmo dono | idem | idem |
| `history_log` | só o próprio | ✗ (só gatilhos) | ✗ | ✗ |
| `ai_requests` | só os próprios | só os próprios, sempre como "pendente" | só arquivar | ✗ |
| `ai_proposed_changes` | só os próprios | ✗ (só a Edge Function) | ✗ direto; aprovar/rejeitar por função dedicada | ✗ |
| Storage `avatars` | só a própria pasta | só na própria pasta | só na própria pasta | só na própria pasta |

### 5.3 Defesa em camadas contra "misturar donos"

Mesmo que alguém tente gravar uma atividade com o próprio `user_id` e a área de outra pessoa:
1. o RLS aceita o `user_id` (é o dela);
2. **mas a chave composta `(area_id, user_id)` não encontra nenhuma área com esse par**, e o banco recusa.

Assim fica cumprida a decisão D8 sem depender de código no navegador.

---

## 6. Exclusão segura, arquivamento e dados demo

### 6.1 Arquivar (padrão)

- "Arquivar" preenche `archived_at`. O registro some das listas, mas continua no banco e pode ser **restaurado** em Configurações → Arquivados.
- Arquivar uma área **não** arquiva automaticamente o que está dentro dela. Os registros continuam existindo, apenas ficam "escondidos" junto com a área. Na tela, a pessoa poderá escolher "arquivar também as N atividades e M categorias". Essa ação é explícita e fica registrada no histórico.

### 6.2 Excluir definitivamente

Fluxo para áreas (o mesmo vale para categorias):
1. A tela chama uma função `area_delete_preview(area_id)` que **conta** o que está ligado à área: categorias, itens, metas e atividades (ativos e arquivados, separadamente).
2. **Se houver qualquer vínculo:** a exclusão é bloqueada. A tela mostra, por exemplo: "Esta área tem 3 categorias, 12 atividades e 2 metas. Arquive a área ou mova/exclua esses registros primeiro." O botão oferecido é **Arquivar**.
3. **Se não houver vínculos:** pede confirmação ("Excluir 'Viagens & Experiências' para sempre?") e exclui. O histórico guarda uma cópia.
4. **Segunda barreira no banco:** mesmo que a tela falhe, a chave estrangeira impede excluir uma área com registros ligados. Nada é apagado em cascata.

"Excluir tudo junto" (em cascata, com confirmação digitada) **não** entra na Etapa 1. Se você quiser, pode ser avaliado depois.

Itens, metas e atividades podem ser excluídos definitivamente (com confirmação), porque não têm "filhos" importantes. Os lembretes de uma atividade vão junto com ela, porque fazem parte dela. O histórico permanece.

### 6.3 Dados demo

- Botão **"Carregar exemplos"** (boas-vindas e Configurações): chama `seed_demo_data()`, que cria categorias, itens, metas e atividades de exemplo com `is_demo = true`, nas áreas existentes e com datas **relativas a hoje** (para o dashboard ficar interessante em qualquer dia).
- Selo visual "exemplo" nos cartões demo.
- Botão **"Remover dados de exemplo"** em Configurações: chama `remove_demo_data()`, que primeiro mostra quantos registros serão removidos e depois apaga **somente** o que tem `is_demo = true`, na ordem correta (lembretes → atividades → itens/metas → categorias).
- **Caso delicado:** se ela criou um registro real dentro de uma categoria de exemplo, essa categoria não pode ser apagada sem afetar dado real. A função **não apaga** essa categoria; ela a converte em real (`is_demo = false`) e avisa: "1 categoria de exemplo foi mantida porque contém seus registros".
- As 10 áreas iniciais **não** são demo, então não são removidas.

---

## 7. Datas, horários e fusos

🐾 **O problema clássico:** um compromisso marcado para "06/11 às 23:30" em Belém pode aparecer como "07/11" se o app confundir fusos. Ou uma data salva como "10/11" aparece como "09/11 às 21:00". Para isso não acontecer:

### 7.1 Regras

1. **Guardamos instantes** (`timestamptz`). O banco guarda o momento exato no tempo (internamente em UTC). Esse momento é o mesmo em qualquer lugar do mundo.
2. **Guardamos também o fuso da atividade** (`event_timezone`), que diz em qual "relógio de parede" ela foi pensada.
3. **Mostramos no fuso do perfil**, que pode ser alterado em Configurações. Se a atividade tiver um fuso diferente do perfil (ex.: uma viagem a Lisboa), a tela mostra os dois horários: "14:00 (Lisboa) · 10:00 (seu horário)".
4. **Todo cálculo de data passa por um único arquivo** (`src/lib/dates.ts`). Nenhuma tela converte datas por conta própria. Assim, um erro de fuso se corrige num lugar só.
5. **Formato brasileiro:** `dd/MM/yyyy` e `HH:mm` (24h), com nomes de meses e dias em português (`date-fns` com `ptBR`).

### 7.2 Como cada caso é tratado

| Situação | Ao salvar | Ao mostrar |
|---|---|---|
| Compromisso com hora | Data + hora digitadas são interpretadas **no `event_timezone`** e convertidas no instante exato. | Convertido para o fuso do perfil. |
| Dia inteiro (`all_day`) | Guarda 00:00 do dia no `event_timezone` (início) e 00:00 do dia seguinte (fim). | Mostrado sempre no **`event_timezone`**, para que "aniversário em 10/11" nunca vire 09/11. |
| Prazo só com data (`due_has_time = false`) | Guarda 23:59:59 daquele dia no `event_timezone`. | Mostra só a data ("até 10/11"). |
| Meta semanal (`period_start`) | `date` simples (dia do calendário). | Sem conversão. |
| "Hoje", "esta semana" no dashboard | O app calcula o início e o fim do dia **no fuso do perfil** e pede ao banco tudo entre esses dois instantes. | |
| Repetição (RRULE) | A regra é guardada junto com o `event_timezone`. | As ocorrências são geradas **no relógio local** do fuso da atividade, para que "toda segunda às 09:00" continue 09:00 mesmo quando um país muda de horário de verão. |

### 7.3 Armadilhas que o código vai evitar

- Nunca criar data a partir de texto só com dia (`"2026-11-10"` é lido como meia-noite **UTC**, que em Belém é 21:00 do dia **anterior**).
- Nunca usar o fuso do aparelho em vez do fuso do perfil. Ela pode estar viajando, e o relógio do celular mudar.
- O banco sempre trabalha em UTC. Funções que precisam do conceito de "hoje" recebem o fuso como parâmetro.
- Fuso inválido é recusado pelo banco (validação contra a lista oficial do PostgreSQL).
- Testes automáticos de datas para Belém (sem horário de verão), São Paulo, Lisboa e Nova York (com horário de verão), incluindo dias de virada de horário.

### 7.4 Risco remanescente (baixo)

Se um governo mudar as regras de horário de um fuso **depois** que um compromisso futuro foi salvo (por exemplo, o Brasil voltar a ter horário de verão), o instante guardado continua o mesmo e o horário mostrado pode mudar em 1 hora. Isso é raro. Como guardamos o `event_timezone`, será possível recalcular esses compromissos caso aconteça.

---

## 8. Integração das futuras áreas e módulos

🐾 O núcleo (perfis, áreas, categorias, itens, metas, atividades, histórico) funciona como uma "tomada padrão". Cada módulo futuro (Pets, Fazenda, Financeiro...) se encaixa nessa tomada sem mudar o núcleo.

### 8.1 Regras para todo módulo novo

1. **Tabelas próprias**, com prefixo do módulo. Exemplos:
   - Pets: `pets`, `pet_vaccines`, `pet_weights`
   - Fazenda: `farm_animals`, `farm_lots`, `farm_productions`
   - Financeiro: `finance_accounts`, `finance_transactions`, `finance_budgets`
   - Estudos: `study_courses`, `study_subjects`, `study_sessions`
2. Toda tabela de módulo segue as **regras comuns** (3.2): `user_id`, datas de controle, `archived_at`, `is_demo`, RLS com o mesmo modelo, chaves compostas para área e categoria.
3. **O módulo aponta para o núcleo, nunca o contrário.** Exemplo: a vacina do pet tem `activity_id` apontando para a atividade "Vacina da Mimi — 15/11". O calendário unificado continua lendo só `activities` e mostra tudo, sem precisar conhecer cada módulo.
4. **`module_key` da área** diz qual módulo "mora" naquela área. Quando ela abre a área de `module_key = pets`, o app carrega a tela especializada de Pets. Áreas sem `module_key` usam a tela genérica (categorias + itens + atividades). Se ela renomear "Meus Pets" para "Meus Bichinhos", nada quebra.
5. **Carregamento sob demanda:** cada módulo é carregado só quando aberto, para o app continuar leve.
6. **Histórico:** o mesmo gatilho genérico de histórico é ligado às tabelas novas.
7. **Valores em dinheiro** (Financeiro): `numeric(14,2)`, nunca números com casas decimais imprecisas, e sempre com a moeda (`BRL` como padrão).
8. Cada módulo novo entra como **novos scripts numerados**. Nada do núcleo é reescrito.

### 8.2 Integrações externas (futuro)

- **Google Calendar:** Edge Function com OAuth do Google. Os tokens ficam numa tabela acessível só pelo servidor. A sincronização usa `external_source`, `external_id` e `external_etag`, já reservados em `activities`.
- **Google Drive/Planilhas:** decisão adiada, conforme a especificação.

---

## 9. Preparação para a Miau IA

🐾 A Miau IA vai **sugerir**, nunca agir sozinha. Toda sugestão vira uma "proposta" que ela precisa aprovar, uma por uma ou todas de uma vez.

### 9.1 Fluxo planejado (futuro)

```
1. Ela pede: "Organize minha semana"
2. O app cria um ai_request (status: pendente)
3. O app chama a Edge Function miau-ia (com o token de login dela)
4. A Edge Function:
   • confere quem é ela e se não passou do limite diário
   • lê SÓ os dados dela necessários (respeitando o RLS)
   • chama o Gemini com a chave secreta (guardada no servidor)
   • grava a resposta e as propostas em ai_proposed_changes
5. A tela mostra as propostas: "Mover 'Estudar inglês' para quarta 19:00?"  [Aprovar] [Rejeitar]
6. Ao aprovar, uma função do banco aplica a mudança:
   • confere se a proposta é dela e ainda está pendente
   • confere se o registro alvo não mudou desde a proposta
   • aplica e anota no histórico com actor = miau_ia
```

### 9.2 Tabelas

**`ai_requests`**: `id`, `user_id`, `request_type` (`organizar_tarefas`, `sugerir_prioridades`, `planejar_estudos`, `montar_rotina`, `detectar_conflitos`, `distribuir_semana`, `analisar_metas`, `livre`), `prompt`, `context_summary` (jsonb: o que foi enviado ao Gemini, para transparência), `status` (`pendente`, `processando`, `concluido`, `erro`, `cancelado`), `response_text`, `model`, `tokens_input`, `tokens_output`, `error_message`, `created_at`, `updated_at`, `archived_at`.

**`ai_proposed_changes`**: `id`, `user_id`, `ai_request_id`, `operation` (`criar`, `atualizar`, `arquivar`; **nunca excluir**), `target_table` (lista fechada: `activities`, `items`, `goals`, `categories`), `target_id`, `payload` (jsonb com os campos propostos), `explanation` (texto amigável), `target_updated_at` (para detectar se o registro mudou depois), `status` (`pendente`, `aprovada`, `rejeitada`, `aplicada`, `falhou`, `expirada`), `decided_at`, `applied_at`, `created_at`, `updated_at`.

### 9.3 Na Etapa 1

- As tabelas e as políticas RLS são criadas.
- A página **Miau IA** existe, com visual completo, explicando o que a IA fará, e o campo de pedido **desativado** com o aviso "Em breve".
- A Edge Function `miau-ia` existe só como esqueleto, respondendo "ainda não disponível". **Não haverá nenhuma resposta simulada fingindo ser IA.**
- A chave do Gemini **não** é necessária nesta etapa.

---

## 10. Navegação, telas e identidade visual

### 10.1 Rotas (endereços)

| Endereço | Página | Login? |
|---|---|---|
| `/entrar`, `/cadastro`, `/recuperar-senha`, `/redefinir-senha` | Autenticação | Não |
| `/` | Início (dashboard) | Sim |
| `/calendario` | Calendário (na Etapa 1: agenda em lista por dia, com filtros) | Sim |
| `/areas` | Todas as áreas | Sim |
| `/areas/:id` | Uma área: categorias, atividades e itens | Sim |
| `/miau-ia` | Miau IA (estrutura) | Sim |
| `/configuracoes` | Perfil, fuso, preferências, áreas (reordenar), arquivados, dados demo, sair | Sim |

### 10.2 Navegação responsiva

- **Computador/tablet largo:** menu lateral fixo com Início, Calendário, Miau IA, uma seção "Minhas áreas" (as áreas ativas, na ordem dela) e Configurações no rodapé.
- **Celular:** barra inferior com exatamente 5 atalhos: **Início · Calendário · Áreas · Miau IA · Mais**. "Mais" abre uma folha deslizante com Configurações, Arquivados, Histórico e Sair.
- Respeita a área segura do iPhone (a "barrinha" de baixo e o recorte da câmera).
- Acessível: navegável por teclado, rótulos para leitores de tela, alvos de toque de pelo menos 44 px.

### 10.3 Dashboard (Início)

| Bloco | Conteúdo |
|---|---|
| Saudação | "Bom dia, Thaís 🐾" (manhã/tarde/noite **no fuso do perfil**) |
| Data | "Sexta-feira, 06/11/2026" |
| Resumo do dia | Compromissos de hoje, tarefas que vencem hoje, atrasadas |
| Cartões das áreas | Ícone, cor, nome e um número de **atenção** |
| Pendências | Lista curta das atrasadas e das que vencem hoje |
| Próximos compromissos | Os 5 próximos, com hora e local |
| Metas semanais | Barras de progresso |
| Atalhos | Calendário e Miau IA |

**Definição do número de "atenção" de cada área** (não é o total de registros):
atividades **atrasadas** + atividades que **vencem hoje** + **compromissos e tarefas agendadas de hoje** ainda não concluídos.
O número é calculado por uma única função do banco (`dashboard_summary`), que recebe o fuso do perfil. Assim a tela faz **uma** consulta em vez de dez.

### 10.4 Identidade visual

**Personalidade:** kawaii adulto. Delicado e acolhedor, mas organizado e legível.

| Token | Uso | Tom aproximado |
|---|---|---|
| `blush` | Destaques, botão principal | #F2B8B5 |
| `rosa-pastel` | Fundos de cartões suaves | #F9DCE1 |
| `lilas` | Elementos secundários, IA | #D8CCF0 |
| `creme` | Fundo geral | #FBF5EC |
| `salvia` | Sucesso, concluído, saúde | #B8CBB2 |
| `pessego` | Avisos suaves, prazos | #F7D2BA |
| `branco-quente` | Superfície dos cartões | #FFFCF8 |
| `ameixa` | **Texto** (escuro, para contraste) | #4A3644 |

- **Contraste:** tons pastel só como fundo. Texto sempre escuro (`ameixa`), para atingir o nível AA de acessibilidade. Cores de alerta (atrasado) com tom um pouco mais forte e sempre acompanhadas de ícone ou texto, nunca só cor.
- **Tipografia:** **Nunito** (arredondada, amigável e muito legível), com os arquivos da fonte guardados no próprio app (sem depender de serviço externo; funciona no PWA).
- **Formas:** cartões com cantos de 16–24 px, sombras discretas e espaçamento generoso.
- **Detalhes:** patinhas e um gatinho de linha fina em estados vazios ("Nada para hoje 🐾"), carregamento e boas-vindas. Desenhos próprios, em SVG.
- **Animações:** suaves (150–250 ms) e desligadas automaticamente se o aparelho pedir "reduzir movimento".
- **Componentes reutilizáveis:** Botão (principal, secundário, perigo, só ícone), Cartão, Campo de texto, Seletor (data, hora, cor, ícone), Modal, Folha deslizante (celular), Aviso (toast), Barra de progresso, Selo ("exemplo", "atrasado"), Estado vazio, Esqueleto de carregamento, Confirmação de exclusão com contagem.

---

## 11. Hospedagem, PWA e as duas versões

🐾 **Será um único app, com um único endereço.** Ele funciona de dois jeitos:

| Versão | Como acessar |
|---|---|
| **Computador** (para a Thaís analisar) | Abrir o link no Chrome, Edge ou Safari. No Chrome/Edge, também dá para "instalar" como janela própria. |
| **iPhone** | Abrir o link no **Safari** → botão Compartilhar → **"Adicionar à Tela de Início"**. Vira um ícone, abre em tela cheia, sem barra do navegador. O link pode ser enviado por WhatsApp. |
| Android | Chrome → "Instalar app". |

**Vercel:**
- Cada envio ao GitHub gera uma **versão de pré-visualização** com link próprio (ótimo para a Thaís testar antes). A branch principal gera a versão oficial.
- Variáveis configuradas no painel da Vercel: `VITE_SUPABASE_URL` e `VITE_SUPABASE_ANON_KEY`. A chave anônima/"publishable" é **pública por natureza**: ela só permite o que o RLS permite.
- Arquivo `vercel.json` para que recarregar `/calendario` não dê erro 404.

**PWA:**
- Manifesto com nome "Miau Life", cores e ícones (incluindo o ícone de 180 px para iPhone).
- O *service worker* guarda apenas os **arquivos do app** (telas, fontes, ícones), para abrir rápido. **Dados não ficam guardados offline na Etapa 1.** Sem internet, o app abre e mostra um aviso amigável.
- Aviso "Nova versão disponível, toque para atualizar" quando publicarmos melhorias.

**Importante no iPhone:** o app instalado na tela inicial tem um "armazenamento" separado do Safari. Por isso, ela precisará **fazer login uma vez dentro do app instalado**, mesmo já estando logada no Safari. Isso é normal e será explicado na tela de boas-vindas.

---

## 12. Escalabilidade: possíveis problemas

| Possível problema | Quando aparece | Como o plano já previne |
|---|---|---|
| RLS lento em tabelas grandes | Muitos milhares de linhas | `(select auth.uid())` + índice começando por `user_id` em todas as tabelas. |
| Dashboard com muitas consultas | Desde o início | Uma função (`dashboard_summary`) que devolve tudo de uma vez. |
| Listas enormes | Histórico, itens antigos | Paginação (carregar aos poucos) em todas as listas que crescem. |
| `history_log` crescendo sem fim | Anos de uso | Só campos que mudaram; sem registros de demo ou de reordenação. No futuro: particionar por ano ou arquivar históricos antigos. |
| Uso excessivo de JSONB | Se tudo for parar em `data` | Regra: JSONB só para dados leves sem regra própria. O resto vira coluna ou tabela de módulo. Limite de tamanho no banco. |
| Repetições (RRULE) | Calendário com muitos eventos recorrentes | Expandir só a janela visível (mês/semana). `recurrence_until` ajuda a filtrar. Se ficar pesado, mover a expansão para uma função de servidor. |
| App pesado no celular | Com muitos módulos | Cada página/módulo carregado sob demanda. |
| Limites do plano gratuito do Supabase | 500 MB de banco, e-mails limitados | Folga grande para uso pessoal. Ver riscos na seção 13. |
| Custos do Gemini | Quando a IA existir | Limite diário por usuário na Edge Function e registro de tokens em `ai_requests`. |
| Muitos usuários no futuro | Se o app for aberto a outras pessoas | O modelo já é multiusuário desde o início (tudo isolado por `user_id`). |

---

## 13. Riscos técnicos e decisões arquiteturais

### 13.1 Riscos

| Risco | Impacto | Mitigação |
|---|---|---|
| **Projeto gratuito do Supabase "pausa" após ~7 dias sem uso** | O app para de funcionar até alguém reativar no painel. | Explicarei como reativar. Para uso diário isso não acontece. Plano Pro (~US$ 25/mês) elimina o problema. |
| **E-mails do Supabase gratuito têm limite baixo** (poucos por hora) e podem cair no spam | Confirmação de cadastro ou recuperação de senha podem atrasar. | Suficiente para 1–2 pessoas. Se precisar, configurar um provedor de e-mail próprio (SMTP), com guia. |
| **Vercel Hobby (grátis) é para uso pessoal, não comercial** | Se o app virar produto, precisa do plano Pro (~US$ 20/mês). | Ok para a fase atual. Reavaliar se for comercializar. |
| Login separado no app do iPhone | Confusão no primeiro uso | Aviso na tela de boas-vindas. |
| Notificações/lembretes no iPhone | Só funcionam em PWA instalado (iOS 16.4+) e exigem servidor de envio | Fora da Etapa 1. Os lembretes são apenas guardados. Planejar na etapa do Calendário. |
| Sem modo offline | Sem internet, não dá para ver ou criar dados | Aviso amigável. Offline real é complexo (conflitos de sincronização) e fica para avaliar depois. |
| Gatilho de cadastro com erro bloqueia novos cadastros | Ninguém consegue se cadastrar | Teste específico antes de publicar; a função é curta e simples. |
| Rodar SQL manualmente na ordem errada | Banco incompleto | Scripts numerados, cada um com verificação no fim ("deu certo se aparecer..."), e um guia passo a passo. |
| Gemini gratuito pode usar os dados enviados para melhorar o modelo | Privacidade (dados pessoais, financeiros) | Antes de ativar a IA, decidir entre plano pago (que não usa dados) ou enviar só resumos. Fica para a etapa da IA. |
| LGPD / exclusão de conta | Direito de apagar todos os dados | O banco já apaga tudo em cascata a partir da conta. Falta a Edge Function "Excluir minha conta" (pendência registrada). |
| Mudança de regra de fuso (7.4) | Horário futuro desloca 1h | Raro; `event_timezone` permite corrigir. |
| Eu não consigo testar o Supabase real daqui | Algo pode funcionar no build, mas falhar no banco real | Vou testar o SQL num PostgreSQL local, simulando o ambiente do Supabase (se for possível instalar aqui), e te entregar um roteiro de teste com duas contas. |

### 13.2 Decisões arquiteturais (resumo)

| Decisão | Motivo | Alternativa descartada |
|---|---|---|
| SPA (app de página única) na Vercel + Supabase | Simples, barato, ideal para PWA | Next.js com servidor: mais complexo e sem ganho real aqui |
| Segurança no banco (RLS + chaves compostas) | Protege mesmo se o código da tela tiver falha | Validar só no navegador: inseguro |
| Tabelas específicas + JSONB leve | Cada coisa com regras próprias, sem engessar | Tabela "genérica de tudo": vira bagunça e fica lenta |
| `text` + `CHECK` para listas de opções | Fácil de ampliar | `enum` do PostgreSQL: difícil de alterar |
| Quatro tipos de atividade numa tabela só | O calendário e o dashboard leem tudo junto | Quatro tabelas: consultas mais complexas |
| Instante (`timestamptz`) + `event_timezone` | Correto em qualquer fuso (D1) | Data e hora separadas: fonte clássica de erros |
| Cor e ícone guardados como "nome" | Permite redesenhar e criar modo escuro | Guardar o código hexadecimal |
| Histórico por gatilho, sem chave estrangeira | Não se perde nada, mesmo após excluir | Histórico feito pelo app: pode ser esquecido |
| IA só propõe e o banco aplica com conferência | Controle total da usuária | IA gravando direto: perigoso |
| Sem `localStorage` para dados | Exigência da especificação; dados no banco | |

---

## 14. Como vamos testar (Fase B)

**O que eu consigo testar aqui:**
1. Instalação das dependências e **build** do app (`npm run build`) sem erros.
2. Verificação de tipos (TypeScript) e de padrões de código (lint).
3. Testes automáticos das funções de data e fuso (Belém, São Paulo, Lisboa, Nova York, dias de virada de horário).
4. Os scripts SQL num PostgreSQL local, se for possível instalá-lo neste ambiente, incluindo um teste de isolamento: "usuário A tenta ler, alterar e excluir dados do usuário B" → deve falhar em todos os casos. Também: "tentar ligar atividade à área de outra pessoa" → deve falhar.
5. Abrir as telas num navegador automatizado, no tamanho de celular e de computador, e tirar capturas para conferir o visual.

**O que depende de você** (com roteiro passo a passo em `docs/TESTES_MANUAIS.md`):
1. Rodar os scripts no Supabase real e conferir as mensagens de sucesso.
2. Criar **duas contas de teste** e confirmar que uma não vê os dados da outra.
3. Testar cadastro, confirmação de e-mail, login, logout e recuperação de senha com e-mails reais.
4. Instalar no iPhone e testar.

Não vou dizer que algo "funciona" sem ter testado. Se algo só puder ser testado por você, vou dizer exatamente isso.

---

## 15. O que você vai precisar fazer manualmente (prévia)

Na Fase B, cada item terá um guia com prints descritos e "como saber se deu certo":

1. **Criar conta no Supabase** (supabase.com) e um projeto. Região sugerida: **São Paulo** (mais perto, mais rápido). Guardar a senha do banco num lugar seguro.
2. **Copiar duas informações** do painel: a URL do projeto e a chave pública (anon/publishable).
3. **Executar os scripts SQL** no "SQL Editor", na ordem numerada.
4. **Configurar a autenticação** (URLs, confirmação de e-mail, e-mails em português).
5. **Criar conta na Vercel** com o login do GitHub e importar o repositório THAISAPP.
6. **Colar as duas variáveis** no painel da Vercel e publicar.
7. **Testar** com o roteiro de testes manuais.

A chave **service_role** e a chave do **Gemini** não serão usadas na Etapa 1. Quando forem necessárias, vão direto para o painel do Supabase (área de "secrets"), **nunca** para arquivos do projeto.

---

## 16. Pontos que preciso que você aprove

### 16.1 Ajuste técnico na decisão D5 (RESTRICT → NO ACTION)

**O que é:** a decisão D5 diz "ON DELETE RESTRICT em áreas e categorias". Proponho usar a variante **`NO ACTION`**, que para o uso normal **bloqueia a exclusão exatamente do mesmo jeito**: uma área com registros ligados não pode ser excluída.

**Por que mudar:** existe uma diferença técnica de *momento* da verificação. Com `RESTRICT`, o banco verifica imediatamente, linha por linha. Quando uma **conta inteira** for excluída (exigência da LGPD), o banco apaga tudo daquela pessoa de uma vez, e o `RESTRICT` pode "travar" no meio, porque encontra uma área que ainda tem categorias que estão prestes a ser apagadas na mesma operação. Com `NO ACTION`, a verificação acontece **no fim** da operação, quando tudo já foi apagado, e a exclusão da conta funciona.

| | RESTRICT | NO ACTION (proposto) |
|---|---|---|
| Impede excluir área com registros ligados | ✅ | ✅ |
| Permite excluir a conta inteira (LGPD) | ❌ pode falhar | ✅ |
| Custo/complexidade | igual | igual |

### 16.2 Escopo da Etapa 1 (confirmar)

Entram:
- Áreas (CRUD completo, reordenar, arquivar/restaurar, exclusão segura).
- Categorias (CRUD).
- Atividades e itens com **formulário básico** de criação e edição dentro de cada área (para o dashboard mostrar dados reais, e não só exemplos).
- Metas semanais (criar e atualizar progresso).
- Calendário como **agenda em lista** (dia/semana). A grade mensal completa e a recorrência expandida ficam para a etapa do Calendário unificado.
- Miau IA só com estrutura.
- Configurações: perfil, foto, fuso, início da semana, arquivados, dados demo, sair.

Não entram: notificações reais, recorrência com exceções, Google Calendar, Gemini ativo, exclusão de conta, modo offline, módulos especializados.

### 16.3 Ferramentas de desenvolvimento extras (não vão para o app final)

- **Vitest**: para os testes automáticos de datas e fusos. Roda só no desenvolvimento; não aumenta o tamanho do app. Alternativa sem ele: testes manuais, menos confiáveis para fusos.
- **Supabase CLI** (opcional): gerar automaticamente os "tipos" do banco para o TypeScript. Sem ele, eu escrevo os tipos à mão (funciona, mas exige cuidado ao mudar o banco).

### 16.4 Pequenas escolhas (já propostas; mude se quiser)

- Fonte **Nunito**.
- Semana começando no **domingo** (configurável).
- Confirmação de e-mail **ligada** no cadastro.
- Salvar `ESPECIFICACAO.md` no repositório (você colou o texto no chat; eu não criei o arquivo para respeitar a regra de só criar o `PLANO.md`).

---

*Fim do plano. Aguardando aprovação para iniciar a Fase B.*
