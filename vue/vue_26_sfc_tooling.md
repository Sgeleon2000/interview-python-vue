# Vue 3 — Однофайловые компоненты (SFC) и инструментарий — конспект и вопросы

## О чём раздел

Этот раздел посвящён **однофайловым компонентам** (Single-File Components, SFC) — файлам с расширением `.vue`, которые объединяют шаблон, логику и стили одного компонента в одном файле. Также разбираем современный синтаксис `<script setup>`, изоляцию стилей (`<style scoped>`), CSS-модули, реактивный `v-bind()` в CSS и весь инструментарий вокруг Vue 3: Vite, `vue-tsc`, Volar (Vue — Official), ESLint, npm-скрипты, HMR.

SFC — это фундамент любого реального Vue-проекта. На собеседовании middle/senior уровня обязательно спросят про `<script setup>`, отличие `scoped` от `:deep()` и почему Vue CLI больше не рекомендуется.

## Ключевые концепции

- **SFC (`.vue`)** — файл из трёх блоков: `<template>`, `<script>`, `<style>`. Компилируется в обычный JS-модуль сборщиком (Vite/Webpack) через `@vue/compiler-sfc`.
- **`<script setup>`** — синтаксический сахар над `setup()`: весь код выполняется как тело функции `setup`, переменные верхнего уровня автоматически доступны в шаблоне. Меньше boilerplate, лучше вывод типов TypeScript, лучше производительность компиляции.
- **Макросы компилятора** — `defineProps`, `defineEmits`, `defineExpose`, `defineOptions`, `defineModel`, `withDefaults`. Это не функции рантайма, а инструкции компилятору; их не нужно импортировать.
- **`<style scoped>`** — изоляция стилей по компоненту через добавление уникального атрибута `data-v-xxxxxx`.
- **`:deep()`, `:slotted()`, `:global()`** — псевдоклассы для управления областью видимости scoped-стилей.
- **CSS-модули (`<style module>`)** — стили как объект JS с локально хешированными именами классов.
- **`v-bind()` в CSS** — связывание CSS-значения с реактивным состоянием компонента.
- **Vite** — рекомендованный сборщик/dev-сервер (быстрый старт, нативный ESM, мгновенный HMR). **Vue CLI** (на Webpack) — в режиме поддержки (maintenance mode), для новых проектов не рекомендуется.
- **Volar / Vue — Official** — расширение VS Code для поддержки `.vue`. **`vue-tsc`** — типовая проверка `.vue` из командной строки.

## Подробный разбор с примерами кода

### 1. Структура SFC

```vue
<!-- UserCard.vue -->
<template>
  <!-- Блок шаблона: ровно один на файл -->
  <div class="card">
    <h2>{{ name }}</h2>
    <button @click="like">Лайков: {{ likes }}</button>
  </div>
</template>

<script setup>
// Блок логики (Composition API со script setup)
import { ref } from 'vue'

const name = ref('Андрей')
const likes = ref(0)
function like() {
  likes.value++
}
</script>

<style scoped>
/* Блок стилей, изолированный для этого компонента */
.card {
  padding: 16px;
  border: 1px solid #ddd;
}
</style>
```

Правила:
- `<template>` — **ровно один** корневой блок на файл (внутри Vue 3 разрешён фрагмент — несколько корневых узлов).
- `<script>` и `<script setup>` — можно использовать вместе (обычный `<script>` нужен для кода, который должен выполниться один раз на уровне модуля, или для `export` из модуля).
- `<style>` — блоков может быть несколько (например, один глобальный и один scoped).
- Поддерживаются пре-процессоры через атрибут `lang`: `<script lang="ts">`, `<style lang="scss">`, `<template lang="pug">`.

### 2. `<script setup>` — что делает и преимущества

`<script setup>` — это компилируемый синтаксис. Код внутри становится телом `setup()`:

```vue
<script setup>
import { ref, computed, onMounted } from 'vue'

// Любая переменная/функция верхнего уровня доступна в шаблоне без return
const count = ref(0)
const double = computed(() => count.value * 2)

onMounted(() => {
  console.log('Компонент смонтирован')
})

function inc() {
  count.value++
}
</script>
```

