---
name: Доступ к Jira/GitLab Sergek через Chrome MCP
description: Рабочие endpoint'ы и приёмы извлечения данных из jira.srgdev.kz и git.sergek.kz — для повторного использования на других командах
type: reference
originSessionId: e96a774c-5a99-4922-a9c0-fc1b5c2fddee
---
**Контекст:** Jira и GitLab корпоративные, требуют аутентификации. Достучаться через WebFetch нельзя. Работающий путь — Chrome MCP + cookies-сессия пользователя в браузере.

## Рабочие endpoint'ы

### Jira (jira.srgdev.kz)

Получить тикеты, закрытые в периоде:
```
GET http://jira.srgdev.kz/rest/api/2/search?
  jql=project = {{KEY}} AND resolutiondate >= "{{YYYY-MM-DD}}" AND resolutiondate <= "{{YYYY-MM-DD}}"
  &fields=summary,status,issuetype,priority,assignee,resolutiondate,created,components,labels,fixVersions
  &maxResults=200
```

Получить статус конкретных тикетов:
```
GET /rest/api/2/search?jql=key in (KEY-1,KEY-2,...)&fields=summary,status,issuetype,assignee,resolutiondate
```

### GitLab (git.sergek.kz)

Сначала открыть страницу проекта чтобы посмотреть Project ID (отображается под названием проекта на странице репо).

Коммиты за период (нужно идти по страницам):
```
GET /api/v4/projects/{{PROJECT_ID}}/repository/commits?since=2026-XX-XX&until=2026-YY-YY&all=true&per_page=100&page=N
```

MR'ы за период:
```
GET /api/v4/projects/{{PROJECT_ID}}/merge_requests?state=all&created_after=...&created_before=...&per_page=100
```

## Доступ к git-репо команд (актуально на 30.04.2026)

`i.bozhitov` теперь member 13 проектов в git.sergek.kz. Целевые репо для отчётности по командам блока разработки:

| Команда | Репо | Project ID | Доступ |
|---|---|---|---|
| VMS | pc-team/vms-core-go | 271 | ✓ |
| VMS | pc-team/vms-desktop | 291 | ✓ |
| VMS | pc-team/vms-analytics | 290 | ✓ |
| VMS | sergekteam/vms_frontend | 206 | ✓ |
| Core / RS | sergekteam/sergek_v3 | 174 | ✓ (с апр.2026) |
| Core / CST | sergekteam/sergek_v4 | 468 | ✓ (с апр.2026) |
| Core / CST | sergekteam/sergek_v4_front | 518 | ✓ (с апр.2026) |
| Patrol | sergekteam/patrol-v2 | 342 | ✓ (с апр.2026) — служебный сервер |
| Patrol | patrol/radarpatrol_cpp | 304 | ⚠ membership есть, но `/repository/commits` → 403 (нужно повышение прав) |
| Platform | platform/big_platform | 603 | ✓ |
| ITS | its/its-backend | 481 | ✓ |

**Раньше (память до апреля)** — было только 7 репо доступно (vms-core-go и vms-* плюс vms_frontend). Сейчас доступ существенно расширен — **всегда проверять membership заново**, не полагаться на старую заметку «доступа нет».

Как проверить: `GET https://git.sergek.kz/api/v4/projects?membership=true&per_page=100&simple=true` → массив проектов с `path_with_namespace`.

Исключение по radarpatrol_cpp: даже если в списке membership, перед использованием — пинг `/repository/commits` (если 403, доступа на код нет — можно запросить отдельно).

## Кросс-командные коммиты (важно для разреза по командам)

В git-репо часто коммитят разработчики из соседних команд — это нормально для тесно связанной кодовой базы, но размывает «активность по командам». Известные паттерны:

- **sergek_v3 (RS)** — коммитят и Stanislav Khomenko (CST), и Assel Zholdasbekova (RS).
- **sergek_v4 (CST)** — основной автор Хоменко (CST), но активна и Жолдасбекова (RS).
- **patrol-v2** — служебный сервер, основной код моделей в radarpatrol_cpp; коммитят преимущественно RS-разработчики.
- **big_platform** — основной автор Абай Скаков, изредка Хоменко (CST).

При построении отчёта — атрибутировать коммиты по людям, не по репо. Для корректного разреза нужен per-author roster команд.

## Приёмы и грабли

- **fetch() через JS на странице Jira/GitLab UI таймаутит** (CDP timeout 45с). Решение: переходить навигацией прямо на REST URL и парсить `document.body.innerText` как JSON.
- **window.* теряется между navigation'ами**, использовать `localStorage.setItem` для накопления страниц.
- **document.body.innerText** возвращается обрезанным (~7-8 строк JSON), нужно дробить вывод на чанки `slice(start, end)` по 5-7 элементов.
- **Парсинг ключей VMSSCRUM-N в коммитах**: regex `/VMSSCRUM[-\s_]*(\d+)/gi` — ловит и `VMSSCRUM-123`, и `VMSSCRUM 123`, и `VMSSCRUM_123`.
- **Команды нередко не используют MR** — `state=merged` может вернуть 0; не считать это ошибкой. Также часть коммитов идёт без префикса тикета — это типично, фиксировать как точку внимания «гигиена коммитов».

## Алгоритм для новой команды

1. Открыть в Chrome доску Jira → авторизация уже есть.
2. Получить ключ проекта Jira из URL (projectKey=...).
3. Открыть страницу репо в GitLab → срисовать Project ID.
4. Дёргать Jira REST API (search by JQL) и GitLab API (commits + MRs).
5. Парсить ключи тикетов в коммитах через regex `[A-Z]+-\d+` (адаптировать под префикс команды).
6. Запустить скрипты `build_report.js` / `build_exec.js`, подставив значения команды.
