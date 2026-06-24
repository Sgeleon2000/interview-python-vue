# Vue 3 — Входные параметры (Props) — конспект и вопросы

## О чём раздел

**Props** (properties) — это входные параметры компонента: способ передать данные **от родителя к потомку**. Props — основной «вход» компонента, формирующий его публичный контракт. Раздел подробно разбирает объявление props через `defineProps` (рантайм-форма и на основе типов TS), правила именования, передачу разных типов значений, валидацию, особенности Boolean-приведения и — самое важное для собеседования — **однонаправленный поток данных** и почему **нельзя мутировать props**.

Это один из самых частых блоков вопросов на собеседовании по Vue, особенно тема one-way data flow.

## Ключевые концепции

- **Props объявляются** через макрос `defineProps` (в `<script setup>`) или опцию `props` (Options API).
- **Поток данных однонаправленный (one-way / top-down)**: родитель → потомок. Изменение prop у родителя обновляет потомка, но **не наоборот**.
- **Props доступны только для чтения** (read-only). Мутировать их внутри потомка нельзя.
- **Именование**: в JS — `camelCase`, в шаблоне-передаче — допустим `kebab-case`.
- **Статические vs динамические**: `title="..."` (строка-литерал) vs `:title="expr"` (`v-bind`, любое выражение).
- **Валидация**: `type`, `required`, `default`, `validator`.
- **Boolean** имеет особое приведение: наличие атрибута без значения = `true`.
- **`withDefaults`** — задание значений по умолчанию для props, объявленных через типы TS.

## Подробный разбор с примерами кода

### Объявление через `defineProps` — рантайм-форма

`defineProps` — это **макрос времени компиляции**, его не нужно импортировать в `<script setup>`. Простейшая форма — массив имён:

```vue
<script setup>
// Массив имён — без валидации
const props = defineProps(['title', 'likes'])

// Доступ к значениям
console.log(props.title)
</script>

<template>
  <!-- В шаблоне props доступны напрямую (без префикса props.) -->
  <h4>{{ title }} — {{ likes }} лайков</h4>
</template>
```

Более правильная форма — **объект с описанием типов** (рекомендуется):

```vue
<script setup>
const props = defineProps({
  title: String,
  likes: Number,
})
</script>
```

### Объявление на основе типов TS

В TypeScript можно объявлять props через дженерик-аргумент `defineProps`, что даёт полноценную типизацию:

```vue
<script setup lang="ts">
// Объявление через тип — компилятор выведет рантайм-валидацию по типам
const props = defineProps<{
  title?: string
  likes: number
}>()
</script>
```

Часто тип выносят в интерфейс:

```vue
<script setup lang="ts">
interface Props {
  title?: string
  likes: number
  tags?: string[]
}

const props = defineProps<Props>()
</script>
```

Важный нюанс: при объявлении через типы **нельзя задать значения по умолчанию прямо в типе** — для этого есть `withDefaults` (см. ниже).

### Именование props (camelCase в JS / kebab-case в шаблоне)

Объявляем в `camelCase`:

```vue
<script setup>
defineProps({
  greetingMessage: String,
})
</script>

<template>
  <span>{{ greetingMessage }}</span>
</template>
```

При передаче из родителя можно использовать оба варианта, но конвенция — **kebab-case в шаблоне** (ближе к HTML-атрибутам):

```vue
<template>
  <!-- Рекомендуется kebab-case при передаче -->
  <MyComponent greeting-message="привет" />
  <!-- Тоже работает, но менее принято -->
  <MyComponent greetingMessage="привет" />
</template>
```

Vue автоматически сопоставляет `greeting-message` ↔ `greetingMessage`.

### Статические и динамические props

```vue
<template>
  <!-- Статический: значение — строковый литерал -->
  <BlogPost title="Заголовок поста" />

  <!-- Динамический: v-bind (:) — значение это JS-выражение -->
  <BlogPost :title="post.title" />
  <BlogPost :title="post.title + ' (черновик)'" />
  <BlogPost :title="isDraft ? 'Черновик' : post.title" />
</template>
```

