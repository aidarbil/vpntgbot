• Updated Plan
  └ ✔ Guard server assignment and subscription flow against missing servers or clients
    ✔ Harden inbound lookup and limit handling in VPN service
    ✔ Ensure payment handlers gracefully handle missing/duplicate transactions

  - app/bot/services/server_pool.py:63 теперь корректно обрабатывает отсутствие inbound’ов и выдаёт понятное
  сообщение; при назначении сервера пользователю добавлена проверка на пустой пул с безопасным возвратом и
  логированием.
  - app/bot/services/vpn.py:147 учитывает ошибки назначения сервера и отсутствие inbound ID; при обновлении
  клиента добавлены проверки лимита устройств, чтобы избежать TypeError.
  - app/bot/routers/subscription/subscription_handler.py:84 усилил логику продления: если клиент или лимит
  устройств недоступны, пользователь получает уведомление без падения обработчика.
  - app/bot/payment_gateways/_gateway.py:70 и app/bot/routers/subscription/payment_handler.py:104 устойчиво
  обрабатывают повторные/отсутствующие транзакции и дубли вебхуков; при пропаже юзера разработчик получает
  уведомление, а бот не падает.

  Дальше

  1. Прогнать ключевые сценарии (создание/продление подписки, отмена платежа) на тестовом стенде, чтобы
  убедиться, что все ветки логирования выполняются как ожидается.

▌ Прогони

> Автоматических тестов для этих сценариев в проекте нет, а интеграционные проверки требуют реально работающих
  3X‑UI, Redis и платёжных шлюзов — в текущей среде их нет. Если дадите тестовое окружение (URL панели, Redis,
  тестовые ключи платёжек), могу расписать точные команды и последовательность ручной проверки, чтобы вы
  прогнали их у себя.

▌ Я купил сервер, что мне теперь делать и как его подключить в нашего бота чтобы он уже выдавал доступы. пока
▌ без оплаты, можно сделать временную заглушку. Напиши план действий

• I’m laying out a step-by-step plan to help the user connect a new server to their bot without payment
  required initially. This includes preparing the server environment, installing the 3x-ui panel with SSL,
  configuring network access, setting environment variables to disable payments, and adding the server through
  the bot’s admin panel with a trial or manual account issuance option. The goal is to give clear and practical
  instructions for a smooth temporary setup.

> Готовим сервер

  - OS и доступ: обновите систему (apt update && apt upgrade), создайте отдельного пользователя, настройте SSH-
  keys, закройте root‑логин, включите брандмауэр (разрешите 22/80/443/2096 и порт панели 3X‑UI).
  - 3X‑UI: установите панель (скрипт из их репо), зайдите в веб‑интерфейс, включите HTTPS (Let’s Encrypt/свой
  сертификат), создайте один inbound (тот, что будет в списке первым), включите Subscription Service на 2096 со
  шляхом /user/, отключите шифрование конфигов — бот должен получать реальные UUID и expiry.

  Домен и сертификат

  - Пропишите DNS для домена панели (например, panel.example.com) и домена бота (например, bot.example.com) на
  IP сервера.
  - В 3X‑UI укажите FQDN для панели и subscription‑сервиса, убедитесь, что сертификаты обновлены.

  Готовим бота

  - На хосте бота (можно тот же сервер) клонируйте проект, создайте plans.json из plans.example.json, задайте
  цены/длительности.
  - Скопируйте .env.example → .env и заполните: BOT_TOKEN, BOT_DEV_ID, BOT_SUPPORT_ID,
  BOT_DOMAIN=bot.example.com, включите только нужные платежи. Для «заглушки» поставьте
  SHOP_PAYMENT_STARS_ENABLED=False и оставьте остальные False, чтобы бот предлагал только бесплатный триал/
  промокоды. Временно можно выставить SHOP_TRIAL_ENABLED=True и подготовить промокоды для ручной выдачи.
  - Укажите доступ к панели: XUI_USERNAME, XUI_PASSWORD, XUI_SUBSCRIPTION_PORT=2096, XUI_SUBSCRIPTION_PATH=/
  user/. Если включили токен в 3X‑UI — пропишите XUI_TOKEN.
  - Настройте Redis (либо внешний, либо локальный контейнер) и добавьте его параметры в .env (REDIS_HOST,
  REDIS_PASSWORD, если требуется).

  Запуск

  - Соберите и поднимите окружение: docker compose build, затем docker compose up -d. Проверьте docker compose
  logs -f bot, убедитесь, что Alembic миграции прошли, и webhook выставлен.
  - Oткройте в браузере https://bot.example.com/webhook – должен отвечать 200/OK (Traefik), значит Telegram
  видит хук.

  Привязка сервера

  - В Telegram войдите в бота как разработчик (указанный BOT_DEV_ID), откройте админ-панель → «Server Manager» →
  «Добавить сервер».
  - Введите название, host = https://panel.example.com (или IP с https), логин/пароль 3X‑UI, максимальное
  количество клиентов. После добавления нажмите «Проверить» или «Синхронизировать», в логах увидите Server X
  added to pool successfully.

  Выдача доступов без оплаты

  - Включите в .env пробный период (SHOP_TRIAL_ENABLED=True) и укажите длительность (SHOP_TRIAL_PERIOD), тогда
  пользователи смогут получить доступ через кнопку «Попробовать бесплатно».
  - Для ручной выдачи доступа создайте промокоды (админ меню → Promocode editor) с нужным сроком; бот активирует
  их через 3X‑UI.
  - Если нужно ограничить трафик/устройства, проверьте план в plans.json (атрибут devices и total_gb, если
  используете).

  Проверка

  - Сначала сами пройдите сценарий: старт бота → получить триал → бот создаёт клиента в 3X‑UI → проверьте, что в
  панели появился пользователь и ссылка (/user/<uuid>) работает.
  - Убедитесь, что бот корректно удаляет/продлевает подписку (кнопки «Продлить», «Изменить») и что при удалении
  клиента на стороне 3X‑UI бот выдаёт понятную ошибку (на случай ручных манипуляций).

  Дальше

  - Как только решите включить оплату, активируйте нужный шлюз (SHOP_PAYMENT_*_ENABLED=True), добавьте ключи и
  настройте вебхуки на https://bot.example.com/<gateway>.
  - Настройте резервное копирование БД (кнопка в админке) и мониторинг логов, чтобы отслеживать ошибки.

▌ Implem