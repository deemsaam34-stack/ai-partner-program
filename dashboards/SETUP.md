# Dashboard Setup — dmass.site/dashboards

Инструкция по настройке внутреннего дашборда финансов 1xaff. Всего 7 шагов, ориентировочно 30–45 минут с нуля (если Cloudflare-аккаунта ещё нет — +10 минут на регистрацию).

Статус файлов на текущий момент:
- `dashboards/index.html` — код дашборда, готов. Два плейсхолдера внутри:
  - `__GOOGLE_SHEETS_API_KEY__` — вставить свой API-ключ (шаг 3)
  - `__PASSWORD_HASH__` — вставить SHA-256 хеш пароля (шаг 4)
- `dashboards/SETUP.md` — этот файл

---

## Шаг 1 · Google Sheet: открыть на чтение по ссылке

По умолчанию реестр партнёров приватный. Чтобы API-ключ (анонимный) мог читать его, надо разрешить публичное чтение по ссылке. На сами данные это не влияет: в дашборд выведен только агрегат, а приватные данные защищены паролем на уровне страницы.

1. Открой реестр: https://docs.google.com/spreadsheets/d/19p4Pvm_R8L7imGhV3CtAL8kMXp-CJJLyvz5xm6MwGdI/edit
2. Правый верхний угол — синяя кнопка **Share**.
3. В появившемся окне — секция «General access», выпадающее меню «Restricted» — переключи на **«Anyone with the link»**.
4. Справа от этого меню — второе выпадающее меню с правами. Убедись, что стоит **«Viewer»** (не Editor).
5. Нажми **Done**.

Проверить, что сработало: открой ту же ссылку в режиме инкогнито (Ctrl+Shift+N). Должен видеть лист без логина.

⚠️ Это значит: любой, у кого есть ID листа, может прочитать данные. ID в коде страницы не секрет, он виден через View source. Пароль на дашборд — первый барьер; Cloudflare Access (шаг 6–7) — второй. Если озабочен — после настройки Cloudflare можно оставить Sheet приватным и подключить к дашборду сервисный аккаунт. Для v1 — достаточно этого.

---

## Шаг 2 · Google Cloud Console: создать API-ключ

1. Открой https://console.cloud.google.com/ — войди под тем же аккаунтом, где лежит Google Sheet (deemsaam34@gmail.com).
2. В верхней строке — селектор проекта. Если проекта нет — «New Project», имя `1xaff-dashboard`, Create. Если уже есть `1xaff` или похожий — выбери его.
3. Слева — навигационное меню («три полоски» ≡ в левом верхнем углу) → **APIs & Services** → **Library**.
4. В поиске набери **«Google Sheets API»** → открой результат → нажми **Enable**. Дождись «API enabled».
5. Вернись на **APIs & Services** → **Credentials**.
6. Сверху — **+ CREATE CREDENTIALS** → **API key**. Скопируй ключ (строка вида `AIzaSy...`), сохрани во временный блокнот.
7. Сразу же в появившемся окне нажми **«Edit API key»** (или позже открой этот ключ из списка Credentials).
8. В разделе **Application restrictions** выбери **«HTTP referrers (web sites)»**.
   - В поле «Website restrictions» добавь два значения:
     - `https://dmass.site/*`
     - `https://dmass.site` (без слэша, на всякий)
9. В разделе **API restrictions** выбери **«Restrict key»** → в выпадающем меню отметь только **«Google Sheets API»**.
10. Нажми **Save**.

Проверка: ключ теперь ограничен referrer'ом dmass.site. Если вставить его в любую чужую страницу — запрос отклонится с 403.

---

## Шаг 3 · Вставить API-ключ в dashboard.html

1. Открой `dashboards/index.html` в редакторе (VS Code / любом).
2. Найди строку (ищи по `__GOOGLE_SHEETS_API_KEY__`):
   ```js
   const GOOGLE_SHEETS_API_KEY = "__GOOGLE_SHEETS_API_KEY__";
   ```
