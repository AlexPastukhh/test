# ChatGPT ↔ Windows Bridge — протокол v0.2-test

**Дата:** 2026-10-08. **Область применения:** только синтетический тест `AlexPastukhh/test`. Это рабочий прототип, **не** универсальный безопасный транспорт для реальных репозиториев.

## 1. Цель и границы

Отвязать число чтений/правок больших файлов от количества вызовов Tunnel. Через Secure MCP Tunnel делаются пакетные операции; основная работа происходит с настоящей копией файлов внутри ChatGPT. Нет лимита «строго два вызова» — дополнительные статусные/проверочные обращения допускаются.

```
HOST (Bridge.ps1 snapshot -Repo test)
  → public synthetic GitHub repo / Actions (Artifact)
  → GitHub plugin download_workflow_artifact(artifact_id)
  → ChatGPT filesystem: unpack + check manifest + heavy work
  → patch generated in ChatGPT
  → HOST (Bridge.ps1 apply ... -CheckOnly / apply)
```

## 2. Установленные компоненты

- Windows: `C:\Users\alexa\chatgpt-bridge\Bridge.ps1` — PowerShell frontend.
- `bridge.py` — Python stdlib core; Windows Git и `gh` требуются.
- `repos.json` — локальный allowlist **только** `test`.
- `state/<snapshot_id>.json`, `state/<snapshot_id>.zip` — данные восстановления и исходный архив **вне проекта**.
- GitHub: `.github/workflows/snapshot.yml` — извлекает ZIP в staging и публикует односслойный Artifact `chatgpt-snapshot`.

Команды (через Windows Tunnel `start_process`):

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File C:\Users\alexa\chatgpt-bridge\Bridge.ps1 snapshot -Repo test
powershell -NoProfile -ExecutionPolicy Bypass -File C:\Users\alexa\chatgpt-bridge\Bridge.ps1 status -SnapshotId <id>
powershell -NoProfile -ExecutionPolicy Bypass -File C:\Users\alexa\chatgpt-bridge\Bridge.ps1 apply -SnapshotId <id> -PatchBase64 <base64> -PatchSha256 <hex> -CheckOnly
powershell -NoProfile -ExecutionPolicy Bypass -File C:\Users\alexa\chatgpt-bridge\Bridge.ps1 apply -SnapshotId <id> -PatchBase64 <base64> -PatchSha256 <hex>
```

`snapshot` возвращает JSON с `snapshot_id`, `status`, `base_commit`, `file_count`, `snapshot_sha256` (для ZIP на Windows), `run_id`, `artifact_id` при готовности. `status` позволяет узнать `ready`, `waiting_artifact`, `error`, а также последнее применение.

## 3. Что включено в snapshot

Только разрешённые пути по `repos.json`, включая tracked и подходящие untracked неигнорируемые файлы. Сохраняются **оригинальные байты**, а не результат `git archive HEAD`. Не экспортируются `.git`, `snapshots/`, игнорируемые и запрещённые пути; symlinks запрещены. Лимиты объёма и эвристический фильтр приватных ключей нужны для fail-closed тестового канала, но не заменяют полноценную проверку секретов.

В доставленном Artifact один ZIP-слой с `repo/<relative-path>` и `_bridge/manifest.json`. У каждого файла в manifest свой SHA-256. Поскольку GitHub Actions переупаковывает содержимое, **SHA всего скачанного Artifact ZIP может отличаться** от `snapshot_sha256` исходного ZIP. Сверяй файл-за-файлом и идентификатор snapshot, а не сравнивай два ZIP побайтово.

## 4. Работа в ChatGPT и патчи

После скачивания ZIP в ChatGPT распакуй проект локально, baseline оставь неизменным. Работай с этим деревом независимо от Tunnel, сформируй стандартный Git diff patch относительно baseline. Для небольшого патча передай Base64 и SHA-256 через Tunnel. При необходимости сначала `-CheckOnly`.

`apply` сверяет хеши preimage каждого затронутого файла с manifest и выполняет `git apply --check`. При конфликте останавливается. Пользовательские изменения не коммитятся и не отправляются на GitHub. Из-за `core.autocrlf` не смешивай побайтовое и содержательное совпадение. Перед повторным применением скрипт проверяет сохранённые postimage-хеши.

**Ограничения v0.2:** для patch-файлов запрещены нестандартные пробельные пути, бинарные diff, rename/copy и очень большие патчи. Применение многофайлового патча не обладает полноценной файловой транзакционностью/rollback. Готовность GitHub Artifact может занять десятки секунд.

## 5. Сбои и восстановление

- Если вызов Tunnel оборвался, не предполагай, что операция не выполнилась. Проверяй `status -SnapshotId` и текущее Git-состояние.
- Повторный `snapshot` с тем же `-SnapshotId` возвращает существующую запись, не создаёт новую публикацию. Ошибочное состояние v0.2 не восстанавливает автоматически push — нужна диагностика.
- `apply` с тем же patch SHA после успеха возвращает `already_applied` лишь при совпадении postimage-хешей; изменение этих файлов после apply — ошибка.
- Не применяй патч при расхождении source SHA, конфликте, непроверенном ZIP или неверном `snapshot_id`.
- Служебный публичный коммит хранит ZIP в истории Git. **Не использовать для реальных репозиториев.** Для них нужен отдельный приватный временный transport adapter без записи конфиденциальных архивов в публичную историю.

## 6. Подтверждённый результат теста

На `test` выполнено: `snapshot` рабочего дерева с локальными изменениями, GitHub Actions Artifact `11568766102`, скачивание в файловое пространство ChatGPT, проверка SHA-256 шести файлов, включая untracked `local-only.txt`; создание тестового файла/патча в ChatGPT; `apply -CheckOnly`; `apply` на Windows с postimage SHA; повторный `apply` → `already_applied`. `demo.txt` и `local-only.txt` остались локальными пользовательскими изменениями без коммита.

## 7. Следующая версия

Приватный временный транспорт, более строгая проверка секретов, операции multi-repo, восстановление после сбоя publish, большие/бинарные патчи, надёжная preflight/apply-транзакция и более строгие гарантии snapshot-consistency. Ничего из этого не объявлять реализованным.