Главное правило: **без `:` значение всегда строка**, с `:` — это JS-выражение нужного типа.

### Передача разных типов

Числа, булевы, массивы, объекты **обязательно** передавать через `v-bind` (`:`), иначе они станут строками:

```vue
<template>
  <!-- Число: без : это была бы строка "42" -->
  <BlogPost :likes="42" />

  <!-- Булево: важная разница -->
  <BlogPost :is-published="false" />   <!-- boolean false -->
  <BlogPost is-published="false" />    <!-- строка "false" → truthy! ОШИБКА -->

  <!-- Массив -->
  <BlogPost :comment-ids="[234, 266, 273]" />

  <!-- Объект -->
  <BlogPost :author="{ name: 'Вероника', company: 'Vue' }" />
</template>
```

### Привязка объекта целиком через `v-bind`

Если нужно передать сразу все свойства объекта как отдельные props, используют `v-bind` без аргумента:

```vue
<script setup>
const post = {
  id: 1,
  title: 'Мой пост',
  likes: 10,
}
</script>

<template>
  <!-- Эквивалентно :id="post.id" :title="post.title" :likes="post.likes" -->
  <BlogPost v-bind="post" />

  <!-- Развёрнутая запись — то же самое -->
  <BlogPost :id="post.id" :title="post.title" :likes="post.likes" />
</template>
```

### Однонаправленный поток данных (one-way) — почему нельзя мутировать props

Это **топ-вопрос на собеседовании**. Разберём детально.

**Суть:** все props формируют **однонаправленную связь сверху вниз** (one-way-down binding). Когда у родителя меняется значение, оно «течёт» вниз к потомку, обновляя его. Но **обратного потока нет**: потомок не должен изменять props, иначе поток данных стал бы двунаправленным и неконтролируемым.

```
   ┌──────────┐  prop (данные текут вниз)   ┌──────────┐
   │ Родитель │ ─────────────────────────▶ │ Потомок  │
   │ (источник│                             │ (только  │
   │  истины) │ ◀───── НЕТ обратного ─────  │  чтение) │
   └──────────┘        потока через prop    └──────────┘
```

**Почему так сделано:**
1. **Предсказуемость и отладка.** Источник истины — всегда родитель. Если бы любой потомок мог менять prop, было бы невозможно понять, кто и когда изменил данные. Однонаправленный поток делает движение данных трассируемым.
2. **Перезапись изменений.** При каждом ререндере родителя prop **пересчитывается заново** и затирает любые локальные изменения, сделанные потомком. То есть мутация prop в потомке всё равно будет «съедена» при следующем обновлении родителя — это просто не работает надёжно.

**Vue предупреждает** в консоли при попытке мутации:

```vue
<script setup>
const props = defineProps(['count'])

function bad() {
  // ❌ Warning: "Set operation on key 'count' failed: target is readonly"
  props.count++
}
</script>
```

Объекты/массивы в props технически можно мутировать «вглубь» (т.к. передаётся ссылка), но это **антипаттерн**: вы тайно меняете состояние родителя в обход явных каналов, что ломает предсказуемость.

```vue
<script setup>
const props = defineProps(['user'])

function bad() {
  // ❌ Технически сработает (мутация по ссылке), но это плохо:
  // меняем состояние родителя скрытно
  props.user.name = 'Новое имя'
}
</script>
```

**Как «изменять» prop правильно — два сценария:**

**Сценарий 1. Prop задаёт начальное значение локального состояния.** Делаем локальную копию через `ref`:

```vue
<script setup>
import { ref } from 'vue'

const props = defineProps(['initialCounter'])

// Локальное состояние, инициализированное из prop.
// Дальнейшие изменения prop НЕ влияют на counter (только начальное значение).
const counter = ref(props.initialCounter)

function increment() {
  counter.value++ // ✅ меняем локальную копию, не prop
}
</script>
```

**Сценарий 2. Prop нужно трансформировать.** Используем `computed`:

```vue
<script setup>
import { computed } from 'vue'

const props = defineProps(['size'])

// Производное значение пересчитывается при изменении prop.
const normalizedSize = computed(() => props.size.trim().toLowerCase())
</script>
```

