# Vue 3 — Provide / Inject — конспект и вопросы

## О чём раздел

`provide` / `inject` — механизм передачи данных от компонента-предка к любому потомку **на любой глубине вложенности**, минуя промежуточные компоненты. Это решение проблемы **prop drilling** — когда prop приходится «протаскивать» через множество компонентов, которым он не нужен, только чтобы донести его до глубоко вложенного потребителя.

Предок один раз **предоставляет** (provide) значение, и любой потомок в его поддереве может **внедрить** (inject) это значение, не зная о промежуточных слоях. Это похоже на «контекст» в других фреймворках.

Ключевые темы раздела (с упором на senior): почему provide/inject ≠ глобальное хранилище, как сохранять **реактивность** передаваемых значений, почему мутировать состояние нужно через **функции-мутаторы** у предка, как использовать **Symbol-ключи** против коллизий, `app.provide` на уровне приложения и типизация через `InjectionKey`.

## Ключевые концепции

- **prop drilling** — антипаттерн прокидывания props через цепочку компонентов-посредников.
- **`provide(key, value)`** — вызывается в `setup`/`<script setup>` предка; делает `value` доступным потомкам по ключу `key`.
- **`inject(key, default?)`** — вызывается в потомке; возвращает предоставленное значение или дефолт.
- **Реактивность** — provide/inject сам по себе НЕ делает значение реактивным. Чтобы потомок реагировал на изменения, нужно предоставлять `ref`/`reactive`/`computed`.
- **Функции-мутаторы** — рекомендуется отдавать вместе со значением функции для его изменения, чтобы мутации происходили там же, где объявлено состояние (инкапсуляция, отслеживаемость).
- **`readonly`** — оборачивание предоставляемого значения, чтобы потомки не могли его мутировать напрямую.
- **Symbol-ключи** — уникальные ключи во избежание коллизий имён, особенно в библиотеках.
- **`app.provide(key, value)`** — provide на уровне всего приложения; доступно во всех компонентах.
- **`InjectionKey<T>`** — типизированный ключ (TypeScript) для строгой типизации provide/inject.

## Подробный разбор с примерами кода

### 1. Проблема prop drilling

```vue
<!-- БЕЗ provide/inject: prop "theme" тащим через все уровни -->
<!-- App -> Layout -> Sidebar -> Menu -> MenuItem (только тут нужен theme) -->
<template>
  <Layout :theme="theme" />        <!-- Layout не использует theme -->
</template>
```

Каждый промежуточный компонент обязан принимать и пробрасывать `theme`, хотя сам им не пользуется. Это шумно, хрупко и тяжело рефакторить.

### 2. Базовый provide / inject

Предок:

```vue
<script setup>
import { provide } from 'vue'

// Предоставляем значение по строковому ключу
provide('theme', 'dark')
provide('appName', 'Моё приложение')
</script>

<template>
  <Layout />
</template>
```

Потомок на любой глубине:

```vue
<script setup>
import { inject } from 'vue'

// Внедряем по тому же ключу
const theme = inject('theme')        // 'dark'
const appName = inject('appName')    // 'Моё приложение'
</script>

<template>
  <div :class="theme">{{ appName }}</div>
</template>
```

### 3. Значения по умолчанию у inject

Если ключ не был предоставлен ни одним предком, `inject` вернёт `undefined`. Чтобы задать дефолт:

```vue
<script setup>
import { inject } from 'vue'

// 2-й аргумент — значение по умолчанию
const theme = inject('theme', 'light')

// Для дорогих/объектных дефолтов — фабрика (3-й аргумент true),
// чтобы не создавать объект, если он не нужен, и не шарить один объект между инстансами
const config = inject('config', () => ({ retries: 3 }), true)
</script>
```

Без дефолта Vue в dev-режиме предупредит, что injection не найдена.

### 4. Реактивность передаваемых значений — КЛЮЧЕВАЯ ТЕМА

`provide`/`inject` **не добавляет реактивности**. Если предоставить «голое» значение, потомок не узнает об изменениях:

```vue
<script setup>
import { provide, ref } from 'vue'

let count = 0
provide('count', count)   // ПЛОХО: примитив-снимок, потомок не увидит изменений
count = 5                  // потомок по-прежнему получит 0
</script>
```

Правильно — предоставлять **реактивный источник** (`ref`, `reactive`, `computed`):

```vue
<script setup>
import { provide, ref } from 'vue'

const count = ref(0)
provide('count', count)   // ХОРОШО: предоставляем сам ref

function increment() {
  count.value++           // потомки реактивно увидят новое значение
}
provide('increment', increment)
</script>
```

