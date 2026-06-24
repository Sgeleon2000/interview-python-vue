# Vue 3 — TypeScript во Vue — конспект и вопросы

## О чём раздел

TypeScript во Vue 3 — это первоклассная (first-class) поддержка типов в шаблонах,
компонентах, сторе и реактивных примитивах. Vue 3 переписан на TypeScript,
а компилятор шаблонов (через **Volar / Vue Language Tools**) умеет проверять типы
прямо внутри `<template>`. В этом разделе разберём настройку проекта, типизацию
`defineProps`/`defineEmits`, реактивных переменных, `computed`, ref'ов на компоненты,
`provide/inject`, стора Pinia, generic-компоненты и типизацию `$attrs`.

Современный стиль — `<script setup lang="ts">` + Composition API.

## Ключевые концепции

- **vue-tsc** — CLI-обёртка над `tsc`, которая умеет проверять типы в `.vue`-файлах
  (обычный `tsc` про `.vue` ничего не знает).
- **Volar (Vue - Official)** — расширение для редактора, дающее типизацию и
  автодополнение в шаблонах. С версии 2.x работает в режиме "Hybrid Mode" вместе с
  плагином TypeScript.
- **Type-based объявления** — `defineProps<T>()`, `defineEmits<T>()` принимают
  generic-параметр с типами вместо runtime-объекта. Компилятор сам генерирует
  runtime-валидацию.
- **`withDefaults`** — функция для задания значений по умолчанию при type-based props.
- **`InjectionKey<T>`** — типизированный ключ для `provide`/`inject`.
- **`InstanceType<typeof Comp>`** — извлечение типа экземпляра компонента для ref'ов.
- **generic в `<script setup>`** — атрибут `generic="T"` для компонентов-дженериков.

## Подробный разбор с примерами кода

### Настройка проекта

`tsconfig.json` (фрагмент, типично из `create-vue`):

```jsonc
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,            // обязательно для качественной типизации
    "jsx": "preserve",         // если используете JSX/TSX
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": ["ESNext", "DOM", "DOM.Iterable"]
  }
}
```

`package.json` — проверка типов как отдельный шаг сборки:

```jsonc
{
  "scripts": {
    "build": "vue-tsc --noEmit && vite build",
    "type-check": "vue-tsc --noEmit"
  }
}
```

> Важно: Vite сам по себе НЕ проверяет типы (esbuild только транспилирует, отбрасывая
> типы). Поэтому проверку типов выносят в `vue-tsc --noEmit`, обычно через
> `npm-run-all` / плагин `vite-plugin-checker`.

### Типизация props через `defineProps<T>()`

```vue
<script setup lang="ts">
// Type-based объявление — самый идиоматичный способ
interface Props {
  title: string
  count?: number          // опциональный => не required
  tags: string[]
  user: { id: number; name: string }
}

const props = defineProps<Props>()
// props.title: string, props.count: number | undefined
</script>
```

#### `withDefaults` — значения по умолчанию

```vue
<script setup lang="ts">
interface Props {
  size?: 'sm' | 'md' | 'lg'
  items?: string[]
}

// withDefaults оборачивает defineProps и задаёт дефолты
const props = withDefaults(defineProps<Props>(), {
  size: 'md',
  // для объектов/массивов дефолт ОБЯЗАТЕЛЬНО через фабрику
  items: () => ['a', 'b'],
})
// props.size: 'sm' | 'md' | 'lg' (уже не undefined)
</script>
```

> С Vue 3.5+ можно использовать **Reactive Props Destructure** с нативными
> дефолтами JS — это предпочтительный современный способ:

```vue
<script setup lang="ts">
// Vue 3.5+: деструктуризация props с дефолтами через = 
// Компилятор сохраняет реактивность (не теряется при деструктуризации!)
const { size = 'md', items = [] } = defineProps<{
  size?: 'sm' | 'md' | 'lg'
  items?: string[]
}>()
</script>
```

### Типизация emits через `defineEmits<T>()`

```vue
<script setup lang="ts">
// Современный синтаксис (Vue 3.3+) — call signature через объект
const emit = defineEmits<{
  change: [id: number]              // именованный кортеж аргументов
  update: [value: string, ok: boolean]
  close: []                          // событие без аргументов
}>()

emit('change', 42)
emit('update', 'hello', true)
emit('close')
</script>
```

Старый (но валидный) синтаксис через call signatures:

```ts
const emit = defineEmits<{
  (e: 'change', id: number): void
  (e: 'update', value: string): void
}>()
```