**Сценарий 3. Нужно реально «менять prop» так, чтобы изменение ушло наверх.** Это и есть случай `v-model` / явного события: потомок не мутирует prop, а **порождает событие**, родитель сам меняет источник истины:

```vue
<!-- Потомок -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])

function onInput(e) {
  // ✅ просим родителя обновить значение
  emit('update:modelValue', e.target.value)
}
</script>

<template>
  <input :value="modelValue" @input="onInput" />
</template>
```

```vue
<!-- Родитель -->
<template>
  <!-- v-model = :modelValue + @update:modelValue -->
  <CustomInput v-model="text" />
</template>
```

Это сохраняет однонаправленный поток: данные вниз через prop, запрос на изменение — вверх через событие.

### Props доступны только для чтения

В Vue 3 объект props **реактивен и read-only**. Прямое присваивание (`props.x = ...`) даёт предупреждение и игнорируется. Деструктуризация в `<script setup>` (Vue 3.5+) сохраняет реактивность, но также не позволяет переопределять значения как источник истины.

```vue
<script setup>
// Vue 3.5+: реактивная деструктуризация с дефолтом
const { title = 'Без названия', likes } = defineProps(['title', 'likes'])
// title и likes остаются реактивными при доступе в нужном контексте,
// но переприсваивать их не следует
</script>
```

> Примечание: в более старых версиях Vue 3 деструктуризация props теряла реактивность — нужно было обращаться через `props.title`. С Vue 3.5 это исправлено компилятором.

### Валидация props (type / required / default / validator)

Полная объектная форма даёт валидацию (предупреждения в dev-режиме):

```vue
<script setup>
defineProps({
  // Базовая проверка типа
  propA: Number,

  // Несколько допустимых типов
  propB: [String, Number],

  // Обязательный prop
  propC: {
    type: String,
    required: true,
  },

  // Обязательный, но допускает null
  propD: {
    type: [String, null],
    required: true,
  },

  // Число со значением по умолчанию
  propE: {
    type: Number,
    default: 100,
  },

  // Объект/массив — default ОБЯЗАТЕЛЬНО через фабричную функцию!
  propF: {
    type: Object,
    default() {
      return { message: 'привет' }
    },
  },
  propArr: {
    type: Array,
    default: () => [],
  },

  // Кастомный валидатор: вернуть true/false
  propG: {
    validator(value) {
      return ['success', 'warning', 'danger'].includes(value)
    },
  },

  // default может быть функцией-фабрикой,
  // получающей "сырые" props первым аргументом
  propH: {
    type: Object,
    default(rawProps) {
      return { theme: rawProps.size === 'big' ? 'dark' : 'light' }
    },
  },

  // Функция как значение по умолчанию: оборачивать в ещё одну функцию
  propFn: {
    type: Function,
    // НЕ default() { ... } — это была бы фабрика; нужно вернуть функцию
    default() {
      return () => console.log('значение по умолчанию')
    },
  },
})
</script>
```

Ключевые правила:
- `type` может быть `String`, `Number`, `Boolean`, `Array`, `Object`, `Date`, `Function`, `Symbol`, любым классом-конструктором, либо массивом из них.
- Для `Object`/`Array` `default` — **только функция-фабрика** (иначе все экземпляры разделят один объект).
- Валидация работает **только в dev-режиме** (предупреждения), в production отключена для производительности.
- `required: true` без `default` → пропуск prop даёт warning.

### Особенности Boolean-приведения

Boolean — единственный тип со специальным «casting»-поведением, имитирующим нативные булевы атрибуты HTML:

```vue
<script setup>
defineProps({
  disabled: Boolean,
})
</script>
```

```vue
<template>
  <!-- Эквивалентно :disabled="true" -->
  <MyComponent disabled />

  <!-- Эквивалентно :disabled="false" -->
  <MyComponent />

  <!-- Явные формы -->
  <MyComponent :disabled="true" />
  <MyComponent :disabled="false" />

  <!-- ВНИМАНИЕ: строка → truthy -->
  <MyComponent disabled="false" /> <!-- НЕ Boolean → стань осторожен -->
</template>
```

