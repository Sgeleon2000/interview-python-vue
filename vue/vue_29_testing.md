# Vue 3 — Тестирование — конспект и вопросы

## О чём раздел

Тестирование — обязательная тема для middle/senior. Интервьюер проверяет, понимаете ли вы
**пирамиду тестов** (много дешёвых unit-тестов внизу, меньше дорогих e2e наверху), умеете ли
писать тесты компонентов через **Vue Test Utils** и **Vitest**, и — главное — тестируете ли вы
**поведение**, а не детали реализации. Хороший тест переживает рефакторинг внутренностей
компонента; плохой ломается от любого переименования внутренней переменной.

Современный стек Vue: **Vitest** (быстрый раннер на Vite, совместимый по API с Jest) + **Vue Test
Utils** (официальная библиотека для монтирования компонентов) для unit/component-тестов, и
**Cypress** или **Playwright** для e2e.

## Ключевые концепции

- **Уровни тестов:**
  - *Unit* — изолированная логика: composables, утилиты, чистые функции, отдельные сторы.
  - *Component* — компонент целиком (рендер, пропсы, события, взаимодействие), но без реального
    браузера/бэкенда.
  - *E2E (end-to-end)* — приложение целиком в настоящем браузере, имитируя пользователя.
- **Vitest** — раннер: `describe`/`it`/`expect`, моки (`vi.fn`, `vi.mock`), фейковые таймеры,
  покрытие. Идеален для composables и утилит — там не нужен DOM.
- **Vue Test Utils (VTU):** `mount` (полный рендер с детьми) и `shallowMount` (дети заглушаются).
  Возвращает `wrapper` с методами поиска (`find`, `get`, `findComponent`), проверки (`text`,
  `html`, `props`, `emitted`) и взаимодействия (`trigger`, `setValue`).
- **Асинхронность:** Vue обновляет DOM асинхронно. После действия нужно дождаться обновления:
  `await wrapper.trigger(...)`, `await nextTick()`, либо `flushPromises()` для зависших промисов.
- **Тестирование Pinia:** `@pinia/testing` создаёт тестовую Pinia с автозамоканными actions.
- **Что тестировать:** наблюдаемое поведение (что видит пользователь и что эмитит компонент),
  а не приватные методы и внутреннее состояние.

## Подробный разбор с примерами кода

### Vitest: unit-тест чистой утилиты

```js
// utils/format.js
export function formatPrice(value, currency = '₽') {
  return `${value.toFixed(2)} ${currency}`
}
```

```js
// utils/format.test.js
import { describe, it, expect } from 'vitest'
import { formatPrice } from './format'

describe('formatPrice', () => {
  it('форматирует число с валютой по умолчанию', () => {
    expect(formatPrice(10)).toBe('10.00 ₽')
  })

  it('поддерживает кастомную валюту', () => {
    expect(formatPrice(10, '$')).toBe('10.00 $')
  })
})
```

### Vitest: тест composable

Composable, использующий реактивность Vue, можно тестировать напрямую — DOM не нужен, но нужна
реактивная среда. Если composable не использует lifecycle-хуки (`onMounted` и т.п.), достаточно
просто вызвать его:

```js
// composables/useCounter.js
import { ref } from 'vue'
export function useCounter(initial = 0) {
  const count = ref(initial)
  const inc = () => count.value++
  const dec = () => count.value--
  return { count, inc, dec }
}
```

```js
// composables/useCounter.test.js
import { describe, it, expect } from 'vitest'
import { useCounter } from './useCounter'

describe('useCounter', () => {
  it('инкрементирует и декрементирует', () => {
    const { count, inc, dec } = useCounter(5)
    expect(count.value).toBe(5)
    inc()
    expect(count.value).toBe(6)
    dec()
    dec()
    expect(count.value).toBe(4)
  })
})
```

> Если composable использует lifecycle-хуки или `provide`/`inject`, его нужно вызывать внутри
> компонента-обёртки или с помощью хелпера `withSetup`, иначе хуки не сработают.

### Vue Test Utils: mount vs shallowMount