**Преимущества по сравнению с обычным `setup()`:**
1. Меньше шаблонного кода — не нужен `return { ... }`.
2. Лучший вывод типов TypeScript (props/emits типизируются напрямую).
3. Лучшая производительность — шаблон компилируется в render-функцию в том же скоупе, без промежуточного прокси.
4. Импортированные компоненты доступны в шаблоне автоматически (без секции `components`).

```vue
<script setup>
// Просто импортируем — регистрировать не нужно
import BaseButton from './BaseButton.vue'
</script>

<template>
  <BaseButton>Готово</BaseButton>
</template>
```

### 3. defineProps / defineEmits

```vue
<script setup>
// Вариант 1: рантайм-объявление
const props = defineProps({
  title: { type: String, required: true },
  count: { type: Number, default: 0 }
})

// События, которые компонент может испускать
const emit = defineEmits(['update', 'close'])

function onClick() {
  emit('update', props.count + 1)
}
</script>
```

С TypeScript — типовое объявление + `withDefaults`:

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
}

// Типовая форма defineProps
const props = withDefaults(defineProps<Props>(), {
  count: 0
})

// Типизированные события
const emit = defineEmits<{
  (e: 'update', value: number): void
  (e: 'close'): void
}>()

// Vue 3.3+: более короткий синтаксис emits
// const emit = defineEmits<{ update: [value: number]; close: [] }>()
</script>
```

### 4. defineModel (Vue 3.4+) — двусторонняя привязка

```vue
<!-- Дочерний компонент: CustomInput.vue -->
<script setup>
// Создаёт ref, синхронизированный с v-model родителя
const model = defineModel()
// С опциями: const model = defineModel({ type: String, default: '' })
</script>

<template>
  <input v-model="model" />
</template>
```

```vue
<!-- Родитель -->
<CustomInput v-model="text" />
```

### 5. defineExpose — публичный API компонента

По умолчанию компонент со `<script setup>` **закрыт**: родитель не видит его внутренние переменные через `ref`/template ref. `defineExpose` явно открывает нужное:

```vue
<!-- Child.vue -->
<script setup>
import { ref } from 'vue'

const count = ref(0)
function reset() {
  count.value = 0
}

// Открываем наружу только reset (count останется приватным)
defineExpose({ reset })
</script>
```

```vue
<!-- Parent.vue -->
<script setup>
import { ref, onMounted } from 'vue'
import Child from './Child.vue'

const childRef = ref(null)

onMounted(() => {
  childRef.value.reset() // доступен, т.к. экспортирован
})
</script>

<template>
  <Child ref="childRef" />
</template>
```

### 6. Top-level await

В `<script setup>` можно использовать `await` на верхнем уровне — компонент станет асинхронным и должен оборачиваться в `<Suspense>`:

```vue
<script setup>
// Верхнеуровневый await — компонент превращается в async-компонент
const res = await fetch('/api/user')
const user = await res.json()
</script>

<template>
  <p>{{ user.name }}</p>
</template>
```

```vue
<!-- Использование async-компонента требует Suspense -->
<template>
  <Suspense>
    <UserProfile />
    <template #fallback>Загрузка...</template>
  </Suspense>
</template>
```

> Важно: после `await` теряется текущий instance, поэтому регистрировать `onMounted` и подобные хуки нужно **до** первого `await`.

### 7. `<style scoped>` — как работает изоляция

При `scoped` компилятор:
1. Добавляет каждому элементу шаблона уникальный атрибут, например `data-v-7ba5bd90`.
2. Переписывает селекторы стиля, добавляя этот атрибут: `.card` → `.card[data-v-7ba5bd90]`.

Так стили не «протекают» в дочерние и сторонние компоненты.

```vue
<style scoped>
/* Скомпилируется в .title[data-v-xxxxxx] */
.title { color: teal; }
</style>
```

**Важная деталь:** scoped-стиль родителя НЕ влияет на внутренности дочернего компонента, но **влияет на корневой элемент** дочернего (у корневого узла дочернего есть и его, и родительский атрибут — для удобства верстки лейаута).

### 8. `:deep()` — проникновение в дочерние компоненты

Чтобы scoped-стиль воздействовал на внутренние элементы дочернего компонента (или содержимое сторонней библиотеки), используют `:deep()`:

```vue
<style scoped>
/* Без :deep() селектор бы получил атрибут текущего компонента
   и не нашёл бы .child-inner внутри дочернего компонента */
