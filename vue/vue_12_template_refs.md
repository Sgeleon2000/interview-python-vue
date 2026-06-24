# Vue 3 — Ссылки на элементы (Template Refs) — конспект и вопросы

## О чём раздел

Template ref — это механизм получить **прямой доступ к DOM-элементу или экземпляру дочернего компонента** из шаблона. Нужен, когда декларативного рендеринга недостаточно: фокус на input, измерение размеров, интеграция со сторонними JS-библиотеками (графики, карты, редакторы), ручной вызов методов дочернего компонента, скролл, анимации.

Во Vue 3.5+ появился рекомендуемый API `useTemplateRef()`. До него (и сейчас тоже) работает «старый» способ: объявить `ref` с тем же именем, что и в атрибуте `ref="..."` шаблона.

## Ключевые концепции

- В шаблоне ставится атрибут `ref="имя"` на элемент/компонент.
- **`useTemplateRef('имя')`** (3.5+) — возвращает ref, который после монтирования содержит элемент. Имя в строке должно совпадать с `ref="имя"`.
- **Старый способ**: создать `const el = ref(null)` с именем, совпадающим с `ref="el"` — Vue сам положит элемент в `el.value`.
- **Доступ только после монтирования** — в `onMounted`, не в `setup`/`onBeforeMount` (там `null`).
- **`v-for` + ref** — ссылка становится массивом элементов (или используется функциональный ref).
- **Ref на компонент** даёт его публичный экземпляр; для `<script setup>` методы/данные доступны только через **`defineExpose`**.
- **Функциональный ref** — `:ref="(el) => ..."`: колбэк получает элемент/`null` при размонтировании.

## Подробный разбор с примерами кода

### Новый API — `useTemplateRef()` (Vue 3.5+)

```vue
<script setup>
import { useTemplateRef, onMounted } from 'vue'

// Строка 'input' должна совпадать с ref="input" в шаблоне
const inputRef = useTemplateRef('input')

onMounted(() => {
  // Доступ к DOM-элементу только после монтирования
  inputRef.value.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

Плюсы `useTemplateRef`: имя ref не обязано совпадать с именем переменной, можно динамически передать имя, лучше типизируется в TypeScript.

### Старый способ — одноимённый `ref`

```vue
<script setup>
import { ref, onMounted } from 'vue'

// Имя переменной ДОЛЖНО совпадать с ref="input"
const input = ref(null)

onMounted(() => {
  input.value.focus()
})
</script>

<template>
  <input ref="input" />
</template>
```

Работает за счёт договорённости: при компиляции Vue связывает `ref="input"` с переменной `input` из области видимости setup. Это всё ещё валидно, но в 3.5+ рекомендуется `useTemplateRef`.

### Options API-аналог — `this.$refs`

```vue
<script>
export default {
  mounted() {
    this.$refs.input.focus() // доступ через this.$refs.<имя>
  }
}
</script>

<template>
  <input ref="input" />
</template>
```

### Доступ только после монтирования

```vue
<script setup>
import { useTemplateRef, onBeforeMount, onMounted } from 'vue'

const el = useTemplateRef('box')

console.log(el.value) // null — DOM ещё не создан (тело setup)

onBeforeMount(() => console.log(el.value)) // null — всё ещё нет
onMounted(() => console.log(el.value))     // <div> — доступен
</script>

<template>
  <div ref="box">…</div>
</template>
```

Если элемент за `v-if` и сейчас скрыт, ref будет `null`, пока условие не станет истинным. Чтобы реагировать на появление/исчезновение, можно следить через `watchEffect` с `flush: 'post'`:

```js
import { watchEffect } from 'vue'
watchEffect(() => {
  if (el.value) el.value.focus()
}, { flush: 'post' }) // post — DOM уже обновлён
```

### Refs внутри `v-for`

Если `ref` стоит на элементе внутри `v-for`, после монтирования ref содержит **массив** элементов.

```vue
<script setup>
import { ref, useTemplateRef, onMounted } from 'vue'

const items = ref([1, 2, 3])
const listRefs = useTemplateRef('items') // будет массивом

onMounted(() => {
  console.log(listRefs.value) // [<li>, <li>, <li>]
  listRefs.value.forEach((li) => console.log(li.textContent))
})
</script>

<template>
  <ul>
    <li v-for="i in items" :key="i" ref="items">{{ i }}</li>
  </ul>
</template>
```

> Важно: порядок элементов в массиве **не гарантированно** соответствует порядку в исходном массиве (особенно при перестановках). Если нужна точная привязка элемент↔данные — используйте функциональные refs (см. ниже).

### Функциональные refs

`:ref` можно привязать к функции — она вызывается при монтировании (с элементом) и при размонтировании (с `null`). Удобно для построения собственной Map элемент↔ключ.

```vue
<script setup>
import { ref } from 'vue'

const items = ref(['a', 'b', 'c'])
const itemMap = new Map()

