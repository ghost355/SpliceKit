# SpliceKit (форк) — установка на другой машине

Короткая инструкция для агента. Всё проверено на macOS 26.7.3 / FCP 12.4 (build 454072) / Xcode 26.6.

## 1. Что нужно до начала

- macOS 14+, Xcode Command Line Tools (`xcode-select -p` должен что-то вернуть).
- Final Cut Pro, установленный из App Store (любая редакция: `Final Cut Pro.app` или `Final Cut Pro Creator Studio.app`).
- ~10 ГБ свободного места (копия FCP — 6.5 ГБ + сборка).
- Python 3.11–3.13 (`python3.12` или `python3.13` в PATH).

## 2. Клон

```bash
cd ~/Projects
git clone https://github.com/ghost355/SpliceKit.git
cd SpliceKit
```

Если форка нет — клонировать upstream и накатить наши коммиты:

```bash
git clone https://github.com/elliotttate/SpliceKit.git
git remote add fork https://github.com/ghost355/SpliceKit.git
git fetch fork && git checkout -b main fork/main
```

Наши коммиты поверх `v3.3.9`: `.mcp.json` с локальными путями, Sentry в патчере, сборка через `make all`, заглушки BRAW без SDK, переподпись после правки Info.plist, непрозрачная Command Palette.

## 3. Сборка (без FCP)

```bash
make all
```

- Сам скачает `Sentry.framework` в `patcher/Frameworks/` (нужен интернет).
- Соберёт `build/lua/liblua.a` и `build/SpliceKit` (universal, arm64 + x86_64).
- Проверит, что нет неразрешённых `SpliceKit_*` — если есть, значит файла нет в `Sources/SOURCES.txt`.

Без Blackmagic RAW SDK сборка работает: BRAW выключен, в `SpliceKitBRAW.mm` срабатывает ветка `#else` с заглушками. Для BRAW поставить SDK в `/Applications/Blackmagic RAW/` и пересобрать.

## 4. Патч FCP

**FCP должен быть закрыт.**

```bash
./patcher/patch_fcp.sh
```

Что делает: копирует установленный FCP в `~/Applications/SpliceKit/`, ставит туда `SpliceKit.framework`, инжектит `LC_LOAD_DYLIB`, переподписывает с entitlements без sandbox, правит `Info.plist`, пишет `.mcp.json` в текущую директорию. Оригинал в `/Applications` не трогается.

Если FCP лежит не по стандартному пути:

```bash
./patcher/patch_fcp.sh --source "/Applications/Мой FCP.app"
```

Передеплой после правок кода (без повторного копирования 6.5 ГБ):

```bash
./patcher/patch_fcp.sh --no-copy --rebuild
```

Удалить всё: `./patcher/patch_fcp.sh --uninstall`.

## 5. MCP

```bash
make mcp-setup     # venv ~/.venvs/splicekit-mcp + mcp 1.30.0
make mcp-doctor    # 3 проверки ok, четвёртая ждёт запущенный FCP
```

Проверить `.mcp.json` в корне репозитория — пути должны быть абсолютные и **локальные**:

```json
{
  "mcpServers": {
    "splicekit": {
      "command": "/Users/<user>/.venvs/splicekit-mcp/bin/python",
      "args": ["/Users/<user>/Projects/SpliceKit/mcp/server.py"]
    }
  }
}
```

Патчер перезаписывает этот файл при каждом запуске — после патча перепроверить.

## 6. Запуск и проверка

```bash
open "$HOME/Applications/SpliceKit/Final Cut Pro.app"
```

Через 15–20 секунд мост должен слушать порт:

```bash
lsof -nP -iTCP:9876 -sTCP:LISTEN
printf '%s\n' '{"jsonrpc":"2.0","method":"system.version","id":1}' | nc -w 5 127.0.0.1 9876
```

В ответе — `fcp_version`, `splicekit_version`, `pid`. Если порт молчит — смотреть `~/Library/Logs/SpliceKit/splicekit.log`.

Смоук-тест на живом проекте:

```bash
# открыть библиотеку
open -a "$HOME/Applications/SpliceKit/Final Cut Pro.app" ~/Movies/Моя.fcpbundle
# список проектов (заведомо неверное имя вернёт список доступных)
printf '%s\n' '{"jsonrpc":"2.0","method":"project.open","params":{"name":"__none__"},"id":2}' | nc -w 8 127.0.0.1 9876
# открыть проект и прочитать таймлайн
printf '%s\n' '{"jsonrpc":"2.0","method":"project.open","params":{"name":"Eda"},"id":3}' | nc -w 10 127.0.0.1 9876
printf '%s\n' '{"jsonrpc":"2.0","method":"timeline.getDetailedState","id":4}' | nc -w 10 127.0.0.1 9876
# скриншоты
printf '%s\n' '{"jsonrpc":"2.0","method":"timeline.capture","params":{"path":"/tmp/tl.png"},"id":5}' | nc -w 10 127.0.0.1 9876
```

## 7. Что важно знать

- Библиотеки и медиа общие для обеих копий. **Одну библиотеку нельзя держать открытой в двух FCP одновременно.**
- Контент Apple с серверов в пропатченной копии недоступен: `CloudContentCatalog`/`CloudContentFeatureFlag` заменены, иначе CloudKit роняет приложение без энтайтлментов. Уже установленные титры/эффекты с диска видны.
- Патчер пишет в общий домен `com.apple.FinalCut`: `FFCloudContentDisabled = 1`. Это действует и на стоковый FCP.
- BRAW в сборке без SDK выключен; VP9/MKV работают.
- Обновился FCP — копию надо перепатчить (`--no-copy` не подойдёт, нужен полный прогон с новым `--source`).
- Внутри FCP: `Cmd+Shift+P` — Command Palette; три кнопки в тулбаре (LiveCam, Palette, Transcript).

## 8. Если что-то сломалось

| Симптом | Что делать |
|---|---|
| `'Sentry/Sentry.h' file not found` | `bash Scripts/ensure_sentry_framework.sh`, затем `make all` |
| `undefined SpliceKit_* symbols` | файла нет в `Sources/SOURCES.txt`; добавить и пересобрать |
| `.mm` компилируется как C (`cmath` not found) | собирать только через `make all`, не одним вызовом clang |
| `invalid Info.plist (plist or signature…)` | переподписать после правок plist: `./patcher/patch_fcp.sh --no-copy --rebuild` |
| FCP стартует и сразу закрывается | `~/Library/Logs/SpliceKit/splicekit.log` + `~/Library/Logs/DiagnosticReports/Final Cut Pro-*.ips` |
| Порт 9876 не слушается | мост стартует через ~15 с после запуска; проверить, что запущена именно пропатченная копия |
| MCP: «`mcp` package not installed» | клиент запускает системный python вместо `~/.venvs/splicekit-mcp/bin/python` |

## 9. Цикл разработки

```bash
make all
./patcher/patch_fcp.sh --no-copy --rebuild   # FCP должен быть закрыт
open "$HOME/Applications/SpliceKit/Final Cut Pro.app"
```

Логи: `~/Library/Logs/SpliceKit/splicekit.log`. MCP-сервер: `mcp/server.py` (~205 тулзов). Мост: 83 метода, `bridge.describe` вернёт список с классификацией безопасности.
