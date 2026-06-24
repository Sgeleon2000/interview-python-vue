# Vue 3 — Хуки жизненного цикла — конспект и вопросы

## О чём раздел

Жизненный цикл компонента — это набор этапов, через которые проходит экземпляр компонента: создание, монтирование в DOM, обновление при изменении реактивных данных и размонтирование. На каждом этапе Vue вызывает соответствующий **хук жизненного цикла** (lifecycle hook), куда вы встраиваете свою логику: загрузку данных, подписки, работу с DOM, очистку ресурсов.

В Composition API хуки регистрируются функциями `onMounted`, `onUnmounted` и т.д. внутри `setup()` (или `<script setup>`). В Options API это методы объекта компонента: `mounted`, `unmounted` и т.д.

## Ключевые концепции

- **Регистрация хука — это побочный эффект `setup()`.** Хук связывается с текущим активным экземпляром, поэтому хуки должны вызываться синхронно в `setup`.
- **Composition ↔ Options.** Почти все хуки имеют префикс `on` в Composition API. Исключение — нет `onSetup`/`onCreated`: сам код `<script setup>` выполняется на стадии `created`.
- **DOM доступен начиная с `onMounted`.** До монтирования `this.$el` / template refs ещё `null`.
- **Порядок parent/child.** При монтировании: `beforeMount` родителя → (рекурсивно дети) → `mounted` детей → `mounted` родителя. То есть дети монтируются раньше родителя.
- **Очистка ресурсов** (таймеры, слушатели, сокеты) обязательно в `onBeforeUnmount` / `onUnmounted`, иначе утечки памяти.

## Подробный разбор с примерами кода

### Полный список хуков и соответствие Options API

| Composition API      | Options API       | Когда вызывается                                              | DOM доступен |
|----------------------|-------------------|--------------------------------------------------------------|--------------|
| (код `<script setup>`)| `beforeCreate`+`created` | Инициализация реактивности, до рендера              | Нет          |
| `onBeforeMount`      | `beforeMount`     | Перед первым рендером в DOM                                   | Нет          |
| `onMounted`          | `mounted`         | После вставки компонента в DOM                                | **Да**       |
| `onBeforeUpdate`     | `beforeUpdate`    | Перед патчингом DOM из-за изменения реактивных данных         | Да (старый)  |
| `onUpdated`          | `updated`         | После того как DOM обновлён                                   | Да (новый)   |
| `onBeforeUnmount`    | `beforeUnmount`   | Перед удалением компонента (экземпляр ещё полностью рабочий)  | Да           |
| `onUnmounted`        | `unmounted`       | После удаления компонента и всех его эффектов                 | Нет (удалён) |
| `onErrorCaptured`    | `errorCaptured`   | Перехват ошибки из дочернего компонента                      | —            |
| `onActivated`        | `activated`       | Компонент внутри `<KeepAlive>` активирован (показан)          | Да           |
| `onDeactivated`      | `deactivated`     | Компонент внутри `<KeepAlive>` деактивирован (скрыт, не удалён)| Да          |
| `onRenderTracked`    | `renderTracked`   | (dev) Отслежена реактивная зависимость при рендере           | —            |
| `onRenderTriggered`  | `renderTriggered` | (dev) Зависимость вызвала ре-рендер                          | —            |
| `onServerPrefetch`   | `serverPrefetch`  | SSR: дождаться async-данных перед рендером на сервере         | —            |

### Диаграмма жизненного цикла словами

```
1. Создаётся экземпляр компонента.
2. Инициализируется реактивное состояние (props, setup()).
   --> Выполняется тело <script setup> / setup() (аналог created).
3. Компилируется render-функция.
4. [onBeforeMount] — DOM ещё не создан.
5. Создаётся реальный DOM, монтируются дочерние компоненты.
6. [onMounted] — DOM в документе, refs доступны.
   ... компонент живёт, реагирует на изменения ...
7. Изменились реактивные данные:
   [onBeforeUpdate] --> патчинг Virtual DOM --> реальный DOM --> [onUpdated]
   ... может повторяться много раз ...
8. Компонент удаляется:
   [onBeforeUnmount] — экземпляр ещё работает.
   --> Vue останавливает все эффекты (watchers, computed), размонтирует детей.
   [onUnmounted] — компонент полностью удалён.
```

### Базовый пример (`<script setup>`)