```js
import { mount, shallowMount } from '@vue/test-utils'
import Parent from './Parent.vue'

// mount — рендерит компонент И все дочерние компоненты по-настоящему
const full = mount(Parent)

// shallowMount — дочерние компоненты заменяются заглушками <child-stub />.
// Полезно для изоляции тестируемого компонента от логики детей.
const shallow = shallowMount(Parent)
```

### Передача пропсов, поиск элементов, проверка текста

```vue
<!-- Greeting.vue -->
<script setup>
defineProps({ name: { type: String, required: true } })
</script>
<template>
  <h1 class="title">Привет, {{ name }}!</h1>
</template>
```

```js
import { mount } from '@vue/test-utils'
import Greeting from './Greeting.vue'

it('рендерит имя из пропса', () => {
  const wrapper = mount(Greeting, {
    props: { name: 'Аня' },             // передаём пропсы
  })

  // find возвращает wrapper (или пустой, если не найдено); get кидает ошибку если не найдено
  const title = wrapper.find('.title')
  expect(title.exists()).toBe(true)
  expect(title.text()).toBe('Привет, Аня!')
  expect(wrapper.text()).toContain('Аня')

  // Поиск по data-test атрибуту — устойчивее к изменениям вёрстки
  // wrapper.find('[data-test="title"]')
})
```

### Триггер событий и проверка эмитов

```vue
<!-- Counter.vue -->
<script setup>
import { ref } from 'vue'
const emit = defineEmits(['change'])
const count = ref(0)
function inc() {
  count.value++
  emit('change', count.value)   // эмитим событие наружу
}
</script>
<template>
  <button data-test="inc" @click="inc">{{ count }}</button>
</template>
```

```js
import { mount } from '@vue/test-utils'
import Counter from './Counter.vue'

it('увеличивает счётчик и эмитит change', async () => {
  const wrapper = mount(Counter)
  const btn = wrapper.get('[data-test="inc"]')

  // trigger возвращает промис — ОБЯЗАТЕЛЬНО await, чтобы дождаться обновления DOM
  await btn.trigger('click')

  expect(btn.text()).toBe('1')

  // emitted() возвращает объект всех эмитов: { change: [[1]] }
  expect(wrapper.emitted('change')).toBeTruthy()
  expect(wrapper.emitted('change')).toHaveLength(1)
  expect(wrapper.emitted('change')[0]).toEqual([1])  // аргументы первого эмита
})
```

### Работа с формами и v-model

```js
it('обновляет значение по вводу', async () => {
  const wrapper = mount(SearchInput)
  const input = wrapper.get('input')

  await input.setValue('vue')        // устанавливает value и триггерит input-событие

  expect(wrapper.emitted('update:modelValue')[0]).toEqual(['vue'])
})
```

### Асинхронность: nextTick и flushPromises

```js
import { mount, flushPromises } from '@vue/test-utils'
import { nextTick } from 'vue'

it('показывает данные после загрузки', async () => {
  const wrapper = mount(UserList)

  // 1) Если изменение реактивного состояния синхронно — хватает nextTick,
  //    чтобы дождаться перерисовки DOM
  await nextTick()

  // 2) Если внутри компонента висят промисы (fetch, await),
  //    flushPromises «прогоняет» все ожидающие микрозадачи
  await flushPromises()

  expect(wrapper.findAll('[data-test="user"]')).toHaveLength(3)
})
```

> Правило: `nextTick` — дождаться обновления DOM после изменения реактивных данных;
> `flushPromises` — дождаться завершения асинхронных операций (промисов). Часто нужны оба.

### Мокинг запросов и зависимостей в Vitest

```js
import { vi, it, expect } from 'vitest'

it('мокаем fetch', async () => {
  // подменяем глобальный fetch
  global.fetch = vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve([{ id: 1, name: 'Аня' }]),
  })

  const wrapper = mount(UserList)
  await flushPromises()

  expect(fetch).toHaveBeenCalledWith('/api/users')
  expect(wrapper.text()).toContain('Аня')
})

// Мок целого модуля
vi.mock('@/api/client', () => ({
  getUsers: vi.fn(() => Promise.resolve([])),
}))
```

