# Vue 3 — Асинхронные компоненты — конспект и вопросы

## О чём раздел

Асинхронные компоненты позволяют загружать компонент **лениво** — не при старте приложения, а только когда он действительно нужен. Это основа **code splitting**: вместо одного большого бандла приложение разбивается на чанки, которые подгружаются по требованию. Результат — меньший начальный размер бандла и быстрее первая отрисовка (TTI/FCP).

Vue 3 предоставляет `defineAsyncComponent` — обёртку, которая принимает функцию-загрузчик (обычно динамический `import()`) и возвращает компонент-плейсхолдер. Vue сам управляет состояниями загрузки: ожидание, успех, ошибка. Дополнительно можно задать компонент-«заглушку» на время загрузки, компонент ошибки, задержки и таймауты.

Темы раздела: `defineAsyncComponent`, ленивая загрузка и code splitting, состояния `loading`/`error` с `delay`/`timeout`, связь с `<Suspense>`, ленивые маршруты во Vue Router.

## Ключевые концепции

- **`defineAsyncComponent(loader)`** — создаёт асинхронный компонент из функции-загрузчика, возвращающей `Promise` с компонентом.
- **Динамический `import()`** — стандартный механизм, который бандлер (Vite/webpack) превращает в отдельный чанк.
- **Code splitting** — разбиение бандла на части, подгружаемые по требованию.
- **Расширенные опции**: `loadingComponent`, `errorComponent`, `delay`, `timeout`, `onError`, `suspensible`.
- **`delay`** — задержка перед показом loading-компонента (чтобы не мигал при быстрой загрузке).
- **`timeout`** — время, после которого загрузка считается провалившейся → показывается errorComponent.
- **`<Suspense>`** — встроенный компонент для координации асинхронных зависимостей (async setup и async-компонентов) с единым fallback'ом.
- **Lazy routes** — асинхронная загрузка компонентов маршрутов во Vue Router через `component: () => import(...)`.

## Подробный разбор с примерами кода

### 1. Базовый defineAsyncComponent

```js
import { defineAsyncComponent } from 'vue'

// Загрузчик возвращает Promise (динамический import).
// Бандлер вынесет HeavyChart в отдельный чанк.
const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)
```

Использование такое же, как у обычного компонента:

```vue
<script setup>
import { defineAsyncComponent, ref } from 'vue'

const showChart = ref(false)

const HeavyChart = defineAsyncComponent(() =>
  import('./components/HeavyChart.vue')
)
</script>

<template>
  <button @click="showChart = true">Показать график</button>
  <!-- Чанк HeavyChart загрузится только при первом рендере -->
  <HeavyChart v-if="showChart" />
</template>
```

### 2. Ленивая загрузка и code splitting

Динамический `import('./X.vue')` — это сигнал бандлеру создать отдельный чанк. До тех пор, пока компонент не понадобится, его код не загружается. Это уменьшает начальный бандл.

Типичные кандидаты на ленивую загрузку:
- тяжёлые компоненты (графики, редакторы, карты);
- модалки и диалоги, которые открываются по действию;
- содержимое вкладок/табов;
- редко используемые экраны и админ-панели.

### 3. Состояния loading / error с расширенными опциями

Вместо функции можно передать **объект-конфиг**:

```js
import { defineAsyncComponent } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'
import ErrorBlock from './ErrorBlock.vue'

const AsyncDashboard = defineAsyncComponent({
  // Функция-загрузчик
  loader: () => import('./Dashboard.vue'),

  // Компонент, показываемый во время загрузки
  loadingComponent: LoadingSpinner,
  // Задержка перед показом loading (мс). По умолчанию 200.
  // Если компонент успел загрузиться быстрее — спиннер не мигнёт.
  delay: 200,

  // Компонент ошибки (если загрузка упала или вышел timeout)
  errorComponent: ErrorBlock,
  // Таймаут загрузки (мс). По умолчанию: Infinity (не истекает).
  timeout: 3000
})
```

Логика состояний:
1. Сразу после монтирования стартует загрузка.
2. Если за `delay` мс не загрузилось — показывается `loadingComponent`.
3. Если загрузилось — рендерится сам компонент.
4. Если упало с ошибкой или прошло `timeout` мс — показывается `errorComponent`. В `errorComponent` приходит prop `error` с объектом ошибки.