```vue
<script setup>
import { ref, onBeforeMount, onMounted, onBeforeUpdate, onUpdated, onBeforeUnmount, onUnmounted } from 'vue'

const count = ref(0)
const el = ref(null) // template ref

// Аналог created — просто код в теле setup
console.log('created: реактивность готова, DOM ещё нет')

onBeforeMount(() => {
  console.log('beforeMount: el.value =', el.value) // null
})

onMounted(() => {
  console.log('mounted: el.value =', el.value)     // <p> доступен
})

onBeforeUpdate(() => {
  console.log('beforeUpdate: DOM ещё со старым значением')
})

onUpdated(() => {
  console.log('updated: DOM уже обновлён')
})

onBeforeUnmount(() => {
  console.log('beforeUnmount: можно ещё читать состояние')
})

onUnmounted(() => {
  console.log('unmounted: компонент удалён, чистим ресурсы')
})
</script>

<template>
  <p ref="el">{{ count }}</p>
  <button @click="count++">+1</button>
</template>
```

### Тот же компонент на Options API (для сравнения)

```vue
<script>
export default {
  data() {
    return { count: 0 }
  },
  created() {
    console.log('created')
  },
  mounted() {
    console.log('mounted', this.$refs.el)
  },
  updated() {
    console.log('updated')
  },
  unmounted() {
    console.log('unmounted')
  }
}
</script>
```

### Типичные задачи в `onMounted` и очистка в `onUnmounted`

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const users = ref([])
const width = ref(window.innerWidth)
let timerId = null

// 1. Запрос данных (компонент уже в DOM, можно показать лоадер)
onMounted(async () => {
  const res = await fetch('/api/users')
  users.value = await res.json()
})

// 2. Подписка на глобальные события
const onResize = () => { width.value = window.innerWidth }
onMounted(() => {
  window.addEventListener('resize', onResize)
  timerId = setInterval(() => console.log('tick'), 1000)
})

// 3. ОБЯЗАТЕЛЬНАЯ очистка — иначе утечка памяти и «зомби»-слушатели
onUnmounted(() => {
  window.removeEventListener('resize', onResize)
  clearInterval(timerId)
})
</script>
```

> Хороший паттерн — выносить «подписка + очистка» в composable: подписку делаем в `onMounted`, очистку в `onUnmounted` внутри одной функции `useEventListener(...)`.

### Порядок выполнения parent/child

```
МОНТИРОВАНИЕ (mount):
  Parent setup/created
  Parent onBeforeMount
    Child setup/created
    Child onBeforeMount
    Child onMounted        <-- дети монтируются ПЕРВЫМИ
  Parent onMounted         <-- родитель ПОСЛЕДНИМ

РАЗМОНТИРОВАНИЕ (unmount):
  Parent onBeforeUnmount   <-- родитель ПЕРВЫМ начинает
    Child onBeforeUnmount
    Child onUnmounted
  Parent onUnmounted       <-- родитель ПОСЛЕДНИМ завершает

ОБНОВЛЕНИЕ (update):
  Parent onBeforeUpdate
    Child onBeforeUpdate (если проп изменился)
    Child onUpdated
  Parent onUpdated
```

Логика: чтобы `onMounted` родителя гарантированно видел смонтированный DOM детей, дети должны смонтироваться раньше.

### `onErrorCaptured` — перехват ошибок

```vue
<script setup>
import { ref, onErrorCaptured } from 'vue'

const error = ref(null)

onErrorCaptured((err, instance, info) => {
  // err — объект ошибки, instance — компонент-источник, info — тип хука/контекст
  error.value = err
  console.error('Поймали ошибку из потомка:', err, info)
  return false // вернуть false — остановить всплытие ошибки выше
})
</script>

<template>
  <div v-if="error">Что-то пошло не так</div>
  <slot v-else />
</template>
```

Особенности: ловит ошибки из дочерних компонентов (рендер, хуки, обработчики событий, watchers). Не ловит синхронные ошибки в самом теле `setup` этого же компонента. Возврат `false` останавливает дальнейшее всплытие.

### `onActivated` / `onDeactivated` и `<KeepAlive>`

При оборачивании компонента в `<KeepAlive>` он не размонтируется при переключении, а **кешируется**. Вместо `unmounted/mounted` срабатывают `onDeactivated/onActivated`.

```vue
<script setup>
import { onActivated, onDeactivated, onMounted, onUnmounted } from 'vue'

