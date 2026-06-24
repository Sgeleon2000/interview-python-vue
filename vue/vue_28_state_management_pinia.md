# Vue 3 — Управление состоянием (Pinia) — конспект и вопросы

## О чём раздел

По мере роста приложения состояние перестаёт «жить» внутри одного компонента: одни и те же
данные (текущий пользователь, корзина, тема, флаги) нужны в разных частях дерева компонентов.
Передавать их через пропсы вниз и события вверх становится тяжело — это **prop drilling**
(пробрасывание пропсов через 5–6 уровней компонентов, которым эти данные не нужны, лишь чтобы
донести их до глубокого потомка). Глобальный стор решает эту проблему: состояние хранится в
одном месте, а любой компонент подписывается на нужный кусок напрямую.

**Pinia** — официальная библиотека управления состоянием для Vue 3 (рекомендована командой Vue
вместо Vuex). Она построена на реактивности Vue (`reactive`/`ref`), имеет минимальный API,
полноценную поддержку TypeScript, devtools, плагинов и поддерживает оба стиля объявления стора:
Options-подобный (option stores) и функциональный на Composition API (setup stores).

На собеседовании по этой теме почти гарантированно спросят про `storeToRefs` (почему нельзя
просто деструктурировать стор) и про отличия Pinia от Vuex — этим вопросам ниже посвящены
отдельные подробные разделы.

## Ключевые концепции

- **Стор (store)** — это объект, хранящий состояние и логику его изменения, доступный из любого
  компонента. Создаётся через `defineStore(id, ...)`, где `id` — уникальный строковый идентификатор.
- **Три кита стора**: `state` (реактивные данные), `getters` (производные значения, аналог
  `computed`), `actions` (методы, в т.ч. асинхронные, аналог методов/мутаций).
- В Pinia **нет отдельных мутаций** (в отличие от Vuex). State можно менять напрямую или внутри
  actions — обе операции реактивны и отслеживаются.
- **`storeToRefs`** превращает свойства стора в `ref`-ы, сохраняя реактивность при деструктуризации.
  Прямая деструктуризация `const { count } = store` ломает реактивность.
- **Setup stores** (стиль `<script setup>`): `ref` → state, `computed` → getters, функции →
  actions. Гибче option stores: можно использовать `watch`, локальные переменные, любые composables.
- **Модульность**: каждый стор — отдельный файл/функция. Сторы можно использовать внутри других
  сторов, просто вызвав `useOtherStore()` внутри.
- **Плагины Pinia** позволяют расширять каждый стор (persist в localStorage, логирование, доп. свойства).
- Альтернатива «лёгкого веса» для простых случаев — общий `reactive()`-объект + `provide`/`inject`
  или просто экспортируемый из модуля.

## Подробный разбор с примерами кода

### Установка и подключение

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
// createPinia() создаёт корневой инстанс Pinia (контейнер всех сторов)
app.use(createPinia())
app.mount('#app')
```

### Option store (Options-подобный синтаксис)

```js
// stores/counter.js
import { defineStore } from 'pinia'

// Первый аргумент — уникальный id стора (используется в devtools и для SSR)
export const useCounterStore = defineStore('counter', {
  // state — ОБЯЗАТЕЛЬНО функция-фабрика, возвращающая объект.
  // Функция нужна, чтобы каждый раз создавалось своё состояние (важно для SSR).
  state: () => ({
    count: 0,
    name: 'Счётчик',
  }),

  // getters — производные значения, кешируются как computed.
  getters: {
    // Первый аргумент — state
    doubleCount: (state) => state.count * 2,

    // Чтобы обратиться к другому геттеру, используем this (тип выводится автоматически)
    doublePlusOne() {
      return this.doubleCount + 1
    },
  },

  // actions — методы (синхронные и асинхронные). this указывает на стор.
  actions: {
    increment() {
      this.count++ // мутируем state напрямую — это реактивно
    },
    async fetchAndAdd() {
      const res = await fetch('/api/value')
      const { value } = await res.json()
      this.count += value
    },
  },
})
```

### Setup store (Composition API синтаксис) — современный предпочтительный стиль

```js
// stores/counter.js
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCounterStore = defineStore('counter', () => {
  // ref/reactive  -> state
  const count = ref(0)
  const name = ref('Счётчик')

  // computed -> getters
  const doubleCount = computed(() => count.value * 2)

  // функции -> actions
  function increment() {
    count.value++
  }

  async function fetchAndAdd() {
    const res = await fetch('/api/value')
    const { value } = await res.json()
    count.value += value
  }

  // ВАЖНО: нужно вернуть всё, что должно быть доступно снаружи стора.
  // Не возвращённое останется приватным (полезно для внутренних helper-ов).
  return { count, name, doubleCount, increment, fetchAndAdd }
})
```

> Когда что выбирать: **option store** ближе к Vuex/Options API, проще для новичков. **Setup
> store** мощнее: внутри можно использовать `watch`, `watchEffect`, другие composables, делать
> приватное состояние — поэтому в новых проектах чаще выбирают именно его.

### Использование стора в компоненте

```vue
<script setup>
import { useCounterStore } from '@/stores/counter'