function setItemRef(el, key) {
  if (el) itemMap.set(key, el)   // монтирование
  else itemMap.delete(key)       // размонтирование (el === null)
}
</script>

<template>
  <ul>
    <li
      v-for="key in items"
      :key="key"
      :ref="(el) => setItemRef(el, key)"
    >
      {{ key }}
    </li>
  </ul>
</template>
```

### Ссылка на компонент и доступ к его методам (`defineExpose`)

Ref на дочерний компонент даёт его **публичный экземпляр**. Но компонент на `<script setup>` по умолчанию **закрыт**: его внутренние переменные и методы наружу не видны. Чтобы открыть конкретные методы/данные, используйте `defineExpose`.

**Дочерний компонент** `Child.vue`:

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)
function increment() {
  count.value++
}
function reset() {
  count.value = 0
}

// Явно открываем наружу только нужное
defineExpose({
  count,      // открываем реактивное значение
  increment,  // и методы
  reset,
})
</script>

<template>
  <p>Счётчик ребёнка: {{ count }}</p>
</template>
```

**Родительский компонент**:

```vue
<script setup>
import { useTemplateRef, onMounted } from 'vue'
import Child from './Child.vue'

const childRef = useTemplateRef('child')

onMounted(() => {
  // Доступны только методы/данные, переданные в defineExpose
  childRef.value.increment()
  console.log(childRef.value.count) // читаем открытое значение
})

function resetChild() {
  childRef.value.reset()
}
</script>

<template>
  <Child ref="child" />
  <button @click="resetChild">Сбросить ребёнка</button>
</template>
```

Без `defineExpose` `childRef.value` будет пустым прокси — методы недоступны. Это сделано осознанно: инкапсуляция компонента. Открывайте наружу минимальный нужный набор.

> Для компонентов на Options API инкапсуляции нет: через ref доступны все публичные свойства и методы экземпляра. `defineExpose` нужен именно для `<script setup>`.

### Типизация (TypeScript, кратко)

```vue
<script setup lang="ts">
import { useTemplateRef } from 'vue'
import Child from './Child.vue'

// DOM-элемент
const input = useTemplateRef<HTMLInputElement>('input')

// Экземпляр компонента — тип берётся через InstanceType
const child = useTemplateRef<InstanceType<typeof Child>>('child')
</script>
```

## Полный рабочий пример

`VideoPlayer.vue` (дочерний, открывает методы управления):

```vue
<script setup>
import { ref, defineExpose } from 'vue'

const videoEl = ref(null)
const playing = ref(false)

function play() {
  videoEl.value.play()
  playing.value = true
}
function pause() {
  videoEl.value.pause()
  playing.value = false
}

// Открываем наружу управляющий API плеера
defineExpose({ play, pause, playing })
</script>

<template>
  <video ref="videoEl" src="/demo.mp4" width="320" />
</template>
```

`App.vue` (родитель управляет плеером через ref):