### Типизация `ref<T>()` и `reactive`

```ts
import { ref, reactive } from 'vue'

// 1. Вывод типа автоматически
const count = ref(0)            // Ref<number>
count.value++                   // ok

// 2. Явный generic — когда нужен union или начальное значение неточное
const msg = ref<string | null>(null)
msg.value = 'hi'

// 3. Сложный тип
interface Todo { id: number; done: boolean }
const todos = ref<Todo[]>([])

// reactive — тип выводится из объекта, generic указывать НЕ принято
const state = reactive({ open: false, name: 'Vue' })
// при необходимости — интерфейс:
const state2: { open: boolean } = reactive({ open: false })
```

> Нюанс: `ref` разворачивает (unwrap) вложенные ref'ы. Тип `Ref<T>` — это про
> `.value`. В шаблоне `.value` не пишут — Vue разворачивает автоматически.

### Типизация `computed`

```ts
import { ref, computed } from 'vue'

const count = ref(0)

// Тип выводится из return
const double = computed(() => count.value * 2)  // ComputedRef<number>

// Явный generic, если нужно сузить/расширить тип
const label = computed<string>(() => `count: ${count.value}`)

// Writable computed
const fullName = computed<string>({
  get: () => 'John Doe',
  set: (val) => { /* ... */ },
})
```

### Типизация template refs и ссылок на компоненты

#### `useTemplateRef` (Vue 3.5+) — рекомендуемый способ

```vue
<script setup lang="ts">
import { useTemplateRef, onMounted } from 'vue'

// generic — тип DOM-элемента или экземпляра компонента
const inputRef = useTemplateRef<HTMLInputElement>('input')

onMounted(() => {
  inputRef.value?.focus()   // value может быть null до монтирования
})
</script>

<template>
  <input ref="input" />
</template>
```

#### Классический ref + `InstanceType`

```vue
<script setup lang="ts">
import { ref } from 'vue'
import MyModal from './MyModal.vue'

// InstanceType извлекает тип экземпляра из типа компонента
const modalRef = ref<InstanceType<typeof MyModal> | null>(null)

function open() {
  // доступны publicly exposed методы/свойства
  modalRef.value?.open()
}
</script>

<template>
  <MyModal ref="modalRef" />
</template>
```

Чтобы наружу были видны методы, в дочернем компоненте используют `defineExpose`:

```vue
<script setup lang="ts">
function open() { /* ... */ }
function close() { /* ... */ }

// expose делает методы доступными через ref родителя и определяет публичный тип
defineExpose({ open, close })
</script>
```

### Типизация `provide` / `inject` через `InjectionKey`

```ts
// keys.ts — выносим ключи отдельно
import type { InjectionKey, Ref } from 'vue'

export interface UserContext {
  user: Ref<string>
  logout: () => void
}

// InjectionKey<T> связывает строковый/Symbol-ключ с типом значения
export const userKey = Symbol() as InjectionKey<UserContext>
```

```vue
<!-- Provider -->
<script setup lang="ts">
import { provide, ref } from 'vue'
import { userKey } from './keys'

const user = ref('Andrei')
provide(userKey, { user, logout: () => { user.value = '' } })
// тип проверяется: forget logout => ошибка компиляции
</script>
```

```vue
<!-- Consumer -->
<script setup lang="ts">
import { inject } from 'vue'
import { userKey } from './keys'

// ctx: UserContext | undefined
const ctx = inject(userKey)

// с дефолтом => UserContext (без undefined)
const ctx2 = inject(userKey, { user: ref(''), logout: () => {} })
</script>
```

### Типизация стора Pinia

Setup-store даёт лучший вывод типов "из коробки":

```ts
// stores/counter.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCounterStore = defineStore('counter', () => {
  const count = ref(0)                              // state
  const double = computed(() => count.value * 2)    // getter
  function increment() { count.value++ }            // action

  return { count, double, increment }
})
```

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCounterStore } from '@/stores/counter'

const store = useCounterStore()
// storeToRefs сохраняет реактивность state/getters при деструктуризации
const { count, double } = storeToRefs(store)
// actions берём напрямую (они не реактивны и в обёртке не нуждаются)
const { increment } = store
</script>
```

Для Options-store типы тоже выводятся, но `this` иногда требует аккуратности.

### Generic-компоненты

Атрибут `generic` в `<script setup>` делает компонент дженериком:

```vue
<script setup lang="ts" generic="T">
// T выводится из переданных props на месте использования
defineProps<{
  items: T[]
  selected: T
}>()

