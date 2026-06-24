# Vue 3 — v-model на компонентах — конспект и вопросы

## О чём раздел

`v-model` на нативных элементах (`<input v-model="text">`) — это синтаксический сахар для двусторонней привязки. На **компонентах** `v-model` позволяет реализовать собственную двустороннюю связь: родитель передаёт значение вниз и автоматически получает обновления наверх.

В разделе разбираем:
- современный `defineModel()` (Vue 3.4+) и во что он разворачивается;
- «старый» способ через проп `modelValue` + событие `update:modelValue`;
- аргументы `v-model` (`v-model:title`);
- несколько `v-model` на одном компоненте;
- модификаторы — встроенные (`.trim`, `.number`, `.lazy`) и кастомные;
- как делать собственные компоненты с двусторонней привязкой.

## Ключевые концепции

- **`v-model="x"` на компоненте** = передача пропа + слушатель события обновления.
- **`defineModel()`** — макрос (3.4+), возвращающий ref, который синхронизирован с родителем: чтение даёт текущее значение пропа, запись испускает событие обновления.
- **Конвенция по умолчанию**: проп `modelValue` + событие `update:modelValue`.
- **Аргумент**: `v-model:title` → проп `title` + событие `update:title`.
- **Несколько v-model**: на одном компоненте можно навесить `v-model:firstName` и `v-model:lastName`.
- **Модификаторы**: `v-model.trim`, `.number`, `.lazy` (встроенные для нативных) и кастомные, доступные через `defineModel`.

## Подробный разбор с примерами кода

### 1. Современный способ — `defineModel()` (Vue 3.4+)

```vue
<!-- CustomInput.vue -->
<script setup>
// model — это ref, двусторонне связанный с родителем
const model = defineModel()
</script>

<template>
  <!-- Читаем и пишем как в обычный ref -->
  <input v-model="model" />
</template>
```

Использование у родителя ровно как с нативным элементом:

```vue
<script setup>
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'
const text = ref('Привет')
</script>

<template>
  <CustomInput v-model="text" />
  <p>{{ text }}</p>
</template>
```

### 2. ВО ЧТО РАЗВОРАЧИВАЕТСЯ `defineModel()` (главное!)

`defineModel()` — это макрос компилятора. На этапе сборки он разворачивается в **проп + событие**. Грубо говоря, этот код:

```vue
<script setup>
const model = defineModel()
</script>
```

эквивалентен такому:

```vue
<script setup>
import { computed } from 'vue'

// 1) Объявляется проп modelValue
const props = defineProps(['modelValue'])
// 2) Объявляется событие update:modelValue
const emit = defineEmits(['update:modelValue'])

// 3) Создаётся writable computed-обёртка
const model = computed({
  get() {
    return props.modelValue
  },
  set(value) {
    emit('update:modelValue', value)
  }
})
</script>
```

То есть `model` — это writable `computed`-подобный ref: при чтении возвращает `props.modelValue`, при записи испускает `emit('update:modelValue', value)`. Родительский `v-model="text"` сам разворачивается в `:model-value="text" @update:model-value="text = $event"`. Замечание: в реальной реализации Vue использует не буквально `computed`, а специальный внутренний ref для корректной обработки локального состояния и модификаторов, но семантически это именно «геттер из пропа, сеттер через emit».

### 3. «Старый» способ (до 3.4) — modelValue + update:modelValue

До появления `defineModel` двустороннюю привязку писали руками:

```vue
<!-- CustomInput.vue (классический способ) -->
<script setup>
defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

Это полезно знать: `defineModel` — лишь сахар над этой парой «проп + событие». На собеседовании часто просят написать именно классический вариант, чтобы показать понимание механики.

Эквивалентная развёртка родительского `v-model`:

```vue
<!-- Эти две строки эквивалентны -->
<CustomInput v-model="text" />
<CustomInput :model-value="text" @update:model-value="text = $event" />
```

### 4. Аргументы v-model

По умолчанию используется проп `modelValue`. Аргумент меняет имя:

```vue
<!-- Родитель -->
<MyComponent v-model:title="bookTitle" />
<!-- разворачивается в: -->
<MyComponent :title="bookTitle" @update:title="bookTitle = $event" />
```

```vue
<!-- MyComponent.vue -->
<script setup>
// Передаём имя модели аргументом
const title = defineModel('title')
</script>

<template>
  <input v-model="title" />
</template>
```

Классический эквивалент:

```vue
<script setup>
defineProps(['title'])
const emit = defineEmits(['update:title'])
</script>
```

### 5. Несколько v-model на одном компоненте

Благодаря аргументам можно навесить несколько независимых привязок:

```vue
<!-- UserName.vue -->
<script setup>
const firstName = defineModel('firstName')
const lastName = defineModel('lastName')
</script>