```vue
<script setup>
import { useTemplateRef, onMounted } from 'vue'
import VideoPlayer from './VideoPlayer.vue'

const playerRef = useTemplateRef('player')
const searchInput = useTemplateRef('search')

onMounted(() => {
  // ставим фокус на поле поиска после монтирования
  searchInput.value.focus()
})

function start() {
  playerRef.value.play()   // вызываем открытый метод дочернего компонента
}
function stop() {
  playerRef.value.pause()
}
</script>

<template>
  <input ref="search" placeholder="Поиск…" />

  <VideoPlayer ref="player" />

  <button @click="start">▶ Играть</button>
  <button @click="stop">⏸ Пауза</button>
  <p>Статус: {{ playerRef?.playing ? 'играет' : 'пауза' }}</p>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Зачем нужны template refs, если есть реактивный рендеринг?**
Для императивных операций, недоступных декларативно: фокус/выделение, измерение размеров и позиции, скролл, интеграция со сторонними JS-библиотеками, ручной вызов методов дочернего компонента, медиа-управление.

**2. В чём разница `useTemplateRef` и старого одноимённого `ref`?**
`useTemplateRef('name')` (3.5+) — имя ref задаётся строкой и не обязано совпадать с именем переменной, можно динамически выбирать имя, лучше типизируется. Старый способ требует, чтобы имя переменной совпадало с `ref="..."`. Функционально для статичного случая эквивалентны.

**3. Когда становится доступен элемент в ref?**
Только после монтирования — в `onMounted` или позже. В теле `setup` и `onBeforeMount` ref ещё `null`.

**4. Что лежит в ref при использовании внутри `v-for`?**
Массив элементов. Порядок не гарантирован при перестановках — для точной привязки используйте функциональные refs и собственную Map.

**5. Почему я не могу вызвать метод дочернего компонента через ref?**
Если ребёнок написан на `<script setup>`, он по умолчанию закрыт. Нужно явно открыть методы через `defineExpose({ ... })`. Без этого `ref.value` — пустой прокси.

**6. Что такое `defineExpose` и зачем он нужен?**
Макрос для `<script setup>`, который указывает, какие свойства/методы компонента видны родителю через template ref. По умолчанию `<script setup>` инкапсулирован; `defineExpose` выборочно открывает публичный API.

**7. Нужен ли `defineExpose` для Options API компонента?**
Нет. У Options API компонента через ref доступны все публичные члены экземпляра. `defineExpose` — только для `<script setup>`.

**8. Что такое функциональный ref?**
`:ref="(el) => {...}"`: функция вызывается при монтировании (передаётся элемент) и размонтировании (`null`). Даёт полный контроль, удобно строить Map ключ↔элемент в `v-for`.

**9. Как реагировать на появление элемента за `v-if`?**
Пока условие ложно — ref `null`. Использовать `watchEffect(() => { if (el.value) ... }, { flush: 'post' })`, чтобы среагировать после появления в DOM.

**10. Реактивен ли template ref?**
Да, это обычный `ref`. Можно использовать в `computed`/`watch`/шаблоне (с осторожностью, т.к. до монтирования он `null`).

**11. Как типизировать ref на компонент в TS?**
Через `InstanceType<typeof Child>`. Доступными окажутся только члены из `defineExpose`.

**12. Почему `flush: 'post'` важен при работе с refs в watch?**
Чтобы наблюдатель видел уже обновлённый DOM. При `flush: 'pre'` (по умолчанию) DOM ещё в старом состоянии, и элемент/контент могут быть неактуальны.

**13. Можно ли поставить `ref` на компонент и на элемент одинаково?**
Синтаксис одинаков (`ref="x"`), но на элементе вы получаете DOM-узел, а на компоненте — публичный экземпляр (с учётом `defineExpose`).

**14. Сохраняется ли значение ref после размонтирования?**
Нет — Vue сбрасывает его в `null` при размонтировании элемента/компонента (а функциональный ref вызывается с `null`).

## Подводные камни (gotchas)

- **Доступ к ref в `setup`/`onBeforeMount`** — там `null`. Только `onMounted` и далее.
- **Имя не совпадает**: для старого способа имя переменной должно совпадать с `ref="..."`; для `useTemplateRef` — строка-аргумент с атрибутом.
- **Методы ребёнка не видны** без `defineExpose` (в `<script setup>`).
- **`v-for` refs**: это массив, порядок не гарантирован — используйте функциональные refs при необходимости точной привязки.
- **Элемент за `v-if`**: ref `null`, пока условие ложно.
- **Чтение ref в шаблоне до монтирования** (`playerRef.playing`) — добавляйте `?.`, иначе ошибка обращения к `null`.
- **`watch` за DOM через ref** — используйте `flush: 'post'`, иначе DOM устаревший.
- **Чрезмерное использование refs** — признак императивного кода там, где хватило бы декларативного (props/events/v-model).

## Лучшие практики

- В новых проектах (3.5+) предпочитайте `useTemplateRef`.
- Обращайтесь к refs в `onMounted` или через `watchEffect({ flush: 'post' })`.
- Открывайте наружу через `defineExpose` минимально необходимый API ребёнка.
- Предпочитайте props/emit/`v-model` императивному управлению через refs; refs — для случаев, где декларативность не подходит.
- Для коллекций в `v-for` с привязкой элемент↔данные используйте функциональные refs.
- В TS типизируйте refs (`HTMLInputElement`, `InstanceType<typeof Child>`).
- Защищайте чтение в шаблоне опциональной цепочкой `?.`.

## Шпаргалка

```text
ШАБЛОН:          <input ref="input" />
НОВЫЙ API (3.5+): const r = useTemplateRef('input'); r.value.focus()
СТАРЫЙ API:       const input = ref(null) // имя == ref="input"
OPTIONS API:      this.$refs.input

ДОСТУП:          только после монтирования (onMounted), иначе null
v-FOR:           ref → массив элементов (порядок не гарантирован)
ФУНКЦИОНАЛЬНЫЙ:  :ref="(el) => map.set(key, el)"  // el===null при unmount

КОМПОНЕНТ:
  ребёнок (<script setup>) ЗАКРЫТ → defineExpose({ method, value })
  родитель: childRef.value.method()
  Options API ребёнок: всё публичное доступно без defineExpose

WATCH за DOM:    watchEffect(() => {...}, { flush: 'post' })
TS:              useTemplateRef<HTMLInputElement>('x')
                 useTemplateRef<InstanceType<typeof Child>>('child')
ШАБЛОН-ЧТЕНИЕ:   childRef?.value  / playerRef?.playing  // защита от null
```

```vue
<script setup>
import { useTemplateRef, onMounted } from 'vue'
import Child from './Child.vue'

const child = useTemplateRef('child')
onMounted(() => child.value.doSomething()) // метод открыт через defineExpose в Child
</script>

<template>
  <Child ref="child" />
</template>
```