Правила приведения:
- **присутствие атрибута без значения** → `true` (как `disabled` у `<input>`);
- **пустая строка** `disabled=""` → `true`;
- отсутствие атрибута → `false` (если тип Boolean).

Если объявлено несколько типов, порядок влияет на приведение:

```js
// Boolean раньше String: пустая строка приводится к true как Boolean
{ type: [Boolean, String] }
// String раньше Boolean: disabled="" останется строкой ""
{ type: [String, Boolean] }
```

### `withDefaults` — дефолты для type-based объявления

При объявлении props через типы TS нельзя задать default в самом типе. Для этого есть `withDefaults`:

```vue
<script setup lang="ts">
interface Props {
  title?: string
  likes?: number
  tags?: string[]
}

// Оборачиваем defineProps в withDefaults
const props = withDefaults(defineProps<Props>(), {
  title: 'Без названия',
  likes: 0,
  // для объектов/массивов — фабрика
  tags: () => [],
})
</script>
```

В Vue 3.5+ дефолты можно задавать прямо в реактивной деструктуризации, что часто заменяет `withDefaults`:

```vue
<script setup lang="ts">
interface Props {
  title?: string
  likes?: number
}

// Дефолты через деструктуризацию (Vue 3.5+)
const { title = 'Без названия', likes = 0 } = defineProps<Props>()
</script>
```

## Полный рабочий пример

Компонент-бейдж `StatusBadge` с валидацией, Boolean-prop, дефолтами и корректной локальной трансформацией.

```vue
<!-- components/StatusBadge.vue -->
<script setup>
import { computed } from 'vue'

const props = defineProps({
  // Текст бейджа — обязателен
  label: {
    type: String,
    required: true,
  },
  // Статус с валидацией допустимых значений
  status: {
    type: String,
    default: 'info',
    validator(value) {
      return ['info', 'success', 'warning', 'danger'].includes(value)
    },
  },
  // Boolean-prop: <StatusBadge outlined /> → true
  outlined: {
    type: Boolean,
    default: false,
  },
  // Объект-prop с фабричным default
  meta: {
    type: Object,
    default: () => ({ icon: null }),
  },
})

// Трансформация prop через computed — НЕ мутируем props
const cssClass = computed(() => {
  return [
    'badge',
    `badge--${props.status}`,
    { 'badge--outlined': props.outlined },
  ]
})
</script>

<template>
  <span :class="cssClass">
    <span v-if="meta.icon" class="badge__icon">{{ meta.icon }}</span>
    {{ label }}
  </span>
</template>

<style scoped>
.badge { padding: 2px 8px; border-radius: 8px; }
.badge--success { background: #e6f4ea; }
.badge--danger { background: #fce8e6; }
.badge--outlined { background: transparent; border: 1px solid currentColor; }
</style>
```

```vue
<!-- App.vue -->
<script setup>
import StatusBadge from './components/StatusBadge.vue'
import { ref } from 'vue'

const orderStatus = ref('success')
</script>

<template>
  <!-- статический + динамический + Boolean + объект -->
  <StatusBadge label="Новый" />
  <StatusBadge label="Оплачено" :status="orderStatus" outlined />
  <StatusBadge
    label="Срочно"
    status="danger"
    :meta="{ icon: '🔥' }"
  />
</template>
```

### Тот же компонент через типы TS + withDefaults