3. Замени плейсхолдер на ключ из шага 2:
   ```js
   const GOOGLE_SHEETS_API_KEY = "AIzaSy...твой-ключ...";
   ```
4. Сохрани файл.

---

## Шаг 4 · Установить пароль доступа (SHA-256 хеш)

Пароль не хранится в коде напрямую — хранится его SHA-256 хеш. Зная хеш, пароль восстановить нельзя (в разумное время).

1. Открой в браузере любую страницу (например, google.com) — это нужно только для доступа к консоли браузера.
2. Нажми **F12** (Windows/Linux) или **Cmd+Option+I** (Mac) → вкладка **Console**.
3. Вставь следующую команду, заменив `МОЙ_ПАРОЛЬ` на выбранный пароль (любая строка, например `1xaff2026!` — желательно не короткая и не словарная):
   ```js
   crypto.subtle.digest('SHA-256', new TextEncoder().encode('МОЙ_ПАРОЛЬ')).then(b => console.log(Array.from(new Uint8Array(b)).map(x => x.toString(16).padStart(2,'0')).join('')))
   ```
4. Нажми **Enter**. В консоль выведется строка из 64 hex-символов, например:
   ```
   a7b2c4...d9f0e1
   ```
5. Скопируй эту строку.
6. Открой `dashboards/index.html`, найди строку:
   ```js
   const PASSWORD_HASH = "__PASSWORD_HASH__";
   ```
7. Замени плейсхолдер на свой хеш:
   ```js
   const PASSWORD_HASH = "a7b2c4...d9f0e1";
   ```
8. Сохрани файл.

Проверка: потом, когда страница задеплоится (шаг 7), откроешь её — появится окно с паролем; ввод правильного пароля → дашборд откроется.

⚠️ Никому не отправляй сам пароль по открытым каналам. Хеш в коде — не секрет, но пароль — секрет.

---

## Шаг 5 · Подключить домен dmass.site к Cloudflare

(Если аккаунт Cloudflare уже есть и домен уже в нём — пропустить.)

1. Зарегистрируйся на https://dash.cloudflare.com/sign-up (бесплатный план).
2. После входа — «Add a Site» → введи `dmass.site` → Continue.
3. Выбери **Free** план → Continue.
4. Cloudflare просканирует текущие DNS-записи домена. Проверь, что видны CNAME/A записи, указывающие на GitHub Pages (обычно `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153` — это IP-адреса GitHub Pages для корня домена, плюс CNAME для www на `deemsaam34-stack.github.io`). Если нет — добавь их по гайду GitHub Pages.
5. Cloudflare выдаст два nameserver, например:
   - `alice.ns.cloudflare.com`
   - `bob.ns.cloudflare.com`
6. Зайди в панель своего доменного регистратора (там, где покупал dmass.site — GoDaddy / Namecheap / REG.RU / и т.д.) → раздел «Nameservers» / «DNS» → замени текущие nameserver на два от Cloudflare → Save.
7. Подожди 5–60 минут (обычно 10). Cloudflare пришлёт email «Your site is active on Cloudflare».

---

## Шаг 6 · Cloudflare Access: ограничить /dashboards по email

1. В левом меню Cloudflare → **Zero Trust** (бесплатно до 50 пользователей).
2. Первый раз — создать Team name (любое, например `1xaff-team`) → Free plan → Continue.
3. В Zero Trust меню → **Access** → **Applications** → **Add an application**.
4. Тип: **Self-hosted**.
5. **Application Configuration**:
   - Application name: `1xaff Dashboard`
   - Session Duration: **24 hours**
   - Application domain:
     - Subdomain: (пусто)
     - Domain: `dmass.site`
     - Path: `dashboards`
6. Остальное по умолчанию → **Next**.
7. **Add policies**:
   - Policy name: `Allowed emails`
   - Action: **Allow**
   - Configure rules: Include → **Emails** → введи свои email-адреса через запятую (свой + доверенных коллег). Например:
     ```
     top1plint@gmail.com, deemsaam34@gmail.com
     ```