`errorComponent` получает информацию об ошибке:

```vue
<!-- ErrorBlock.vue -->
<script setup>
defineProps({
  // Vue передаёт сюда объект ошибки
  error: { type: Error, default: null }
})
</script>

<template>
  <div class="error">
    Не удалось загрузить компонент: {{ error?.message }}
  </div>
</template>
```

### 4. Обработка ошибок и повторные попытки (onError)

```js
const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  onError(error, retry, fail, attempts) {
    // Например, повторяем загрузку до 3 раз при сетевой ошибке
    if (attempts <= 3) {
      retry()
    } else {
      fail()
    }
  }
})
```

`retry()` — повторить загрузку; `fail()` — окончательно провалить; `attempts` — номер попытки.

### 5. Связь с `<Suspense>`

`<Suspense>` — встроенный компонент, который координирует **несколько асинхронных зависимостей** в поддереве и показывает единый fallback, пока они не разрешатся. Он работает с:
- компонентами с **async `setup()`** (использующими `await` на верхнем уровне);
- асинхронными компонентами (`defineAsyncComponent`).

```vue
<template>
  <Suspense>
    <!-- Основной контент: может содержать async-компоненты / async setup -->
    <template #default>
      <AsyncDashboard />
    </template>

    <!-- Fallback, пока зависимости грузятся -->
    <template #fallback>
      <LoadingSpinner />
    </template>
  </Suspense>
</template>
```

Важный нюанс взаимодействия:
- По умолчанию async-компонент **внутри** `<Suspense>` становится «suspensible»: его собственные `loadingComponent`/`delay`/`timeout` **игнорируются**, а управление загрузкой берёт на себя `<Suspense>` (показывает свой `#fallback`).
- Чтобы async-компонент управлял своим состоянием сам даже внутри Suspense, задают опцию `suspensible: false`.

Компонент с async setup, работающий с Suspense:

```vue
<!-- Dashboard.vue -->
<script setup>
// Топ-левел await делает setup асинхронным.
// <Suspense> покажет fallback, пока промис не разрешится.
const res = await fetch('/api/stats')
const stats = await res.json()
</script>

<template>
  <div>Всего: {{ stats.total }}</div>
</template>
```

Обработку ошибок async setup ловят через `onErrorCaptured` на родителе или `errorComponent`/error-границы.

### 6. Использование с роутером (lazy routes)

Vue Router нативно поддерживает ленивую загрузку — достаточно передать функцию, возвращающую `import()`. `defineAsyncComponent` для маршрутов **не нужен**: роутер сам обрабатывает промис.

```js
// router.js
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/',
    // Обычный (eager) компонент — попадёт в основной бандл
    component: () => import('./views/Home.vue')
  },
  {
    path: '/admin',
    // Ленивый чанк: загрузится только при переходе на /admin
    component: () => import('./views/Admin.vue')
  },
  {
    path: '/reports',
    // Группировка чанков (webpack magic comment); Vite использует свою стратегию
    component: () => import(/* webpackChunkName: "reports" */ './views/Reports.vue')
  }
]

export const router = createRouter({
  history: createWebHistory(),
  routes
})
```

Состояния загрузки маршрутов обычно показывают через глобальный индикатор (`router.beforeEach` + прогресс-бар) или оборачивают `<RouterView>` в `<Suspense>`:

```vue
<template>
  <Suspense>
    <template #default>
      <RouterView />
    </template>
    <template #fallback>
      <PageLoader />
    </template>
  </Suspense>
</template>
```

## Полный рабочий пример

Ленивый «тяжёлый» компонент с loading/error-состояниями, ретраями и интеграцией с Suspense на уровне страницы.

```vue
<!-- LoadingSpinner.vue -->
<template>
  <div class="spinner">Загрузка…</div>
</template>
```

```vue
<!-- ErrorBlock.vue -->
<script setup>
defineProps({ error: { type: Error, default: null } })
</script>

<template>
  <div class="error">Ошибка загрузки: {{ error?.message }}</div>
</template>
```

