# Vue 3 — Основы реактивности — конспект и вопросы

## О чём раздел

Реактивность — сердце Vue. Когда вы меняете данные, Vue автоматически обновляет
DOM и пересчитывает зависимости. В Vue 3 (Composition API) реактивное состояние
создаётся двумя основными API: `ref()` и `reactive()`. Понимание их различий,
ограничений и типичных ошибок (особенно **потери реактивности при
деструктуризации**) — обязательный минимум для middle/senior-разработчика и
классика собеседований.

## Ключевые концепции

- **`ref()`** — оборачивает **любое** значение (примитив или объект) в реактивную
  «коробку» с полем `.value`.
- **`reactive()`** — делает реактивным **только объект** (включая массивы и
  коллекции `Map`/`Set`) через `Proxy`.
- **Разворачивание ref (unwrapping)** — в шаблоне (на верхнем уровне) и внутри
  reactive-объектов `.value` подставляется автоматически.
- **Ограничения `reactive`** — нельзя использовать примитивы, нельзя заменять
  объект целиком, теряется реактивность при деструктуризации.
- **`ref` универсальнее** — работает с примитивами, переживает переприсваивание,
  можно деструктурировать без потери реактивности (через `toRefs`).
- **`<script setup>`** — автоматически экспонирует объявления верхнего уровня в шаблон.
- **Мутация vs замена** — `reactive` любит мутации, плохо переносит замену.
- **Реактивность массивов и коллекций** — работает через перехват методов.
- **`nextTick()`** — дождаться обновления DOM после изменения состояния.
- **`toRef` / `toRefs`** — превратить свойства reactive в ref без потери связи.

## Подробный разбор с примерами кода

### `ref()` — реактивная ссылка

`ref()` принимает значение и возвращает объект-обёртку с единственным свойством
`.value`:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

console.log(count)       // объект-ref { value: 0 }
console.log(count.value) // 0 — в JS обращаемся через .value

function increment() {
  count.value++          // мутируем через .value
}
</script>

<template>
  <!-- В шаблоне .value НЕ нужен — Vue разворачивает ref автоматически -->
  <button @click="increment">Счётчик: {{ count }}</button>
</template>
```

Почему `.value`? JS не позволяет перехватывать чтение/запись обычных переменных.
`ref` оборачивает значение в объект, чтобы отслеживать доступ через геттер/сеттер
`.value`. Это даёт работу с примитивами и возможность передавать реактивную ссылку
между функциями, не теряя связь.

`ref` глубоко реактивен: если внутри лежит объект/массив, он тоже становится
реактивным (внутри используется `reactive`).

```vue
<script setup>
import { ref } from 'vue'

const obj = ref({ nested: { count: 0 }, list: [1, 2] })

function mutate() {
  obj.value.nested.count++   // глубокая реактивность работает
  obj.value.list.push(3)
}
</script>
```

### `reactive()` — реактивный объект

`reactive()` делает объект реактивным через `Proxy`. Доступ к свойствам идёт
напрямую, без `.value`:

```vue
<script setup>
import { reactive } from 'vue'

const state = reactive({ count: 0, user: { name: 'Анна' } })

function increment() {
  state.count++             // без .value
  state.user.name = 'Боб'   // глубокая реактивность
}
</script>

<template>
  <p>{{ state.count }} — {{ state.user.name }}</p>
</template>
```

Важно: `reactive()` возвращает **Proxy**, а не исходный объект. Они НЕ равны по
ссылке, и для корректной работы реактивности нужно использовать именно прокси:

```js
const raw = {}
const proxy = reactive(raw)
console.log(proxy === raw)            // false
console.log(reactive(raw) === proxy)  // true — повторный вызов вернёт тот же прокси
console.log(reactive(proxy) === proxy)// true — оборачивать прокси безопасно
```

### Разворачивание ref (unwrapping)

**В шаблоне** ref разворачивается автоматически, но только если это **свойство
верхнего уровня** в контексте рендера:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
const object = { id: ref(1) }  // обычный объект, НЕ реактивный
</script>

<template>
  {{ count }}        <!-- 0  — развернулось -->
  {{ object.id }}    <!-- ref-объект, НЕ развернулось (id не верхнего уровня) -->
  {{ object.id.value }} <!-- 1 — нужен .value -->

  <!-- Лайфхак: деструктурируем заранее -->
  <!-- const { id } = object -> тогда {{ id }} развернётся -->
</template>
```