### Тестирование с Pinia

Компонент, использующий стор, нужно монтировать с активной Pinia. `@pinia/testing` даёт
`createTestingPinia`, который по умолчанию **замокивает все actions** (они не выполняются, но
фиксируются вызовы) и позволяет задать начальный state:

```js
import { mount } from '@vue/test-utils'
import { createTestingPinia } from '@pinia/testing'
import { useCartStore } from '@/stores/cart'
import CartWidget from './CartWidget.vue'

it('рендерит товары из стора и вызывает action', async () => {
  const wrapper = mount(CartWidget, {
    global: {
      plugins: [
        createTestingPinia({
          // начальное состояние стора
          initialState: {
            cart: { items: [{ id: 1, title: 'Книга', price: 100, qty: 2 }] },
          },
          // actions замоканы по умолчанию; createSpy подставляет vi.fn
          stubActions: true,
        }),
      ],
    },
  })

  expect(wrapper.text()).toContain('Книга')

  // Получаем стор внутри теста (Pinia уже активна)
  const cart = useCartStore()
  await wrapper.get('[data-test="remove"]').trigger('click')

  // Проверяем, что action был вызван (он замокан, реально не выполнялся)
  expect(cart.removeItem).toHaveBeenCalledWith(1)
})
```

Если нужно протестировать сам стор (его actions/getters), монтировать компонент не нужно —
активируем обычную Pinia и работаем со стором напрямую:

```js
import { setActivePinia, createPinia } from 'pinia'
import { beforeEach, it, expect } from 'vitest'
import { useCounterStore } from '@/stores/counter'

beforeEach(() => {
  setActivePinia(createPinia())   // свежая Pinia перед каждым тестом
})

it('increment увеличивает count и doubleCount', () => {
  const store = useCounterStore()
  store.increment()
  expect(store.count).toBe(1)
  expect(store.doubleCount).toBe(2)
})
```

### E2E кратко: Cypress / Playwright

E2E запускают приложение в настоящем браузере и проверяют пользовательские сценарии целиком
(вместе с бэкендом или его моками). Они самые медленные и хрупкие, поэтому их немного — только
критичные пути (логин, оформление заказа).

```js
// Playwright
import { test, expect } from '@playwright/test'

test('пользователь логинится', async ({ page }) => {
  await page.goto('/login')
  await page.getByLabel('Email').fill('user@example.com')
  await page.getByLabel('Пароль').fill('secret')
  await page.getByRole('button', { name: 'Войти' }).click()
  await expect(page).toHaveURL('/dashboard')
  await expect(page.getByText('Добро пожаловать')).toBeVisible()
})
```

```js
// Cypress — похожий сценарий
describe('login', () => {
  it('логинит пользователя', () => {
    cy.visit('/login')
    cy.get('[data-test="email"]').type('user@example.com')
    cy.get('[data-test="password"]').type('secret')
    cy.get('[data-test="submit"]').click()
    cy.url().should('include', '/dashboard')
    cy.contains('Добро пожаловать')
  })
})
```

> Cypress — удобный DX, работает в браузере, есть Component Testing. Playwright — быстрее,
> мультибраузерность (Chromium/Firefox/WebKit), хорош для параллельного прогона в CI.

## Полный рабочий пример

Компонент списка пользователей с загрузкой и его тест:

```vue
<!-- UserList.vue -->
<script setup>
import { ref, onMounted } from 'vue'

const users = ref([])
const loading = ref(false)
const error = ref(null)

async function load() {
  loading.value = true
  error.value = null
  try {
    const res = await fetch('/api/users')
    if (!res.ok) throw new Error('Ошибка загрузки')
    users.value = await res.json()
  } catch (e) {
    error.value = e.message
  } finally {
    loading.value = false
  }
}

onMounted(load)
</script>

<template>
  <div>
    <p v-if="loading" data-test="loading">Загрузка…</p>
    <p v-else-if="error" data-test="error">{{ error }}</p>
    <ul v-else>
      <li v-for="u in users" :key="u.id" data-test="user">{{ u.name }}</li>
    </ul>
  </div>
</template>
```