// Вызываем хук ВНУТРИ setup — Pinia вернёт единый инстанс стора (синглтон на приложение)
const counter = useCounterStore()

// Доступ к state и getters — как к обычным свойствам реактивного объекта
console.log(counter.count)       // 0
console.log(counter.doubleCount) // 0

// Вызов action
counter.increment()
</script>

<template>
  <p>Счёт: {{ counter.count }}</p>
  <p>Удвоенный: {{ counter.doubleCount }}</p>
  <button @click="counter.increment">+1</button>
</template>
```

### storeToRefs — главная ловушка (топ-вопрос собеседования)

Стор — это **реактивный объект** (по сути обёртка над `reactive`). Если деструктурировать его
напрямую, мы вытащим «сырые» значения и **потеряем реактивность** — компонент перестанет
обновляться:

```vue
<script setup>
import { storeToRefs } from 'pinia'
import { useCounterStore } from '@/stores/counter'

const counter = useCounterStore()

// ❌ НЕПРАВИЛЬНО: реактивность теряется.
// count становится просто числом 0, оторванным от стора.
const { count, doubleCount } = counter

// ✅ ПРАВИЛЬНО: storeToRefs оборачивает каждое реактивное свойство (state и getters) в ref,
// сохраняя связь со стором. В шаблоне .value разворачивается автоматически.
const { count: countRef, doubleCount: doubleRef } = storeToRefs(counter)

// ВАЖНО: actions деструктурировать можно напрямую — это обычные функции, реактивность им не нужна.
const { increment } = counter
</script>

<template>
  <!-- countRef обновляется, а count из прямой деструктуризации — НЕТ -->
  <p>{{ countRef }} / {{ doubleRef }}</p>
  <button @click="increment">+1</button>
</template>
```

**Почему так происходит.** `reactive`-объект реактивен как целое: реактивность даёт Proxy,
который перехватывает обращения к свойствам. Когда вы пишете `const { count } = store`, JS
читает свойство `count` один раз, получает примитив и кладёт его в новую переменную — Proxy
больше не участвует, отслеживания зависимостей нет. `storeToRefs` обходит свойства стора и для
каждого state/getter создаёт `toRef`-обёртку: переменная становится `ref`, при чтении `.value`
обращается обратно к стору через Proxy, и связь сохраняется. (Аналогично ведёт себя `toRefs` для
обычного `reactive`.) `storeToRefs` отличается от `toRefs` тем, что пропускает методы (actions) и
не оборачивает их.

### Прямая мутация и патчинг state

```js
const store = useCounterStore()

// 1) Прямая мутация — самый частый и нормальный способ
store.count++

// 2) $patch объектом — сразу несколько полей одной «транзакцией»
store.$patch({ count: 10, name: 'Новый' })

// 3) $patch функцией — удобно для массивов/сложных изменений
store.$patch((state) => {
  state.count++
  state.list.push({ id: 1 })
})

// 4) Полная замена state
store.$state = { count: 0, name: 'reset' }

// 5) Сброс к начальному состоянию (работает «из коробки» только для option stores)
store.$reset()
```

### Async-actions

Actions — обычные функции, поэтому могут быть `async` без всяких ограничений:

```js
export const useUsersStore = defineStore('users', {
  state: () => ({ list: [], loading: false, error: null }),
  actions: {
    async fetchUsers() {
      this.loading = true
      this.error = null
      try {
        const res = await fetch('/api/users')
        if (!res.ok) throw new Error('Ошибка сети')
        this.list = await res.json()
      } catch (e) {
        this.error = e.message
      } finally {
        this.loading = false
      }
    },
  },
})
```

В компоненте можно дождаться завершения, т.к. action возвращает промис:

```js
const users = useUsersStore()
await users.fetchUsers()
```

### Использование одного стора внутри другого (модульность)

```js
// stores/cart.js
import { defineStore } from 'pinia'
import { useAuthStore } from './auth'