.wrapper :deep(.child-inner) {
  color: red;
}
</style>
```

### 9. `:slotted()` — стилизация контента слота

Контент, переданный через слот, принадлежит **родителю**, поэтому по умолчанию scoped-стили дочернего на него не действуют. `:slotted()` это исправляет:

```vue
<!-- В компоненте, который РЕНДЕРИТ слот -->
<style scoped>
:slotted(.item) {
  margin: 4px 0;
}
</style>
```

### 10. `:global()` — глобальный селектор внутри scoped

```vue
<style scoped>
/* Применится глобально, без атрибута data-v */
:global(.dark-theme body) {
  background: #111;
}
</style>
```

### 11. CSS-модули

`<style module>` компилирует CSS в объект, доступный в шаблоне как `$style`:

```vue
<template>
  <p :class="$style.red">Текст</p>
</template>

<style module>
.red { color: red; }
</style>
```

С именованным модулем и доступом из `<script setup>`:

```vue
<script setup>
import { useCssModule } from 'vue'
// Доступ к объекту классов в JS
const classes = useCssModule() // или useCssModule('myStyles') для <style module="myStyles">
</script>

<template>
  <p :class="classes.red">Текст</p>
</template>

<style module>
.red { color: red; }
</style>
```

### 12. `v-bind()` в CSS — реактивные стили

```vue
<script setup>
import { ref } from 'vue'
const themeColor = ref('#42b883')
</script>

<template>
  <button class="btn" @click="themeColor = '#ff0000'">Сменить цвет</button>
</template>

<style scoped>
.btn {
  /* Связывает CSS-свойство с реактивным состоянием.
     Vue подставит inline CSS-переменную и будет её обновлять. */
  background: v-bind(themeColor);
}
</style>
```

Можно обращаться к выражениям/полям объекта (в кавычках):

```vue
<style scoped>
.box { font-size: v-bind('theme.fontSize'); }
</style>
```

### 13. Инструментарий

- **Vite** — основной сборщик. Создание проекта:
  ```bash
  npm create vue@latest
  # интерактивно: TypeScript, Router, Pinia, ESLint и т.д.
  ```
- **Vue CLI** — устаревший (maintenance mode, на Webpack). Для новых проектов используйте Vite/`create-vue`.
- **vue-tsc** — типовая проверка `.vue`-файлов в CLI (обёртка над `tsc`):
  ```bash
  vue-tsc --noEmit
  ```
- **Volar / «Vue — Official»** — официальное расширение VS Code (заменило Vetur). Для TS поддержки `.vue` нужен Take Over Mode или плагин `@vue/typescript-plugin`.
- **ESLint** — `eslint-plugin-vue` для линтинга `.vue`.
- **HMR (Hot Module Replacement)** — Vite заменяет изменённый модуль без перезагрузки страницы и **сохраняет состояние** компонента. Работает «из коробки» для SFC.
- **Prerender / SSG** — для статической генерации применяют надстройки (Nuxt, vite-plugin-ssr/vite-plugin-pages, vite-ssg). Чистый Vite даёт SPA; SSR/SSG настраивается отдельно.

### 14. Типичные npm-скрипты

```json
{
  "scripts": {
    "dev": "vite",                       // dev-сервер + HMR
    "build": "vue-tsc --noEmit && vite build",  // проверка типов + сборка
    "preview": "vite preview",           // локальный предпросмотр собранного билда
    "lint": "eslint . --fix",
    "type-check": "vue-tsc --noEmit"
  }
}
```

## Полный рабочий пример

```vue
<!-- ThemedCounter.vue -->
<script setup lang="ts">
import { ref, computed } from 'vue'

interface Props {
  /** Стартовое значение счётчика */
  start?: number
  /** Заголовок карточки */
  title: string
}

const props = withDefaults(defineProps<Props>(), {
  start: 0
})

// Типизированные события наружу
const emit = defineEmits<{
  change: [value: number]
}>()

const count = ref(props.start)
const color = ref('#42b883')

const isEven = computed(() => count.value % 2 === 0)

function inc() {
  count.value++
  emit('change', count.value) // уведомляем родителя
}