```js
// UserList.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mount, flushPromises } from '@vue/test-utils'
import UserList from './UserList.vue'

describe('UserList', () => {
  beforeEach(() => {
    vi.restoreAllMocks()
  })

  it('отображает список пользователей после успешной загрузки', async () => {
    global.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve([
        { id: 1, name: 'Аня' },
        { id: 2, name: 'Борис' },
      ]),
    })

    const wrapper = mount(UserList)
    // onMounted уже вызвал load(); дожидаемся промисов и перерисовки
    await flushPromises()

    const items = wrapper.findAll('[data-test="user"]')
    expect(items).toHaveLength(2)
    expect(items[0].text()).toBe('Аня')
    expect(wrapper.find('[data-test="loading"]').exists()).toBe(false)
  })

  it('показывает ошибку при сбое запроса', async () => {
    global.fetch = vi.fn().mockResolvedValue({ ok: false })

    const wrapper = mount(UserList)
    await flushPromises()

    expect(wrapper.find('[data-test="error"]').text()).toContain('Ошибка')
  })
})
```

## Частые вопросы на собеседовании (Q/A)

**1. Какие уровни тестов вы знаете и в чём разница?**
Unit (изолированная логика — функции, composables, сторы), component (компонент целиком без
браузера), e2e (всё приложение в реальном браузере как пользователь). Пирамида: много unit,
меньше component, минимум e2e — они дорогие и медленные.

**2. Чем `mount` отличается от `shallowMount`?**
`mount` рендерит компонент и всех его детей по-настоящему. `shallowMount` заменяет дочерние
компоненты заглушками. `shallowMount` изолирует тестируемый компонент; `mount` ближе к реальному
поведению и предпочтителен по умолчанию.

**3. Что значит «тестировать поведение, а не реализацию»?**
Проверять то, что видит пользователь и что компонент эмитит наружу (вывод в DOM, события), а не
внутренние методы, имена переменных или приватное состояние. Тогда рефакторинг внутренностей не
ломает тесты. Антипаттерн — вызывать `wrapper.vm.privateMethod()` и проверять внутренние ref-ы.

**4. Зачем нужны `nextTick` и `flushPromises`?**
Vue обновляет DOM асинхронно. `nextTick` ждёт перерисовки после изменения реактивных данных.
`flushPromises` прогоняет все ожидающие промисы/микрозадачи (нужно для `fetch`/`await` внутри
компонента). `trigger` и `setValue` уже возвращают промис, поэтому их `await`-ят.

**5. Как проверить, что компонент эмитнул событие?**
Через `wrapper.emitted('eventName')` — вернёт массив вызовов с аргументами, например
`[[payload]]`. Если событие не эмитилось — `undefined`.

**6. Как передать пропсы в тесте?**
В опции `props` при `mount`: `mount(Comp, { props: { name: 'Аня' } })`. Изменить позже —
`wrapper.setProps({...})` (с `await`).

**7. Как тестировать composable?**
Если без lifecycle-хуков — вызвать напрямую и проверять возвращаемые ref-ы/функции. Если есть
`onMounted`/`provide` — обернуть в тестовый компонент (`withSetup`-хелпер), иначе хуки не
выполнятся.

**8. Как тестировать компонент со стором Pinia?**
Монтировать с `createTestingPinia()` из `@pinia/testing` в `global.plugins`, задать `initialState`.
По умолчанию actions замоканы (`stubActions`), их вызовы можно проверять через `toHaveBeenCalled`.

**9. Как тестировать сам стор?**
Не нужен компонент: `setActivePinia(createPinia())` в `beforeEach`, затем вызвать `useStore()`,
выполнить action и проверить state/getters.

**10. Как мокать API/модули в Vitest?**
`vi.fn()` для отдельных функций (например, `global.fetch`), `vi.mock('module')` для целого
модуля, `mockResolvedValue`/`mockRejectedValue` для промисов. `vi.restoreAllMocks()` в `beforeEach`
для чистоты.