export const useCartStore = defineStore('cart', {
  state: () => ({ items: [] }),
  actions: {
    checkout() {
      // Просто вызываем хук другого стора внутри action
      const auth = useAuthStore()
      if (!auth.isLoggedIn) throw new Error('Нужна авторизация')
      // ... логика оформления
    },
  },
})
```

### Подписки и наблюдение за стором

```js
const store = useCounterStore()

// Подписка на любые изменения state (аналог watch для всего стора)
store.$subscribe((mutation, state) => {
  // mutation.type: 'direct' | 'patch object' | 'patch function'
  localStorage.setItem('counter', JSON.stringify(state))
})

// Подписка на вызовы actions (логирование, метрики)
store.$onAction(({ name, args, after, onError }) => {
  console.log(`action ${name} вызван с`, args)
  after((result) => console.log(`${name} успешно`, result))
  onError((err) => console.warn(`${name} упал`, err))
})
```

### Плагины Pinia

Плагин — функция, получающая контекст и расширяющая каждый стор. Классический пример —
персистентность в localStorage:

```js
// plugins/persist.js
export function persistPlugin({ store }) {
  // Восстанавливаем сохранённое состояние при создании стора
  const saved = localStorage.getItem(store.$id)
  if (saved) store.$patch(JSON.parse(saved))

  // Сохраняем при каждом изменении
  store.$subscribe((_, state) => {
    localStorage.setItem(store.$id, JSON.stringify(state))
  })

  // Можно добавить общее свойство во ВСЕ сторы
  return { sharedHello: 'hi' }
}

// main.js
const pinia = createPinia()
pinia.use(persistPlugin)
```

> На практике для персистентности обычно берут готовый `pinia-plugin-persistedstate`, но понимать
> устройство плагина важно для собеседования.

### Альтернатива без Pinia: reactive + provide/inject

Для небольших приложений или локального «состояния поддерева» глобальный стор избыточен. Можно
сделать простой реактивный стор вручную:

```js
// store/simple.js
import { reactive, readonly } from 'vue'

const state = reactive({ count: 0 })

function increment() {
  state.count++
}

// readonly запрещает менять state мимо action-ов (как «инкапсуляция»)
export const simpleStore = {
  state: readonly(state),
  increment,
}
```

Или через `provide`/`inject`, чтобы состояние было доступно только в поддереве:

```vue
<!-- Provider.vue -->
<script setup>
import { reactive, provide } from 'vue'

const store = reactive({ user: null, login(u) { this.user = u } })
// Ключ лучше делать Symbol, чтобы избежать коллизий
provide('userStore', store)
</script>
```

```vue
<!-- Child.vue (любой глубины) -->
<script setup>
import { inject } from 'vue'
const store = inject('userStore')
</script>
```

Плюсы: ноль зависимостей. Минусы: нет devtools, нет SSR-безопасности «из коробки», нет
структуры getters/actions, нет плагинов — для крупного проекта это всё придётся изобретать заново,
поэтому в реальных приложениях выбирают Pinia.

## Полный рабочий пример

Стор корзины с асинхронной загрузкой товаров, геттерами и персистентностью:

```js
// stores/cart.js
import { ref, computed } from 'vue'
import { defineStore } from 'pinia'

export const useCartStore = defineStore('cart', () => {
  // --- state ---
  const items = ref([])      // [{ id, title, price, qty }]
  const loading = ref(false)
  const error = ref(null)

  // --- getters ---
  const totalCount = computed(() =>
    items.value.reduce((sum, it) => sum + it.qty, 0)
  )
  const totalPrice = computed(() =>
    items.value.reduce((sum, it) => sum + it.price * it.qty, 0)
  )
  const isEmpty = computed(() => items.value.length === 0)

  // --- actions ---
  function addItem(product) {
    const existing = items.value.find((i) => i.id === product.id)
    if (existing) existing.qty++
    else items.value.push({ ...product, qty: 1 })
  }

  function removeItem(id) {
    items.value = items.value.filter((i) => i.id !== id)
  }

  async function loadFromServer() {
    loading.value = true
    error.value = null
    try {
      const res = await fetch('/api/cart')
      if (!res.ok) throw new Error('Не удалось загрузить корзину')
      items.value = await res.json()
    } catch (e) {
      error.value = e.message
    } finally {
      loading.value = false
    }
  }

  return {
    items, loading, error,
    totalCount, totalPrice, isEmpty,
    addItem, removeItem, loadFromServer,
  }
})
```

```vue
<!-- CartWidget.vue -->
<script setup>
import { onMounted } from 'vue'
import { storeToRefs } from 'pinia'
import { useCartStore } from '@/stores/cart'

const cart = useCartStore()