Потомок:

```vue
<script setup>
import { inject } from 'vue'

const count = inject('count')        // это ref
const increment = inject('increment')
</script>

<template>
  <!-- ref в шаблоне разворачивается автоматически -->
  <button @click="increment">{{ count }}</button>
</template>
```

Аналогично с `reactive`:

```js
import { provide, reactive } from 'vue'

const store = reactive({ user: null, isLoggedIn: false })
provide('store', store)   // реактивный объект целиком
```

### 5. Работа с реактивностью: функции-мутаторы и readonly

Лучшая практика: **держать мутации внутри компонента, который объявил состояние**, и отдавать потомкам как данные, так и функции для их изменения. Так логика изменений инкапсулирована и легко отслеживается.

```vue
<!-- ThemeProvider.vue (предок) -->
<script setup>
import { provide, ref, readonly } from 'vue'

const theme = ref('light')

// Функция-мутатор: единственный «легальный» способ менять состояние
function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

provide('theme', {
  // readonly: потомок не сможет мутировать ref напрямую (предупреждение в dev)
  theme: readonly(theme),
  // меняем только через мутатор
  toggleTheme
})
</script>

<template>
  <slot />
</template>
```

Потомок:

```vue
<script setup>
import { inject } from 'vue'

const { theme, toggleTheme } = inject('theme')
</script>

<template>
  <div :class="theme">
    <button @click="toggleTheme">Переключить тему: {{ theme }}</button>
  </div>
</template>
```

Почему так:
- **Инкапсуляция**: вся логика изменения темы в одном месте.
- **Отслеживаемость**: понятно, кто и как меняет состояние.
- **Безопасность**: `readonly` не даёт случайно мутировать из глубины дерева, что иначе превратило бы поток данных в неуправляемый.

### 6. Symbol-ключи против коллизий

Строковые ключи могут конфликтовать (особенно в больших приложениях и библиотеках). `Symbol` уникален всегда:

```js
// keys.js — выносим ключи в отдельный модуль
export const themeKey = Symbol('theme')
export const userKey = Symbol('user')
```

```js
// предок
import { provide } from 'vue'
import { themeKey } from './keys'
provide(themeKey, themeData)

// потомок
import { inject } from 'vue'
import { themeKey } from './keys'
const theme = inject(themeKey)
```

Symbol гарантирует, что два разных провайдера с «одинаковым» именем не перетрут друг друга.

### 7. provide на уровне приложения (app.provide)

`app.provide` делает значение доступным **во всех компонентах** приложения — удобно для глобальных вещей (конфиг, i18n, API-клиент):

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import { configKey } from './keys'

const app = createApp(App)

// Доступно ЛЮБОМУ компоненту приложения
app.provide('appVersion', '1.0.0')
app.provide(configKey, { apiBase: '/api', locale: 'ru' })

app.mount('#app')
```

В любом компоненте:

```js
const version = inject('appVersion')
```

Отличие от компонентного provide: `app.provide` действует на всё приложение, а `provide` в компоненте — только на его поддерево.

### 8. Типизация через InjectionKey (TypeScript)

`InjectionKey<T>` — это `Symbol` с привязанным типом значения. Он обеспечивает строгую типизацию: TS выведет тип результата `inject` и проверит тип в `provide`.

```ts
// keys.ts
import type { InjectionKey, Ref } from 'vue'

interface ThemeContext {
  theme: Readonly<Ref<string>>
  toggleTheme: () => void
}

// Типизированный ключ
export const themeKey: InjectionKey<ThemeContext> = Symbol('theme')
```

```ts
// предок
import { provide, ref, readonly } from 'vue'
import { themeKey } from './keys'

const theme = ref('light')
const toggleTheme = () => { theme.value = theme.value === 'light' ? 'dark' : 'light' }

// TS проверит, что объект соответствует ThemeContext
provide(themeKey, { theme: readonly(theme), toggleTheme })
```

```ts
// потомок
import { inject } from 'vue'
import { themeKey } from './keys'

// Тип выведется как ThemeContext | undefined
const ctx = inject(themeKey)
// Можно задать дефолт, тогда тип — ThemeContext без undefined
const ctx2 = inject(themeKey, { theme: readonly(ref('light')), toggleTheme: () => {} })
```

## Полный рабочий пример

Глобальный «провайдер пользователя» с реактивным состоянием, мутаторами, Symbol-ключом и типизацией. Демонстрирует prop-free доступ к данным на любой глубине.

```ts
// userContext.ts
import type { InjectionKey, Ref } from 'vue'

