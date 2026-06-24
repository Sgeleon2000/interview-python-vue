# Vue 3 — Прод и доступность — конспект и вопросы

## О чём раздел

Раздел охватывает две темы выпуска приложения в реальный мир:

1. **Продакшен-сборка и деплой** — как собрать приложение для продакшена (минификация, `NODE_ENV=production`, отключение dev-инструментов, source maps), как деплоить и кэшировать ресурсы, как отлавливать ошибки в проде (`app.config.errorHandler`).
2. **Доступность (a11y, accessibility)** — как сделать приложение пригодным для всех пользователей, включая тех, кто работает с клавиатуры и скринридеров: семантика HTML, skip-links, управление фокусом при роутинге, ARIA, корректные формы с `label`, проверка с клавиатуры, контраст.

Обе темы часто недооценивают, но они отличают продакшен-готовый продукт от прототипа и регулярно встречаются на собеседованиях уровня middle/senior.

## Ключевые концепции

**Продакшен:**
- **`NODE_ENV=production`** — включает оптимизации, убирает dev-предупреждения и проверки.
- **Минификация / tree-shaking** — уменьшение размера бандла.
- **Отключение devtools** в проде (`app.config.devtools`, исторически).
- **Source maps** — отладка прод-ошибок без раскрытия исходников публично.
- **Деплой и кэширование** — хешированные имена файлов + правильные Cache-Control заголовки.
- **`app.config.errorHandler`** — глобальный перехват ошибок для систем мониторинга (Sentry и т.п.).

**Доступность:**
- **Семантика** — правильные HTML-теги (`<button>`, `<nav>`, `<main>`, заголовки).
- **Skip-links** — ссылка «перейти к контенту» для клавиатуры/скринридеров.
- **Focus management** — перенос фокуса при SPA-навигации.
- **ARIA** — атрибуты для случаев, когда семантики HTML не хватает.
- **Формы и `label`** — каждое поле должно иметь связанный лейбл.
- **Проверка с клавиатуры** — всё доступно без мыши.
- **Контраст** — соответствие WCAG (4.5:1 для текста).

## Подробный разбор с примерами кода

### 1. Продакшен-сборка

#### NODE_ENV и сборка

Vite автоматически выставляет `NODE_ENV=production` при `vite build`. Это включает:
- Удаление dev-only кода Vue (предупреждения, проверки реактивности, читаемые имена варнингов).
- Минификацию (esbuild/terser), tree-shaking.
- Замену `__DEV__` на `false`, благодаря чему бандл становится меньше и быстрее.

```bash
# Продакшен-сборка
npm run build      # => vite build, NODE_ENV=production автоматически

# Локальный предпросмотр прод-сборки
npm run preview
```

```js
// vite.config.js
export default {
  build: {
    // Минификация (по умолчанию esbuild — быстрее; terser — агрессивнее)
    minify: 'esbuild',
    // Source maps: 'hidden' — генерировать, но не ссылаться из бандла
    // (загружаются только в систему мониторинга, не публикуются)
    sourcemap: 'hidden',
    rollupOptions: {
      output: {
        // Хеши в именах для кэш-бастинга (см. ниже)
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash].[ext]',
      },
    },
  },
}
```

#### Отключение devtools в проде

Vue DevTools в продакшен-сборке отключаются автоматически. Дополнительно убедитесь, что не включаете флаги отладки:

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Профилирование/перформанс-метки — только в dev
if (import.meta.env.DEV) {
  app.config.performance = true
}

app.mount('#app')
```

#### Source maps

Source maps позволяют отлаживать минифицированный код по исходникам. В проде:
- `sourcemap: 'hidden'` — генерирует `.map`-файлы, но **не** добавляет в бандл комментарий-ссылку. Файлы загружаются в Sentry/мониторинг, но не доступны публично.
- Не публикуйте source maps на CDN, если не хотите раскрывать исходный код.

### 2. Деплой и кэширование

Стратегия: имена файлов с **content hash** позволяют кэшировать ассеты «навсегда», а `index.html` — никогда.

```
# Заголовки на сервере / CDN:

# Хешированные ассеты — иммутабельны, кэшируем на год
/assets/*.js   Cache-Control: public, max-age=31536000, immutable
/assets/*.css  Cache-Control: public, max-age=31536000, immutable

# index.html — всегда свежий, чтобы подтянуть новые хеши
/index.html    Cache-Control: no-cache
```

При новой сборке меняются хеши → меняются имена файлов → браузер скачивает новые версии. `index.html` не кэшируется, поэтому всегда ссылается на актуальные хеши. Для SPA с history-режимом настройте сервер на возврат `index.html` для всех неизвестных путей (fallback).

### 3. Отслеживание ошибок: app.config.errorHandler

Глобальный обработчик ловит ошибки из рендеров, watcher'ов, хуков жизненного цикла. Идеален для интеграции с системами мониторинга.

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

// Глобальный перехват ошибок компонентов
app.config.errorHandler = (err, instance, info) => {
  // err — объект ошибки
  // instance — компонент, где произошла ошибка
  // info — строка с типом источника (например, 'render function')

  // Отправляем в систему мониторинга (Sentry, свой бэкенд)
  reportToMonitoring(err, { component: instance?.$options.name, info })

  // В dev дополнительно выводим в консоль
  if (import.meta.env.DEV) {
    console.error(`[errorHandler] ${info}:`, err)
  }
}

// Перехват предупреждений (только dev)
app.config.warnHandler = (msg, instance, trace) => {
  if (import.meta.env.DEV) console.warn(msg, trace)
}

app.mount('#app')
```

Дополняйте это:
- `window.addEventListener('error', ...)` и `'unhandledrejection'` — для ошибок вне Vue (async, промисы).
- `onErrorCaptured` хук — локальная обработка ошибок в поддереве компонентов (error boundary).

```js
// Компонент-граница ошибок
import { onErrorCaptured, ref } from 'vue'

const hasError = ref(false)
onErrorCaptured((err) => {
  hasError.value = true
  // Вернуть false, чтобы остановить всплытие ошибки выше
  return false
})
```

### 4. Доступность (a11y)

#### Семантика HTML

Используйте правильные теги — скринридеры и клавиатура полагаются на семантику. Не делайте кнопки из `<div>`.

```vue
<template>
  <!-- ПЛОХО: div не фокусируется, не реагирует на Enter/Space,
       скринридер не объявит его как кнопку -->
  <div class="btn" @click="submit">Отправить</div>

  <!-- ХОРОШО: нативная кнопка — фокус, клавиатура, роль из коробки -->
  <button type="button" @click="submit">Отправить</button>

  <!-- Структура страницы семантичными тегами -->
  <header>...</header>
  <nav aria-label="Главное меню">...</nav>
  <main>
    <h1>Заголовок страницы</h1>
    <!-- Иерархия заголовков без пропусков уровней -->
    <h2>Раздел</h2>
  </main>
  <footer>...</footer>
</template>
```

#### Skip-links — пропуск навигации

Skip-link позволяет клавиатурным пользователям сразу перейти к основному контенту, минуя навигацию.

```vue
<template>
  <!-- Видим только при фокусе (Tab), уводит к #main -->
  <a href="#main" class="skip-link">Перейти к содержимому</a>

  <nav>...длинное меню...</nav>

  <main id="main" tabindex="-1">
    <h1>Контент</h1>
  </main>
</template>

<style scoped>
.skip-link {
  position: absolute;
  left: -9999px; /* скрыт визуально */
}
.skip-link:focus {
  left: 8px;
  top: 8px; /* появляется при Tab */
}
</style>
```

#### Управление фокусом при роутинге (SPA-специфика)

В SPA навигация не перезагружает страницу, поэтому фокус и объявление скринридера не происходят автоматически. Нужно вручную переносить фокус на заголовок/контент после перехода.

```js
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes: [/* ... */],
})

router.afterEach((to) => {
  // Ждём отрисовку нового маршрута, затем переносим фокус
  // на основной контейнер, чтобы скринридер объявил новую страницу
  requestAnimationFrame(() => {
    const main = document.getElementById('main')
    if (main) {
      main.setAttribute('tabindex', '-1')
      main.focus()
    }
    // Обновляем заголовок вкладки — важно для скринридеров
    document.title = to.meta.title || 'Моё приложение'
  })
})

export default router
```

#### ARIA — когда семантики не хватает

ARIA-атрибуты дополняют семантику для динамических/кастомных компонентов. Правило ARIA №1: **не используйте ARIA, если есть нативный HTML-элемент** с нужной семантикой.

```vue
<script setup>
import { ref } from 'vue'
const open = ref(false)
const liveMessage = ref('')
</script>