```vue
<!-- App.vue -->
<script setup>
import { defineAsyncComponent, ref } from 'vue'
import LoadingSpinner from './LoadingSpinner.vue'
import ErrorBlock from './ErrorBlock.vue'

const show = ref(false)

// Асинхронный компонент с полным набором опций
const HeavyChart = defineAsyncComponent({
  loader: () => import('./HeavyChart.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorBlock,
  delay: 200,        // не мигать спиннером при быстрой загрузке
  timeout: 5000,     // через 5с считаем загрузку проваленной
  // Вне Suspense будет использовать loadingComponent;
  // внутри Suspense по умолчанию делегирует ему управление
  suspensible: true,
  onError(error, retry, fail, attempts) {
    // Ретраим сетевые ошибки до 2 раз
    if (attempts <= 2) retry()
    else fail()
  }
})
</script>

<template>
  <button @click="show = !show">
    {{ show ? 'Скрыть' : 'Показать' }} график
  </button>

  <!-- Чанк HeavyChart загрузится при первом show=true -->
  <HeavyChart v-if="show" />
</template>
```

```vue
<!-- Вариант с Suspense на уровне страницы -->
<!-- DashboardPage.vue -->
<script setup>
import { defineAsyncComponent } from 'vue'

// Внутри Suspense это поведение координируется единым fallback'ом
const AsyncWidget = defineAsyncComponent(() => import('./Widget.vue'))
</script>

<template>
  <Suspense>
    <template #default>
      <AsyncWidget />
    </template>
    <template #fallback>
      <p>Подготавливаем дашборд…</p>
    </template>
  </Suspense>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое асинхронный компонент и зачем он нужен?**
Компонент, который загружается лениво — только когда понадобится, а не при старте приложения. Создаётся через `defineAsyncComponent`. Нужен для code splitting: уменьшает начальный бандл и ускоряет первую отрисовку.

**2. Как `defineAsyncComponent` связан с code splitting?**
Он принимает функцию-загрузчик с динамическим `import()`. Бандлер (Vite/webpack) видит `import()` и выносит компонент в отдельный чанк, который подгружается по требованию.

**3. Какие расширенные опции принимает defineAsyncComponent?**
`loader`, `loadingComponent`, `errorComponent`, `delay` (задержка перед спиннером), `timeout` (лимит загрузки), `onError` (обработчик с retry/fail), `suspensible`.

**4. Зачем нужен `delay`?**
Чтобы избежать мигания loading-компонента при быстрой загрузке. Спиннер показывается, только если загрузка длится дольше `delay` (по умолчанию 200мс).

**5. Что происходит при `timeout`?**
Если компонент не загрузился за `timeout` мс, загрузка считается провалившейся и показывается `errorComponent`. По умолчанию `timeout` = бесконечность.

**6. Как errorComponent узнаёт о причине ошибки?**
Ему передаётся prop `error` с объектом ошибки.

**7. Что такое `<Suspense>` и как он связан с async-компонентами?**
Встроенный компонент, координирующий асинхронные зависимости (async `setup()` и async-компоненты) в поддереве и показывающий единый `#fallback`, пока они не разрешатся. Async-компонент внутри Suspense по умолчанию делегирует управление загрузкой ему.

**8. Что произойдёт с loadingComponent/delay у async-компонента внутри Suspense?**
По умолчанию они игнорируются — Suspense сам показывает свой fallback. Чтобы вернуть управление компоненту, ставят `suspensible: false`.

**9. Как реализуется async setup и при чём тут Suspense?**
Компонент с топ-левел `await` в `<script setup>` имеет асинхронный setup. Его нужно оборачивать в `<Suspense>`, который покажет fallback до разрешения промиса.

**10. Нужен ли `defineAsyncComponent` для ленивых маршрутов?**
Нет. Vue Router сам поддерживает ленивую загрузку: `component: () => import('./View.vue')`. Роутер обрабатывает промис самостоятельно.

**11. Как обрабатывать ошибки/ретраи загрузки?**
Через опцию `onError(error, retry, fail, attempts)`: можно повторить (`retry`), окончательно провалить (`fail`) или принять решение по номеру попытки.

**12. Где разумно применять async-компоненты?**
Тяжёлые виджеты (графики, редакторы, карты), модалки по требованию, содержимое вкладок, редкие экраны, админ-панели.