```vue
<script setup lang="ts">
import { computed } from 'vue'

type Status = 'info' | 'success' | 'warning' | 'danger'

interface Props {
  label: string
  status?: Status
  outlined?: boolean
  meta?: { icon: string | null }
}

const props = withDefaults(defineProps<Props>(), {
  status: 'info',
  outlined: false,
  meta: () => ({ icon: null }),
})

const cssClass = computed(() => [
  'badge',
  `badge--${props.status}`,
  { 'badge--outlined': props.outlined },
])
</script>
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое props и зачем они нужны?**
Входные параметры компонента — способ передать данные от родителя к потомку. Формируют публичный контракт компонента, делают его переиспользуемым и конфигурируемым.

**2. Почему нельзя мутировать props внутри потомка? (топ-вопрос)**
Из-за **однонаправленного потока данных**: источник истины — родитель. Причины: (1) предсказуемость/отладка — иначе непонятно, кто менял данные; (2) при ререндере родителя prop пересчитывается и затирает локальные изменения. Vue выдаёт warning, props — read-only. Менять нужно через локальную копию, `computed` или событие наверх (`v-model`).

**3. Как правильно «изменить» prop?**
Три способа: (a) локальная копия `ref(props.x)` — если prop задаёт лишь начальное значение; (b) `computed(() => transform(props.x))` — если нужно производное значение; (c) `emit('update:x', newVal)` / `v-model` — если изменение должно уйти к родителю как источнику истины.

**4. Что такое однонаправленный поток данных?**
Данные текут только сверху вниз: родитель → потомок через props. Обратно данные идут только через события. Это делает движение данных трассируемым и предсказуемым.

**5. Чем отличается `title="x"` от `:title="x"`?**
Без `:` значение — строковый литерал (всегда строка). С `:` (`v-bind`) — это JS-выражение, поэтому числа, булевы, массивы, объекты передаются как есть. `:likes="42"` — число, `likes="42"` — строка «42».

**6. Как передать число/булево/массив/объект в prop?**
Только через `v-bind`: `:likes="42"`, `:active="true"`, `:items="[1,2]"`, `:user="{...}"`. Без `:` всё станет строкой, что критично для Boolean (`disabled="false"` — truthy-строка).

**7. Какие правила именования у props?**
Объявляем в `camelCase` (`greetingMessage`). При передаче рекомендуется `kebab-case` (`greeting-message`). Vue сопоставляет их автоматически.

**8. Как задать значение по умолчанию для объекта/массива?**
Через **фабричную функцию**: `default: () => ({})` или `default: () => []`. Если задать объект напрямую, все экземпляры компонента разделят одну ссылку (баг с общим состоянием).

**9. Расскажи про валидацию props.**
В объектной форме: `type` (конструктор или массив), `required: true`, `default` (значение или фабрика), `validator(value)` (возвращает boolean). Валидация работает только в dev-режиме как предупреждения; в production отключена.

**10. В чём особенность Boolean-props?**
Имитируют нативные булевы атрибуты: наличие атрибута без значения (`disabled`) или `disabled=""` → `true`; отсутствие → `false`. При нескольких типах порядок (`[Boolean, String]`) влияет на приведение пустой строки.

**11. Что такое `withDefaults` и когда нужен?**
Функция для задания дефолтов при объявлении props через **типы TS** (`defineProps<Props>()`), т.к. в типе default указать нельзя. В Vue 3.5+ дефолты можно задавать прямо в реактивной деструктуризации, заменяя `withDefaults`.

**12. Можно ли мутировать объект, переданный в prop?**
Технически да (передаётся ссылка, глубокая мутация сработает), но это **антипаттерн**: вы скрытно меняете состояние родителя в обход явных каналов, ломая предсказуемость. Если нужно изменить — порождайте событие, чтобы родитель сам обновил данные.

**13. Реактивны ли props? Что происходит при их деструктуризации?**
Объект props реактивен и read-only. До Vue 3.5 деструктуризация теряла реактивность (нужно было `props.x`). С Vue 3.5 компилятор сохраняет реактивность деструктурированных props и позволяет задавать дефолты прямо там.

**14. Чем рантайм-объявление отличается от объявления через типы (TS)?**
Рантайм (`defineProps({ ... })`) задаёт реальную валидацию в рантайме. Type-based (`defineProps<Props>()`) даёт строгую типизацию на этапе компиляции; Vue выводит из типов минимальную рантайм-проверку. Type-based не позволяет default в типе — используется `withDefaults`. Смешивать обе формы в одном `defineProps` нельзя.

## Подводные камни (gotchas)

- **Boolean-строка**: `disabled="false"` — это truthy-строка, а не `false`. Передавайте `:disabled="false"`.
- **Default для объекта/массива без фабрики**: `default: {}` — все экземпляры делят один объект. Используйте `default: () => ({})`.
- **Мутация props**: прямое присваивание → warning и игнор; глубокая мутация объекта-prop «работает», но скрытно меняет родителя — антипаттерн.
- **Передача числа без `:`**: `likes="42"` даёт строку «42», ломая арифметику.
- **`default` для prop-функции**: `default()` — это фабрика, она должна *вернуть* функцию: `default() { return () => {} }`. Иначе сама фабрика станет значением.
- **Деструктуризация в старом Vue 3 (<3.5)** теряла реактивность — частая ловушка в легаси.
- **Валидации нет в production** — не полагайтесь на `validator` как на защиту в рантайме у пользователя.
- **Смешение рантайм- и type-формы** `defineProps` запрещено: либо объект, либо дженерик.
- **camelCase в шаблоне**: `greetingMessage="..."` работает, но `greeting-message` — конвенция; будьте последовательны.
- **`required` + `default`** одновременно бессмысленны: если prop обязателен, default не нужен.

## Лучшие практики

- Всегда объявляйте props с **типом и валидацией** (объектная форма) или через **типы TS** — это документирует контракт.
- Делайте props **read-only по дисциплине**: никогда не мутируйте их; используйте `ref`-копию, `computed` или `emit`.
- Для объектов/массивов задавайте `default` **только фабрикой**.
- Передавайте не-строковые значения через **`v-bind`** (`:`).
- Именуйте props в `camelCase`, передавайте в `kebab-case`.
- Для двусторонней связи используйте **`v-model`** (`modelValue` + `update:modelValue`), а не мутацию prop.
- В TS предпочитайте **type-based объявление** с `withDefaults` (или дефолты в деструктуризации в 3.5+).
- Не передавайте слишком много props — это сигнал, что компонент стоит разбить или использовать слоты/объект-конфиг.

## Шпаргалка

```vue
<!-- Объявление: рантайм -->
<script setup>
const props = defineProps({
  title:   { type: String, required: true },
  likes:   { type: Number, default: 0 },
  active:  { type: Boolean, default: false },
  items:   { type: Array,  default: () => [] },      // фабрика!
  meta:    { type: Object, default: () => ({}) },    // фабрика!
  status:  { type: String, validator: v => ['a','b'].includes(v) },
})
</script>
```

```vue
<!-- Объявление: типы TS + withDefaults -->
<script setup lang="ts">
interface Props { title: string; likes?: number }
const props = withDefaults(defineProps<Props>(), { likes: 0 })
</script>
```

```vue
<!-- Передача -->
<Comp title="статика" />          <!-- строка-литерал -->
<Comp :title="expr" />            <!-- динамика (v-bind) -->
<Comp :likes="42" />              <!-- число -->
<Comp :active="true" />           <!-- булево -->
<Comp disabled />                 <!-- Boolean → true -->
<Comp :items="[1,2,3]" />         <!-- массив -->
<Comp :user="{ id: 1 }" />        <!-- объект -->
<Comp v-bind="postObject" />      <!-- весь объект как набор props -->
```

| Тема | Правило |
|------|---------|
| Поток данных | сверху вниз (родитель → потомок), read-only |
| Изменить prop | НЕЛЬЗЯ; копия/`computed`/`emit` (`v-model`) |
| Не-строки | только через `v-bind` (`:`) |
| Boolean | наличие атрибута = `true`; `="false"` = строка |
| Object/Array default | только фабрика `() => ...` |
| Именование | объявление camelCase, передача kebab-case |
| Валидация | dev-режим: `type`/`required`/`default`/`validator` |
| TS дефолты | `withDefaults` или деструктуризация (3.5+) |
```

| Способ «менять» prop | Когда |
|----------------------|-------|
| `ref(props.x)` | prop — лишь начальное значение |
| `computed(() => f(props.x))` | нужно производное значение |
| `emit('update:x', v)` / `v-model` | изменение уходит к источнику истины (родителю) |