**Внутри reactive-объекта** ref тоже разворачивается автоматически (можно
обращаться без `.value`):

```vue
<script setup>
import { ref, reactive } from 'vue'

const count = ref(0)
const state = reactive({ count })

console.log(state.count)  // 0 — развернулось, .value не нужен
state.count = 5
console.log(count.value)  // 5 — связь двусторонняя

// НО: в массивах и нативных коллекциях (Map/Set) разворачивания НЕТ
const arr = reactive([ref(0)])
console.log(arr[0].value) // нужен .value
</script>
```

### Ограничения `reactive`

`reactive` имеет ряд существенных ограничений, из-за которых Vue официально
рекомендует `ref` как основной инструмент.

**1. Только объекты.** Примитивы (`string`, `number`, `boolean`) нельзя обернуть:

```js
// БЕСПОЛЕЗНО: reactive игнорирует примитивы
const n = reactive(0) // вернёт 0, без реактивности
```

**2. Нельзя заменять объект целиком** — потеряется ссылка на прокси и реактивность:

```vue
<script setup>
import { reactive } from 'vue'

let state = reactive({ count: 0 })

function reset() {
  // ОШИБКА реактивности: заменяем сам объект,
  // старый прокси (за которым следит Vue) выброшен, связь утеряна
  state = reactive({ count: 0 })
}
function resetOk() {
  // ПРАВИЛЬНО: мутируем существующий объект
  Object.assign(state, { count: 0 })
}
</script>
```

**3. Теряется реактивность при деструктуризации** (см. отдельный раздел ниже).

### Почему `ref` универсальнее

| Критерий | `ref` | `reactive` |
|---|---|---|
| Примитивы | да | нет |
| Объекты/массивы/коллекции | да | да |
| Замена значения целиком | да (`x.value = новое`) | нет (теряет реактивность) |
| Деструктуризация | сохраняет реактивность (через сам ref) | теряет реактивность |
| Передача между функциями | сохраняет связь | теряет при разворачивании в примитив |
| Доступ | через `.value` (минус) | напрямую (плюс) |

Главный недостаток `ref` — `.value`, но в шаблоне он не нужен, а в коде это
небольшая цена за универсальность и устойчивость к потере реактивности.

### `<script setup>` и автоматическое раскрытие

`<script setup>` — синтаксический сахар, который автоматически делает все
объявления верхнего уровня (переменные, функции, импорты) доступными в шаблоне —
без `return` и без объекта `setup()`:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)        // доступно в шаблоне автоматически
function inc() { count.value++ }  // тоже доступно
</script>

<template>
  <button @click="inc">{{ count }}</button>
</template>
```

Без `<script setup>` (обычный Composition API) пришлось бы:

```vue
<script>
import { ref } from 'vue'
export default {
  setup() {
    const count = ref(0)
    return { count }  // вручную возвращаем для шаблона
  }
}
</script>
```

### Мутации vs замена

Принцип:
- `reactive` любит **мутации** (`state.x = 1`, `arr.push(...)`) и не переносит
  **замену** самого объекта.
- `ref` спокойно переносит **замену значения**: `data.value = новыйМассив`.

```vue
<script setup>
import { ref, reactive } from 'vue'

const listRef = ref([1, 2, 3])
const stateReactive = reactive({ list: [1, 2, 3] })

function replaceAll(newList) {
  listRef.value = newList            // OK — ref переживает замену
  stateReactive.list = newList       // OK — заменяем СВОЙСТВО, не сам прокси
  // stateReactive = reactive({...}) // НЕ OK — потеря реактивности
}
</script>
```

### Реактивность массивов и коллекций

И `ref` (если внутри массив), и `reactive` делают массивы глубоко реактивными.
Vue перехватывает мутирующие методы (`push`, `pop`, `splice`, `sort`...) и
изменения по индексу/длине:

```vue
<script setup>
import { reactive, ref } from 'vue'