**13. Можно ли использовать именованный экспорт в загрузчике?**
Да, нужно вернуть промис, резолвящийся в объект с дефолтным компонентом: `.then(m => m.MyComponent)` или `import(...).then(m => ({ default: m.MyComponent }))`.

**14. Чем lazy route отличается от async-компонента?**
Lazy route — это ленивая загрузка на уровне маршрута, управляемая роутером (без defineAsyncComponent). Async-компонент — ленивый компонент в любом месте шаблона с собственными состояниями loading/error. Их можно сочетать.

## Подводные камни (gotchas)

- **`import('./X.vue')` без `defineAsyncComponent` не делает компонент асинхронным** в шаблоне — он просто промис. Для использования как компонента нужен `defineAsyncComponent` (кроме роутера, который сам это обрабатывает).
- **loadingComponent/delay/timeout игнорируются внутри Suspense** по умолчанию. Если нужны собственные состояния — `suspensible: false`.
- **Создание async-компонента в render-функции/inline.** Не вызывайте `defineAsyncComponent` внутри рендера или `v-for` — он пересоздаётся на каждый рендер и теряет кэш загрузки. Определяйте на уровне модуля/setup.
- **Ошибки async setup нужно ловить отдельно** — через `onErrorCaptured` на родителе; они не попадают в `errorComponent` async-компонента автоматически.
- **`timeout` по умолчанию бесконечен** — без явного значения «зависшая» загрузка никогда не покажет errorComponent.
- **Мигание спиннера** при слишком маленьком `delay` для быстрых сетей. По умолчанию 200мс — обычно норма.
- **SSR + Suspense** имеет нюансы (Suspense всё ещё помечен как экспериментальный API) — проверяйте поведение при server-side rendering.
- **Дублирование чанков**: один и тот же `import()` в разных местах бандлер обычно дедуплицирует, но следите за стратегией чанкинга для оптимального кэширования.

## Лучшие практики

- Лениво грузите **тяжёлые и редко используемые** компоненты (графики, редакторы, модалки, админка), а не всё подряд — мелкие компоненты дешевле держать в основном бандле.
- Задавайте **`errorComponent` и `timeout`**, чтобы пользователь не «завис» на бесконечной загрузке.
- Используйте **`delay`** (по умолчанию 200мс), чтобы спиннер не мигал при быстрой загрузке.
- Для **маршрутов** используйте `() => import(...)` напрямую — не оборачивайте в `defineAsyncComponent`.
- Объявляйте async-компоненты **на уровне модуля/setup**, а не в рендере, чтобы сохранить кэш.
- Сочетайте с **`<Suspense>`** для скоординированных загрузок и единого fallback'а на уровне страницы/секции.
- Добавляйте **ретраи** через `onError` для устойчивости к сетевым сбоям.
- Следите за **стратегией чанков** (имена/группировка) ради эффективного кэширования.

## Шпаргалка

```js
import { defineAsyncComponent } from 'vue'

// Минимально
const A = defineAsyncComponent(() => import('./A.vue'))

// С опциями
const B = defineAsyncComponent({
  loader: () => import('./B.vue'),
  loadingComponent: Spinner,   // во время загрузки
  errorComponent: ErrorView,   // при ошибке/timeout (получает prop error)
  delay: 200,                  // задержка перед спиннером, мс
  timeout: 5000,               // лимит загрузки, мс
  suspensible: true,           // делегировать управление Suspense (по умолч.)
  onError(err, retry, fail, attempts) { attempts <= 2 ? retry() : fail() }
})
```

```vue
<!-- Suspense -->
<Suspense>
  <template #default><AsyncComp /></template>
  <template #fallback><Loader /></template>
</Suspense>
```

```js
// Lazy route — БЕЗ defineAsyncComponent
{ path: '/admin', component: () => import('./Admin.vue') }
```

Краткие правила:
- `defineAsyncComponent(() => import(...))` = ленивый компонент + чанк.
- `delay`/`timeout`/`loadingComponent`/`errorComponent` управляют состояниями.
- `<Suspense>` координирует async зависимости и перекрывает локальный loading (если `suspensible`).
- Маршруты лениво грузятся через `() => import(...)` без обёртки.