<template>
  <input v-model="firstName" placeholder="Имя" />
  <input v-model="lastName" placeholder="Фамилия" />
</template>
```

```vue
<!-- Родитель -->
<UserName
  v-model:first-name="first"
  v-model:last-name="last"
/>
```

### 6. Модификаторы v-model

**Встроенные** модификаторы для нативного `v-model`: `.lazy` (синхронизация по `change`, а не `input`), `.number` (приведение к числу), `.trim` (обрезка пробелов).

**Кастомные** модификаторы для своих компонентов доступны через `defineModel`. Макрос возвращает вторым элементом объект модификаторов:

```vue
<!-- MyInput.vue -->
<script setup>
// Деструктурируем: model — значение, modifiers — объект модификаторов
const [model, modifiers] = defineModel({
  // Опции пропа можно передать тут же
  // set вызывается при записи значения — реализуем модификатор
  set(value) {
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
    return value
  }
})
</script>

<template>
  <input v-model="model" />
</template>
```

```vue
<!-- Родитель использует модификатор .capitalize -->
<MyInput v-model.capitalize="text" />
```

`modifiers` будет `{ capitalize: true }`. Для именованной модели имя модификатора задаётся так: `v-model:title.capitalize`, а в компоненте — `const [title, titleModifiers] = defineModel('title', { ... })`.

### 7. Геттер/сеттер и опции в defineModel

`defineModel` принимает опции, как `defineProps` (`type`, `required`, `default`), плюс `get`/`set` для трансформации:

```vue
<script setup>
const model = defineModel({
  type: String,
  default: '',
  required: true,
  // Трансформация при чтении из родителя
  get(value) {
    return value.trim()
  },
  // Трансформация при записи к родителю
  set(value) {
    return value.toLowerCase()
  }
})
</script>
```

### 8. Типизация в TypeScript

```vue
<script setup lang="ts">
// Тип значения модели
const model = defineModel<string>()

// С опциями и required
const count = defineModel<number>({ required: true })

// Именованная модель
const title = defineModel<string>('title', { default: '' })
</script>
```

## Полный рабочий пример

```vue
<!-- MoneyInput.vue — собственный input с двусторонней привязкой и модификатором -->
<script setup>
// Кастомный модификатор .uppercase для текстового описания
const [amount, modifiers] = defineModel('amount', {
  type: Number,
  default: 0,
  set(value) {
    // Запрещаем отрицательные значения при записи наверх
    return Math.max(0, Number(value) || 0)
  }
})

const description = defineModel('description', {
  type: String,
  default: ''
})

// модификаторы для description нужно объявлять отдельной моделью при необходимости
</script>

<template>
  <div class="money-input">
    <input v-model.number="amount" type="number" placeholder="Сумма" />
    <input v-model="description" placeholder="Описание" />
  </div>
</template>
```

```vue
<!-- PaymentForm.vue — родитель с несколькими v-model -->
<script setup>
import { ref } from 'vue'
import MoneyInput from './MoneyInput.vue'

const sum = ref(100)
const note = ref('Оплата')
</script>

<template>
  <MoneyInput
    v-model:amount="sum"
    v-model:description="note"
  />
  <p>Сумма: {{ sum }} ₽, описание: {{ note }}</p>
</template>
```

Классическая реализация того же компонента (без `defineModel`) для сравнения:

```vue
<!-- MoneyInputClassic.vue -->
<script setup>
const props = defineProps({
  amount: { type: Number, default: 0 },
  description: { type: String, default: '' }
})
const emit = defineEmits(['update:amount', 'update:description'])
</script>

<template>
  <input
    :value="amount"
    type="number"
    @input="emit('update:amount', Math.max(0, Number($event.target.value) || 0))"
  />
  <input
    :value="description"
    @input="emit('update:description', $event.target.value)"
  />
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Во что разворачивается `v-model` на компоненте?**
`v-model="x"` → проп `modelValue` + слушатель `@update:modelValue`, то есть `:model-value="x" @update:model-value="x = $event"`. С аргументом `v-model:title="x"` → проп `title` + событие `update:title`.

**2. Что делает `defineModel()` и во что он компилируется?**
Это макрос (3.4+), возвращающий writable ref. Компилятор разворачивает его в `defineProps(['modelValue'])` + `defineEmits(['update:modelValue'])` + writable computed: геттер читает `props.modelValue`, сеттер делает `emit('update:modelValue', value)`. Чтение ref = значение пропа, запись = испускание события.

**3. Как было до Vue 3.4?**
Руками: объявляли проп `modelValue`, событие `update:modelValue`, и в шаблоне `:value="modelValue" @input="emit('update:modelValue', $event.target.value)"`.