export interface User {
  id: number
  name: string
}

export interface UserContext {
  user: Readonly<Ref<User | null>>
  isLoggedIn: Readonly<Ref<boolean>>
  login: (u: User) => void
  logout: () => void
}

export const userKey: InjectionKey<UserContext> = Symbol('user')
```

```vue
<!-- UserProvider.vue -->
<script setup lang="ts">
import { provide, ref, readonly, computed } from 'vue'
import { userKey, type User } from './userContext'

const user = ref<User | null>(null)
const isLoggedIn = computed(() => user.value !== null)

// Функции-мутаторы — единственный способ менять состояние
function login(u: User) {
  user.value = u
}
function logout() {
  user.value = null
}

provide(userKey, {
  user: readonly(user),
  isLoggedIn: readonly(isLoggedIn),
  login,
  logout
})
</script>

<template>
  <slot />
</template>
```

```vue
<!-- main подключение -->
<!-- App.vue -->
<script setup>
import UserProvider from './UserProvider.vue'
import Header from './Header.vue'
import Page from './Page.vue'
</script>

<template>
  <UserProvider>
    <Header />
    <Page />
  </UserProvider>
</template>
```

```vue
<!-- DeepProfile.vue — глубоко вложенный потомок, без prop drilling -->
<script setup lang="ts">
import { inject } from 'vue'
import { userKey } from './userContext'

// Дефолт-объект на случай использования вне провайдера
const ctx = inject(userKey)
if (!ctx) throw new Error('UserProvider не найден в дереве')

const { user, isLoggedIn, login, logout } = ctx
</script>

<template>
  <div>
    <template v-if="isLoggedIn">
      Привет, {{ user?.name }}!
      <button @click="logout">Выйти</button>
    </template>
    <button v-else @click="login({ id: 1, name: 'Андрей' })">
      Войти
    </button>
  </div>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Какую проблему решает provide/inject?**
Prop drilling — необходимость прокидывать props через цепочку компонентов-посредников, которым эти данные не нужны. Provide/inject позволяет предку напрямую передать данные любому потомку на любой глубине.

**2. Делает ли provide/inject значение реактивным?**
Нет. Сам механизм не добавляет реактивности. Чтобы потомок реагировал на изменения, нужно предоставлять реактивный источник: `ref`, `reactive` или `computed`. Если предоставить примитив-снимок, потомок не увидит изменений.

**3. Где можно вызывать provide и inject?**
Синхронно в `setup()` / `<script setup>` во время инициализации компонента. `inject` должен вызываться синхронно в setup-контексте (не внутри коллбэков/async после await), иначе теряется привязка к текущему инстансу.

**4. Как задать значение по умолчанию для inject?**
Вторым аргументом: `inject('key', 'default')`. Для объектных/дорогих дефолтов — фабрика с третьим аргументом `true`: `inject('key', () => ({...}), true)`.

**5. Почему рекомендуют отдавать функции-мутаторы вместе со значением?**
Чтобы все изменения состояния происходили в том же компоненте, который его объявил — это инкапсулирует логику, делает поток данных предсказуемым и отслеживаемым. Потомки меняют состояние только через предоставленные функции.

**6. Как запретить потомку мутировать предоставленное состояние?**
Обернуть значение в `readonly()` перед provide. Потомок получит read-only-версию; попытка мутации вызовет предупреждение в dev-режиме.

**7. Зачем нужны Symbol-ключи?**
Чтобы избежать коллизий имён ключей, особенно в библиотеках и больших приложениях. `Symbol` всегда уникален, поэтому два провайдера не перетрут друг друга.

**8. Чем `app.provide` отличается от компонентного `provide`?**
`app.provide` делает значение доступным во всём приложении (всем компонентам). Компонентный `provide` — только потомкам данного компонента. `app.provide` удобен для глобальных зависимостей: конфиг, i18n, API-клиент.

**9. Что такое InjectionKey и зачем он?**
Это типизированный Symbol-ключ (`InjectionKey<T>`) в TypeScript. Он связывает ключ с типом значения, благодаря чему `inject` возвращает правильно типизированный результат, а `provide` проверяется на соответствие типу.

**10. Может ли потомок переопределить (затенить) предоставленное значение для своих потомков?**
Да. Если потомок сам вызовет `provide` с тем же ключом, для его поддерева значение будет переопределено (shadowing), как в лексической области видимости.