// state и getters — через storeToRefs (сохраняем реактивность при деструктуризации)
const { items, totalCount, totalPrice, loading, isEmpty } = storeToRefs(cart)
// actions — напрямую
const { removeItem, loadFromServer } = cart

onMounted(loadFromServer)
</script>

<template>
  <div class="cart">
    <p v-if="loading">Загрузка…</p>
    <p v-else-if="isEmpty">Корзина пуста</p>
    <ul v-else>
      <li v-for="it in items" :key="it.id">
        {{ it.title }} × {{ it.qty }} — {{ it.price * it.qty }} ₽
        <button @click="removeItem(it.id)">Удалить</button>
      </li>
    </ul>
    <p>Товаров: {{ totalCount }}, на сумму {{ totalPrice }} ₽</p>
  </div>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Зачем нужен стор, если есть пропсы и события?**
Пропсы/события хороши для связи «родитель ↔ прямой потомок». Когда данные нужны множеству
компонентов на разных уровнях, появляется prop drilling (пробрасывание через промежуточные
компоненты) и хрупкая система событий. Стор хранит общее состояние в одном месте, и любой
компонент обращается к нему напрямую, минуя иерархию.

**2. Что такое prop drilling и как стор его решает?**
Это передача данных через цепочку компонентов, которым сами данные не нужны — они лишь
«транзитные». Стор устраняет цепочку: глубокий потомок просто вызывает `useStore()`.

**3. Почему нельзя деструктурировать стор напрямую и для чего `storeToRefs`?**
Стор — реактивный объект на основе Proxy. Деструктуризация читает свойство один раз и кладёт
примитив в переменную — связь с Proxy теряется, реактивности нет. `storeToRefs` оборачивает
каждое state/getter в `ref`, сохраняя связь со стором. Actions при этом можно деструктурировать
напрямую — это обычные функции.

**4. Чем `storeToRefs` отличается от `toRefs`?**
`toRefs` оборачивает все свойства `reactive`-объекта. `storeToRefs` — специально для сторов:
оборачивает только state и getters, **пропуская actions** (методы), чтобы не превращать функции
в бесполезные ref-ы.

**5. В чём разница между option store и setup store?**
Option store описывается объектом с секциями `state`/`getters`/`actions` (как Vuex/Options API).
Setup store — функцией: `ref` → state, `computed` → getters, функции → actions. Setup store
гибче: внутри доступны `watch`, любые composables, приватное состояние. Но `$reset()` для него
нужно реализовывать вручную.

**6. Есть ли в Pinia мутации, как в Vuex?**
Нет. Это сознательное упрощение. State меняется напрямую или внутри actions, обе операции
реактивны и видны в devtools. Слой мутаций из Vuex был признан лишним бойлерплейтом.

**7. Как делать асинхронные операции?**
Прямо в actions: action может быть `async`, использовать `await`, возвращать промис. Отдельной
концепции (как `actions` vs `mutations` в Vuex) не требуется.

**8. Почему `state` — это функция-фабрика, а не объект?**
Чтобы для каждого экземпляра приложения (особенно при SSR — новый инстанс на каждый запрос)
создавалось своё, изолированное состояние. Общий объект привёл бы к утечке состояния между
пользователями/запросами.

**9. Pinia vs Vuex — почему Pinia стала стандартом?**
См. подробно ниже. Кратко: проще API (нет мутаций, нет namespaced-модулей со строками),
отличный TypeScript-вывод типов без обёрток, модульность «из коробки» (каждый стор отдельный),
меньше бойлерплейта, лучше работает с Composition API. Vuex 4 в режиме поддержки, Pinia —
официальная рекомендация.

**10. Как организовать модульность в Pinia?**
Каждый стор — отдельный `defineStore` в своём файле с уникальным id. «Связи» между сторами
делаются вызовом `useOtherStore()` внутри action/getter. Никаких namespaced-модулей и строковых
путей, как в Vuex, не нужно.

**11. Что такое плагин Pinia и зачем он?**
Функция `({ store, app, pinia, options }) => {...}`, которую регистрируют через `pinia.use(...)`.
Она выполняется для каждого стора и позволяет добавлять общие свойства, перехватывать изменения
(персистентность, логирование, метрики). Пример — `pinia-plugin-persistedstate`.

**12. Можно ли обойтись без Pinia?**
Для маленьких приложений — да: экспортируемый `reactive()`-объект или `provide`/`inject`.
Но вы потеряете devtools, SSR-безопасность, структуру и плагины. Для среднего/крупного проекта
это невыгодно.