**4. Чем v-model в Vue 3 отличается от Vue 2?**
В Vue 2 по умолчанию был проп `value` + событие `input`, и только один v-model на компонент (для второго использовали `.sync`). В Vue 3 по умолчанию `modelValue` + `update:modelValue`, поддержка нескольких v-model через аргументы, `.sync` удалён.

**5. Как сделать несколько v-model на одном компоненте?**
Через аргументы: `v-model:firstName` и `v-model:lastName`, в компоненте — `defineModel('firstName')` и `defineModel('lastName')`. Каждая модель = свой проп + своё событие `update:xxx`.

**6. Как задать значение по умолчанию для модели?**
`defineModel({ default: 0 })` или с типом `defineModel({ type: String, default: '' })`. Аналогично опциям пропа.

**7. Как реализовать кастомный модификатор v-model?**
Деструктурировать макрос: `const [model, modifiers] = defineModel({ set(v) { return modifiers.capitalize ? cap(v) : v } })`. Родитель применяет `v-model.capitalize="x"`, а `modifiers` будет `{ capitalize: true }`.

**8. Что делают `get`/`set` в опциях `defineModel`?**
`get(value)` трансформирует значение при чтении из родителя; `set(value)` трансформирует перед испусканием события обновления. Удобно для нормализации (trim, lowercase, ограничения диапазона).

**9. Можно ли просто мутировать значение, полученное от `defineModel`?**
Да — `model.value = 'new'` автоматически испускает `update:modelValue`. Это и есть удобство: ref ведёт себя как локальное состояние, но синхронизирован с родителем.

**10. Зачем computed-обёртка, почему не мутировать проп напрямую?**
Пропсы read-only (однонаправленный поток). Прямая мутация пропа — антипаттерн и вызовет warning. Writable computed/`defineModel` корректно «спрашивает» родителя обновить значение через событие.

**11. Что если родитель не слушает `update:modelValue`?**
Тогда привязка станет односторонней: значение придёт вниз, но обновления потеряются. На практике `v-model` всегда вешает слушатель, так что проблема возникает лишь при ручной передаче только `:model-value`.

**12. Как затипизировать модель в TS?**
`const model = defineModel<string>()`, `defineModel<number>({ required: true })`, для именованной — `defineModel<string>('title')`.

## Подводные камни (gotchas)

- **`defineModel` требует Vue 3.4+**. На более старых версиях используйте `modelValue` + `update:modelValue` вручную.
- **Имя события строго `update:xxx`** — иначе двусторонняя привязка не сработает. Для `modelValue` это `update:modelValue`.
- **Не мутируйте проп напрямую** в классическом варианте — только через `emit('update:...')`.
- **`v-model:firstName` в шаблоне пишут как `v-model:first-name`** (kebab), а проп объявляют `firstName`/`first-name` — Vue нормализует, но событие `update:firstName`.
- **Модификаторы — отдельно на каждую модель**: при нескольких v-model имя модификатора привязано к конкретной модели (`v-model:title.cap` → модификаторы только для `title`).
- **`get`/`set` влияют на направление**: путаница get vs set — частая ошибка. get = из родителя в компонент, set = из компонента в родителя.
- **Реактивность default**: для объектов/массивов в `default` используйте фабрику, как в обычных пропсах.

## Лучшие практики

- Для новых проектов на 3.4+ — используйте `defineModel()`: меньше шаблонного кода.
- Для нормализации значений применяйте `get`/`set`, а не вотчеры.
- Давайте моделям осмысленные имена через аргументы (`v-model:title`), когда их несколько.
- Объявляйте тип и default модели для надёжности.
- Понимайте развёртку (проп + `update:` событие) — это помогает дебажить и писать совместимый код.
- В TypeScript всегда типизируйте `defineModel<T>()`.

## Шпаргалка

```vue
<!-- defineModel (3.4+) -->
<script setup>
const model = defineModel()                       // modelValue
const title = defineModel('title')                // именованная
const count = defineModel({ type: Number, default: 0 })
const [m, mods] = defineModel({ set: v => ... })  // + модификаторы
const tm = defineModel('title', { get, set })     // get/set
</script>
```

```vue
<!-- Классика (эквивалент defineModel()) -->
<script setup>
defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
const model = computed({
  get: () => props.modelValue,
  set: v => emit('update:modelValue', v)
})
</script>
```

| Запись | Разворачивается в |
|---|---|
| `v-model="x"` | `:model-value="x" @update:model-value="x=$event"` |
| `v-model:title="x"` | `:title="x" @update:title="x=$event"` |
| `v-model.cap="x"` | + объект `modifiers = { cap: true }` |
| несколько | `v-model:a` + `v-model:b` (независимые) |
