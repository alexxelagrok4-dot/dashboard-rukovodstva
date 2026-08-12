# ДАШБОРД-РУКОВОДСТВА

Еженедельный управленческий дашборд для планёрок с руководителями: план/факт по выручке, утилизации, стоимости аренды, грузообороту, человеко-часам и себестоимости операции — раздельно опалубка (тонны) и бытовки (штуки).

Структура и логика показателей зафиксированы в артефакте-черновике (дерево показателей, схема воронки лид→сделка, открытые решения) — см. историю обсуждения. Ключевой принцип: «Выручка» — это физическое поступление денег за неделю (касса, не начисление), и раскладывается не на «причины», а на **сумму четырёх независимо планируемых денежных потоков**, у каждого своя ответственная служба:

| Поток | Служба |
|---|---|
| Новые клиенты | Продажи |
| Повторные сделки | Продажи (клиентский менеджер) |
| Продления договоров | Продажи (клиентский менеджер) |
| Собрано по долгу | Безопасность |

Файл `index.html` — самодостаточный (как и остальные инструменты компании), использует тот же Supabase-проект и тот же общий логин/пароль, что и «ПРОДЛЕНИЕ-ВЗЫСКАНИЕ» — отдельного аккаунта заводить не нужно.

## v1: ручной ввод

Автоматических выгрузок пока нет ни по одному потоку (кроме того, что уже есть в DAYA) — первая версия работает на **еженедельном ручном вводе** 32 показателей через форму. Источники по каждому потоку подключаются по мере готовности, один за другим — так же поэтапно, как развивался инструмент СБ (см. его changelog).

Известные источники на будущее:
- **Опалубка** — данные по сделкам/договорам/движениям уже есть в DAYA.
- **Бытовки** — учёт ведётся в 1С (формат выгрузки уточняется).
- **Лиды / целевые лиды** — источник будет определён позже (вероятно Bitrix24).

## Настройка базы (один раз)

Используется тот же Supabase-проект, что и у инструмента «ПРОДЛЕНИЕ-ВЗЫСКАНИЕ» — просто добавляются две новые таблицы. В **SQL Editor** этого проекта выполнить:

```sql
create extension if not exists pgcrypto;

create table public.dash_weeks (
  id uuid primary key default gen_random_uuid(),
  week_start date not null unique,
  created_by uuid references auth.users(id),
  created_by_email text,
  created_at timestamptz not null default now()
);

create table public.dash_entries (
  id uuid primary key default gen_random_uuid(),
  week_id uuid references public.dash_weeks(id) on delete cascade,
  metric_key text not null,
  value numeric,
  updated_by_email text,
  updated_at timestamptz not null default now(),
  unique(week_id, metric_key)
);
create index dash_entries_week_id_idx on public.dash_entries(week_id);

alter table public.dash_weeks enable row level security;
alter table public.dash_entries enable row level security;

create policy "read for authenticated" on public.dash_weeks for select using (auth.role() = 'authenticated');
create policy "insert for authenticated" on public.dash_weeks for insert with check (auth.role() = 'authenticated');

create policy "read for authenticated" on public.dash_entries for select using (auth.role() = 'authenticated');
create policy "insert for authenticated" on public.dash_entries for insert with check (auth.role() = 'authenticated');
create policy "update for authenticated" on public.dash_entries for update using (auth.role() = 'authenticated');
```

Значения хранятся построчно (метрика → число за неделю), а не жёсткими колонками — так проще добавлять новые показатели позже (например, ветку лидов), не меняя структуру таблицы.

## Список показателей (32 поля ввода)

Генерируются в коде из справочников `REVENUE_STREAMS` × `PRODUCTS` (выручка, 16 полей), `PRODUCTS` (утилизация, 4 поля), `PRODUCTS` × `TURNOVER_STAGES` (грузооборот, 8 полей), плюс человеко-часы и ФОТ (4 поля). Единый источник списка полей для формы ввода и для плиток — расхождение между ними исключено конструктивно (оба берут данные из одного и того же списка в коде).

«Стоимость аренды» и «Себестоимость операции» — не вводятся вручную, а вычисляются из остальных полей.