const emit = defineEmits<{
  select: [item: T]
}>()
</script>

<template>
  <ul>
    <li v-for="(item, i) in items" :key="i" @click="emit('select', item)">
      <slot :item="item" />
    </li>
  </ul>
</template>
```

Использование — `T` выведется как `User`:

```vue
<GenericList :items="users" :selected="users[0]" @select="onSelect" />
```

Можно добавить ограничения: `generic="T extends { id: number }"`.

### Типизация событий DOM и `$attrs`

```vue
<script setup lang="ts">
// Типизация нативных событий — указываем тип события явно
function onInput(e: Event) {
  // target — EventTarget | null, нужно привести
  const value = (e.target as HTMLInputElement).value
}

function onSubmit(e: SubmitEvent) {
  e.preventDefault()
}
</script>

<template>
  <input @input="onInput" />
</template>
```

Типизация `$attrs` / fallthrough-атрибутов:

```vue
<script setup lang="ts">
import { useAttrs } from 'vue'

// attrs слабо типизирован (Record<string, unknown>) — это by design,
// т.к. набор атрибутов произволен. Для строгости явно объявляйте props/emits.
const attrs = useAttrs()
</script>
```

### Options API — кратко

В Options API используют `defineComponent` для корректного вывода `this`:

```ts
import { defineComponent } from 'vue'

export default defineComponent({
  props: {
    title: { type: String, required: true },
    count: { type: Number, default: 0 },
  },
  data() {
    return { local: 0 as number }
  },
  computed: {
    doubled(): number {
      return this.count * 2   // this типизирован
    },
  },
  methods: {
    inc() { this.local++ },
  },
})
```

Для сложных props-типов используют `PropType<T>`:

```ts
import { defineComponent, type PropType } from 'vue'

