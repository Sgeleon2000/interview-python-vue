# Vue 3 — Пользовательские события компонентов (Component Events) — конспект и вопросы

## О чём раздел

Компоненты в Vue общаются «вниз» через пропсы (родитель → потомок) и «вверх» через события (потомок → родитель). Пользовательские (кастомные) события — это механизм, с помощью которого дочерний компонент сообщает родителю, что что-то произошло (нажали кнопку, изменилось значение, форма отправлена), и при необходимости передаёт данные.

В этом разделе разбираем:
- как испускать события через `emit` (Composition API, `<script setup>`) и `$emit` (в шаблоне);
- именование событий: `camelCase` в JS / `kebab-case` в шаблоне;
- передачу аргументов вместе с событием;
- декларацию и валидацию событий через `defineEmits`;
- сравнение «событие vs callback-проп»;
- особенности всплытия: у кастомных событий нет нативного bubbling.

## Ключевые концепции

- **`defineEmits`** — макрос компилятора (только в `<script setup>`), который объявляет, какие события компонент может испускать, и возвращает функцию `emit`.
- **`emit(eventName, ...args)`** — вызывает событие. Родитель слушает его через `@event-name`.
- **`$emit`** — то же самое, доступно прямо в `<template>` без объявления переменной.
- **Однонаправленный поток**: пропсы вниз, события вверх. Дочерний компонент не мутирует пропсы напрямую, а просит родителя это сделать через событие.
- **Именование**: в JS принято `camelCase` (`updateCount`), в шаблоне слушатель — `kebab-case` (`@update-count`). Vue автоматически сопоставляет их.
- **Нет нативного всплытия**: кастомное событие приходит только прямому родителю. Оно не «всплывает» через дерево компонентов, как нативные DOM-события.

## Подробный разбор с примерами кода

### 1. Базовый emit через `defineEmits`

```vue
<!-- BlogPost.vue -->
<script setup>
// Объявляем список испускаемых событий и получаем функцию emit
const emit = defineEmits(['enlarge-text'])

function onClick() {
  // Испускаем событие — родитель его поймает
  emit('enlarge-text')
}
</script>

<template>
  <div class="blog-post">
    <h4>Заголовок поста</h4>
    <!-- В шаблоне можно вызывать $emit напрямую, без defineEmits-переменной -->
    <button @click="$emit('enlarge-text')">Увеличить текст (через $emit)</button>
    <button @click="onClick">Увеличить текст (через emit из JS)</button>
  </div>
</template>
```

Родитель слушает событие:

```vue
<!-- Parent.vue -->
<script setup>
import { ref } from 'vue'
import BlogPost from './BlogPost.vue'

const fontSize = ref(1)
</script>

<template>
  <div :style="{ fontSize: fontSize + 'em' }">
    <!-- @enlarge-text — слушатель кастомного события -->
    <BlogPost @enlarge-text="fontSize += 0.1" />
  </div>
</template>
```

### 2. Передача аргументов с событием

Любые аргументы после имени события передаются обработчику родителя.

```vue
<!-- Child.vue -->
<script setup>
const emit = defineEmits(['increase-by'])

function add() {
  // Передаём число вторым аргументом
  emit('increase-by', 5)
}
</script>

<template>
  <button @click="add">+5</button>
  <!-- В шаблоне аргументы передаются так же -->
  <button @click="$emit('increase-by', 10)">+10</button>
</template>
```

```vue
<!-- Parent.vue -->
<template>
  <!-- Вариант 1: инлайн, $event — это первый переданный аргумент -->
  <Child @increase-by="(n) => count += n" />

  <!-- Вариант 2: метод-обработчик получает аргументы напрямую -->
  <Child @increase-by="onIncrease" />
</template>

<script setup>
import { ref } from 'vue'
import Child from './Child.vue'

const count = ref(0)

function onIncrease(amount) {
  count.value += amount
}
</script>
```

Можно передавать несколько аргументов:

```js
emit('submit', formData, isValid) // обработчик: (data, valid) => {...}
```

### 3. Именование событий: camelCase в JS / kebab-case в шаблоне

```vue
<script setup>
// Объявляем в camelCase
const emit = defineEmits(['updateCount'])
emit('updateCount', 1)
</script>
```

В шаблоне родителя слушаем в `kebab-case` (рекомендованный стиль, согласованный с HTML):

```vue
<!-- Рекомендуется kebab-case -->
<Child @update-count="..." />
```

В отличие от пропсов и компонентов, имена событий **не** проходят автоматическое преобразование регистра при сопоставлении. Однако `v-on` в шаблоне автоматически преобразует `@update-count` в слушатель события `updateCount`. Поэтому на практике: объявляйте события в camelCase, слушайте в kebab-case — и всё совпадёт.

### 4. Декларация и валидация событий

Объявление событий улучшает читаемость, документирует «контракт» компонента и позволяет Vue понять, что это слушатели событий, а не fallthrough-атрибуты (про fallthrough — отдельный раздел).

Объектный синтаксис позволяет **валидировать** payload. Функция-валидатор получает аргументы, переданные в `emit`, и должна вернуть `boolean` (валидно/нет). Возврат `false` приводит к предупреждению в консоли (в dev-режиме), но не блокирует событие.

```vue
<script setup>
const emit = defineEmits({
  // Без валидации — payload не проверяется
  click: null,

  // С валидацией: проверяем структуру payload
  submit: ({ email, password }) => {
    if (email && password) {
      return true
    } else {
      console.warn('Невалидный payload события submit!')
      return false
    }
  }
})

function onSubmit() {
  emit('submit', { email: 'a@b.c', password: '123' })
}
</script>
```

### 5. Типизация событий в TypeScript

С TypeScript используется типовой (type-only) синтаксис `defineEmits`:

```vue
<script setup lang="ts">
// Старый «вызовный» синтаксис
const emit = defineEmits<{
  (e: 'change', id: number): void
  (e: 'update', value: string): void
}>()

// Новый, более лаконичный (Vue 3.3+) — кортежи аргументов
const emit2 = defineEmits<{
  change: [id: number]
  update: [value: string]
}>()
</script>
```

### 6. События vs callback-пропсы

В Vue, в отличие от React, идиоматично передавать события, а не callback-пропсы. Но технически можно и так:

```vue
<!-- Callback-проп: родитель передаёт функцию -->
<Child :on-submit="handleSubmit" />
```

```vue
<script setup>
const props = defineProps(['onSubmit'])
props.onSubmit?.(data) // вызываем callback
</script>
```

Сравнение:

| Аспект | Событие (`emit`) | Callback-проп |
|---|---|---|
| Идиоматичность во Vue | Да, рекомендуется | Реже, но допустимо |
| Несколько слушателей | Легко: `@evt="a" @evt="b"`... (см. ниже) | Один callback на проп |
| Декларация/валидация | `defineEmits` | через `defineProps` |
| Модификаторы (`.once`, `.stop`) | Поддерживаются | Нет |
| Интеграция с `v-model` | Да (`update:modelValue`) | Нет |

Замечание: на одном слушателе нельзя «магически» подвесить два обработчика одинаковым `@evt`, но callback-пропы тоже ограничены. Главные плюсы событий — модификаторы `v-on` и нативная интеграция с `v-model`.

### 7. Отсутствие нативного всплытия (no bubbling)

Кастомные события Vue **не всплывают** по дереву компонентов. Если `Grandchild` испускает событие, его поймает только прямой родитель `Child`, но не `Parent`. Чтобы «пробросить» событие выше, нужно ретранслировать его вручную:

```vue
<!-- Child.vue — ретрансляция события наверх -->
<script setup>
const emit = defineEmits(['notify'])
</script>

<template>
  <!-- Ловим notify от внука и пробрасываем дальше родителю -->
  <Grandchild @notify="emit('notify', $event)" />
</template>
```

