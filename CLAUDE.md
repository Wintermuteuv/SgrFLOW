# SgrFLOW — передача контекста в Claude Code

## Что это

Внутренний продукт компании Sergek (Казахстан) — **knowledge graph** с 3D-визуализацией связей между людьми, проектами и продуктами. Одна страница, живые пузыри, размер = приоритет.

**Репозиторий:** https://github.com/Wintermuteuv/SgrFLOW

---

## Текущее состояние

### Что уже есть в репо

```
prototype/index.html   — самодостаточный прототип, открывается в браузере без сервера
README.md              — описание, стек, roadmap
docs/concept.md        — концепция, схема данных, пользователи
```

### Прототип (prototype/index.html)

- Библиотека: `3d-force-graph@1.73.0` (CDN, WebGL / Three.js)
- Тёмная тема (`#0f1117`)
- Три типа узлов: 👤 Люди (`#7F77DD`), 📦 Продукты (`#1D9E75`), 🔧 Проекты (`#D85A30`)
- Размер сферы = `Math.pow(priority, 1.4) * 2.2`
- Canvas-лейблы на каждом узле через `THREE.Sprite`
- Фильтры по типу (кнопки в шапке)
- Клик по узлу → инфо-панель справа (приоритет, связи с тегами)
- Данные захардкожены внутри HTML: 8 людей, 4 продукта, 6 проектов, 24 связи

---

## Задача для Claude Code

Превратить прототип в полноценное приложение. Стек: **Vite + React + TypeScript**.

### Шаг 1 — инициализировать проект

```bash
npm create vite@latest . -- --template react-ts
npm install 3d-force-graph
npm install --save-dev @types/three
```

### Шаг 2 — структура

```
src/
  components/
    Graph.tsx        — 3D граф (обёртка над ForceGraph3D)
    InfoPanel.tsx    — боковая панель при клике на узел
    FilterBar.tsx    — кнопки фильтрации
  data/
    graph.json       — узлы и связи (вынести из prototype/index.html)
  types/
    index.ts         — типы Node, Link, NodeType
  App.tsx
  main.tsx
```

### Шаг 3 — схема данных (graph.json)

```typescript
type NodeType = 'person' | 'product' | 'project'

interface GraphNode {
  id: string
  name: string
  type: NodeType
  priority: 1 | 2 | 3 | 4 | 5
  dept?: string      // для person
  domain?: string    // для product
  status?: string    // для project
}

interface GraphLink {
  source: string
  target: string
  label: string
}
```

### Шаг 4 — деплой на GitHub Pages

```bash
npm install --save-dev gh-pages
```

В `package.json`:
```json
{
  "homepage": "https://wintermuteuv.github.io/SgrFLOW",
  "scripts": {
    "predeploy": "npm run build",
    "deploy": "gh-pages -d dist"
  }
}
```

---

## Важные детали реализации

**ForceGraph3D в React** — библиотека мутирует DOM напрямую, нужен `useRef`:

```tsx
const containerRef = useRef<HTMLDivElement>(null)
useEffect(() => {
  if (!containerRef.current) return
  const graph = ForceGraph3D()(containerRef.current)
    .graphData(data)
    // ... конфиг
  return () => graph._destructor?.()
}, [data])
```

**nodeThreeObject** — кастомные сферы с canvas-лейблами, реализация в `prototype/index.html` (функция `mkLabel`), перенести как есть.

**Resize** — слушать `window resize`, вызывать `graph.width(w).height(h)`.

**Фильтрация** — при смене фильтра пересчитывать `graphData`: узлы выбранного типа + все смежные узлы (любого типа).

---

## Цветовая схема

| Тип | Цвет | Hex |
|-----|------|-----|
| person | фиолетовый | `#7F77DD` |
| product | зелёный | `#1D9E75` |
| project | оранжевый | `#D85A30` |
| bg | тёмный | `#0f1117` |
| surface | тёмно-серый | `#1a1d27` |

---

## Контекст о продукте

- Пользователи: руководители и тимлиды Sergek
- Рабочий язык UI: русский
- Данные сейчас: JSON вручную; цель — подключить Jira REST API и GitLab API (endpoint'ы уже известны, описаны в `context/reference_jira_gitlab_apis.md`)
