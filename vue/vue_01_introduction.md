# Vue 3 — Введение — конспект и вопросы

## О чём раздел

Этот раздел даёт целостное представление о том, что такое Vue, какие задачи он решает и почему его называют «прогрессивным фреймворком». Здесь разбираются базовые идеи (декларативный рендеринг, реактивность), формат Single-File Component (SFC), две парадигмы написания компонентов (Options API и Composition API), варианты подключения Vue к проекту (CDN, сборка, SSR/SSG, веб-компоненты), коротко — Virtual DOM, а также экосистема инструментов (Router, Pinia, Vite, Nuxt).

Цель — сформировать «карту местности», чтобы дальнейшие темы (реактивность, компоненты, директивы и т. д.) ложились на понятный фундамент.

## Ключевые концепции

- **Vue** — JavaScript-фреймворк для построения пользовательских интерфейсов поверх стандартных HTML, CSS и JavaScript.
- **Прогрессивность** — Vue можно внедрять постепенно: от «оживления» куска статической страницы через CDN до полноценного SPA/SSR-приложения.
- **Декларативный рендеринг** — вы описываете, *как должен выглядеть UI* для конкретного состояния, а Vue сам синхронизирует DOM при изменении состояния.
- **Реактивность** — Vue отслеживает изменения JavaScript-состояния и автоматически эффективно обновляет DOM.
- **Single-File Component (SFC)** — файл `.vue`, объединяющий шаблон (`<template>`), логику (`<script setup>`) и стили (`<style>`) одного компонента.
- **Две парадигмы API**: Options API (объект с опциями `data`, `methods`, `computed`…) и Composition API (функции `ref`, `reactive`, `computed`, чаще всего в `<script setup>`).
- **Virtual DOM** — лёгкое представление реального DOM в памяти; Vue сравнивает его версии («diff») и применяет к реальному DOM минимальные изменения.
- **Экосистема**: Vite (сборка/дев-сервер), Vue Router (маршрутизация), Pinia (управление состоянием), Nuxt (мета-фреймворк для SSR/SSG).

## Подробный разбор с примерами кода

### Что такое Vue и его «прогрессивность»

Vue построен вокруг стандартных веб-технологий и расширяет их декларативным синтаксисом шаблонов. «Прогрессивный» означает, что фреймворк масштабируется под задачу. Типичные сценарии использования:

- Встраивание как «Web Components» без шага сборки.
- «Оживление» части статической страницы (островная архитектура / progressive enhancement).
- Полноценное SPA (Single-Page Application).
- SSR/SSG через Nuxt или собственный SSR-слой.
- Десктоп, мобайл (через Capacitor/NativeScript), WebGL, даже терминал.

Один и тот же набор знаний (реактивность, компоненты, API) переиспользуется во всех этих сценариях.

### Декларативный рендеринг

Вместо ручного управления DOM (как в jQuery) вы связываете состояние с разметкой.

```vue
<script setup>
// Composition API + <script setup> — основной современный подход
import { ref } from 'vue'

// ref создаёт реактивную «обёртку» вокруг значения
const count = ref(0)

// функция-обработчик меняет состояние
function increment() {
  // внутри <script setup> к ref обращаемся через .value
  count.value++
}
</script>

<template>
  <!-- {{ }} — интерполяция текста: при изменении count Vue сам обновит DOM -->
  <p>Счётчик: {{ count }}</p>
  <!-- @click — сокращение для v-on:click -->
  <button @click="increment">+1</button>
</template>
```

Ключевая идея: вы НЕ пишете `document.querySelector(...).textContent = ...`. Вы описываете «что показать», а Vue решает «как обновить».

### Реактивность

Реактивность — это система, которая «знает», какие части UI зависят от какого состояния, и обновляет только нужное.