**11. Provide/inject — это замена Pinia/Vuex?**
Нет. Это инструмент передачи зависимостей по дереву, без devtools, time-travel, модулей и глобального доступа из любого места. Для сложного глобального состояния используют Pinia. Provide/inject хорош для «контекста» (тема, локаль, текущий пользователь в поддереве, DI).

**12. Что вернёт inject, если ключ не предоставлен и нет дефолта?**
`undefined`, плюс предупреждение в dev-режиме. Поэтому для обязательных зависимостей часто проверяют результат и кидают ошибку.

**13. Реактивен ли результат inject, если предоставили ref?**
Да — потомок получает тот же ref, и при изменении `.value` у предка обновится представление потомка. В шаблоне ref разворачивается автоматически.

**14. Можно ли отслеживать, какой компонент предоставил значение?**
Inject ищет ближайшего предка вверх по дереву, предоставившего данный ключ (как scope chain). Это динамическая связь по дереву компонентов, а не по импортам.

## Подводные камни (gotchas)

- **Голые примитивы не реактивны.** `provide('count', 5)` или `provide('count', count)` где `count` — обычная переменная, не дадут потомку реакции на изменения. Нужен `ref`/`reactive`.
- **Деструктуризация reactive ломает реактивность.** `const { user } = inject('store')` где store — `reactive`, оторвёт `user` от реактивности. Либо предоставляйте `ref`/`toRefs`, либо обращайтесь через `store.user`.
- **inject вне setup-контекста.** Вызов `inject` после `await` или внутри коллбэка теряет привязку к инстансу. Вызывайте синхронно в setup.
- **Неконтролируемые мутации из глубины.** Если не использовать `readonly` и мутаторы, потомки могут менять состояние напрямую, и станет неясно, откуда пришло изменение. Используйте `readonly` + функции-мутаторы.
- **Коллизии строковых ключей.** Два разных провайдера с ключом `'config'` перетрут друг друга. Используйте Symbol.
- **provide/inject не виден в props/devtools так же явно**, как props — связи менее очевидны, что усложняет отладку. Не злоупотребляйте.
- **Дефолт-объект без фабрики шарится между инстансами.** `inject('cfg', {})` создаёт один объект-дефолт; используйте фабрику `() => ({})` с `true`, если объект мутабельный.

## Лучшие практики

- Предоставляйте **реактивные источники** (`ref`/`reactive`/`computed`), если потомкам нужна реакция на изменения.
- Оборачивайте отдаваемое состояние в **`readonly`** и давайте **функции-мутаторы** — инкапсуляция и предсказуемость.
- Используйте **Symbol-ключи**, вынесенные в отдельный модуль (`keys.ts`), особенно в библиотеках.
- Типизируйте через **`InjectionKey<T>`** в TypeScript.
- Для обязательных зависимостей **проверяйте результат inject** и кидайте понятную ошибку, если провайдер не найден.
- Для глобальных синглтонов (конфиг, API-клиент, i18n) используйте **`app.provide`**.
- Не подменяйте provide/inject полноценный **state management** (Pinia) для сложного глобального состояния.
- Оформляйте провайдер как **отдельный компонент-обёртку** (`ThemeProvider`, `UserProvider`) или composable (`useProvideUser` / `useUser`) для чистого API.

## Шпаргалка

```js
// ПРЕДОК
import { provide, ref, readonly } from 'vue'
const count = ref(0)
const inc = () => count.value++
provide('count', readonly(count))   // реактивно + readonly
provide('inc', inc)                 // функция-мутатор

// ПОТОМОК
import { inject } from 'vue'
const count = inject('count', 0)    // с дефолтом
const inc   = inject('inc')

// SYMBOL + ТИПИЗАЦИЯ (TS)
import type { InjectionKey } from 'vue'
export const key: InjectionKey<MyType> = Symbol('my')
provide(key, value)                 // тип проверяется
const v = inject(key)               // тип выводится

// УРОВЕНЬ ПРИЛОЖЕНИЯ
app.provide('config', { ... })      // доступно всем
```

Краткие правила:
- provide/inject решает **prop drilling**.
- Сам по себе **НЕ реактивен** — отдавайте `ref`/`reactive`/`computed`.
- Меняйте состояние через **функции-мутаторы**, защищайте `readonly`.
- **Symbol-ключи** против коллизий, **InjectionKey** для типов.
- **`app.provide`** — для глобальных зависимостей.
- Это **DI/контекст**, а не замена Pinia.
