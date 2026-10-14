# 🏋️ Meu Treino

A lightweight, mobile-first workout tracker that runs entirely in the browser. Log your sets, reps and weights, browse your training history, and watch your progress on a chart — all backed by [Supabase](https://supabase.com) for authentication and storage.

The whole app is a single `index.html` file: no build step, no bundler, no server code.

## Features

- **Accounts** — sign up and sign in with email and password (Supabase Auth). Each user only sees their own data.
- **Log workouts** — pick a muscle group, then one of its exercises (listed alphabetically) and the date, then enter each set with its own weight (kg) and reps; add or remove sets as needed, plus an optional note. The form is pre-filled with the sets from your last entry for that exercise.
- **Workout plans** — in the *Fichas* tab, create plans such as *Treino A (Push)*: give it a name and an optional description, add exercises (pick a muscle group, then the exercise) and put them in order with the ↑ / ↓ buttons. In the *Registrar* tab, choose a plan to log it exercise by exercise: the list shows which ones you've already logged on the selected date (✓), and saving moves on to the next pending one. You can tap any exercise to skip ahead or go back, or pick *Exercício avulso* to log a single exercise as before.
- **Custom exercises** — in the *Exercícios* tab, add your own exercises (each one belongs to a muscle group), see the full list and edit the name or muscle group. The muscle groups are a fixed list stored in the database, with no screen to manage them.
- **History** — your latest 150 entries grouped by day, with the option to delete an entry.
- **Progress** — per-exercise chart with one line per set (switch between weight and reps), a list of what changed in each set from one session to the next (e.g. set 2 went from 18 to 20 kg, set 3 from 11 to 12 reps), and a summary of your personal record, best estimated one-rep max (Epley formula), change since the first entry and number of sessions.
- **Light and dark mode** — follows your system preference.

> The user interface is in Brazilian Portuguese.

## Tech stack

| Piece | Used for |
| --- | --- |
| HTML, CSS, vanilla JavaScript (ES modules) | The app itself |
| [supabase-js v2](https://supabase.com/docs/reference/javascript) | Auth and database access |
| [Chart.js v4](https://www.chartjs.org/) | Progress chart |

Both libraries are loaded from the jsDelivr CDN.

## Getting started

### 1. Create a Supabase project

Create a free project at [supabase.com](https://supabase.com).

### 2. Create the database tables

Open the **SQL Editor** in your Supabase project and run the script below. It creates the five tables the app uses, fills in the muscle groups and enables Row Level Security so that every user can only read and change their own rows. The muscle groups are shared by all users and are read-only from the app.

```sql
create table public.grupos_musculares (
  id   bigint generated always as identity primary key,
  nome text not null unique
);

insert into public.grupos_musculares (nome) values
  ('Abdômen'), ('Abdutores'), ('Adutores'), ('Antebraço'), ('Bíceps'),
  ('Costas'), ('Glúteos'), ('Lombar'), ('Ombros'), ('Panturrilha'),
  ('Peito'), ('Posterior de coxa'), ('Quadríceps'), ('Trapézio'), ('Tríceps');

create table public.exercicios (
  id                bigint generated always as identity primary key,
  user_id           uuid not null default auth.uid() references auth.users on delete cascade,
  nome              text not null,
  grupo_muscular_id bigint not null references public.grupos_musculares,
  created_at        timestamptz not null default now(),
  unique (user_id, nome)
);

create table public.registros (
  id           bigint generated always as identity primary key,
  user_id      uuid not null default auth.uid() references auth.users on delete cascade,
  exercicio_id bigint not null references public.exercicios on delete cascade,
  data         date not null default current_date,
  series       jsonb not null check (jsonb_typeof(series) = 'array' and jsonb_array_length(series) > 0),
  observacao   text,
  created_at   timestamptz not null default now()
);

create table public.fichas (
  id         bigint generated always as identity primary key,
  user_id    uuid not null default auth.uid() references auth.users on delete cascade,
  nome       text not null,
  descricao  text,
  created_at timestamptz not null default now(),
  unique (user_id, nome)
);

create table public.ficha_exercicios (
  ficha_id     bigint not null references public.fichas on delete cascade,
  exercicio_id bigint not null references public.exercicios on delete cascade,
  ordem        int not null,
  primary key (ficha_id, exercicio_id)
);

alter table public.grupos_musculares enable row level security;
alter table public.exercicios        enable row level security;
alter table public.registros         enable row level security;
alter table public.fichas            enable row level security;
alter table public.ficha_exercicios  enable row level security;

create policy "read muscle groups" on public.grupos_musculares
  for select to authenticated using (true);

create policy "own exercises" on public.exercicios
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "own entries" on public.registros
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "own plans" on public.fichas
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- A plan's exercises belong to whoever owns the plan, and can only point to that user's exercises
create policy "own plan exercises" on public.ficha_exercicios
  for all
  using (exists (select 1 from public.fichas f where f.id = ficha_id and f.user_id = auth.uid()))
  with check (
    exists (select 1 from public.fichas f where f.id = ficha_id and f.user_id = auth.uid())
    and exists (select 1 from public.exercicios e where e.id = exercicio_id and e.user_id = auth.uid())
  );
```

| Table | Purpose |
| --- | --- |
| `grupos_musculares` | Muscle groups (`nome` = name). Filled in by the script above; to add or rename one, edit the table in Supabase |
| `exercicios` | Exercises (`nome` = name, `grupo_muscular_id` = its muscle group) |
| `registros` | Workout entries (`data` = date, `observacao` = note). `series` holds the sets in order, each with its own weight in kg and reps: `[{"carga": 14, "repeticoes": 15}, {"carga": 18, "repeticoes": 12}]` |
| `fichas` | Workout plans (`nome` = name, `descricao` = optional description) |
| `ficha_exercicios` | The exercises in each plan, in order (`ordem` = position, starting at 1). An exercise appears at most once per plan |

#### Upgrading an existing database

Run the upgrades your database still needs, in this order, in the SQL Editor **before** deploying the new `index.html`.

**Sets stored as a list.** Earlier versions stored a single `series` count plus one `repeticoes` and `carga` value per entry. If your database was created with that schema, run this once. Each old entry becomes a list of identical sets, so no history is lost.

```sql
begin;

alter table public.registros add column if not exists series_novo jsonb;

-- Repeats { carga, repeticoes } once per set: 3 sets → a list with 3 identical items
update public.registros
   set series_novo = to_jsonb(array_fill(
     jsonb_build_object('carga', carga, 'repeticoes', repeticoes),
     array[series]
   ));

alter table public.registros
  drop column series,
  drop column repeticoes,
  drop column carga;

alter table public.registros rename column series_novo to series;

alter table public.registros
  alter column series set not null,
  add constraint registros_series_lista
    check (jsonb_typeof(series) = 'array' and jsonb_array_length(series) > 0);

commit;
```

**Muscle groups as a table.** Earlier versions stored the muscle group as optional free text in `exercicios.grupo_muscular`. This upgrade creates the `grupos_musculares` table and links each exercise to the group with the same name, ignoring upper and lower case:

```sql
begin;

create table public.grupos_musculares (
  id   bigint generated always as identity primary key,
  nome text not null unique
);

insert into public.grupos_musculares (nome) values
  ('Abdômen'), ('Abdutores'), ('Adutores'), ('Antebraço'), ('Bíceps'),
  ('Costas'), ('Glúteos'), ('Lombar'), ('Ombros'), ('Panturrilha'),
  ('Peito'), ('Posterior de coxa'), ('Quadríceps'), ('Trapézio'), ('Tríceps');

alter table public.grupos_musculares enable row level security;

create policy "read muscle groups" on public.grupos_musculares
  for select to authenticated using (true);

alter table public.exercicios
  add column grupo_muscular_id bigint references public.grupos_musculares;

update public.exercicios e
   set grupo_muscular_id = g.id
  from public.grupos_musculares g
 where lower(trim(e.grupo_muscular)) = lower(g.nome);

commit;

-- Exercises that still have no muscle group
select id, nome, grupo_muscular from public.exercicios where grupo_muscular_id is null;
```

The last query lists the exercises that couldn't be matched. These are exercises that had no group, had a typo in it, or were tagged *Pernas*, which is replaced by *Quadríceps*, *Posterior de coxa* and *Glúteos*. Give each one a group, either in the app's *Exercícios* tab, where they show "Sem grupo muscular", or in SQL. Until then they don't appear in the *Registrar* tab. When the query returns no rows, make the group required and drop the old text column:

```sql
alter table public.exercicios
  alter column grupo_muscular_id set not null,
  drop column grupo_muscular;
```

**Workout plans.** Creates the `fichas` and `ficha_exercicios` tables used by the *Fichas* tab. Existing data is not changed:

```sql
begin;

create table public.fichas (
  id         bigint generated always as identity primary key,
  user_id    uuid not null default auth.uid() references auth.users on delete cascade,
  nome       text not null,
  descricao  text,
  created_at timestamptz not null default now(),
  unique (user_id, nome)
);

create table public.ficha_exercicios (
  ficha_id     bigint not null references public.fichas on delete cascade,
  exercicio_id bigint not null references public.exercicios on delete cascade,
  ordem        int not null,
  primary key (ficha_id, exercicio_id)
);

alter table public.fichas           enable row level security;
alter table public.ficha_exercicios enable row level security;

create policy "own plans" on public.fichas
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "own plan exercises" on public.ficha_exercicios
  for all
  using (exists (select 1 from public.fichas f where f.id = ficha_id and f.user_id = auth.uid()))
  with check (
    exists (select 1 from public.fichas f where f.id = ficha_id and f.user_id = auth.uid())
    and exists (select 1 from public.exercicios e where e.id = exercicio_id and e.user_id = auth.uid())
  );

commit;
```

### 3. Configure the app

In your Supabase project, open **Connect** (or **Project Settings → API**) and copy the project URL and the **publishable** (or `anon`) key. Paste them at the top of the `<script type="module">` block in `index.html`:

```js
const SUPABASE_URL = 'https://your-project.supabase.co';
const SUPABASE_KEY = 'your-publishable-key';
```

The publishable key is designed to be exposed in client-side code. Your data is protected by the Row Level Security policies above, so make sure they are in place. Never put the `service_role` / secret key in this file.

### 4. Run it

Because the app uses ES modules, serve it over HTTP instead of opening the file directly. Any static server works, for example:

```bash
npx serve .
# or
python -m http.server 8000
```

Then open the printed address in your browser.

### Deploying

Since it's a single static file, you can host it on GitHub Pages, Netlify, Vercel, Cloudflare Pages or any other static host. If you use email confirmation, add your site's URL under **Authentication → URL Configuration** in Supabase so the confirmation link redirects correctly.

## Branching model

- `master` — stable, production-ready code.
- `develop` — integration branch for ongoing work. Create feature branches from `develop` and merge them back via pull request.
