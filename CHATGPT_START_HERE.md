# ChatGPT Bridge — быстрый старт (v0.2-test)

**Главная инструкция:** [docs/CHATGPT_TRANSFER_PROTOCOL.md](docs/CHATGPT_TRANSFER_PROTOCOL.md).

Скрипты уже установлены в `C:\Users\alexa\chatgpt-bridge\`. Версия 0.2 работает **только** с синтетическим `C:\Users\alexa\test` и публичным `AlexPastukhh/test`. Никогда не публикуй реальные репозитории, секреты или персональные данные в этот канал.

## Для нового чата

1. Прочитай [полный протокол](docs/CHATGPT_TRANSFER_PROTOCOL.md) через GitHub-плагин.
2. Создай актуальный snapshot рабочего дерева через Tunnel:
   `powershell -NoProfile -ExecutionPolicy Bypass -File C:\Users\alexa\chatgpt-bridge\Bridge.ps1 snapshot -Repo test`
3. Из компактного JSON-ответа возьми **текущий** `artifact_id`; если `status=waiting_artifact`, вызови `Bridge.ps1 status -SnapshotId <id>`.
4. Через GitHub-плагин `download_workflow_artifact(repo_full_name="AlexPastukhh/test", artifact_id=<id>)` получи ZIP в **файловую систему текущего ChatGPT**, не на ПК.
5. Распакуй единственный Artifact ZIP: `repo/...` и `_bridge/manifest.json`. Проверь SHA-256 каждого файла по манифесту, `snapshot_id`, количество файлов. Сохраняй baseline неизменным.
6. Работай с локальными файлами в ChatGPT, не вызывая Tunnel на чтение каждого файла.
7. Если нужны правки на ПК, создай Git-compatible текстовый patch относительно baseline, передай как Base64 в `Bridge.ps1 apply -SnapshotId <id> -PatchBase64 <base64> -PatchSha256 <sha256> -CheckOnly`; затем без `-CheckOnly`. Версии 0.2 запрещены binary/rename/copy и большие патчи.
8. Проверь JSON `applied`, `post_sha256`, `status`. Повторное применение того же патча должно отвечать `already_applied`.

Никаких автоматических коммитов пользовательских локальных файлов. Bridge публикует в GitHub **только тестовый транспортный ZIP**, поэтому в тестовом репозитории появятся служебные коммиты. Реальные проекты пока не поддерживаются.

Документация: 2026-10-08; полный сквозной тест v0.2: snapshot → GitHub Artifact → ChatGPT Workspace → patch → HOST apply.
