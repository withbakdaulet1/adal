# ADAL TARGET — Жиналыс кестесі

Мидл таргетологтар мен РОМ-ға арналған күндік/айлық отчет дашборды. Бастапқыда Claude Artifact ретінде жасалды, енді Vercel-ге дербес сайт ретінде көшірілуде.

## Қазіргі жағдай

`adal_dashboard.html` — толық жұмыс істейтін UI (Күндік / Айлық жиынтық / Жоспарлар / Проекттер беттері, есептеу логикасы, жасыл/қызыл статус) дайын. Бірақ дерек сақтау мен авторизация әзірге **Claude-ге тән API-мен** жазылған (`window.claude.use('db')`, `window.claude.use('user')`) — бұлар тек claude.ai ішінде жұмыс істейді.

## Не істеу керек (Vercel-ге көшіру)

Файл ішінде `// === MIGRATE: ... ===` деп белгіленген **9 функция** бар. Соларды Supabase-пен ауыстыру керек:

| Функция | Немен ауыстырылады |
|---|---|
| `initStorage()` | Supabase Auth session (`supabase.auth.getUser()`) |
| `loadRoles()` | `users` кестесінен `role` бағанын оқу |
| `setRole()` | `users` кестесін жаңарту |
| `loadAll()` | `projects`, `entries`, `monthlyPlans` кестелерін оқу |
| `setMonthlyPlan()` | `monthly_plans` кестесіне жазу |
| `saveEntry()` | `entries` кестесіне жазу |
| `addProject()` / `deleteProject()` | `projects` кестесіне жазу/өшіру |
| `wireRoleSearch()` | `users` кестесінен атымен іздеу |

## Ұсынылған дерекқор құрылымы (Supabase Postgres)

```sql
-- Пайдаланушылар (Supabase Auth-пен байланысты)
create table users (
  id uuid primary key references auth.users(id),
  name text,
  role text default 'midl' check (role in ('midl', 'rom')) -- жаңа тіркелген = midl
);

create table projects (
  id uuid primary key default gen_random_uuid(),
  name text not null
);

create table entries (
  date date not null,
  project_id uuid references projects(id) on delete cascade,
  revenue numeric default 0,
  cost numeric default 0,
  problem text default '',
  primary key (date, project_id)
);

create table monthly_plans (
  month_key text not null,        -- '2026-09' форматында
  project_id uuid references projects(id) on delete cascade,
  plan numeric default 0,
  primary key (month_key, project_id)
);
```

## Рөл логикасы

- Жаңа тіркелген адам әдепкі бойынша `role = 'midl'` — «Проекттер» бетін көрмейді.
- РОМ-ды белгілеу үшін Supabase Table Editor-де сол адамның `role` мәнін `'rom'` етіп қолмен өзгертесіз (бір рет).
- Row Level Security (RLS) саясаттарын қосу ұсынылады: `entries`/`monthly_plans` кестелеріне жазу — кез келген аутентификацияланған пайдаланушыға; `projects`/`users.role` кестесіне жазу — тек `role = 'rom'` адамдарға.

## Деплой қадамдары

1. Осы репозиторийді Vercel-ге қосу (GitHub → Import Project)
2. Supabase жобасын жасау (тегін жоспар жеткілікті), `SUPABASE_URL` мен `SUPABASE_ANON_KEY`-ды Vercel Environment Variables-қа қосу
3. Жоғарыдағы SQL-ды Supabase SQL Editor-де іске қосу
4. `adal_dashboard.html` ішіндегі 9 `MIGRATE` блогын Supabase JS клиентімен ауыстыру
5. Бірінші РОМ-ды Supabase Table Editor арқылы қолмен белгілеу
