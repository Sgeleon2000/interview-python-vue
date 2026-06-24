# Vue 3 — Плагины (plugins) — конспект и вопросы

## О чём раздел

**Плагин** — это самостоятельный код, который добавляет в приложение Vue **функциональность уровня всего приложения (app-level)**. Плагин подключается один раз через `app.use(plugin, options)` и может настраивать сразу всё приложение.

Когда писать плагин: когда нужно сделать что-то **глобально для всего приложения**, например:
- зарегистрировать глобальные компоненты или директивы;
- сделать ресурс доступным всем компонентам через `provide`;
- добавить глобальные свойства/методы в `app.config.globalProperties`;
- подключить библиотеку, которой нужна установка во всём приложении (Vue Router, Pinia, i18n, Vuetify — все они плагины).

Плагин — это либо **объект с методом `install`**, либо просто **функция**, которая сама работает как `install`.

```js
import { createApp } from 'vue'
import App from './App.vue'
import myPlugin from './plugins/myPlugin'

const app = createApp(App)

// Подключаем плагин; вторым аргументом — необязательные опции
app.use(myPlugin, { /* options */ })

app.mount('#app')
```

## Ключевые концепции

- **Плагин = объект с `install(app, options)`** или функция `(app, options) => {}`.
- **`install` получает**: `app` (экземпляр приложения из `createApp`) и `options` (то, что передали вторым аргументом в `app.use`).
- **`app.use(plugin, options)`** — устанавливает плагин. Возвращает сам `app`, поэтому вызовы можно чейнить.
- **Идемпотентность**: один и тот же плагин Vue установит только один раз (повторные `app.use` игнорируются).
- **Что обычно делает плагин**:
  - `app.component(...)` — глобальная регистрация компонентов;
  - `app.directive(...)` — глобальная регистрация директив;
  - `app.provide(...)` — внедрение ресурса для всего дерева (через `inject`);
  - `app.config.globalProperties.$x = ...` — глобальные свойства, доступные в любом компоненте.
- **Порядок установки важен**, если один плагин зависит от другого.
- Плагин **привязан к конкретному экземпляру `app`**, а не глобально к Vue (в отличие от Vue 2, где `Vue.use` был глобальным).

## Подробный разбор с примерами кода

### 1. Анатомия плагина

```js
// plugins/myPlugin.js
export default {
  // install вызывается при app.use(myPlugin, options)
  install(app, options) {
    // app — экземпляр приложения
    // options — объект, переданный вторым аргументом в app.use

    // (1) Глобальный компонент
    app.component('MyButton', SomeButtonComponent)

    // (2) Глобальная директива
    app.directive('focus', { mounted: (el) => el.focus() })

    // (3) Provide ресурса для inject во всём дереве
    app.provide('config', options)

    // (4) Глобальное свойство/метод
    app.config.globalProperties.$myMethod = () => { /* ... */ }
  }
}
```

### 2. Плагин как функция

Если в плагине нужен только `install`, можно передать функцию напрямую:

```js
// Функция-плагин: сама играет роль install
export default function myPlugin(app, options) {
  app.config.globalProperties.$translate = (key) => key
}
```

```js
app.use(myPlugin, { locale: 'ru' })
```

### 3. globalProperties — глобальные свойства

`app.config.globalProperties` добавляет свойство/метод, доступное **в любом компоненте через `this`** (Options API) и в шаблоне:

```js
app.config.globalProperties.$translate = (key) => {
  return translationsTable[key] ?? key
}
```

```vue
<!-- В Options API доступно через this -->
<script>
export default {
  mounted() {
    console.log(this.$translate('greeting'))
  }
}
</script>

<template>
  <!-- В шаблоне — напрямую -->
  <p>{{ $translate('greeting') }}</p>
</template>
```

**Важно для Composition API:** в `<script setup>` нет `this`, поэтому `globalProperties` напрямую недоступны. Для Composition API предпочтительнее `provide`/`inject` или композабл. Глобальные свойства начинают с `$`, чтобы не конфликтовать с пользовательскими данными.

### 4. provide / inject из плагина

Более «Composition-friendly» способ — `app.provide`, затем `inject` в компонентах:

```js
// plugin install
install(app, options) {
  // Делаем объект доступным всему дереву компонентов
  app.provide('i18n', options)
}
```

