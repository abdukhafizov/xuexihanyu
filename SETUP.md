# Настройка Supabase (аккаунты + облачное хранение)

Приложение умеет работать двумя способами:

- **Без аккаунта (гостевой режим)** — всё как раньше, данные хранятся в `localStorage` браузера. Ничего настраивать не нужно, просто открой `index.html`.
- **С аккаунтом** — уроки, слова и прогресс хранятся в облаке (Supabase Postgres) и доступны с любого устройства. Для этого нужно один раз создать бесплатный проект в Supabase и вписать пару значений в файл `config.js`.

Пока `config.js` не заполнен, приложение автоматически работает в гостевом режиме — экран входа даже не показывается. Это безопасный дефолт: ничего не сломается, пока ты не пройдёшь шаги ниже.

Всё бесплатно (free tier Supabase: 500 МБ базы, 50 000 monthly active users — для личного проекта с запасом).

---

## 1. Создать проект в Supabase

1. Зайди на [supabase.com](https://supabase.com) и зарегистрируйся (можно через GitHub).
2. На дашборде нажми **New project**.
3. Заполни:
   - **Name** — например `xuexihanyu`.
   - **Database Password** — придумай и сохрани где-нибудь (пароль от самой базы, в приложении не используется напрямую).
   - **Region** — ближайший к тебе регион.
4. Нажми **Create new project** и подожди пару минут, пока Supabase поднимет базу.

## 2. Взять URL и anon key

1. В проекте открой **Settings → API** (иконка шестерёнки в левом меню, затем вкладка API).
2. Скопируй два значения:
   - **Project URL** — выглядит как `https://xxxxxxxxxxxx.supabase.co`
   - **anon public** ключ (в разделе Project API keys) — длинная строка, начинается с `eyJ...`

   > Это публичный ключ — его можно смело хранить в коде фронтенда и коммитить в git. Он не даёт прямого доступа к данным: реальная защита обеспечивается политиками Row Level Security (RLS), которые мы включим в шаге 4. Секретным является только **service_role** ключ — его в приложении мы вообще не используем, никогда никуда не вставляй его.

## 3. Создать таблицы (SQL Editor)

1. В левом меню открой **SQL Editor** → **New query**.
2. Вставь целиком следующий скрипт и нажми **Run**:

```sql
create extension if not exists pgcrypto;

create table public.lessons (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  position int not null default 0,
  created_at timestamptz not null default now()
);

create table public.words (
  id uuid primary key default gen_random_uuid(),
  lesson_id uuid not null references public.lessons(id) on delete cascade,
  hanzi text not null,
  pinyin text,
  translation text,
  created_at timestamptz not null default now()
);

create table public.word_progress (
  id uuid primary key default gen_random_uuid(),
  word_id uuid not null references public.words(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  interval int not null default 0,
  ease_factor numeric not null default 2.5,
  repetitions int not null default 0,
  next_review_date date,
  status text,
  updated_at timestamptz not null default now(),
  unique (word_id, user_id)
);

create table public.daily_stats (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  date date not null,
  cards_reviewed_count int not null default 0,
  unique (user_id, date)
);

create index words_lesson_id_idx on public.words(lesson_id);
create index word_progress_word_user_idx on public.word_progress(word_id, user_id);
create index daily_stats_user_date_idx on public.daily_stats(user_id, date);
```

Должно появиться сообщение `Success. No rows returned`.

> **Уже создавал таблицы раньше и заводишь `position` только сейчас?** Выполни отдельным запросом в SQL Editor:
> ```sql
> alter table public.lessons add column if not exists position int not null default 0;
> ```
> Это добавит колонку для ручного порядка уроков (стрелки «переместить выше/ниже» в приложении), не трогая уже сохранённые уроки.

## 4. Включить Row Level Security (RLS)

Там же, в SQL Editor, выполни новым запросом:

```sql
alter table public.lessons enable row level security;
alter table public.words enable row level security;
alter table public.word_progress enable row level security;
alter table public.daily_stats enable row level security;

create policy "own lessons" on public.lessons
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());

create policy "own words" on public.words
  for all using (exists (select 1 from public.lessons l where l.id = words.lesson_id and l.user_id = auth.uid()))
  with check (exists (select 1 from public.lessons l where l.id = words.lesson_id and l.user_id = auth.uid()));

create policy "own progress" on public.word_progress
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());

create policy "own stats" on public.daily_stats
  for all using (user_id = auth.uid()) with check (user_id = auth.uid());
```

Это гарантирует: каждый пользователь видит и может менять **только свои** уроки, слова, прогресс и статистику — Postgres проверяет это на уровне базы данных, а не полагается на код фронтенда.

Проверить: **Authentication → Policies** — там должны появиться политики для всех четырёх таблиц.

## 5. Включить вход по email

1. **Authentication → Providers**.
2. Убедись, что **Email** включён (обычно включён по умолчанию).
3. (Опционально) **Authentication → Settings** → если не хочешь, чтобы пользователи подтверждали почту письмом перед входом, можно выключить **Confirm email** — тогда регистрация сразу пускает в приложение. По умолчанию Supabase отправит письмо со ссылкой подтверждения.

## 6. Включить вход через Google (опционально, но приложение его поддерживает)

Это отдельный сервис (Google Cloud Console), шагов немного больше:

1. Зайди в [Google Cloud Console](https://console.cloud.google.com/) → создай новый проект (или используй существующий).
2. **APIs & Services → OAuth consent screen** — заполни минимум (название приложения, email), тип **External**, сохрани.
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID**:
   - Application type: **Web application**.
   - **Authorized redirect URIs** — добавь адрес вида:
     ```
     https://xxxxxxxxxxxx.supabase.co/auth/v1/callback
     ```
     (тот же домен, что и Project URL из шага 2, плюс `/auth/v1/callback` — точный адрес также показан в Supabase на шаге ниже).
4. Нажми **Create** — получишь **Client ID** и **Client Secret**.
5. Вернись в Supabase: **Authentication → Providers → Google** → включи тумблер, вставь Client ID и Client Secret из шага 4 → **Save**.

Если пропустить этот раздел — кнопка «Войти через Google» в приложении просто покажет ошибку от Supabase при клике. Email/пароль при этом работает независимо и не требует Google.

## 7. Вписать ключи в приложение

Открой файл [`config.js`](config.js) в корне проекта и замени плейсхолдеры значениями из шага 2:

```js
window.APP_CONFIG = {
  SUPABASE_URL: "https://xxxxxxxxxxxx.supabase.co",
  SUPABASE_ANON_KEY: "eyJ...твой ключ..."
};
```

Сохрани файл и обнови страницу — на месте автоматического гостевого режима появится экран входа/регистрации.

## Как это работает дальше

- **Гость → аккаунт**: если у тебя уже были уроки в гостевом режиме и ты потом зарегистрируешься — при первом входе приложение предложит перенести локальные уроки в аккаунт (можно отказаться, тогда они просто останутся только в этом браузере).
- **Настройки устройства** (тема, дневная цель, тумблеры вроде «Цвета тонов») специально остаются в `localStorage` даже при входе в аккаунт — это личные предпочтения конкретного браузера/устройства, не часть учебных данных.
- Если пропадёт интернет во время работы с аккаунтом — приложение покажет баннер об этом; все изменения, сделанные офлайн, не сохранятся, пока соединение не восстановится.

## Если что-то не работает

- **«Неверный email или пароль» при регистрации нового аккаунта** — обычно значит, что email уже зарегистрирован; попробуй войти вместо регистрации.
- **Ничего не происходит после регистрации** — проверь, не включено ли подтверждение почты (шаг 5); письмо может попасть в спам.
- **Google-кнопка выдаёт ошибку** — почти всегда несовпадение redirect URI (шаг 6.3) — он должен быть **точно** таким, как в Supabase (Authentication → Providers → Google, там показан правильный адрес).
- **Данные не сохраняются** — открой консоль браузера (F12), там будут видны сетевые ошибки Supabase (неверный URL/ключ или не выполнены шаги 3–4).