Либо использовать паттерны: provide/inject, общий store (Pinia), event bus (mitt). Нативные DOM-события (`click` на `<button>`) всплывают как обычно — речь именно о кастомных событиях компонентов.

## Полный рабочий пример

```vue
<!-- TodoInput.vue — дочерний компонент с несколькими событиями -->
<script setup>
import { ref } from 'vue'

// Декларация и валидация событий
const emit = defineEmits({
  // Событие добавления: payload должен быть непустой строкой
  add: (text) => typeof text === 'string' && text.trim().length > 0,
  // Событие отмены — без payload
  cancel: null
})

const text = ref('')

function submit() {
  if (!text.value.trim()) return
  // Передаём текст вместе с событием
  emit('add', text.value.trim())
  text.value = ''
}
</script>

<template>
  <form @submit.prevent="submit">
    <input v-model="text" placeholder="Новая задача" />
    <button type="submit">Добавить</button>
    <!-- $emit прямо в шаблоне, событие без аргументов -->
    <button type="button" @click="$emit('cancel')">Отмена</button>
  </form>
</template>
```

```vue
<!-- TodoApp.vue — родитель -->
<script setup>
import { ref } from 'vue'
import TodoInput from './TodoInput.vue'

const todos = ref([])

// Обработчик получает payload из emit('add', text)
function addTodo(text) {
  todos.value.push({ id: Date.now(), text })
}

function onCancel() {
  console.log('Ввод отменён')
}
</script>

<template>
  <!-- camelCase-событие add слушаем как @add; kebab — для составных имён -->
  <TodoInput @add="addTodo" @cancel="onCancel" />

  <ul>
    <li v-for="todo in todos" :key="todo.id">{{ todo.text }}</li>
  </ul>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `emit` отличается от `$emit`?**
Это одна и та же возможность. `$emit` доступен в шаблоне без объявлений. `emit` — это функция, возвращённая `defineEmits()` в `<script setup>`, нужна, когда событие испускается из JS-логики. Под капотом оба вызывают внутренний механизм инстанса.

**2. Как объявить события в `<script setup>`?**
Через макрос `defineEmits(['e1', 'e2'])` (массив) или `defineEmits({ e1: validator })` (объект с валидацией). Возвращает функцию `emit`. В TS — `defineEmits<{ change: [id: number] }>()`.

**3. Зачем вообще объявлять события, если можно просто вызвать `$emit`?**
Декларация документирует контракт компонента, включает валидацию payload, помогает IDE/TS и, главное, отличает слушатели событий от fallthrough-атрибутов — объявленные события не попадают в `$attrs` и не вешаются как нативные слушатели на корневой элемент.

**4. Как передать данные вместе с событием?**
Аргументами после имени: `emit('update', value, extra)`. Родитель получит их в обработчике: `@update="(value, extra) => ..."`. В инлайн-выражении первый аргумент доступен как `$event`.

**5. Почему рекомендуют camelCase в JS и kebab-case в шаблоне?**
camelCase — естественный стиль для JS-идентификаторов. kebab-case в шаблоне согласован с HTML-атрибутами и не вызывает проблем с регистром. `v-on` корректно сопоставит `@update-text` с событием `updateText`. ВАЖНО: в отличие от пропсов, имена событий не нормализуются автоматически при ручном вызове, но `v-on` это делает.

**6. Всплывают ли кастомные события?**
Нет. Кастомное событие компонента доходит только до прямого родителя. Для проброса выше нужно ретранслировать (`@evt="emit('evt', $event)"`) или использовать provide/inject, store, event bus. Нативные DOM-события — всплывают как обычно.

**7. Как провалидировать payload события?**
Объектным синтаксисом `defineEmits`: `{ submit: (payload) => boolean }`. Возврат `false` даёт предупреждение в dev-режиме, но не отменяет испускание события.

**8. Событие vs callback-проп — что выбрать?**
Во Vue идиоматичны события: они поддерживают модификаторы `v-on` (`.once`, `.stop`, `.prevent`), интегрируются с `v-model`, чётче читаются. Callback-пропсы допустимы (`:on-action="fn"`), но используются реже — например, для совместимости или императивных API.

**9. Можно ли вызвать `emit` несколько раз / для нескольких событий?**
Да, в `defineEmits` объявляют все события, и `emit('a')`, `emit('b', x)` можно вызывать в любой момент логики компонента.

**10. Как затипизировать события в TypeScript?**
`const emit = defineEmits<{ change: [id: number]; close: [] }>()` — лаконичный синтаксис кортежей (3.3+) или старый вызовный `(e: 'change', id: number): void`.

**11. Что вернёт `emit`?**
Ничего полезного (`undefined`). Это не способ получить результат от родителя — поток однонаправленный. Если нужен «ответ», родитель меняет проп, который придёт обратно вниз.

**12. Сработает ли `@click` на компоненте без объявления события `click`?**
Если у компонента один корневой элемент и `inheritAttrs` не отключён, необъявленный `@click` уйдёт как fallthrough-слушатель на корневой DOM-элемент (нативный click). Если же объявить `click` в `defineEmits`, он перестанет быть нативным и станет кастомным — нужно явно `emit('click')`.

## Подводные камни (gotchas)

- **Регистр имён**: при ручном `emit('updateText')` в шаблоне слушайте как `@update-text` или `@updateText` — `v-on` нормализует. Но если объявите событие в kebab (`'update-text'`) и зовёте `emit('updateText')` — не совпадёт. Будьте последовательны.
- **Объявленное событие исчезает из `$attrs`**: если объявили `click` в `defineEmits`, нативный слушатель `@click` от родителя больше не повесится автоматически на корень — теперь это ваше кастомное событие.
- **Нет bubbling**: типичная ошибка — ждать событие от внука на уровне дедушки. Не придёт без ретрансляции.
- **Валидатор не блокирует**: возврат `false` из валидатора лишь печатает warning в dev. Событие всё равно испускается.
- **`emit` не возвращает значение**: не используйте его как RPC к родителю.
- **`defineEmits` — только в `<script setup>`** и только на верхнем уровне; нельзя вызывать условно или импортировать.

## Лучшие практики

- Всегда объявляйте события через `defineEmits` — это контракт компонента.
- Именуйте события глаголами в прошедшем времени/намерении: `submit`, `update:modelValue`, `delete`, `change`.
- Используйте `camelCase` в JS, `kebab-case` в шаблоне родителя.
- Передавайте минимально необходимый payload; для сложных данных — объект.
- Для двусторонней привязки используйте конвенцию `update:xxx` (см. раздел про v-model).
- В TypeScript типизируйте payload через type-only `defineEmits`.
- Не пытайтесь полагаться на всплытие — ретранслируйте события явно или используйте store.

## Шпаргалка

```vue
<script setup>
// Массив
const emit = defineEmits(['change', 'delete'])

// Объект с валидацией
const emit = defineEmits({
  submit: (payload) => !!payload.email,
  cancel: null
})

// TypeScript (3.3+)
const emit = defineEmits<{
  change: [id: number]
  update: [value: string, ts: number]
}>()

emit('change', 42)            // с аргументом
emit('submit', { email })     // объект-payload
</script>

<template>
  <!-- $emit прямо в шаблоне -->
  <button @click="$emit('change', 1)">+1</button>
</template>
```

```vue
<!-- Родитель -->
<Child
  @change="onChange"               
  @delete="(id) => remove(id)"     
  @some-event="onEvent"            
/>

<!-- Ретрансляция (нет нативного всплытия) -->
<Grandchild @notify="emit('notify', $event)" />
```

| Шаблон (kebab) | JS (camelCase) |
|---|---|
| `@update-count` | `emit('updateCount')` |
| `@enlarge-text` | `emit('enlargeText')` |
| `@submit` | `emit('submit')` |
