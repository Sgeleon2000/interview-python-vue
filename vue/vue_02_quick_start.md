# Vue 3 — Быстрый старт — конспект и вопросы

## О чём раздел

Раздел о том, как начать работать с Vue 3 на практике: создать проект через `npm create vue@latest`, разобраться в структуре проекта на Vite, понять, что делает `createApp` и `.mount()`, как настраивается приложение (app-конфигурация, глобальные плагины/компоненты), а также как использовать Vue без шага сборки (CDN + import maps) и чем отличаются глобальная сборка и ESM-сборка. В конце — про Vue DevTools.

Цель — уметь с нуля поднять рабочее Vue-приложение и осознанно выбрать способ подключения (сборка vs без сборки).

## Ключевые концепции

- **`npm create vue@latest`** — официальный скаффолдер (на базе `create-vue`), генерирует проект на Vite с опциональными фичами (TS, Router, Pinia, тесты, ESLint).
- **Vite** — дев-сервер и сборщик: мгновенный старт, HMR через нативные ESM в разработке, Rollup-сборка для продакшена.
- **`createApp(RootComponent)`** — создаёт экземпляр приложения; одна страница может иметь несколько независимых приложений.
- **`.mount(selector | element)`** — монтирует приложение в DOM-элемент; возвращает экземпляр корневого компонента.
- **App-конфигурация** — `app.config`, `app.use(plugin)`, `app.component(...)`, `app.directive(...)`, `app.provide(...)` — глобальные настройки до монтирования.
- **CDN + import maps** — использование Vue без сборщика прямо в браузере через `importmap`.
- **Global build vs ESM build** — `vue.global.js` (всё в `window.Vue`) против `vue.esm-browser.js` (нативные ES-модули).
- **Vue DevTools** — расширение браузера / отдельное приложение для отладки.

## Подробный разбор с примерами кода

### Создание проекта

```bash
# Запуск официального скаффолдера (нужен Node.js LTS)
npm create vue@latest

# Альтернативы для других пакетных менеджеров:
# pnpm create vue@latest
# yarn create vue@latest
# bun create vue@latest
```

Скаффолдер задаст вопросы (имя проекта, нужны ли TypeScript, JSX, Vue Router, Pinia, Vitest, e2e-тесты, ESLint, Prettier). После генерации:

```bash
cd <имя-проекта>
npm install   # установка зависимостей
npm run dev   # запуск дев-сервера (Vite) с HMR
```

Полезные npm-скрипты (из `package.json`):

```bash
npm run dev      # дев-сервер с горячей перезагрузкой
npm run build    # production-сборка в папку dist/
npm run preview  # локальный предпросмотр production-сборки
```

### Структура проекта

Типичный проект `create-vue` выглядит так:

```text
my-app/
├─ index.html            # точка входа HTML (Vite обслуживает её)
├─ package.json          # зависимости и скрипты
├─ vite.config.js        # конфигурация Vite (плагины, алиасы)
├─ public/               # статика, копируется как есть (favicon и т.п.)
└─ src/
   ├─ main.js            # точка входа JS: createApp(...).mount(...)
   ├─ App.vue            # корневой компонент
   ├─ assets/            # стили/картинки, проходят через сборку
   ├─ components/        # переиспользуемые компоненты
   ├─ views/             # страницы (если выбран Vue Router)
   ├─ router/            # настройка маршрутов (если выбран Router)
   └─ stores/            # хранилища Pinia (если выбрана)
```

Ключевой момент: `index.html` — это «исходник», а не шаблон. Vite обрабатывает его и подключает `src/main.js` как ES-модуль:

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="ru">
  <head>
    <meta charset="UTF-8" />
    <title>My App</title>
  </head>
  <body>
    <!-- сюда монтируется приложение -->
    <div id="app"></div>
    <!-- type="module" — нативные ES-модули -->
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

### `createApp` и `.mount()`

`main.js` — точка входа: создаём приложение из корневого компонента и монтируем его.

```js
// src/main.js
import { createApp } from 'vue'
import App from './App.vue' // корневой компонент

// createApp возвращает «экземпляр приложения»
const app = createApp(App)

// .mount принимает CSS-селектор или DOM-элемент
// и возвращает экземпляр корневого компонента
app.mount('#app')
```

Корневой компонент:

```vue
<!-- src/App.vue -->
<script setup>
import { ref } from 'vue'
const title = ref('Моё первое Vue-приложение')
</script>

<template>
  <h1>{{ title }}</h1>
</template>
```

Можно поднять **несколько независимых приложений** на одной странице — у каждого своя конфигурация:

```js
import { createApp } from 'vue'

// Виджет «корзина» в одном месте страницы
createApp(CartWidget).mount('#cart')

// Виджет «уведомления» — в другом
createApp(Notifications).mount('#notifications')
```

`.mount()` возвращает экземпляр корневого компонента; код после него выполняется уже после монтирования (но не гарантирует завершения async-операций внутри компонента).

### App-конфигурация

Между `createApp(App)` и `app.mount(...)` настраивают приложение глобально. Важно: всё это нужно делать **до** `.mount()`.

```js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'         // Vue Router
import { createPinia } from 'pinia'   // Pinia
import GlobalButton from './components/GlobalButton.vue'

const app = createApp(App)

// 1. Плагины через app.use()
app.use(router)
app.use(createPinia())

// 2. Глобальная регистрация компонента
app.component('GlobalButton', GlobalButton)

// 3. Глобальная директива
app.directive('focus', {
  mounted: (el) => el.focus()
})

// 4. Глобальный provide (доступен через inject в любом потомке)
app.provide('apiBase', 'https://api.example.com')

// 5. Глобальные настройки и обработчик ошибок
app.config.errorHandler = (err, instance, info) => {
  console.error('Глобальная ошибка:', err, info)
}
// Глобальные свойства (использовать осторожно, доступны как this.$x)
app.config.globalProperties.$appName = 'My App'

// Монтируем — ПОСЛЕ всей конфигурации
app.mount('#app')
```

### Использование без сборки: CDN и import maps

Минимальный вариант через **global build** (всё в глобальном `Vue`):

```html
<!DOCTYPE html>
<html lang="ru">
  <body>
    <div id="app">
      <!-- {{ }} работает после монтирования -->
      <button @click="count++">Кликов: {{ count }}</button>
    </div>

    <!-- Global build: создаёт window.Vue -->
    <script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
    <script>
      const { createApp, ref } = Vue
      createApp({
        setup() {
          const count = ref(0)
          return { count } // что вернули — доступно в шаблоне
        }
      }).mount('#app')
    </script>
  </body>
</html>
```

Современный вариант — **ESM build + import maps** (можно писать `import { ... } from 'vue'` без сборщика):

```html
<!DOCTYPE html>
<html lang="ru">
  <body>
    <div id="app">{{ message }}</div>

    <!-- import map сопоставляет имя 'vue' с URL ESM-сборки -->
    <script type="importmap">
      {
        "imports": {
          "vue": "https://unpkg.com/vue@3/dist/vue.esm-browser.js"
        }
      }
    </script>

    <!-- type="module" обязателен для import -->
    <script type="module">
      import { createApp, ref } from 'vue'

      createApp({
        setup() {
          const message = ref('Привет из ESM + importmap!')
          return { message }
        }
      }).mount('#app')
    </script>
  </body>
</html>
```

Import maps позволяют разбивать код на отдельные `.js`-модули и импортировать их без бандлера. Минусы подхода без сборки: нет SFC (`.vue`), нет HMR, нет оптимизаций компилятора, медленнее при множестве модулей.

### Global build vs ESM build (важное различие)

| Сборка | Файл | Как подключать | Доступ к API |
|---|---|---|---|
| **Global** | `vue.global.js` | `<script src=...>` | через глобальный `window.Vue` (`const { createApp } = Vue`) |
| **ESM (browser)** | `vue.esm-browser.js` | `<script type="module">` + import maps | через `import { createApp } from 'vue'` |

Кроме того, для каждой есть **dev** и **prod** версии:
- Dev-версия (`vue.global.js`, `vue.esm-browser.js`) — с предупреждениями и подсказками, тяжелее.
- Prod-версия (`vue.global.prod.js`, `vue.esm-browser.prod.js`) — минифицирована, без предупреждений.

Дополнительно есть **`vue.runtime.*`** сборки — без компилятора шаблонов (компиляция выполняется при сборке проекта). Именно runtime-сборку использует Vite по умолчанию: шаблоны SFC компилируются заранее, поэтому в бандл не тащится компилятор. Полная сборка нужна, когда шаблоны компилируются в браузере (например, строковые `template` в CDN-сценарии).

### Vite — почему он быстрый

- **В разработке**: Vite не бандлит весь код, а отдаёт исходники через нативные ESM, трансформируя файлы по запросу. Старт почти мгновенный независимо от размера проекта.
- **HMR (Hot Module Replacement)**: при изменении файла обновляется только он, состояние приложения по возможности сохраняется.
- **Продакшен**: сборка через Rollup с tree-shaking, code-splitting, минификацией.