```vue
<script setup>
import { inject } from 'vue'

// Получаем то, что плагин сделал доступным
const i18n = inject('i18n')
</script>

<template>
  <p>{{ i18n.greetings.hello }}</p>
</template>
```

### 5. Цепочка установки и порядок

`app.use()` возвращает `app`, что позволяет чейнить:

```js
createApp(App)
  .use(router)   // сначала роутер
  .use(pinia)    // затем стор
  .use(i18n)     // затем локализация
  .mount('#app')
```

**Порядок важен**, если плагин при установке использует другой плагин. Например, если плагин в `install` обращается к роутеру или стору — их надо установить **раньше**. Если зависимостей нет, порядок не критичен.

## Полный рабочий пример: простой i18n-плагин

```js
// plugins/i18n.js
export default {
  install(app, options) {
    // options.messages — таблица переводов вида { ru: { hello: 'Привет' }, ... }
    // options.locale — текущая локаль

    const { messages = {}, locale = 'en' } = options

    // (1) Глобальный метод $t для Options API и шаблонов.
    // Ключи вида "home.title" разворачиваем по точкам.
    app.config.globalProperties.$t = (key) => {
      return key
        .split('.')
        .reduce((obj, part) => obj?.[part], messages[locale]) ?? key
    }

    // (2) provide — чтобы было доступно и в Composition API через inject
    app.provide('i18n', {
      locale,
      messages,
      t: app.config.globalProperties.$t
    })
  }
}
```

Подключение:

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import i18nPlugin from './plugins/i18n'

const app = createApp(App)

app.use(i18nPlugin, {
  locale: 'ru',
  messages: {
    ru: {
      home: { title: 'Главная', welcome: 'Добро пожаловать!' }
    },
    en: {
      home: { title: 'Home', welcome: 'Welcome!' }
    }
  }
})

app.mount('#app')
```

Использование в Options API (через `$t`):

```vue
<template>
  <h1>{{ $t('home.title') }}</h1>
  <p>{{ $t('home.welcome') }}</p>
</template>
```

Использование в Composition API (через `inject`):

```vue
<script setup>
import { inject } from 'vue'

const i18n = inject('i18n')
</script>

<template>
  <h1>{{ i18n.t('home.title') }}</h1>