const arr = reactive([1, 2, 3])
arr.push(4)        // реактивно
arr[0] = 100       // реактивно (Proxy ловит запись по индексу)
arr.length = 1     // реактивно

const set = reactive(new Set())
const map = reactive(new Map())
set.add('x')       // реактивно
map.set('k', 'v')  // реактивно — Vue поддерживает Map/Set/WeakMap/WeakSet
</script>
```

Замена массивов неразрушающими методами (`filter`, `map`, `concat`) для `reactive`
делается через присвоение свойству; для `ref` — через `.value`:

```js
listRef.value = listRef.value.filter(x => x > 1)
```

### `nextTick()`

Vue обновляет DOM **асинхронно**, батчируя изменения в рамках одного «тика». После
изменения состояния DOM ещё не обновлён в той же синхронной строке. Чтобы
дождаться обновления DOM — используйте `nextTick`:

```vue
<script setup>
import { ref, nextTick } from 'vue'

const count = ref(0)

async function increment() {
  count.value++
  // DOM здесь ещё показывает старое значение
  await nextTick()
  // теперь DOM обновлён — можно читать актуальный layout/измерения
  console.log('DOM обновлён')
}
</script>
```

### `refs` и `.value` — резюме правил

- В JS-коде доступ к ref — всегда через `.value`.
- В шаблоне — авто-разворачивание (только для верхнего уровня).
- Внутри reactive-объекта — авто-разворачивание (кроме массивов/коллекций).
- Template ref (ссылка на DOM-элемент) — тоже `ref`, но заполняется после монтирования.

### Типичные ошибки потери реактивности (ТОП-вопрос: деструктуризация)

Это самая частая ошибка и любимый вопрос интервьюеров.

**Деструктуризация `reactive` рвёт связь.** Деструктуризация копирует значение
свойства в локальную переменную, теряя связь с прокси:

```vue
<script setup>
import { reactive } from 'vue'

const state = reactive({ count: 0, name: 'Анна' })

// ПОТЕРЯ РЕАКТИВНОСТИ: count — обычное число, копия, не связано с state
let { count } = state
count++                 // меняет локальную переменную, НЕ state.count
console.log(state.count)// 0 — состояние не изменилось

// Аналогично при передаче примитива в функцию
function useCount(c) { /* c — копия значения, не реактивна */ }
useCount(state.count)
</script>
```

**Решения через `toRefs` / `toRef`:**

```vue
<script setup>
import { reactive, toRefs, toRef } from 'vue'

const state = reactive({ count: 0, name: 'Анна' })

// toRefs: каждое свойство -> ref, связь сохраняется
const { count, name } = toRefs(state)
count.value++            // меняет state.count!
console.log(state.count) // 1

// toRef: один конкретный prop -> ref
const countRef = toRef(state, 'count')
countRef.value = 10
console.log(state.count) // 10
</script>
```

`toRefs` особенно важен в **composables**, чтобы вернуть реактивное состояние,
которое можно деструктурировать на стороне вызова:

```js
// useFeature.js
import { reactive, toRefs } from 'vue'
export function useFeature() {
  const state = reactive({ loading: false, data: null })
  // возвращаем toRefs, чтобы вызывающий код мог деструктурировать без потерь
  return { ...toRefs(state) }
}
```

**Другие способы потерять реактивность:**
- Замена reactive-объекта целиком (`state = reactive({...})`).
- Сохранение примитивного свойства reactive в локальную переменную.
- Передача `state.someValue` (примитива) в функцию вместо самого `ref`.

С `ref` деструктуризация тоже копирует ссылку на ref-объект — и это безопасно,
потому что ref-объект сам по себе реактивен (связь живёт внутри него):

```js
const a = ref(1)
const b = a            // b и a указывают на тот же ref — реактивность сохранена
b.value = 5
console.log(a.value)   // 5
```

## Полный рабочий пример

```vue
<script setup>
import { ref, reactive, toRefs, computed, nextTick } from 'vue'

// ref для примитива и массива
const newTodo = ref('')
const todos = ref([
  { id: 1, text: 'Изучить ref', done: true },
  { id: 2, text: 'Изучить reactive', done: false }
])

// reactive для сгруппированного состояния
const filters = reactive({ onlyActive: false, search: '' })