```vue
<script setup>
import { ref, reactive, computed } from 'vue'

// ref — для примитивов (и не только); доступ через .value в JS
const price = ref(100)
const qty = ref(2)

// reactive — для объектов; доступ к полям без .value
const user = reactive({ name: 'Аня', vip: true })

// computed — вычисляемое значение, кэшируется и пересчитывается
// только при изменении зависимостей (price, qty, user.vip)
const total = computed(() => {
  const base = price.value * qty.value
  return user.vip ? base * 0.9 : base // VIP-скидка 10%
})
</script>

<template>
  <p>{{ user.name }} платит: {{ total }} ₽</p>
</template>
```

В Vue 3 реактивность реализована на `Proxy` (ES2015), что даёт более полное отслеживание изменений, чем `Object.defineProperty` во Vue 2.

### Single-File Component (SFC)

SFC — фирменный формат Vue. Один файл `.vue` содержит три блока:

```vue
<script setup>
// ЛОГИКА компонента
import { ref } from 'vue'
const message = ref('Привет, SFC!')
</script>

<template>
  <!-- РАЗМЕТКА (HTML + директивы Vue) -->
  <h1 class="title">{{ message }}</h1>
</template>

<style scoped>
/* СТИЛИ. scoped — стили применяются только к этому компоненту */
.title {
  color: #42b883; /* фирменный зелёный Vue */
}
</style>
```

Преимущества SFC:
- Логика, разметка и стили одного компонента — рядом, в одном файле.
- `<template>` компилируется в эффективный render-код на этапе сборки.
- `scoped`-стили, CSS-модули, препроцессоры (Sass/Less), `v-bind()` в CSS.
- Полная поддержка TypeScript.

SFC требует шага сборки (Vite), но именно он включает оптимизации компилятора.

### Две парадигмы API: Options API vs Composition API

Это один из самых частых вопросов на собеседовании. Vue 3 поддерживает обе.

**Options API** — компонент описывается объектом с «опциями». Логика разнесена по секциям `data`, `methods`, `computed`, `watch`, хукам жизненного цикла.

```vue
<script>
// Options API — классический стиль (есть и во Vue 2)
export default {
  data() {
    return { count: 0 } // реактивное состояние
  },
  computed: {
    doubled() {
      return this.count * 2 // доступ к состоянию через this
    }
  },
  methods: {
    increment() {
      this.count++
    }
  },
  mounted() {
    console.log('Компонент примонтирован')
  }
}
</script>

<template>
  <button @click="increment">{{ count }} (×2 = {{ doubled }})</button>
</template>
```

**Composition API** — логика строится из импортируемых функций (`ref`, `computed`, `watch`, `onMounted`…). В SFC обычно используется синтаксический сахар `<script setup>`.

```vue
<script setup>
// Composition API — современный стиль
import { ref, computed, onMounted } from 'vue'

const count = ref(0)
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++
}

onMounted(() => {
  console.log('Компонент примонтирован')
})
</script>

<template>
  <button @click="increment">{{ count }} (×2 = {{ doubled }})</button>
</template>
```

**Сравнение (важно для собеседования):**

| Критерий | Options API | Composition API |
|---|---|---|
| Организация кода | По «типу» опции (все data вместе, все methods вместе) | По «логической задаче» (фича целиком) |
| Переиспользование логики | Mixins (конфликты имён, неявность) | Composables — функции, чисто и явно |
| `this` | Активно используется, можно ошибиться с контекстом | Почти не нужен; нет проблем с `this` |
| TypeScript | Работает, но типизация менее естественна | Отличная выводимость типов |
| Порог входа | Ниже для новичков | Чуть выше, но мощнее |
| Tree-shaking | Хуже | Лучше (импортируешь только нужное) |
| Размер крупных компонентов | Логика «размазана» по секциям | Связанная логика рядом |

**Когда что использовать:**
- Composition API + `<script setup>` — рекомендация по умолчанию для новых проектов, особенно средних/крупных и с TypeScript.
- Options API — для простых компонентов, при миграции с Vue 2, в командах, привыкших к нему, или в обучающих целях.
- Важно: это две грани одной системы. Options API внутри реализован поверх Composition API. Можно (но обычно не нужно) смешивать.