<template>
  <!-- Кастомный аккордеон: роль, состояние, связь с контентом -->
  <button
    :aria-expanded="open"
    aria-controls="panel-1"
    @click="open = !open"
  >
    Подробнее
  </button>
  <div id="panel-1" role="region" v-show="open">...</div>

  <!-- Live region: скринридер объявит изменения текста
       (например, "Сохранено") без перемещения фокуса -->
  <div aria-live="polite" class="sr-only">{{ liveMessage }}</div>

  <!-- Иконка-кнопка без видимого текста — нужна метка -->
  <button aria-label="Закрыть" @click="close">
    <svg aria-hidden="true"><!-- иконка декоративна --></svg>
  </button>
</template>

<style scoped>
/* Класс для контента только для скринридеров */
.sr-only {
  position: absolute;
  width: 1px; height: 1px;
  padding: 0; margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap; border: 0;
}
</style>
```

#### Формы и label

Каждое поле должно иметь связанный `<label>` — иначе скринридер не объявит назначение поля, а клик по подписи не сфокусирует ввод.

```vue
<script setup>
import { ref } from 'vue'
const email = ref('')
const emailError = ref('')
</script>

<template>
  <form @submit.prevent="onSubmit">
    <!-- Связь label и input через for/id -->
    <label for="email">Электронная почта</label>
    <input
      id="email"
      v-model="email"
      type="email"
      autocomplete="email"
      :aria-invalid="!!emailError"
      :aria-describedby="emailError ? 'email-error' : undefined"
      required
    />
    <!-- Сообщение об ошибке, связанное с полем и объявляемое -->
    <p
      v-if="emailError"
      id="email-error"
      role="alert"
    >
      {{ emailError }}
    </p>

    <button type="submit">Войти</button>
  </form>
</template>
```

#### Проверка с клавиатуры

Всё интерактивное должно работать без мыши: Tab переходит по фокусируемым элементам, Enter/Space активируют, Esc закрывает модалки, видна индикация фокуса (`:focus-visible`).

```vue
<script setup>
import { ref, watch, nextTick } from 'vue'
const isOpen = ref(false)
const dialogRef = ref(null)

// При открытии модалки — фокус внутрь; при закрытии — обратно
watch(isOpen, async (open) => {
  if (open) {
    await nextTick()
    dialogRef.value?.focus()
  }
})
</script>

<template>
  <!-- Esc закрывает; модалка фокусируема -->
  <div
    v-if="isOpen"
    ref="dialogRef"
    role="dialog"
    aria-modal="true"
    aria-labelledby="dlg-title"
    tabindex="-1"
    @keydown.esc="isOpen = false"
  >
    <h2 id="dlg-title">Диалог</h2>
    <button @click="isOpen = false">Закрыть</button>
  </div>
</template>

<style>
/* Видимая индикация фокуса с клавиатуры */
:focus-visible {
  outline: 2px solid #0066cc;
  outline-offset: 2px;
}
</style>
```

#### Контраст

Текст должен иметь достаточный контраст с фоном по WCAG: 4.5:1 для обычного текста, 3:1 для крупного. Проверяйте в DevTools (Lighthouse, Contrast checker) или axe. Не передавайте информацию только цветом (например, ошибку — ещё и текстом/иконкой).

### 5. Инструменты проверки a11y

- **Lighthouse** (в Chrome DevTools) — аудит доступности, контраста, ARIA.
- **axe DevTools** — расширение, находит нарушения WCAG.
- **eslint-plugin-vuejs-accessibility** — линтер для шаблонов Vue.
- **Проверка вручную**: пройти по приложению только клавиатурой; включить скринридер (VoiceOver/NVDA).

## Полный рабочий пример

```js
// main.js — продакшен-готовая инициализация
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)

// Глобальный обработчик ошибок для мониторинга
app.config.errorHandler = (err, instance, info) => {
  // В проде — отправляем в Sentry/бэкенд; в dev — в консоль
  if (import.meta.env.PROD) {
    sendToMonitoring(err, { info, component: instance?.$options?.name })
  } else {
    console.error(`[Vue error] ${info}`, err)
  }
}

// Перформанс-метки только в dev
if (import.meta.env.DEV) {
  app.config.performance = true
}

// Доступность роутинга: фокус + заголовок при навигации
router.afterEach((to) => {
  requestAnimationFrame(() => {
    const main = document.getElementById('main')
    main?.focus()
    document.title = to.meta.title ? `${to.meta.title} — App` : 'App'
  })
})