function reset() {
  count.value = props.start
}

// Открываем reset для родителя через template ref
defineExpose({ reset })
</script>

<template>
  <section class="counter">
    <h3 class="title">{{ title }}</h3>
    <p :class="$style.value">{{ count }} ({{ isEven ? 'чётное' : 'нечётное' }})</p>
    <button class="btn" @click="inc">+1</button>
    <input v-model="color" type="color" />
  </section>
</template>

<!-- Изолированные стили + реактивный v-bind -->
<style scoped>
.counter {
  padding: 16px;
  border: 1px solid #eee;
  border-radius: 8px;
}
.title {
  color: v-bind(color); /* реактивная привязка цвета */
}
.btn {
  background: v-bind(color);
  color: #fff;
  border: none;
  padding: 8px 12px;
  border-radius: 6px;
}
</style>

<!-- CSS-модуль для одного класса -->
<style module>
.value {
  font-weight: bold;
}
</style>
```

```vue
<!-- App.vue — родитель -->
<script setup>
import { ref, onMounted } from 'vue'
import ThemedCounter from './ThemedCounter.vue'

const counterRef = ref(null)

function onChange(value) {
  console.log('Новое значение:', value)
}

onMounted(() => {
  // Вызов экспортированного метода дочернего компонента
  // counterRef.value.reset()
})
</script>

<template>
  <ThemedCounter
    ref="counterRef"
    title="Демо-счётчик"
    :start="5"
    @change="onChange"
  />