```js
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [vue()], // плагин для обработки .vue-файлов
  resolve: {
    alias: {
      // алиас '@' → папка src (частая практика)
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  }
})
```

### Vue DevTools

- Расширение для Chrome/Firefox/Edge или отдельное приложение.
- Возможности: дерево компонентов, инспекция props/состояния, отладка реактивности, таймлайн событий, инспекция Pinia/Router.
- В Vite-проекте можно подключить плагин `vite-plugin-vue-devtools` для встроенной панели прямо в приложении.

```bash
# Пример установки плагина devtools для Vite
npm i -D vite-plugin-vue-devtools
```

## Полный рабочий пример

Минимальное, но «настоящее» приложение со структурой `src/`.

```js
// src/main.js — точка входа
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Глобальная директива v-focus (демонстрация app-конфигурации)
app.directive('focus', {
  mounted: (el) => el.focus()
})

app.mount('#app')
```

```vue
<!-- src/App.vue — корневой компонент -->
<script setup>
import { ref, computed } from 'vue'
import GreetingCard from './components/GreetingCard.vue'

const name = ref('') // двусторонняя привязка к input

// приветствие пересчитывается при изменении name
const greeting = computed(() =>
  name.value ? `Привет, ${name.value}!` : 'Введите имя'
)
</script>

<template>
  <main>
    <h1>Быстрый старт Vue 3</h1>

    <!-- v-focus — наша глобальная директива, ставит фокус при монтировании -->
    <input v-focus v-model="name" placeholder="Ваше имя" />

    <!-- передаём приветствие в дочерний компонент через prop -->
    <GreetingCard :text="greeting" />
  </main>
</template>
```

```vue
<!-- src/components/GreetingCard.vue — дочерний компонент -->
<script setup>
// объявляем входной prop с типом
defineProps({
  text: {
    type: String,
    required: true
  }
})
</script>

<template>
  <div class="card">{{ text }}</div>
</template>

<style scoped>
.card {
  padding: 12px;
  border: 1px solid #42b883;
  border-radius: 8px;
}
</style>
```

## Частые вопросы на собеседовании (Q/A)

**1. Как создать новый Vue 3 проект?**
Командой `npm create vue@latest` — это официальный скаффолдер на базе Vite. Он спрашивает про TypeScript, Router, Pinia, тесты, линтеры и генерирует готовую структуру. Затем `npm install` и `npm run dev`.

**2. Что делает `createApp` и `mount`?**
`createApp(RootComponent)` создаёт экземпляр приложения с собственной конфигурацией. `.mount(selector|element)` монтирует приложение в DOM и возвращает экземпляр корневого компонента. На одной странице может быть несколько приложений.

**3. Почему конфигурацию приложения делают до `.mount()`?**
Потому что `app.use()`, `app.component()`, `app.directive()`, `app.provide()`, `app.config` влияют на то, как будет создаваться дерево компонентов. После монтирования изменения уже не применятся к текущему рендеру.

**4. Что такое Vite и почему он быстрый?**
Vite — дев-сервер и сборщик. В разработке он использует нативные ESM и трансформирует файлы по запросу (без полного бандла), плюс быстрый HMR. Для продакшена собирает через Rollup с tree-shaking и code-splitting.

**5. Чем отличается global build от ESM build Vue?**
Global build (`vue.global.js`) подключается тегом `<script>` и кладёт API в `window.Vue`. ESM build (`vue.esm-browser.js`) подключается как модуль (`<script type="module">`) и используется через `import` (обычно с import maps).

**6. Что такое import maps и зачем они нужны?**
Это нативная браузерная фича, сопоставляющая «голые» имена модулей (`vue`) с URL. Позволяет писать `import { createApp } from 'vue'` без сборщика. Используется в CDN-сценариях без сборки.

**7. Можно ли использовать Vue без шага сборки?**
Да — через CDN (global или ESM build + import maps). Подходит для прототипов, виджетов, «оживления» страниц. Но нет SFC, HMR и оптимизаций компилятора, поэтому для крупных приложений рекомендуется сборка.

**8. Что такое runtime-сборка и full-сборка Vue?**
Full-сборка включает компилятор шаблонов (нужен, если шаблоны — строки, компилируемые в браузере). Runtime-сборка без компилятора (легче); её использует Vite, потому что шаблоны SFC компилируются заранее на этапе сборки.

**9. Зачем нужен `index.html` в корне проекта Vite?**
Это точка входа: Vite обслуживает её, обрабатывает ссылки на ресурсы и подключает `src/main.js` как ES-модуль. В отличие от webpack, html здесь — реальный исходник, а не шаблон.

