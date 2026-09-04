# qa_claude

Инструменты QA для Claude Code: MCP-серверы Qase и Asana плюс скилл декомпозиции ТЗ.

## Скиллы

**`/qa-decomposition`** — [.claude/skills/qa-decomposition/](.claude/skills/qa-decomposition/)

Разбивает ТЗ фичи на QA-задачи и заводит их подзадачами в Asana. Порядок: найти ТЗ → собрать декомпозицию в `docs/decomposition-<фича>.md` → уточнить неприменимые обязательные блоки → ревью → правки → создание подзадач по присланной ссылке. В Asana ничего не создаётся до явного согласования.

Внутри скилла:
- [SKILL.md](.claude/skills/qa-decomposition/SKILL.md) — порядок работы и правила нарезки ТЗ на блоки
- [reference/template-blocks.md](.claude/skills/qa-decomposition/reference/template-blocks.md) — 38 обязательных разделов из шаблона «(ХХХ) Шаблон. Декомпозиция фичи со стороны QA»
- [reference/format.md](.claude/skills/qa-decomposition/reference/format.md) — формат имени и тела задачи
- [reference/decomposition-template-asana.md](.claude/skills/qa-decomposition/reference/decomposition-template-asana.md) — сырая выгрузка шаблона
- [reference/decomposition-example-idl.md](.claude/skills/qa-decomposition/reference/decomposition-example-idl.md) — готовая декомпозиция IDL на 46 задач как образец

## MCP

Настроены серверы **Qase** и **Asana** ([.mcp.json](.mcp.json)) — Claude Code подключает их автоматически, отдельный `claude mcp add` не нужен. Требуется только один раз положить свои токены в переменные окружения.

Требования: **Node.js 20+**.

---

## Шаг 1. Получить токены

### Qase

1. Откройте [app.qase.io/user/api/token](https://app.qase.io/user/api/token) — или кликните по своей аватарке в правом верхнем углу и выберите **API Tokens**.
2. Нажмите **Create a new API token**.
3. Задайте название (например, `claude-code`) и нажмите **Create**.
4. **Скопируйте токен сразу** — он показывается только один раз. Если потеряли, придётся создавать новый.

Токен выглядит как строка из 64 hex-символов: `aaedf838...eba4`

Полезно знать:
- Токен наследует **ваши права** в воркспейсе. Если ваша роль не даёт создавать тест-раны, через API вы их тоже не создадите.
- Лимит — 600 запросов в минуту, дальше HTTP 429.
- Отозвать токен можно там же, через меню «...» → **Revoke**.
- Для CI лучше использовать **App Token** (Workspace settings → Apps): действия помечаются именем приложения, а не вашим, и права урезаны до минимума.

### Asana

1. Откройте [app.asana.com/0/my-apps](https://app.asana.com/0/my-apps) — консоль разработчика.
2. Создайте **Personal access token**, укажите описание (обязательно).
3. Скопируйте токен.

Формат PAT: `2/<user_gid>/<token_gid>:<секрет>`

---

## Шаг 2. Записать токены в окружение

Claude Code подставляет в `.mcp.json` значения из переменных окружения того терминала, в котором он запущен. Поэтому токены прописываются в конфиг шелла, а не в файлы проекта.

**macOS / Linux (zsh — по умолчанию на macOS):**

```bash
echo 'export QASE_API_TOKEN="сюда_токен_qase"' >> ~/.zshrc
echo 'export ASANA_ACCESS_TOKEN="сюда_токен_asana"' >> ~/.zshrc
source ~/.zshrc
```

Для bash — то же самое, но в `~/.bashrc` (Linux) или `~/.bash_profile` (macOS).

**Windows (PowerShell), разово для пользователя:**

```powershell
[Environment]::SetEnvironmentVariable("QASE_API_TOKEN", "сюда_токен_qase", "User")
[Environment]::SetEnvironmentVariable("ASANA_ACCESS_TOKEN", "сюда_токен_asana", "User")
```

Затем перезапустите терминал.

Проверить, что переменные видны:

```bash
echo $QASE_API_TOKEN
```

Если пусто — токен не подхватился, `source` не выполнен или строка ушла не в тот файл.

> **Важно:** после изменения `~/.zshrc` нужно перезапустить сам `claude` — уже запущенный процесс читает окружение только при старте.

---

## Шаг 3. Подключиться

1. Запустите `claude` в папке проекта.
2. Примите диалог доверия к папке — без него настройка `enableAllProjectMcpServers` из [.claude/settings.json](.claude/settings.json) игнорируется, и серверы останутся в статусе `⏸ Pending approval`.
3. Выполните `/mcp` — оба сервера должны показывать **connected**.

Проверить токен Qase напрямую, не поднимая MCP:

```bash
curl -H "Token: $QASE_API_TOKEN" https://api.qase.io/v1/project
```

Ответ `{"status":true,...}` — токен рабочий. `401` — токен неверный или отозван.

---

## Безопасность

- Токены **не хранятся в репозитории** — в `.mcp.json` стоят плейсхолдеры `${QASE_API_TOKEN}` и `${ASANA_ACCESS_TOKEN}`.
- Не вписывайте токен прямо в `.mcp.json`: файл под контролем версий, и секрет уедет в историю первым же `git commit -a`.
- Если токен где-то засветился — сразу отзовите его и создайте новый.
- VS Code может подсвечивать `${QASE_API_TOKEN}` предупреждением «Variable not found» — это ложная тревога, редактор пытается разобрать синтаксис своих переменных. Claude Code подставляет значение сам.

## Что дают серверы

| Сервер | Возможности |
|---|---|
| `qase` | Тест-кейсы, сюиты, раны, результаты, дефекты, планы, майлстоуны, поиск через QQL |
| `asana` | Задачи, проекты, секции, комментарии, подзадачи, зависимости, поиск |

Источники: [Qase — Token and Project code](https://developers.qase.io/docs/api-token-and-project-code), [Understanding Tokens in Qase](https://docs.qase.io/en/articles/8586564-understanding-tokens-in-qase), [Asana — Personal access token](https://developers.asana.com/docs/personal-access-token)