// computed зависит и от ref, и от reactive
const visibleTodos = computed(() =>
  todos.value.filter((t) => {
    const byStatus = filters.onlyActive ? !t.done : true
    const bySearch = t.text.toLowerCase().includes(filters.search.toLowerCase())
    return byStatus && bySearch
  })
)

const remaining = computed(() => todos.value.filter((t) => !t.done).length)

async function addTodo() {
  const text = newTodo.value.trim()
  if (!text) return
  // мутируем массив внутри ref — реактивно
  todos.value.push({ id: Date.now(), text, done: false })
  newTodo.value = ''            // замена значения ref — ОК
  await nextTick()              // дождаться рендера нового элемента
}

function removeTodo(id) {
  // замена массива целиком через .value — для ref это безопасно
  todos.value = todos.value.filter((t) => t.id !== id)
}

// Деструктуризация reactive БЕЗ потери реактивности — через toRefs
const { onlyActive, search } = toRefs(filters)
</script>

<template>
  <section>
    <h2>Задачи (осталось: {{ remaining }})</h2>

    <input v-model="newTodo" @keyup.enter="addTodo" placeholder="Новая задача" />
    <button @click="addTodo">Добавить</button>

    <!-- onlyActive/search — это ref после toRefs, в шаблоне без .value -->
    <label><input type="checkbox" v-model="onlyActive" /> Только активные</label>
    <input v-model="search" placeholder="Поиск" />

    <ul>
      <li v-for="todo in visibleTodos" :key="todo.id">
        <input type="checkbox" v-model="todo.done" />
        <span :style="{ textDecoration: todo.done ? 'line-through' : 'none' }">
          {{ todo.text }}
        </span>
        <button @click="removeTodo(todo.id)">×</button>
      </li>
    </ul>
  </section>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. В чём ключевое различие `ref` и `reactive`?**
`ref` оборачивает любое значение (включая примитивы) в объект с `.value`.
`reactive` делает реактивным только объект через `Proxy` и доступен напрямую без
`.value`. `ref` универсальнее и устойчивее к потере реактивности.

**2. Почему у `ref` нужен `.value`, а у `reactive` нет?**
JS не позволяет перехватывать доступ к обычным переменным. `ref` оборачивает
значение в объект и отслеживает доступ через геттер/сеттер `.value`. `reactive`
использует `Proxy` объекта, поэтому перехватывает доступ к свойствам напрямую.

**3. Когда `.value` не нужен у ref?**
В шаблоне (авто-разворачивание для свойств верхнего уровня) и при доступе к ref как
к свойству reactive-объекта (кроме массивов и Map/Set).

**4. Почему деструктуризация `reactive` ломает реактивность? Как исправить?**
Деструктуризация копирует текущее значение свойства в локальную переменную,
разрывая связь с прокси. Для примитивов реактивность теряется. Исправление —
`toRefs(state)` (все свойства в ref) или `toRef(state, 'key')` (одно свойство).

**5. Можно ли заменить reactive-объект целиком? Почему нет?**
Нет. Переприсваивание (`state = reactive({...})`) выбрасывает старый прокси, за
которым следит Vue, и связь теряется. Нужно мутировать существующий объект,
например `Object.assign(state, newData)`.

**6. Почему Vue рекомендует `ref` как основной инструмент?**
`ref` работает с примитивами, переживает замену значения, не страдает от потери
реактивности при деструктуризации (сам ref-объект реактивен), его удобно передавать
между функциями и возвращать из composables.

**7. Реактивны ли массивы и коллекции в Vue 3?**
Да. Через `Proxy` Vue перехватывает мутирующие методы и изменения по индексу/длине.
Поддерживаются `Array`, `Map`, `Set`, `WeakMap`, `WeakSet`.

**8. Что такое `nextTick` и зачем он нужен?**
Vue обновляет DOM асинхронно, батчируя изменения. `nextTick()` возвращает Promise,
который резолвится после применения изменений в DOM. Нужен, когда после изменения
состояния требуется работать с актуальным DOM.

**9. Что делает `<script setup>`?**
Это компайл-тайм сахар: все объявления верхнего уровня (state, функции, импорты)
автоматически доступны в шаблоне без явного `return`. Меньше шаблонного кода,
лучше производительность и вывод типов.