</template>
```

## Частые вопросы на собеседовании (Q/A)

**Q: Что такое SFC и из чего он состоит?**
A: Single-File Component — файл `.vue`, объединяющий `<template>`, `<script>` и `<style>` одного компонента. Компилируется сборщиком в обычный JS-модуль через `@vue/compiler-sfc`.

**Q: Чем `<script setup>` лучше обычного `setup()`?**
A: Меньше boilerplate (нет `return`), переменные верхнего уровня и импортированные компоненты автоматически доступны в шаблоне, лучше вывод типов TS, выше производительность компиляции (рендер-функция в том же скоупе без прокси).

**Q: Почему `defineProps`/`defineEmits` не нужно импортировать?**
A: Это макросы компилятора, а не рантайм-функции. Они обрабатываются `@vue/compiler-sfc` на этапе сборки и доступны глобально внутри `<script setup>`.

**Q: Зачем нужен `defineExpose`?**
A: Компонент со `<script setup>` по умолчанию закрыт — родитель не видит его внутренности через template ref. `defineExpose` явно публикует методы/свойства наружу.

**Q: Как работает `<style scoped>`?**
A: Компилятор добавляет каждому элементу уникальный атрибут `data-v-xxxxxx` и дописывает его в селекторы (`.x` → `.x[data-v-xxxxxx]`), изолируя стили в пределах компонента.

**Q: Влияют ли scoped-стили родителя на дочерний компонент?**
A: На внутренности — нет. Но на корневой элемент дочернего влияют, так как ему присваиваются оба атрибута. Для проникновения внутрь используют `:deep()`.

**Q: Когда нужен `:deep()`?**
A: Когда scoped-стиль должен воздействовать на внутренние элементы дочернего компонента или содержимое UI-библиотеки.

**Q: Чем `:slotted()` отличается от обычного scoped-селектора?**
A: Контент слота принадлежит родителю, поэтому scoped-стили дочернего его не затрагивают. `:slotted()` позволяет дочернему компоненту стилизовать переданный в него slot-контент.

**Q: Что делает `v-bind()` в CSS?**
A: Связывает CSS-значение с реактивным состоянием компонента — Vue создаёт inline CSS-переменную и обновляет её при изменении состояния.

**Q: Чем CSS-модули отличаются от scoped?**
A: Scoped изолирует через атрибут data-v и сохраняет имена классов. Модули хешируют сами имена классов и отдают их объектом (`$style`/`useCssModule`), полностью исключая коллизии имён.

**Q: Можно ли совмещать `<script>` и `<script setup>`?**
A: Да. Обычный `<script>` нужен для кода уровня модуля (выполняется один раз) или именованных экспортов; `<script setup>` — для логики компонента.

**Q: Что такое top-level await в `<script setup>` и какие ограничения?**
A: Возможность писать `await` на верхнем уровне — компонент становится async и должен использоваться внутри `<Suspense>`. Хуки жизненного цикла нужно регистрировать до первого `await`.

**Q: Почему Vue CLI больше не рекомендуется?**
A: Он построен на Webpack и переведён в режим поддержки. Для новых проектов рекомендован Vite (`create-vue`) — быстрый старт, нативный ESM, мгновенный HMR.

**Q: Что такое `vue-tsc` и Volar?**
A: `vue-tsc` — типовая проверка `.vue` в CLI. Volar (расширение «Vue — Official» в VS Code) обеспечивает подсветку, автодополнение и TS-интеграцию для `.vue` (заменил Vetur).

## Подводные камни (gotchas)

- **`defineExpose` обязателен** для доступа к методам через template ref — иначе родитель получит «пустой» прокси.
- **Регистрация хуков после `await`** в top-level await теряет instance — хуки не сработают.
- **`:deep()` синтаксис**: старый `::v-deep`/`>>>`/`/deep/` устарел; используйте `:deep()`.
- **scoped не защищает от глобальных стилей** — глобальный CSS (`<style>` без scoped, внешние библиотеки) всё равно может перекрыть селекторы.
- **`v-bind()` в CSS** работает только в SFC и требует, чтобы имя ссылалось на доступное в скоупе реактивное значение.
- **Один `<template>` на файл** — несколько шаблонных блоков недопустимы (но несколько корневых узлов внутри одного `<template>` — можно).
- **CSS-модули и динамические классы**: имена классов хешируются, поэтому нельзя ссылаться на них как на статические строки извне.
- **Vue CLI vs Vite**: миграция со старого Webpack-проекта может потребовать переписать конфиг и заменить специфичные Webpack-лоадеры.

## Лучшие практики

- Используйте `<script setup>` + Composition API для нового кода.
- Типизируйте props/emits через generic-форму (`defineProps<T>()`, `defineEmits<T>()`) при использовании TS.
- По умолчанию держите стили `scoped`; для глобальных — отдельный блок `<style>` без scoped или `:global()`.
- Применяйте `:deep()` точечно, не «обмазывайте» им всё подряд — это нарушает инкапсуляцию.
- Минимизируйте `defineExpose` — открывайте только реально нужный API.
- Для двусторонней привязки в новых версиях используйте `defineModel` вместо ручных `modelValue` + `update:modelValue`.
- В `build`-скрипт добавляйте `vue-tsc --noEmit` для проверки типов перед сборкой.
- Используйте `eslint-plugin-vue` и официальное расширение Volar.

## Шпаргалка

```vue
<script setup lang="ts">
// Макросы (импорт не нужен):
const props = withDefaults(defineProps<{ a: string; b?: number }>(), { b: 0 })
const emit  = defineEmits<{ change: [n: number] }>()
const model = defineModel<string>()      // v-model (3.4+)
defineExpose({ method })                  // публичный API
defineOptions({ name: 'MyComp' })         // имя/опции компонента

// Top-level await -> async-компонент (нужен <Suspense>)
// const data = await fetchData()
</script>

<style scoped>
.a { color: red; }                /* -> .a[data-v-xxx] */
.wrap :deep(.inner) { }           /* проникнуть в дочерний */
:slotted(.x) { }                  /* стилизовать слот-контент */
:global(.y) { }                   /* глобальный селектор */
.b { color: v-bind(color); }      /* реактивный CSS */
</style>

<style module>
.red { color: red; }              /* доступ: $style.red / useCssModule() */
</style>
```

| Псевдокласс  | Назначение                                   |
|--------------|----------------------------------------------|
| `:deep()`    | Проникнуть в дочерний компонент              |
| `:slotted()` | Стилизовать контент слота                    |
| `:global()`  | Глобальный селектор внутри scoped            |

| Инструмент | Роль                                  |
|------------|---------------------------------------|
| Vite       | Сборка/dev-сервер + HMR (рекомендуется)|
| Vue CLI    | Устарел (Webpack, maintenance mode)   |
| vue-tsc    | Проверка типов `.vue` в CLI           |
| Volar      | Официальное расширение VS Code        |
| ESLint     | Линтинг (`eslint-plugin-vue`)         |