8. **Next** → **Add application**.

Теперь при открытии `dmass.site/dashboards/` Cloudflare покажет экран «Sign in with email» → придёт одноразовый код на указанный email → после подтверждения — пустит. Сессия 24 часа.

Базовая JS-защита (шаг 4) остаётся поверх — это двойной замок. При желании можно позже убрать JS-пароль (оставить только Cloudflare Access).

---

## Шаг 7 · Деплой (git push)

Из терминала VPS или локально:

```bash
cd /home/xaff/ai-partner-program
git add dashboards/
git commit -m "feat(dashboard): api key and password hash configured"
git push
```

GitHub Pages деплоит автоматически (master → push → через 1–3 минуты обновится `dmass.site`).

Проверка: открой https://dmass.site/dashboards/ в инкогнито:
1. Cloudflare Access попросит email (если шаг 6 выполнен) → введи свой → получи код на почту → введи код.
2. Появится окно «Доступ к дашборду — введите пароль» → введи пароль из шага 4 → нажми «Войти».
3. Дашборд загрузит данные из Google Sheet и отрисует KPI + таблицы + графики.

Если на любом этапе что-то не так — см. раздел «Диагностика».

---

## Диагностика

### «Ошибка: Google Sheets API: 403 …»

- API-ключ неправильный / не из того проекта.
- Или: ограничение referrer не соответствует домену (проверь, что в HTTP referrers стоит `https://dmass.site/*`).
- Или: Google Sheets API не активирован в том же проекте — вернись в Cloud Console → API Library → Google Sheets API → Enable.

### «Ошибка: Google Sheets API: 400 …»

- Лист не опубликован на чтение (шаг 1 не сделан): `Anyone with the link → Viewer`.
- Или имя листа изменилось с «Источник» на что-то другое — поправить константу `RANGE` в коде.

### «Неверный пароль» при вводе правильного пароля

- В `PASSWORD_HASH` в коде не тот хеш. Перегенерируй по шагу 4.
- Убедись, что хеш — 64 hex-символа (не меньше, не больше).

### «Ошибка: API key не установлен»

- Плейсхолдер `__GOOGLE_SHEETS_API_KEY__` не заменён. Шаг 3.

### Cloudflare Access не просит email

- Проверь, что в Application Configuration путь выставлен как `dashboards` (без начального слэша), домен `dmass.site`.
- Кэш Cloudflare — попробуй через инкогнито.
- Путь чувствителен: `/dashboards` и `/dashboards/` — одно и то же, если в настройках Application указан `dashboards`.

---

## Ориентировочное время на ручные шаги

| Шаг | Действие | Время |
|---:|---|---:|
| 1 | Google Sheet Share → Anyone with link | 1 мин |
| 2 | Google Cloud Console: API key + restriction | 10 мин (первый раз) / 5 мин (если проект уже есть) |
| 3 | Вставить API-key в код | 1 мин |
| 4 | Сгенерировать password-hash, вставить | 3 мин |
| 5 | Подключить домен к Cloudflare (если ещё нет) | 10–60 мин ожидания DNS |
| 6 | Cloudflare Access → application + email policy | 5 мин |
| 7 | git push, проверить в инкогнито | 3 мин |

Итого: **30–45 минут активной работы** + DNS-пропагация Cloudflare (если домен ещё не в нём).

---

## Обновление дашборда в будущем

Данные подтягиваются из Google Sheet **на каждый reload страницы**. То есть:
- Любое изменение в реестре (ручное или через `enrich_registry_with_metrics.py`) отразится на дашборде при следующем открытии.
- Ничего пересчитывать и деплоить не нужно.

Если меняется **структура** реестра (например, добавляется новая колонка, нужная дашборду) — редактировать `COL` в JS и `RANGE` если вышли за пределы A:AM.

Если меняется **дизайн/логика** дашборда — редактировать `index.html` → push → автодеплой 1–3 минуты.