</template>
```

Этот пример показывает обе техники сразу: `globalProperties` для шаблонов/Options API и `provide` для Composition API.

## Частые вопросы на собеседовании (Q/A)

**1. Что такое плагин во Vue 3?**
Самодостаточный код, добавляющий функциональность уровня всего приложения. Это объект с методом `install(app, options)` или функция с такой же сигнатурой, подключаемая через `app.use()`.

**2. Какова сигнатура метода install?**
`install(app, options)`: `app` — экземпляр приложения (из `createApp`), `options` — необязательный объект, переданный вторым аргументом в `app.use(plugin, options)`.

**3. Что обычно делают плагины?**
Регистрируют глобальные компоненты (`app.component`) и директивы (`app.directive`), внедряют ресурсы (`app.provide`), добавляют глобальные свойства/методы (`app.config.globalProperties`), инициализируют сторонние библиотеки на уровне приложения.

**4. Как подключить плагин?**
`app.use(plugin, options)` до `app.mount()`. Метод возвращает `app`, что позволяет чейнить несколько `use`.

**5. Можно ли установить плагин дважды?**
Нет смысла: Vue игнорирует повторную установку одного и того же плагина в один экземпляр приложения (он идемпотентен).

**6. В чём разница globalProperties и provide/inject?**
`globalProperties` доступны через `this` (Options API) и в шаблоне, но **не** в `<script setup>` (там нет `this`). `provide/inject` работает и в Composition API, и лучше типизируется, не «засоряет» все компоненты. Для современного кода предпочтительнее `provide`/`inject` или композабл.

**7. Почему глобальные свойства называют через `$`?**
Соглашение: префикс `$` снижает риск коллизий с данными/методами компонентов и сигнализирует, что свойство — «фреймворковое/глобальное».

**8. Важен ли порядок установки плагинов?**
Да, если один плагин при установке зависит от другого (например, использует роутер или стор внутри `install`). Тогда зависимость надо подключить раньше. Без зависимостей порядок не важен.

**9. Чем плагин Vue 3 отличается от плагина Vue 2?**
В Vue 2 плагин ставился глобально через `Vue.use()` и влиял на все экземпляры. В Vue 3 плагин привязывается к конкретному экземпляру приложения через `app.use()`, что даёт изоляцию (несколько приложений на странице не мешают друг другу).

**10. Приведите примеры плагинов из экосистемы.**
Vue Router, Pinia (стор), Vue I18n, Vuetify/PrimeVue (UI-библиотеки) — все подключаются через `app.use()` и являются плагинами.

**11. Можно ли в плагине регистрировать компоненты и директивы?**
Да, это типичный сценарий: внутри `install` вызывают `app.component(...)` и `app.directive(...)`, после чего они доступны во всём приложении без локального импорта.

**12. Как передать настройки в плагин?**
Вторым аргументом `app.use(plugin, options)`. Этот `options` приходит в `install(app, options)` и используется для конфигурации (локаль, ключи API, набор переводов и т.п.).

## Подводные камни (gotchas)

- **`globalProperties` недоступны в `<script setup>`** — нет `this`. Используйте `inject` или композабл.
- **Установка после `mount`**: подключайте плагины **до** `app.mount('#app')`.
- **Зависимости между плагинами**: если плагин использует другой в `install`, установите зависимость раньше — иначе ошибка/неопределённое поведение.
- **Переопределение глобальных свойств**: два плагина, пишущие в `app.config.globalProperties.$x`, перетрут друг друга молча.
- **Привязка к экземпляру app**: плагин не глобален — если создаёте несколько приложений, каждое нужно конфигурировать отдельно.
- **`provide` на уровне app vs компонента**: `app.provide` виден всему дереву; не путать с компонентным `provide`, который виден только потомкам компонента.
- **Тяжёлая логика в install**: всё в `install` выполняется при старте приложения — не кладите туда дорогие синхронные операции, замедляющие инициализацию.
- **Типизация globalProperties в TS** требует расширения интерфейса `ComponentCustomProperties` — иначе TypeScript не увидит `$t` и подобное.

## Лучшие практики

- **Плагин — для функциональности уровня приложения**; для локальной логики используйте композаблы/компоненты.
- **Предпочитайте `provide`/`inject`** глобальным свойствам — это совместимо с Composition API и лучше типизируется.
- **Делайте плагин конфигурируемым** через `options`, с разумными значениями по умолчанию.
- **Документируйте, что плагин регистрирует** (компоненты, директивы, ключи inject, globalProperties).
- **Учитывайте порядок** установки при зависимостях между плагинами.
- **Префикс `$`** для глобальных свойств, чтобы избежать коллизий.
- **Идемпотентность и отсутствие побочных эффектов вне `install`** — весь код, меняющий приложение, держите в `install`.
- **Для TypeScript** расширяйте `ComponentCustomProperties`, чтобы глобальные свойства были типизированы.
- **Не злоупотребляйте глобальной регистрацией** компонентов — это раздувает бандл; регистрируйте глобально только действительно повсеместно используемые.

## Шпаргалка

```js
// Объектная форма плагина
export default {
  install(app, options) {
    app.component('GlobalComp', Comp)            // глобальный компонент
    app.directive('focus', { mounted: el => el.focus() }) // директива
    app.provide('key', resource)                 // для inject
    app.config.globalProperties.$helper = fn     // глобальное свойство
  }
}

// Функциональная форма
export default function plugin(app, options) { /* как install */ }

// Подключение (до mount, можно чейнить)
createApp(App)
  .use(router)
  .use(pinia)
  .use(myPlugin, { /* options */ })
  .mount('#app')
```

| Что добавляет плагин | API |
|---|---|
| Глобальный компонент | `app.component(name, comp)` |
| Глобальная директива | `app.directive(name, def)` |
| Ресурс для всего дерева | `app.provide(key, value)` |
| Глобальное свойство/метод | `app.config.globalProperties.$x` |

| Доступ к данным плагина | Где работает |
|---|---|
| `this.$x` / `{{ $x }}` (globalProperties) | Options API + шаблон |
| `inject('key')` (provide) | Composition API + Options API |

- `install(app, options)` — сердце плагина.
- `app.use(plugin, options)` — подключение, до `mount`.
- В `<script setup>` — `inject`, а не `this.$x`.
- Порядок важен только при зависимостях.