### Переиспользование логики: mixins vs composables

```js
// composables/useMouse.js — переиспользуемая логика (composable)
import { ref, onMounted, onUnmounted } from 'vue'

// Соглашение: имя начинается с "use"
export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  // возвращаем реактивное состояние наружу
  return { x, y }
}
```

```vue
<script setup>
import { useMouse } from './composables/useMouse'

// явный импорт, явные зависимости, нет конфликтов имён
const { x, y } = useMouse()
</script>

<template>
  <p>Курсор: {{ x }}, {{ y }}</p>
</template>
```

Composables решают главную проблему mixins — неявность и конфликты имён.

### Способы использования Vue

1. **CDN без сборки** — быстро встроить Vue в существующую страницу:

```html
<script src="https://unpkg.com/vue@3/dist/vue.global.js"></script>
<div id="app">{{ message }}</div>
<script>
  const { createApp, ref } = Vue
  createApp({
    setup() {
      const message = ref('Привет из CDN!')
      return { message }
    }
  }).mount('#app')
</script>
```

2. **Полноценная сборка (рекомендуется)** — `npm create vue@latest`, Vite, SFC, HMR, TypeScript.

3. **SSR / SSG** — серверный рендеринг для SEO и быстрого первого экрана. Обычно через **Nuxt** (полноценный мета-фреймворк) или собственный SSR-слой Vue.
   - SSR — HTML генерируется на сервере на каждый запрос.
   - SSG — HTML генерируется заранее (на этапе сборки) для статических страниц.

4. **Web Components** — Vue-компонент можно собрать как нативный кастомный элемент (`defineCustomElement`) и использовать в любом окружении, даже без Vue-приложения.

### Virtual DOM (кратко)

Шаблон Vue компилируется в render-функцию, которая возвращает дерево **VNode** (виртуальное представление DOM). При изменении состояния:
1. Создаётся новое VNode-дерево.
2. Алгоритм сравнения («diffing») находит отличия от предыдущего дерева.
3. К реальному DOM применяются только минимальные изменения.

Это дешевле, чем перерисовывать весь DOM, и абстрагирует ручную работу с DOM. В Vue 3 компилятор делает дополнительные оптимизации (например, «hoisting» статических узлов и «patch flags»), поэтому diffing ещё дешевле.

### Экосистема

- **Vite** — современный сборщик и дев-сервер (молниеносный HMR, ESM в разработке). Стандарт для Vue 3.
- **Vue Router** — официальная маршрутизация для SPA.
- **Pinia** — официальное хранилище состояния (заменило Vuex; проще, типобезопаснее).
- **Nuxt** — мета-фреймворк: SSR/SSG, файловый роутинг, авто-импорты, серверные API-роуты.
- **Vue DevTools** — расширение/приложение для отладки реактивности, компонентов и состояния.

## Полный рабочий пример

Небольшой компонент «список задач», демонстрирующий реактивность, computed, обработку событий и SFC.