onMounted(() => console.log('создан 1 раз при первом показе'))
onActivated(() => console.log('показан (в т.ч. повторно из кеша)'))
onDeactivated(() => console.log('скрыт, но НЕ удалён — состояние сохранено'))
onUnmounted(() => console.log('удалён из кеша (KeepAlive вытеснил/умер)'))
</script>
```

Используйте `onActivated` для возобновления (перезапрос «свежих» данных, запуск таймеров), `onDeactivated` — для паузы (остановить таймеры, не очищая состояние).

### Асинхронная регистрация хуков (ограничения)

Хук **должен** регистрироваться синхронно во время выполнения `setup`, потому что Vue определяет «текущий активный экземпляр» именно в этот момент. После `await` контекст экземпляра теряется.

```vue
<script setup>
import { onMounted } from 'vue'

// ВЕРНО: синхронно в setup
onMounted(() => console.log('ok'))

async function init() {
  const data = await fetch('/api')
  // НЕВЕРНО: после await активного экземпляра уже нет
  onMounted(() => console.log('не сработает / предупреждение')) // ❌
}
init()
</script>
```

Исключение — хуки можно регистрировать синхронно внутри composable, вызванной синхронно из `setup`:

```js
// useFeature.js
import { onMounted } from 'vue'
export function useFeature() {
  onMounted(() => { /* ok: useFeature() вызвана синхронно из setup */ })
}
```

Если нужна асинхронная инициализация с сохранением экземпляра — используйте `<Suspense>` с top-level `await` или сохраняйте экземпляр через `getCurrentInstance()` до `await` (продвинутый приём, обычно не нужен).

## Полный рабочий пример

```vue
<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

// Композабл: подписка на событие с автоматической очисткой
function useEventListener(target, event, handler) {
  onMounted(() => target.addEventListener(event, handler))
  onBeforeUnmount(() => target.removeEventListener(event, handler))
}

const x = ref(0)
const y = ref(0)
const loading = ref(true)
const stats = ref(null)

// 1. Слежение за мышью с гарантированной очисткой
useEventListener(window, 'mousemove', (e) => {
  x.value = e.clientX
  y.value = e.clientY
})

// 2. Загрузка данных после монтирования
onMounted(async () => {
  try {
    const res = await fetch('/api/stats')
    stats.value = await res.json()
  } finally {
    loading.value = false
  }
})
</script>

<template>
  <section>
    <p>Курсор: {{ x }}, {{ y }}</p>
    <p v-if="loading">Загрузка статистики…</p>
    <pre v-else>{{ stats }}</pre>
  </section>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `created` отличается от `mounted`?**
В `created` (тело `setup`) реактивность уже работает, но DOM ещё не создан — `$refs` пусты. В `mounted` компонент уже в DOM, можно обращаться к элементам, измерять размеры, инициализировать сторонние библиотеки.

**2. Где делать API-запрос — в `created`/`setup` или в `mounted`?**
Технически можно в обоих. Запрос в `setup` стартует чуть раньше (до рендера). На практике для CSR разница невелика; запрос в `onMounted` удобен, когда нужна гарантия, что DOM (например, контейнер для лоадера) уже есть. Для SSR данные грузят через `serverPrefetch`/`<Suspense>`, т.к. `mounted` на сервере не вызывается.

**3. Какие хуки НЕ вызываются при SSR?**
На сервере не вызываются `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`, `activated`, `deactivated`. Срабатывают только до-рендерные (`setup`/`created`) и `serverPrefetch`. Поэтому код с `window`/`document` должен жить в `onMounted`.

**4. Каков порядок mounted у родителя и детей?**
Сначала `mounted` у всех детей, потом у родителя. Дети монтируются раньше, чтобы родитель в своём `onMounted` видел готовый DOM поддерева.

**5. Как очистить таймер/подписку при удалении компонента?**
В `onUnmounted` (или `onBeforeUnmount`) вызвать `clearInterval`/`removeEventListener`/закрыть WebSocket. Идеально — рядом с местом подписки, через composable.

**6. В чём разница `onBeforeUnmount` и `onUnmounted`?**
В `onBeforeUnmount` экземпляр ещё полностью функционален (реактивность, DOM на месте) — удобно прочитать финальное состояние. В `onUnmounted` все эффекты компонента (watchers, computed, дочерние компоненты) уже остановлены и удалены.

**7. Можно ли вызвать `onMounted` после `await` в setup?**
Нет. Хук теряет связь с экземпляром после `await`. Регистрируйте все хуки синхронно в начале `setup`, а async-логику кладите внутрь колбэка хука.

**8. Чем `unmounted` отличается от `deactivated`?**
`unmounted` — компонент уничтожен, состояние потеряно. `deactivated` (только под `<KeepAlive>`) — компонент скрыт, но закеширован, состояние сохранено; при возврате сработает `activated`, а не `mounted`.