**10. Разворачивается ли ref внутри массива/Map в reactive-объекте?**
Нет. Авто-разворачивание внутри reactive работает для свойств объекта, но НЕ для
элементов массивов и нативных коллекций — там нужен `.value`.

**11. Чем `toRef` отличается от `toRefs`?**
`toRef(obj, 'key')` создаёт один ref, связанный с конкретным свойством. `toRefs(obj)`
конвертирует ВСЕ свойства reactive-объекта в объект из ref. `toRefs` обычно нужен
при деструктуризации/возврате из composable.

**12. Глубокая или поверхностная реактивность у ref и reactive по умолчанию?**
Обе глубокие по умолчанию (вложенные объекты тоже реактивны). Для поверхностной —
`shallowRef` / `shallowReactive`.

**13. Что произойдёт при `reactive(somePrimitive)`?**
Ничего полезного — вернётся само значение без реактивности. `reactive` работает
только с объектами. Для примитивов используйте `ref`.

**14. Сохраняется ли реактивность при `const b = a`, где `a = ref(1)`?**
Да. `b` указывает на тот же ref-объект, реактивная связь живёт внутри ref. Поэтому
`ref` безопасен к копированию ссылки, в отличие от деструктуризации примитива из
reactive.

## Подводные камни (gotchas)

- **Деструктуризация `reactive`** — главная ловушка: примитивы теряют реактивность.
  Используйте `toRefs`/`toRef`.
- **Замена reactive-объекта целиком** — рвёт связь. Только мутации или замена
  свойств.
- **`reactive` с примитивами** — бесполезен.
- **Забытый `.value` в JS-коде** — ошибка «логика не работает», `count + 1` даст
  `[object Object]1`.
- **Ожидание синхронного обновления DOM** — DOM обновляется асинхронно, нужен
  `nextTick`.
- **Ref внутри массива/Map в reactive** — `.value` всё ещё нужен.
- **Прокси не равен исходнику** — `reactive(obj) !== obj`; следите за тем, какой
  объект используете для отслеживания.
- **Передача `state.primitive` в функцию** — копия, не реактивна.

## Лучшие практики

- По умолчанию используйте **`ref`** — он универсальнее и предсказуемее.
- `reactive` уместен для группировки связанного состояния, но помните о его
  ограничениях.
- В composables возвращайте `toRefs(state)` или набор `ref`, чтобы код-потребитель
  мог безопасно деструктурировать.
- Никогда не деструктурируйте `reactive` напрямую без `toRefs`.
- Для замены коллекций используйте `.value =` (ref) или присвоение свойству
  (reactive), не переприсваивайте сам reactive-объект.
- Используйте `<script setup>` как основной стиль для Composition API.
- Для дорогих неглубоких структур рассмотрите `shallowRef`/`shallowReactive`.
- Используйте `nextTick` при необходимости работать с обновлённым DOM.

## Шпаргалка

```vue
<script setup>
import { ref, reactive, toRef, toRefs, nextTick } from 'vue'

// ref — любое значение, доступ через .value (в шаблоне без .value)
const count = ref(0)
count.value++

// reactive — только объект, доступ напрямую
const state = reactive({ count: 0, user: { name: 'Анна' } })
state.count++

// ПОТЕРЯ реактивности
let { count: c } = state      // c — копия, не реактивна!
state = reactive({})          // НЕЛЬЗЯ — теряет реактивность

// СОХРАНЕНИЕ реактивности
const { count: cc } = toRefs(state)  // cc — ref, связан со state.count
const one = toRef(state, 'count')    // один prop -> ref
Object.assign(state, { count: 5 })   // мутация вместо замены

// ref переживает замену значения
const list = ref([1, 2])
list.value = [3, 4]           // OK

// дождаться DOM
await nextTick()
</script>
```

| Хочу | Использую |
|---|---|
| Реактивный примитив | `ref()` |
| Реактивный объект с прямым доступом | `reactive()` |
| Деструктурировать reactive | `toRefs()` / `toRef()` |
| Заменить значение целиком | `ref` + `.value =` |
| Дождаться обновления DOM | `nextTick()` |
| Универсальный безопасный выбор | `ref()` |
```