```vue
<script setup>
// Composition API + <script setup>
import { ref, computed } from 'vue'

// Реактивный массив задач
const todos = ref([
  { id: 1, text: 'Выучить реактивность', done: true },
  { id: 2, text: 'Понять Composition API', done: false }
])

// Поле ввода новой задачи
const newTodo = ref('')

// Вычисляемое: количество невыполненных задач
const remaining = computed(
  () => todos.value.filter((t) => !t.done).length
)

// Добавление задачи
function addTodo() {
  const text = newTodo.value.trim()
  if (!text) return // не добавляем пустые
  todos.value.push({
    id: Date.now(),
    text,
    done: false
  })
  newTodo.value = '' // очищаем поле
}

// Удаление по id
function removeTodo(id) {
  todos.value = todos.value.filter((t) => t.id !== id)
}
</script>

<template>
  <section class="todo">
    <h2>Задачи (осталось: {{ remaining }})</h2>

    <!-- v-model связывает input с состоянием в обе стороны -->
    <form @submit.prevent="addTodo">
      <input v-model="newTodo" placeholder="Новая задача" />
      <button type="submit">Добавить</button>
    </form>

    <ul>
      <!-- v-for требует :key для эффективного обновления списка -->
      <li v-for="todo in todos" :key="todo.id">
        <!-- v-model на чекбокс правит todo.done -->
        <input type="checkbox" v-model="todo.done" />
        <!-- :class — динамический класс при выполнении -->
        <span :class="{ done: todo.done }">{{ todo.text }}</span>
        <button @click="removeTodo(todo.id)">×</button>
      </li>
    </ul>
  </section>
</template>

<style scoped>
.done {
  text-decoration: line-through;
  color: gray;
}
</style>
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое Vue и почему его называют «прогрессивным фреймворком»?**
Vue — JS-фреймворк для UI поверх HTML/CSS/JS. «Прогрессивный» — потому что внедряется постепенно: от подключения через CDN и «оживления» куска страницы до полноценного SPA/SSR. Один набор концепций масштабируется под любую задачу.

**2. В чём разница между Options API и Composition API?**
Options API организует код по типу опции (`data`, `methods`, `computed`) и опирается на `this`. Composition API строит логику из функций (`ref`, `computed`, `watch`), группируя код по задаче, лучше переиспользуется через composables и лучше типизируется. Options API внутри реализован поверх Composition API.

**3. Когда выбрать Composition API, а когда Options API?**
Composition API — по умолчанию для новых, средних/крупных проектов и TypeScript. Options API — для простых компонентов, при миграции с Vue 2 или если команда к нему привыкла.

**4. Что такое SFC и какие у него преимущества?**
Single-File Component — файл `.vue` с `<template>`, `<script setup>`, `<style>`. Плюсы: всё для одного компонента рядом, компиляция шаблона в оптимизированный код, scoped-стили, поддержка TS и препроцессоров. Минус — нужен шаг сборки.

**5. Что такое декларативный рендеринг?**
Подход, где вы описываете желаемый UI для состояния, а фреймворк сам синхронизирует DOM. Противоположность императивному ручному управлению DOM (как в jQuery).

**6. Как устроена реактивность Vue 3 и чем отличается от Vue 2?**
Vue 3 использует `Proxy` для отслеживания доступа и изменений объектов, что покрывает добавление/удаление свойств и работу с массивами без хаков. Vue 2 использовал `Object.defineProperty`, который не видел новые свойства (нужен был `Vue.set`).

**7. Что такое Virtual DOM и зачем он нужен?**
Лёгкое представление DOM в памяти (дерево VNode). При изменении состояния Vue строит новое дерево, сравнивает со старым (diff) и применяет к реальному DOM минимальные изменения. Это быстрее полной перерисовки и абстрагирует ручную работу с DOM.

**8. Что входит в экосистему Vue?**
Vite (сборка/дев-сервер), Vue Router (маршрутизация), Pinia (состояние), Nuxt (SSR/SSG мета-фреймворк), Vue DevTools (отладка).

**9. Чем composables лучше mixins?**
Composables — это функции с явными импортами и явно возвращаемым состоянием: нет конфликтов имён, видно источник каждой переменной, легко типизировать. Mixins сливаются неявно, что приводит к коллизиям и непрозрачности.

**10. Можно ли использовать Vue без сборки?**
Да, через CDN (`vue.global.js`) и `createApp(...).mount(...)`. Подходит для прототипов и «оживления» страниц, но без HMR, SFC и оптимизаций компилятора.

**11. Что такое SSR и SSG, чем отличаются?**
SSR (Server-Side Rendering) — HTML генерируется на сервере на каждый запрос (хорошо для динамики). SSG (Static Site Generation) — HTML генерируется заранее на этапе сборки (хорошо для статики). Оба улучшают SEO и время первого экрана; чаще всего реализуются через Nuxt.

**12. Что такое `ref` и `reactive`, в чём разница?**
`ref` оборачивает любое значение (часто примитив), доступ в JS через `.value`. `reactive` делает реактивным объект, доступ к полям напрямую. В шаблоне `.value` для ref не нужен (авто-разворачивание).

**13. Зачем нужен Pinia и чем он отличается от Vuex?**
Pinia — официальное хранилище состояния для Vue 3: проще API (нет мутаций), отличная типизация, модульность из коробки, лучше для Composition API. Vuex — предыдущее решение, более многословное.

**14. Можно ли превратить Vue-компонент в нативный Web Component?**
Да, через `defineCustomElement` — компонент собирается в кастомный элемент и работает в любом окружении, даже без Vue-приложения.

## Подводные камни (gotchas)

- **Забыли `.value` у `ref` в JS.** В `<script setup>` обращение к ref-значению — через `.value`. В шаблоне `.value` НЕ пишут (авто-разворачивание).
- **Смешивание стилей без причины.** Технически можно совмещать Options и Composition API, но это запутывает; выбирайте один стиль на компонент.
- **`scoped` ≠ полная изоляция.** Scoped-стили изолируют селекторы, но глобальные стили и каскад всё ещё влияют; для дочерних компонентов нужен `:deep()`.
- **CDN-сборка обманчиво проста.** Без шага сборки нет SFC, HMR и оптимизаций компилятора — для продакшена это редко подходит.
- **Ожидание «магии» от Virtual DOM.** VDOM не делает приложение быстрым автоматически; неэффективные вычисления и лишние ререндеры всё равно вредят.
- **Mixins в новом коде.** В проектах на Composition API mixins — антипаттерн; используйте composables.
- **Реактивность массивов/объектов.** Замена ссылки на `ref`-массив (`todos.value = ...`) реактивна; но мутации вложенных структур через нереактивные копии — частый источник багов.

## Лучшие практики

- Для новых проектов используйте **Composition API + `<script setup>`** и Vite.
- Выносите переиспользуемую логику в **composables** (`useXxx`), а не в mixins.
- Группируйте код **по логической задаче**, а не по типу опций.
- Применяйте **TypeScript** — Composition API даёт хорошую выводимость типов.
- Используйте **Pinia** для глобального состояния и **Vue Router** для маршрутизации.
- Для production выбирайте **сборку** (SFC), а CDN — только для прототипов/виджетов.
- Ставьте **Vue DevTools** для отладки реактивности и компонентов.
- Если нужны SEO и быстрый первый экран — рассмотрите **Nuxt** (SSR/SSG).

## Шпаргалка

```text
Vue = декларативный рендеринг + реактивность поверх HTML/CSS/JS
Прогрессивность: CDN → SPA → SSR/SSG → Web Components

SFC (.vue): <template> + <script setup> + <style scoped>

API:
  Options API     — data/methods/computed/watch, this, mixins
  Composition API — ref/reactive/computed/watch, composables (use*), TS-friendly
  По умолчанию    — Composition API + <script setup>

Реактивность Vue 3 — на Proxy (vs Object.defineProperty во Vue 2)
  ref(x)      → .value в JS, авто-разворачивание в шаблоне
  reactive(o) → поля напрямую (только объекты)
  computed()  → кэшируемое вычисляемое значение

Virtual DOM: state → VNode-дерево → diff → минимальный патч реального DOM

Экосистема:
  Vite        — сборка/дев-сервер
  Vue Router  — маршрутизация
  Pinia       — состояние (замена Vuex)
  Nuxt        — SSR/SSG мета-фреймворк
  DevTools    — отладка

Запуск:
  createApp(App).mount('#app')   // монтирование корневого компонента
```