**13. Как сделать state доступным только для чтения снаружи?**
В ручном сторе — обернуть в `readonly()`. В Pinia инкапсуляция достигается через приватные
переменные setup store (не возвращать их из функции), а менять — только через возвращаемые actions.

**14. Как стор связан с реактивностью Vue?**
Pinia использует внутри `reactive`/`ref`/`computed` Vue. Поэтому getters кешируются как computed,
а изменения state триггерят рендер компонентов так же, как обычное локальное реактивное состояние.

## Pinia vs Vuex (отдельно — топ-вопрос)

| Критерий | Vuex (4) | Pinia |
|---|---|---|
| Мутации | Обязательны (`mutations`), синхронны | Нет — меняем state напрямую/в actions |
| Actions | Отдельно от мутаций, для async | Любые функции, sync и async |
| Модули | Вложенные namespaced-модули, строки-пути | Каждый стор независим, импорт хука |
| TypeScript | Слабый, нужны обёртки/хелперы | Полный вывод типов «из коробки» |
| API | Многословный (`commit`/`dispatch`/`mapState`) | Минимальный, прямой доступ |
| Composition API | Поддержка ограничена | Спроектирован под него |
| Размер | Больше | ~1 КБ, легче |
| Статус | Режим поддержки | Официальная рекомендация Vue |

**Почему Pinia победила.** Vuex требовал много бойлерплейта: строковые типы мутаций, разделение
mutations/actions, namespaced-модули со строковыми путями (`'cart/items'`), плохо типизировался.
Pinia убрала мутации, сделала каждый стор самостоятельным модулем (просто функция + импорт),
дала идеальный TS-вывод и прямой доступ к state/getters. Это меньше кода, меньше ошибок,
естественная работа с `<script setup>`. Команда Vue официально объявила Pinia преемником Vuex.

## Подводные камни (gotchas)

- **Прямая деструктуризация стора** ломает реактивность — всегда `storeToRefs` для state/getters.
- **Вызов `useStore()` вне setup/без активной Pinia** упадёт: до `app.use(createPinia())` или вне
  компонента/setup стор недоступен. В роутер-гвардах и обычных модулях вызывайте `useStore()`
  внутри функции, а не на верхнем уровне модуля.
- **`state` как объект, а не функция** приведёт к общему состоянию между инстансами (баг в SSR).
- **`$reset()` не работает для setup stores** автоматически — нужно реализовать сброс вручную.
- **Стор на верхнем уровне модуля при SSR** — нельзя; инстанс Pinia создаётся на каждый запрос.
- **Геттер с побочными эффектами или асинхронный** — антипаттерн: getter должен быть чистым
  computed. Асинхронность — только в actions.
- **Мутация state снаружи без необходимости** — лучше держать изменения внутри actions для
  предсказуемости и удобства отладки через devtools.

## Лучшие практики

- Один стор — одна предметная область (auth, cart, ui). Имя хука: `useXxxStore`.
- Предпочитайте **setup stores** для новых проектов — больше гибкости и единый стиль с компонентами.
- Деструктурируйте state/getters через `storeToRefs`, actions — напрямую.
- Асинхронную логику и валидацию инкапсулируйте в actions; компонент только вызывает action.
- Не дублируйте в локальном состоянии то, что уже есть в сторе.
- Для персистентности/логирования используйте плагины, а не копипаст в каждом сторе.
- Для маленького локального состояния не тащите Pinia — хватит `ref`/`reactive` + provide/inject.

## Шпаргалка

```js
// Объявление
export const useStore = defineStore('id', () => {        // setup store
  const x = ref(0)
  const double = computed(() => x.value * 2)
  function inc() { x.value++ }
  return { x, double, inc }
})

// Использование
const store = useStore()
store.inc()                                   // action — напрямую
const { x, double } = storeToRefs(store)      // state/getters — через storeToRefs
const { inc } = store                          // actions — деструктуризация ок

// Мутации
store.x++
store.$patch({ x: 10 })
store.$patch((s) => { s.x++ })
store.$reset()                                 // только option store

// Подписки
store.$subscribe((m, s) => {/* на изменения state */})
store.$onAction(({ name, after, onError }) => {/* на вызовы actions */})

// Плагин
pinia.use(({ store }) => { /* расширяем каждый стор */ })
```

**Правила одной строкой:**
- `storeToRefs` — для state и getters; actions деструктурируем напрямую.
- Прямая деструктуризация стора = потеря реактивности.
- `state` — всегда функция-фабрика.
- Async-логика и мутации — в actions; getters — чистые.
- Pinia вместо Vuex: нет мутаций, лучше TS, проще модульность.