**9. Зачем `onUpdated` и почему в нём опасна мутация данных?**
`onUpdated` срабатывает после каждого патчинга DOM. Если в нём менять реактивное состояние — получите новый ре-рендер и потенциально бесконечный цикл. Для реакции на конкретные данные используйте `watch`, а не `onUpdated`.

**10. Как поймать ошибку дочернего компонента?**
`onErrorCaptured((err, instance, info) => {...})` в родителе. Возврат `false` останавливает всплытие. Это основа «error boundary» во Vue.

**11. Доступен ли DOM в `onBeforeUpdate`?**
Да, но со «старым» содержимым (до применения изменений). В `onUpdated` DOM уже отражает новые данные.

**12. Как Options API соответствует Composition?**
`beforeCreate`+`created` → тело `setup`; `mounted` → `onMounted`; `updated` → `onUpdated`; `unmounted` → `onUnmounted` и т.д. Префикс `on` + PascalCase имя хука.

**13. Что произойдёт, если зарегистрировать `onMounted` дважды?**
Оба колбэка выполнятся в порядке регистрации. Множественная регистрация — норма (например, разные composables добавляют свои `onMounted`).

**14. Гарантирован ли в `onMounted` полностью отрендеренный DOM всего приложения?**
DOM самого компонента и его детей — да. Но если нужно дождаться эффектов после следующего цикла рендера (например, после программного изменения состояния в `onMounted`), используйте `await nextTick()`.

## Подводные камни (gotchas)

- **Обращение к `ref` элемента в `onBeforeMount` / `setup`** — там `null`. DOM появляется только в `onMounted`.
- **Забытая очистка** в `onUnmounted` — типичная утечка памяти: таймеры тикают, слушатели висят на `window`, сокеты не закрыты.
- **Регистрация хука после `await`** — молча не работает (в dev — предупреждение).
- **Тяжёлая логика в `onUpdated`** — вызывается на каждый ре-рендер; легко словить лишние вычисления или бесконечный цикл при мутации стейта.
- **Использование `window`/`document` в `setup`** ломает SSR — переносите в `onMounted`.
- **`<KeepAlive>`**: ожидание `mounted` при возврате на закешированный компонент — его не будет, используйте `onActivated`.
- **`nextTick`**: изменили данные и сразу читаете DOM — он ещё старый; ждите `await nextTick()`.

## Лучшие практики

- Регистрируйте все хуки **синхронно** в начале `setup`/`<script setup>`.
- Парность «подписка ↔ очистка»: всё, что создаёте в `onMounted`, убирайте в `onUnmounted`.
- Инкапсулируйте побочные эффекты в composables (`useXxx`) — хуки внутри них тоже срабатывают.
- Для реакции на изменение конкретных данных используйте `watch`/`watchEffect`, а не `onUpdated`.
- DOM-измерения и интеграцию со сторонними JS-библиотеками делайте в `onMounted` (+ `nextTick` при необходимости).
- Для async-инициализации компонента предпочитайте `<Suspense>` и top-level `await`.
- Добавляйте «error boundary» через `onErrorCaptured` на верхних уровнях UI.

## Шпаргалка

```text
СОЗДАНИЕ:    setup()/created  → onBeforeMount → onMounted (DOM готов)
ОБНОВЛЕНИЕ:  onBeforeUpdate (старый DOM) → onUpdated (новый DOM)
УДАЛЕНИЕ:    onBeforeUnmount (ещё живой) → onUnmounted (чистим ресурсы)
KEEPALIVE:   onActivated (показан) / onDeactivated (скрыт, не удалён)
ОШИБКИ:      onErrorCaptured(err, inst, info) → return false = стоп всплытие

DOM доступен:  с onMounted; в onBeforeUpdate — старый, в onUpdated — новый
Порядок mount: дети → родитель;  unmount: родитель начинает первым
SSR:           НЕ вызываются mounted/updated/unmounted; есть serverPrefetch
Запросы:       onMounted (CSR) / serverPrefetch + Suspense (SSR)
Очистка:       onUnmounted (clearInterval, removeEventListener, ws.close)
Ограничение:   хуки регистрировать СИНХРОННО в setup (нельзя после await)
nextTick:      ждать обновлённый DOM после смены реактивного состояния
```

```vue
<script setup>
import { onMounted, onUnmounted } from 'vue'
let id
onMounted(() => { id = setInterval(tick, 1000) })   // запуск
onUnmounted(() => clearInterval(id))                 // очистка
function tick() {}
</script>
```