// Ловим необработанные промисы вне Vue
window.addEventListener('unhandledrejection', (e) => {
  if (import.meta.env.PROD) sendToMonitoring(e.reason, { info: 'unhandledrejection' })
})

app.use(router)
app.mount('#app')
```

```vue
<!-- App.vue — доступный каркас приложения -->
<script setup>
import { RouterView } from 'vue-router'
</script>

<template>
  <!-- Skip-link первым в фокус-порядке -->
  <a href="#main" class="skip-link">Перейти к содержимому</a>

  <header>
    <nav aria-label="Главное меню">
      <RouterLink to="/">Главная</RouterLink>
      <RouterLink to="/about">О нас</RouterLink>
    </nav>
  </header>

  <!-- main фокусируем для переноса фокуса при навигации -->
  <main id="main" tabindex="-1">
    <RouterView />
  </main>

  <footer>...</footer>
</template>

<style>
.skip-link { position: absolute; left: -9999px; }
.skip-link:focus { left: 8px; top: 8px; }
:focus-visible { outline: 2px solid #0066cc; outline-offset: 2px; }
</style>
```

## Частые вопросы на собеседовании (Q/A)

**1. Что меняет продакшен-сборка по сравнению с dev?**
`NODE_ENV=production` убирает dev-предупреждения и проверки реактивности, включает минификацию и tree-shaking, заменяет `__DEV__` на `false`, отключает DevTools. Бандл становится значительно меньше и быстрее.

**2. Как правильно настроить кэширование ассетов?**
Имена файлов с content hash (`[name].[hash].js`) кэшируются «навсегда» (`max-age=31536000, immutable`), а `index.html` — `no-cache`, чтобы всегда подтягивать актуальные хеши. Новая сборка меняет хеши → браузер скачивает новые версии.

**3. Зачем `sourcemap: 'hidden'` в проде?**
Чтобы иметь source maps для отладки в системе мониторинга, но не публиковать их публично и не раскрывать исходный код. `hidden` генерирует `.map`, но не добавляет ссылку в бандл.

**4. Для чего `app.config.errorHandler`?**
Глобальный перехват ошибок из рендеров, watcher'ов и хуков жизненного цикла. Используется для отправки в мониторинг (Sentry). Дополняется `onErrorCaptured` (локальные границы ошибок) и `window` слушателями для ошибок вне Vue.

**5. Чем `onErrorCaptured` отличается от `errorHandler`?**
`errorHandler` — глобальный, один на приложение. `onErrorCaptured` — хук компонента, перехватывает ошибки из его поддерева (error boundary); возврат `false` останавливает всплытие. Позволяет показать локальный fallback вместо падения всего приложения.

**6. Почему важна семантика HTML для доступности?**
Скринридеры и клавиатура опираются на семантику: `<button>` фокусируется и реагирует на Enter/Space из коробки, `<nav>`/`<main>` дают навигацию по landmark'ам, заголовки — структуру. `<div>` с `@click` ничего этого не даёт.

**7. Что такое skip-link и зачем он нужен?**
Скрытая до фокуса ссылка «перейти к содержимому», позволяющая клавиатурным/скринридер-пользователям пропустить повторяющуюся навигацию и сразу попасть в основной контент.

**8. Почему в SPA нужно вручную управлять фокусом при навигации?**
SPA не перезагружает страницу, поэтому фокус остаётся на старом элементе, а скринридер не объявляет новую страницу. В `router.afterEach` нужно перенести фокус на основной контейнер/заголовок и обновить `document.title`.

**9. Когда использовать ARIA, а когда нет?**
Правило: не использовать ARIA, если есть нативный HTML-элемент с нужной семантикой. ARIA нужна для кастомных компонентов (аккордеоны, табы, диалоги), live-региона для динамических объявлений, `aria-label` для иконок-кнопок. Неправильная ARIA хуже её отсутствия.

**10. Как обеспечить доступность форм?**
Каждое поле — со связанным `<label for>`/`id`, `autocomplete`, при ошибке — `aria-invalid` и `aria-describedby` на сообщение с `role="alert"`. Не полагаться только на placeholder вместо label.

**11. Как проверить доступность приложения?**
Автоматически — Lighthouse, axe DevTools, `eslint-plugin-vuejs-accessibility`. Вручную — пройти по приложению только клавиатурой (Tab/Enter/Esc, видимый фокус) и включить скринридер. Проверить контраст (4.5:1 для текста).

**12. Какие требования к контрасту и что важно помнить про цвет?**
WCAG AA: 4.5:1 для обычного текста, 3:1 для крупного. Нельзя передавать информацию только цветом (ошибки/статусы дублировать текстом, иконкой или паттерном) — для дальтоников и скринридеров.

## Подводные камни (gotchas)

- **Публикация source maps на CDN** — раскрытие исходного кода; используйте `hidden`.
- **`index.html` с долгим кэшем** — пользователи застревают на старой версии; должен быть `no-cache`.
- **Отсутствие SPA-fallback** на сервере — прямой заход на `/route` даёт 404.
- **`errorHandler` без отправки в мониторинг** — ошибки в проде теряются.
- **Ошибки вне Vue** (async/промисы) не ловятся `errorHandler` — нужны `window` слушатели.
- **`<div @click>` вместо `<button>`** — недоступно с клавиатуры и для скринридеров.
- **Фокус не переносится при SPA-навигации** — скринридер не объявляет новую страницу.
- **ARIA поверх нативной семантики** — дублирование/конфликт ролей, хуже чем без ARIA.
- **Placeholder вместо label** — поле без объявляемого назначения.
- **Невидимый фокус** (`outline: none` без замены) — клавиатурные пользователи теряются.
- **Информация только цветом** — недоступна дальтоникам/скринридерам.
- **Пропуски уровней заголовков** (h1 → h3) — ломают структуру для скринридеров.
- **`app.config.performance = true` в проде** — лишние накладные расходы.

## Лучшие практики

**Продакшен:**
1. `vite build` для прода; проверяйте размер бандла.
2. Content hash в именах + `immutable` кэш для ассетов, `no-cache` для `index.html`.
3. `sourcemap: 'hidden'`, загрузка карт только в мониторинг.
4. `app.config.errorHandler` + `window` слушатели + `onErrorCaptured` для границ ошибок.
5. Dev-инструменты и `performance` — только под `import.meta.env.DEV`.
6. SPA-fallback на сервере для history-режима.

**Доступность:**
7. Семантичный HTML по умолчанию; ARIA — только когда семантики не хватает.
8. Skip-link к основному контенту.
9. Перенос фокуса и обновление `document.title` при роутинге.
10. Все формы с `label`, ошибки через `aria-invalid`/`role="alert"`.
11. Полная работа с клавиатуры + видимый `:focus-visible`.
12. Контраст по WCAG; информация не только цветом.
13. Аудит: Lighthouse/axe + ручная проверка клавиатурой и скринридером.

## Шпаргалка

| Тема | Приём |
|------|-------|
| Прод-сборка | `vite build`, `NODE_ENV=production` авто |
| Минификация | `build.minify: 'esbuild'` |
| Source maps | `sourcemap: 'hidden'` |
| Кэш ассетов | `[hash]` в имени + `immutable, max-age=1y` |
| Кэш index.html | `no-cache` |
| Глобальные ошибки | `app.config.errorHandler` |
| Границы ошибок | `onErrorCaptured` (возврат `false`) |
| Ошибки вне Vue | `window` `error`/`unhandledrejection` |
| Dev-флаги | под `import.meta.env.DEV` |
| Семантика | `<button>`, `<nav>`, `<main>`, заголовки |
| Skip-link | `<a href="#main">` скрытый до фокуса |
| Фокус при роутинге | `router.afterEach` → `main.focus()` |
| ARIA | только без нативной семантики |
| Live region | `aria-live="polite"` |
| Иконка-кнопка | `aria-label` + `aria-hidden` на svg |
| Формы | `<label for>`, `aria-invalid`, `role="alert"` |
| Клавиатура | Tab/Enter/Esc, `:focus-visible` |
| Контраст | 4.5:1 текст; не только цветом |
| Аудит | Lighthouse, axe, eslint-plugin-vuejs-accessibility |

**Главное:** прод-сборка убирает dev-оверхед, кэширование строится на content hash, ошибки централизуются через `errorHandler`. Доступность начинается с семантичного HTML; ARIA, skip-links, управление фокусом при роутинге, корректные формы и контраст делают приложение пригодным для всех. Проверяйте автоматически (Lighthouse/axe) и вручную (клавиатура/скринридер).