**10. Какие основные npm-скрипты в Vue-проекте?**
`dev` — дев-сервер с HMR, `build` — production-сборка в `dist/`, `preview` — локальный предпросмотр собранного приложения.

**11. Как глобально зарегистрировать компонент или плагин?**
Компонент — `app.component('Name', Comp)`, плагин — `app.use(plugin)`, директива — `app.directive('name', def)`. Всё до `.mount()`.

**12. Как отлаживать Vue-приложение?**
Через Vue DevTools (расширение браузера или приложение): дерево компонентов, props/state, реактивность, таймлайн, Pinia/Router. В Vite можно подключить `vite-plugin-vue-devtools`.

**13. В чём разница между dev- и prod-сборкой Vue?**
Dev — с предупреждениями, проверками и подсказками (тяжелее), удобна при разработке. Prod (`*.prod.js`) — минифицирована, без предупреждений, для боевого окружения. `npm run build` использует prod-вариант.

**14. Что возвращает `.mount()` и где это полезно?**
Возвращает экземпляр корневого компонента. Полезно для доступа к публичным свойствам/методам корня извне приложения, но в большинстве случаев не используется напрямую.

## Подводные камни (gotchas)

- **Конфигурация после `.mount()`** — `app.use`/`app.component` после монтирования не повлияют на уже отрендеренное приложение. Всё настраивайте до `.mount()`.
- **`vue.global.js` не понимает `import`** — global build кладёт API в `window.Vue`; для `import` нужна ESM-сборка и `type="module"`.
- **Забытый `type="module"`** — без него браузер не выполнит `import` и import maps не сработают.
- **Dev-сборка в продакшене** — подключение `vue.global.js` (а не `.prod.js`) на бой увеличивает размер и оставляет предупреждения.
- **Путаница runtime vs full build** — строковые шаблоны (`template: '...'`) требуют full-сборки с компилятором; в SFC-проекте на Vite по умолчанию runtime-сборка.
- **`index.html` как обычный шаблон** — в Vite его нельзя класть в `public/` и ожидать обработки; он должен быть в корне как точка входа.
- **Глобальные `globalProperties`** — удобны, но создают неявные зависимости и хуже типизируются; предпочитайте `provide/inject` или composables.
- **Несколько приложений и общее состояние** — разные `createApp` не делят provide/конфигурацию; для общего состояния нужен общий store или общий plugin.

## Лучшие практики

- Создавайте проекты через **`npm create vue@latest`** и используйте **Vite**.
- Держите всю app-конфигурацию (`use`, `component`, `provide`, `config`) в `main.js` **до `.mount()`**.
- Для продакшена используйте **`npm run build`** (prod-сборка) и проверяйте через `npm run preview`.
- Без сборки выбирайте **ESM build + import maps** вместо global build, если нужны модули.
- Не злоупотребляйте **`app.config.globalProperties`** — берите `provide/inject` или composables.
- Настройте **алиас `@` → `src`** для коротких импортов.
- Установите **Vue DevTools** (или `vite-plugin-vue-devtools`) для отладки.
- На production подключайте только **prod-сборки** Vue.

## Шпаргалка

```text
СОЗДАНИЕ ПРОЕКТА
  npm create vue@latest   # официальный скаффолдер на Vite
  npm install
  npm run dev             # дев-сервер + HMR
  npm run build           # prod-сборка → dist/
  npm run preview         # предпросмотр сборки

СТРУКТУРА
  index.html              # точка входа (в корне!), грузит /src/main.js
  src/main.js             # createApp(App).mount('#app')
  src/App.vue             # корневой компонент
  vite.config.js          # плагины, алиасы

ТОЧКА ВХОДА
  const app = createApp(App)
  app.use(router); app.use(pinia)     // плагины
  app.component('X', X)               // глоб. компонент
  app.directive('focus', {...})       // глоб. директива
  app.provide('key', val)             // глоб. provide
  app.config.errorHandler = ...       // настройки
  app.mount('#app')                   // ВСЕГДА в конце

БЕЗ СБОРКИ
  Global: <script src="vue.global.js">  → window.Vue
  ESM:    <script type="importmap"> { "vue": ".../vue.esm-browser.js" }
          <script type="module"> import { createApp } from 'vue'

СБОРКИ VUE
  global  vs  esm-browser      (тег script vs import)
  dev     vs  prod (.prod.js)  (предупреждения vs минификация)
  full    vs  runtime          (с компилятором vs без; Vite → runtime)

ОТЛАДКА
  Vue DevTools (расширение) / vite-plugin-vue-devtools
```
