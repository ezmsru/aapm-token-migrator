# aapm-token-migrator

Разовая CI-миграция **DATA-38470**: во всех config-репозиториях под группой
`middleconf/itbigdata/**` заменяет в стенд-values ключ `aapmToken` на `aapmSecret`
и выставляет MR в `master`. Разработчики принимают MR сами.

## Что делает

Пайплайн ([.gitlab-ci.yml](.gitlab-ci.yml)), один job `update-aapm-token`:

1. Резолвит группу `middleconf/itbigdata` и рекурсивно (`include_subgroups=true`,
   с пагинацией) собирает **все** проекты под ней. Токен — `GITLAB_MIDDLECONF_TUZ_TOKEN`.
2. В каждом репо в файлах **строго `*-values.yaml`** ищет строки
   `aapmToken: "<UUID>"` и заменяет на
   `aapmSecret: "<pam_token_*>"   # значение AAPM-токена для обращения за секретом.`
3. Где есть изменения — создаёт ветку `feature/DATA-38470.update_token_pam`,
   коммитит и `git push -o merge_request.create` в `master`.

## Маппинг

Замена строго по значению, которое **сейчас** стоит в `aapmToken`. Сами UUID в
репозитории **не хранятся** — они берутся из CI-переменных GitLab, имена которых
совпадают с целевым `aapmSecret` (значение переменной = UUID):

| CI-переменная (значение = UUID) | Новый `aapmSecret`      |
| ------------------------------- | ----------------------- |
| `pam_token_lambda_llm`          | `pam_token_lambda_llm`  |
| `pam_token_prod_llm`            | `pam_token_prod_llm`    |
| `pam_token_sandbox_llm`         | `pam_token_sandbox_llm` |

- UUID, которого нет среди значений этих переменных, **не трогается**.
- Строки, где уже стоит `aapmSecret`, **не трогаются** (идемпотентно).

## Запуск

Только вручную через **Run pipeline** (`workflow: source == web`).

Inputs:

| Input                   | По умолчанию                            | Назначение                                   |
| ----------------------- | --------------------------------------- | -------------------------------------------- |
| `middleconf_group_path` | `middleconf/itbigdata`                  | корневая группа для рекурсивного перебора     |
| `feature_branch`        | `feature/DATA-38470.update_token_pam`   | ветка под MR в каждом репо                     |
| `target_branch`         | `master`                                | ветка-приёмник MR                             |
| `dry_run`               | `true`                                  | `true` — только дифф; `false` — push + MR     |
| `cleanup_mrs`           | `false`                                 | `true` — режим очистки (см. ниже), миграция не выполняется |

**Порядок:** сначала прогнать с `dry_run=true` (покажет диффы по всем репам без
push/MR), убедиться — затем перезапустить с `dry_run=false`.

> Замена правит **только одну строку** и сохраняет оригинальные концы строк
> (CRLF/CR/LF) — поэтому в репах с CRLF дифф остаётся `+1 −1`, а не весь файл.

## Очистка ранее созданных MR

`cleanup_mrs=true` — пайплайн **не мигрирует**, а откатывает следы прошлых
прогонов: по всем репам группы закрывает открытые MR, исходящие из
`feature_branch`, и удаляет саму ветку. Уважает `dry_run`:

- `dry_run=true` — только показать, какие MR/ветки были бы удалены;
- `dry_run=false` — реально закрыть MR (`state_event=close`) и удалить ветки.

## Требования

- CI-переменная `GITLAB_MIDDLECONF_TUZ_TOKEN` (masked) — ТУЗ с правами ≥ Developer
  (push ветки + создание MR) во всех репозиториях группы.
- CI-переменные `pam_token_lambda_llm`, `pam_token_prod_llm`, `pam_token_sandbox_llm`
  (masked) — значения = UUID соответствующих AAPM-токенов. Если переменная
  «protected», прогон должен идти на protected-ветке/теге, иначе значение пустое.
- Раннер с тегом `docker-abd`; образ `alpine:3.22` из `unixrepo`
  (`apk add git curl jq python3`).