**11. Vitest vs Jest — почему Vitest?**
Vitest работает на Vite, поэтому быстрый и использует ту же конфигурацию сборки, что и проект
(алиасы, плагины). API совместим с Jest (`describe`/`it`/`expect`/`vi`), есть HMR-режим watch,
нативный ESM и TypeScript без доп. настройки.

**12. Cypress vs Playwright?**
Cypress — отличный DX, работает внутри браузера, есть component testing. Playwright — быстрее,
поддерживает Chromium/Firefox/WebKit, удобен для параллельного запуска в CI и для нескольких
вкладок/контекстов.

**13. Как искать элементы устойчиво к изменениям?**
По атрибутам `data-test`/`data-testid`, а не по CSS-классам или тексту, которые часто меняются.
Это снижает хрупкость тестов.

**14. Что покрывать e2e, а что unit/component?**
Unit/component — основная масса логики и поведения компонентов (быстро, дёшево). E2e — только
критичные сквозные сценарии (логин, оплата), потому что они медленные и хрупкие.

## Подводные камни (gotchas)

- **Забыли `await`** у `trigger`/`setValue`/`setProps` — DOM не успевает обновиться, ассерт падает.
- **Зависшие промисы без `flushPromises`** — данные после `fetch` не появятся в момент проверки.
- **Тест на приватные методы** (`wrapper.vm.someMethod`) — хрупкий, ломается при рефакторинге.
- **`shallowMount` там, где нужно реальное поведение детей** — заглушки скрывают интеграционные баги.
- **Поиск по тексту/классам** вместо `data-test` — тесты ломаются от косметических правок.
- **Грязные моки между тестами** — без `vi.restoreAllMocks()`/свежей Pinia состояние «протекает».
- **`createTestingPinia` с замоканными actions** — помните, что action реально не выполняется;
  если нужна его логика, отключите `stubActions`.
- **Тестирование `onMounted`-логики без `flushPromises`** — асинхронная инициализация не завершится.

## Лучшие практики

- Стройте пирамиду: много unit, умеренно component, минимум e2e.
- Тестируйте наблюдаемое поведение (DOM + эмиты), а не внутренности.
- Используйте `data-test`-атрибуты для устойчивого поиска элементов.
- `mount` по умолчанию; `shallowMount` — когда нужно изолировать от детей.
- Всегда `await` асинхронные взаимодействия; комбинируйте `nextTick` + `flushPromises`.
- Изолируйте тесты: свежие моки и свежая Pinia в `beforeEach`.
- Для сторов тестируйте actions/getters отдельно от компонентов.
- Не гонитесь за 100% покрытием — покрывайте важную логику и граничные случаи.

## Шпаргалка

```js
// Vitest
import { describe, it, expect, vi, beforeEach } from 'vitest'
vi.fn(); vi.mock('mod'); vi.restoreAllMocks()
mockResolvedValue(); mockRejectedValue()

// Vue Test Utils
import { mount, shallowMount, flushPromises } from '@vue/test-utils'
import { nextTick } from 'vue'

const w = mount(Comp, { props: {...}, global: { plugins: [...], stubs: {...} } })
w.find('[data-test="x"]'); w.get(sel); w.findAll(sel); w.findComponent(Child)
w.text(); w.html(); w.props(); w.exists()
await w.trigger('click'); await w.get('input').setValue('v'); await w.setProps({...})
w.emitted('event')            // [[args], ...]

// Асинхронность
await nextTick()              // дождаться перерисовки DOM
await flushPromises()         // дождаться промисов (fetch/await)

// Pinia
import { createTestingPinia } from '@pinia/testing'      // в компонентах
import { setActivePinia, createPinia } from 'pinia'      // для самих сторов
```

**Правила одной строкой:**
- Тестируй поведение (DOM + эмиты), не реализацию.
- `await` для trigger/setValue; `flushPromises` для fetch-логики.
- `mount` по умолчанию, `shallowMount` для изоляции.
- `createTestingPinia` для компонентов, `setActivePinia(createPinia())` для сторов.
- `data-test` вместо классов; свежие моки в `beforeEach`.
