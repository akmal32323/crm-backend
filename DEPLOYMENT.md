# Развёртывание CRM в продакшн — пошаговый гайд

## Упрощённый путь (рекомендуется): Render Blueprint

В репозитории `crm-backend` уже лежит файл `render.yaml` — он сам описывает, что нужно развернуть:
базу PostgreSQL и backend вместе, одной операцией, без ручного клика по десяткам кнопок.

1. Залить `crm-backend` в репозиторий на GitHub (если ещё не сделано)
2. Зарегистрироваться на [render.com](https://render.com) (можно через GitHub)
3. Dashboard → **New +** → **Blueprint**
4. Выбрать репозиторий с `crm-backend` — Render сам найдёт `render.yaml` и покажет план: 1 база данных + 1 web-сервис
5. Нажать **Apply** — Render создаст всё сам, база и backend окажутся уже связаны (переменная `DATABASE_URL` подставится автоматически)
6. После первого деплоя останется вписать вручную только 3 секретных переменные, которые Render специально оставляет пустыми (в интерфейсе сервиса → **Environment**):
   - `ANTHROPIC_API_KEY`
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_BOT_USERNAME`
7. Сохранить — Render передеплоит сервис с новыми переменными автоматически

Таблицы в БД создадутся сами при каждом деплое (это прописано в `render.yaml`), отдельно запускать `npm run db:sync` не нужно.

**Важно про бесплатный тариф Render**: план `starter` в файле — платный (нужен, потому что Telegram-бот и уведомления должны работать постоянно, а бесплатный тариф "усыпляет" сервис после простоя). Если хочется сначала просто проверить, что всё работает, можно временно поменять `plan: starter` на `plan: free` в `render.yaml` — но тогда бот будет отваливаться при простое.

Если этот путь сработал — остальные шаги гайда (Neon, Railway) не нужны, переходи сразу к **Шагу 5 (Frontend на Vercel)** ниже.

---

## Альтернативный путь (более гибкий, но больше ручных шагов): Neon + Railway

## Шаг 1. Ключи и токены

1. **Anthropic API-ключ**
   console.anthropic.com → Settings → API Keys → Create Key
   Сохраните значение — оно показывается один раз.

2. **Telegram-бот**
   В Telegram: [@BotFather](https://t.me/BotFather) → `/newbot` → имя → username (должен заканчиваться на `bot`)
   BotFather выдаст токен вида `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`.
   Также запомните username бота без `@` (например `agency_crm_bot`) — он понадобится для `TELEGRAM_BOT_USERNAME`.

---

## Шаг 2. База данных на Neon

1. Зарегистрироваться на [neon.tech](https://neon.tech) (без карты)
2. Create Project → выбрать регион (ближе к тому, где будет backend)
3. В Dashboard → Connection Details скопировать connection string вида:
   `postgresql://user:password@ep-xxxx.region.aws.neon.tech/dbname?sslmode=require`
4. Разбить эту строку на составляющие для `.env`:
   - `DB_HOST` = `ep-xxxx.region.aws.neon.tech`
   - `DB_PORT` = `5432`
   - `DB_NAME` = `dbname`
   - `DB_USER` = `user`
   - `DB_PASSWORD` = `password`

   Либо (проще) — заменить в `src/config/db.js` инициализацию Sequelize на вариант с одной строкой:
   ```js
   const sequelize = new Sequelize(process.env.DATABASE_URL, {
     dialect: 'postgres',
     dialectOptions: { ssl: { require: true, rejectUnauthorized: false } },
     logging: false,
   });
   ```
   и в `.env` использовать один `DATABASE_URL` вместо пяти переменных. Neon требует SSL — без `dialectOptions.ssl` подключение не пройдёт.

---

## Шаг 3. Backend + бот на Railway

1. Залить `crm-backend` в отдельный репозиторий на GitHub
2. [railway.app](https://railway.app) → New Project → Deploy from GitHub repo → выбрать репозиторий
3. В Settings сервиса → Variables добавить все переменные из `.env.example`:
   - `PORT` — Railway сам подставит свой, можно оставить 4000 как дефолт в коде
   - `DATABASE_URL` (или пять DB_* переменных) — из шага 2
   - `JWT_SECRET` — любая длинная случайная строка
   - `ANTHROPIC_API_KEY` — из шага 1
   - `ANTHROPIC_MODEL` — `claude-sonnet-4-6`
   - `TELEGRAM_BOT_TOKEN`, `TELEGRAM_BOT_USERNAME` — из шага 1
4. Railway автоматически определит Node.js-проект и задеплоит
5. После первого деплоя выполнить синхронизацию таблиц: в Railway есть вкладка "Shell" на сервисе — запустить `npm run db:sync` один раз (создаст все таблицы по моделям)
6. Скопировать публичный URL сервиса (Settings → Networking → Generate Domain) — он понадобится фронтенду

---

## Шаг 4. Первый сотрудник (руководитель)

Таблицы пустые, войти некому. Проще всего — через Railway Shell выполнить одноразовый скрипт:

```js
// добавьте временно в src/config/, запустите один раз через Railway Shell: node src/config/seedAdmin.js
require('dotenv').config();
const bcrypt = require('bcrypt');
const { sequelize, User } = require('../models');

async function seed() {
  const passwordHash = await bcrypt.hash('ВАШ_ПАРОЛЬ', 10);
  await User.create({
    fullName: 'Имя Руководителя',
    email: 'you@agency.ru',
    passwordHash,
    role: 'admin',
  });
  console.log('Руководитель создан');
  process.exit(0);
}
seed();
```

После первого входа руководитель добавляет остальных сотрудников уже через админку → раздел «Сотрудники».

---

## Шаг 5. Frontend на Vercel

1. Залить `crm-frontend` в отдельный репозиторий на GitHub
2. [vercel.com](https://vercel.com) → Add New Project → импортировать репозиторий
3. Environment Variables → добавить `VITE_API_URL` = URL backend'а из шага 3 + `/api` (например `https://crm-backend-production.up.railway.app/api`)
4. Deploy — Vercel сам определит Vite-проект

---

## Проверка

1. Открыть Vercel-ссылку → `/login` → войти созданным в шаге 4 руководителем
2. Создать тестового клиента → в карточке нажать «Получить ссылку» → перейти по ней в Telegram → должно прийти подтверждение от бота
3. Создать проект с датой окончания через 1–3 дня → должно прийти мгновенное уведомление в тот же Telegram-чат
4. Написать боту в Telegram что-то из базы знаний агентства (если она заполнена) — должен ответить через Claude

---

## Частые проблемы

- **Бот не отвечает** — проверьте `TELEGRAM_BOT_TOKEN` в переменных Railway, посмотрите логи сервиса
- **Ошибка подключения к БД (SSL)** — Neon требует SSL, см. шаг 2 про `dialectOptions`
- **Frontend получает 401/CORS-ошибку** — проверьте, что `VITE_API_URL` указывает на правильный домен backend'а и что там есть `/api` в конце
- **Уведомления о сроках не приходят** — cron сработает только в 10:00 по времени сервера Railway (обычно UTC); для проверки среагирует и мгновенное уведомление при создании/изменении проекта со сроком