defineComponent({
  props: {
    user: { type: Object as PropType<{ id: number; name: string }>, required: true },
    list: { type: Array as PropType<string[]>, default: () => [] },
  },
})
```

## Полный рабочий пример

`UserCard.vue` — типизированный компонент с props, emits, expose и computed:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'

// --- Типы ---
export interface User {
  id: number
  firstName: string
  lastName: string
  role: 'admin' | 'user'
}

// --- Props ---
const props = withDefaults(defineProps<{
  user: User
  highlighted?: boolean
}>(), {
  highlighted: false,
})

// --- Emits ---
const emit = defineEmits<{
  edit: [id: number]
  delete: [user: User]
}>()

// --- Локальное состояние ---
const expanded = ref(false)

// --- Computed ---
const fullName = computed(() => `${props.user.firstName} ${props.user.lastName}`)
const isAdmin = computed(() => props.user.role === 'admin')

// --- Методы ---
function toggle() { expanded.value = !expanded.value }
function onEdit() { emit('edit', props.user.id) }

// --- Публичное API для родителя ---
defineExpose({ expanded, toggle })
</script>

<template>
  <div class="card" :class="{ highlighted }">
    <h3 @click="toggle">{{ fullName }} <span v-if="isAdmin">⭐</span></h3>
    <div v-if="expanded">
      <button @click="onEdit">Изменить</button>
      <button @click="emit('delete', user)">Удалить</button>
    </div>
  </div>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `defineProps<T>()` отличается от `defineProps({...})`?**
Type-based (`<T>`) объявляет props через TS-типы, компилятор сам генерирует
runtime-опции. Runtime-объект (`{...}`) задаёт валидацию вручную. Смешивать в одном
вызове нельзя. Type-based чище, но дефолты — через `withDefaults` (или деструктуризацию
в 3.5+).

**2. Зачем нужен `withDefaults`?**
При type-based объявлении нет места для `default`. `withDefaults` оборачивает
`defineProps<T>()` и задаёт дефолты, при этом убирает `undefined` из выводимых типов.
Для объектов/массивов дефолт задаётся фабрикой `() => [...]`.

**3. Как типизировать ref на дочерний компонент?**
`ref<InstanceType<typeof Child> | null>(null)` или `useTemplateRef<...>('name')`.
Наружу доступны только то, что компонент отдал через `defineExpose`.

**4. Что такое `InjectionKey` и зачем он?**
`InjectionKey<T>` — типизированный ключ (обычно `Symbol`), связывающий `provide`/
`inject` с типом значения. Даёт автодополнение и проверку, что передали правильную
структуру, плюс избегает коллизий строковых ключей.

**5. Почему Vite не ловит ошибки типов и как их проверять?**
Vite использует esbuild, который только удаляет типы без проверки (для скорости).
Проверку выносят в `vue-tsc --noEmit` (CI / pre-build) либо подключают
`vite-plugin-checker` для проверки в dev-режиме.

**6. Что такое generic-компонент и как его объявить?**
Компонент, типы которого параметризованы. Объявляется через
`<script setup lang="ts" generic="T">`. `T` выводится из props на месте
использования — даёт строгую связь между `items: T[]` и событием `select: [T]`.

**7. Как типизировать нативное DOM-событие?**
Параметр обработчика типизируют конкретным типом события (`Event`, `MouseEvent`,
`SubmitEvent`), а `e.target` приводят к нужному элементу: `(e.target as HTMLInputElement).value`.

**8. Почему `$attrs` / `useAttrs()` слабо типизированы?**
Набор fallthrough-атрибутов произволен и неизвестен на этапе компиляции, поэтому тип
— `Record<string, unknown>`. Для строгой типизации лучше явно объявлять props и emits.

**9. Чем отличается типизация setup-store и options-store в Pinia?**
Setup-store (функция, возвращающая ref/computed/функции) даёт лучший автоматический
вывод типов. Options-store тоже типизируется, но требует аккуратности с `this` в
геттерах. `storeToRefs` нужен, чтобы не потерять реактивность при деструктуризации.

**10. Нужен ли `defineComponent` в Composition API?**
В `<script setup>` — нет, типы выводятся сами. `defineComponent` нужен для Options API
(корректный `this`) и для компонентов без `<script setup>`.

**11. Как задать дефолты props без `withDefaults`?**
В Vue 3.5+ — через Reactive Props Destructure: `const { x = 1 } = defineProps<...>()`.
Компилятор сохраняет реактивность.

**12. Что разворачивает (unwrap) `ref` в типах?**
`ref` рекурсивно разворачивает вложенные ref'ы при доступе через `.value`. В шаблоне
ref на верхнем уровне разворачивается автоматически (`.value` не пишем).

## Подводные камни (gotchas)

- **Дефолт-объект/массив без фабрики.** В `withDefaults` для ссылочных типов нужно
  `() => [...]`, иначе ошибка (как и в runtime-props).
- **Смешивание type-based и runtime в одном `defineProps`.** Запрещено — выбрать одно.
- **`InstanceType` не видит приватные методы.** Без `defineExpose` ref на компонент даёт
  только публичный интерфейс; внутренние методы недоступны и не типизированы.
- **`reactive` теряет тип при деструктуризации.** `const { x } = reactive(...)` — `x`
  станет обычным значением без реактивности (тип сохранится, но реакция — нет).
- **`ref(null)` без generic.** Тип станет `Ref<null>`; для последующего присвоения
  объекта укажите `ref<T | null>(null)`.
- **Импорт типов.** Используйте `import type { ... }` для чистого type-only импорта
  (важно при `isolatedModules` / `verbatimModuleSyntax`).
- **Volar старый "Takeover Mode".** В новых версиях не нужен — используется Hybrid Mode;
  не путайте инструкции из старых статей.

## Лучшие практики

- Включайте `"strict": true` — без этого половина пользы от типов теряется.
- Предпочитайте type-based `defineProps`/`defineEmits` runtime-объектам.
- Выносите общие типы (`User`, `Props`) и `InjectionKey` в отдельные `.ts`-файлы.
- Используйте `import type` для типов.
- Проверяйте типы в CI через `vue-tsc --noEmit`.
- Для публичного API компонента всегда используйте `defineExpose` + типы.
- Setup-store в Pinia + `storeToRefs` — для лучшего вывода типов и реактивности.

## Шпаргалка

```ts
// Props
const props = defineProps<{ title: string; n?: number }>()
const props = withDefaults(defineProps<P>(), { n: 0, list: () => [] })
const { n = 0 } = defineProps<P>()            // 3.5+ реактивная деструктуризация

// Emits
const emit = defineEmits<{ change: [id: number]; close: [] }>()

// Ref / computed
const c = ref<string | null>(null)            // Ref<string | null>
const d = computed<number>(() => c.value!.length)

// Template ref на компонент
const m = ref<InstanceType<typeof Modal> | null>(null)
const el = useTemplateRef<HTMLInputElement>('input')   // 3.5+

// Provide / inject
const key = Symbol() as InjectionKey<Ctx>
provide(key, value); const ctx = inject(key)

// Generic-компонент
// <script setup lang="ts" generic="T extends { id: number }">

// Options API
defineComponent({ props: { u: Object as PropType<User> } })
```
