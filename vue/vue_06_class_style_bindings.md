# Vue 3 — Привязка классов и стилей — конспект и вопросы

## О чём раздел

Привязка классов (`:class`) и инлайн-стилей (`:style`) — это специальные расширения директивы
`v-bind` для атрибутов `class` и `style`. Vue понимает, что эти атрибуты особенные, и позволяет
передавать в них не только строки, но и **объекты** и **массивы**, аккуратно объединяя
результат со статическими значениями. Тема почти всегда встречается в практических заданиях
(«подсветить активный пункт», «показать ошибку»), поэтому важно знать все формы синтаксиса.

## Ключевые концепции

- `:class` принимает **строку**, **объект** (`{ имя-класса: булево }`) или **массив**
  (смесь строк, объектов, выражений).
- `:style` принимает **объект** (`{ свойство: значение }`) или **массив** объектов.
- Статический `class`/`style` и динамический `:class`/`:style` **объединяются**, а не
  затирают друг друга.
- Свойства в `:style` можно писать в `camelCase` (`fontSize`) или `kebab-case` в кавычках
  (`'font-size'`).
- Vue автоматически добавляет **вендорные префиксы** для свойств, требующих их.
- Для свойства можно передать **массив значений** — браузер применит последнее поддерживаемое.
- При использовании `:class`/`:style` **на компоненте** классы/стили попадают на корневой
  элемент (наследование атрибутов).

## Подробный разбор с примерами кода

### `:class` с объектом

Ключ — имя класса, значение — булево выражение (включать класс или нет).

```vue
<script setup>
import { ref } from 'vue'

const isActive = ref(true)
const hasError = ref(false)
</script>

<template>
  <!-- класс 'active' будет, 'text-danger' — нет -->
  <div :class="{ active: isActive, 'text-danger': hasError }">Объект</div>

  <!-- статический class объединяется с динамическим -->
  <div class="card" :class="{ active: isActive }">Объединение</div>
  <!-- результат: class="card active" -->
</template>
```

Объект удобно вынести в `computed` или хранить в реактивном состоянии:

```vue
<script setup>
import { reactive, computed } from 'vue'

const state = reactive({ active: true, error: false })

// объект-класс целиком в computed
const classObject = computed(() => ({
  active: state.active && !state.error,
  'text-danger': state.error,
}))
</script>

<template>
  <div :class="classObject">computed-классы</div>
</template>
```

### `:class` с массивом

Массив позволяет комбинировать строки, тернарники и объекты.

```vue
<script setup>
import { ref } from 'vue'

const activeClass = ref('active')
const errorClass = ref('text-danger')
const isActive = ref(true)
</script>

<template>
  <!-- применятся оба класса -->
  <div :class="[activeClass, errorClass]">Массив строк</div>

  <!-- тернарник внутри массива -->
  <div :class="[isActive ? activeClass : '', errorClass]">Тернарник</div>

  <!-- объект внутри массива (можно смешивать) -->
  <div :class="[{ [activeClass]: isActive }, errorClass]">Смешанный</div>
</template>
```

### Динамические имена классов

```vue
<script setup>
import { ref } from 'vue'

const theme = ref('dark')   // 'dark' | 'light'
const size = ref('lg')
</script>

<template>
  <!-- вычисляемые имена классов через шаблонную строку -->
  <button :class="`btn btn--${theme} btn--${size}`">Кнопка</button>

  <!-- или через объект с динамическим ключом -->
  <button :class="{ [`theme-${theme}`]: true }">Тема</button>
</template>
```

### Классы на компонентах

Если у компонента **один** корневой элемент, переданный `class` добавляется к его классам.

```vue
<!-- MyButton.vue -->
<template>
  <button class="btn">Нажми</button>
</template>
```

```vue
<!-- использование -->
<MyButton class="btn--primary" :class="{ active: isActive }" />
<!-- результат на <button>: class="btn btn--primary active" -->
```

Для **нескольких** корневых узлов (фрагмент) нужно явно указать, куда применять класс, через
`$attrs`:

```vue
<!-- MultiRoot.vue: два корня, поэтому Vue не знает, куда вешать class -->
<template>
  <p :class="$attrs.class">Заголовок</p>
  <span>Текст</span>
</template>

<script setup>
// отключаем автоматическое наследование, управляем вручную
defineOptions({ inheritAttrs: false })
</script>
```

### `:style` с объектом

```vue
<script setup>
import { ref } from 'vue'

const activeColor = ref('tomato')
const fontSize = ref(18)
</script>

<template>
  <!-- camelCase -->
  <p :style="{ color: activeColor, fontSize: fontSize + 'px' }">camelCase</p>

  <!-- kebab-case в кавычках -->
  <p :style="{ color: activeColor, 'font-size': fontSize + 'px' }">kebab-case</p>
</template>
```

Объект стилей лучше выносить отдельно:

```vue
<script setup>
import { reactive } from 'vue'

const styleObject = reactive({
  color: 'white',
  background: '#333',
  padding: '8px 12px',
})
</script>

<template>
  <div :style="styleObject">Вынесенный объект</div>
</template>
```

### `:style` с массивом

Массив объединяет несколько объектов стилей (последующие переопределяют предыдущие).

```vue
<script setup>
import { reactive } from 'vue'

const base = reactive({ color: 'white', fontSize: '14px' })
const overrides = reactive({ fontSize: '20px', fontWeight: 'bold' })
</script>

<template>
  <!-- fontSize из overrides победит: 20px -->
  <p :style="[base, overrides]">Слияние стилей</p>
</template>
```

### Автопрефиксы

Если указать свойство, требующее вендорного префикса (например, `transform`, `user-select`),
Vue в рантайме определит нужный префикс и добавит его автоматически.

```vue
<template>
  <!-- Vue сам подставит -webkit-/-ms- при необходимости -->
  <div :style="{ transform: 'rotate(10deg)', userSelect: 'none' }">Префиксы</div>
</template>
```

### Множественные значения одного свойства

Можно передать массив значений — браузер применит **последнее**, которое поддерживает. Полезно
для CSS со степенью деградации (например, `display: flex` с устаревшими префиксами).

```vue
<template>
  <!-- браузер выберет поддерживаемый вариант display -->
  <div :style="{ display: ['-webkit-box', '-ms-flexbox', 'flex'] }">
    Множественные значения
  </div>
</template>
```

## Полный рабочий пример

Карточка задачи с подсветкой статуса, приоритета и инлайн-прогрессом.

```vue
<script setup>
import { ref, reactive, computed } from 'vue'

const task = reactive({
  title: 'Подготовиться к собеседованию',
  done: false,
  priority: 'high', // 'low' | 'medium' | 'high'
  progress: 65,     // 0..100
})

const isOverdue = ref(true)

// Объект классов через computed
const cardClasses = computed(() => ({
  card: true,
  'card--done': task.done,
  'card--overdue': isOverdue.value && !task.done,
  [`card--priority-${task.priority}`]: true,
}))

// Цвет прогресс-бара зависит от значения
const progressColor = computed(() => {
  if (task.progress < 34) return '#e74c3c'
  if (task.progress < 67) return '#f39c12'
  return '#27ae60'
})

// Инлайн-стиль прогресс-бара
const progressStyle = computed(() => ({
  width: `${task.progress}%`,
  background: progressColor.value,
  transition: 'width 0.3s ease',
}))
</script>

<template>
  <!-- статический class 'task' + динамические через объект -->
  <div class="task" :class="cardClasses">
    <h3 :style="{ textDecoration: task.done ? 'line-through' : 'none' }">
      {{ task.title }}
    </h3>

    <!-- массив: базовые стили + переопределение для done -->
    <span
      :class="['badge', `badge--${task.priority}`]"
      :style="[{ padding: '2px 8px' }, task.done ? { opacity: 0.5 } : {}]"
    >
      Приоритет: {{ task.priority }}
    </span>

    <div class="progress">
      <div class="progress__bar" :style="progressStyle"></div>
    </div>

    <button @click="task.done = !task.done">
      {{ task.done ? 'Вернуть в работу' : 'Завершить' }}
    </button>
    <button @click="task.progress = Math.min(100, task.progress + 10)">+10%</button>
  </div>
</template>

<style scoped>
.card { border: 1px solid #ddd; border-radius: 8px; padding: 12px; }
.card--done { opacity: 0.6; }
.card--overdue { border-color: #e74c3c; }
.card--priority-high { box-shadow: 0 0 0 2px rgba(231, 76, 60, 0.3); }
.progress { height: 8px; background: #eee; border-radius: 4px; overflow: hidden; }
.progress__bar { height: 100%; }
</style>
```

## Частые вопросы на собеседовании (Q/A)

**1. Какие формы принимает `:class`?**
Строку, объект `{ имя: булево }` и массив (смесь строк, объектов и выражений). Объект — самый
частый способ переключать классы по условию.

**2. Что происходит со статическим `class` при наличии `:class`?**
Они **объединяются**, а не затираются. `class="a" :class="{ b: true }"` даст `class="a b"`.
То же справедливо для `style`.

**3. Как переключить класс по условию?**
Объектная форма: `:class="{ active: isActive }"`. Класс добавляется, когда выражение истинно.

**4. Как задать динамическое имя класса?**
Через шаблонную строку (`:class="\`btn--${size}\`"`) или объект с вычисляемым ключом
(`{ [\`theme-${theme}\`]: true }`).

**5. Какие формы принимает `:style`?**
Объект `{ свойство: значение }` (camelCase или kebab-case в кавычках) и массив объектов,
которые сливаются с приоритетом у последних.

**6. camelCase или kebab-case в `:style`?**
Можно оба. `fontSize` без кавычек или `'font-size'` в кавычках — эквивалентны. camelCase
удобнее, так как не требует кавычек.

**7. Что такое автопрефиксы в `:style`?**
Vue в рантайме сам добавляет вендорные префиксы (`-webkit-`, `-ms-` и т.д.) для свойств,
которым они нужны, например `transform` или `user-select`.

**8. Как задать несколько значений одного CSS-свойства?**
Передать массив значений: `:style="{ display: ['-webkit-box', 'flex'] }"`. Браузер применит
последнее поддерживаемое значение.

**9. Куда попадают классы, переданные компоненту?**
На корневой элемент компонента (если корень один) — благодаря наследованию атрибутов; классы
объединяются с собственными классами корня.

**10. Что если у компонента несколько корневых элементов?**
Vue не знает, куда вешать класс, и выдаст предупреждение. Нужно отключить `inheritAttrs` и
вручную привязать `:class="$attrs.class"` к нужному узлу.

**11. Можно ли вынести объект классов/стилей в `computed`?**
Да, и это рекомендуется для сложной логики: шаблон остаётся чистым, а кеширование `computed`
даёт производительность.

**12. В чём разница между объектным и массивным синтаксисом `:class`?**
Объект — для **переключения** классов по булевым условиям. Массив — для **комбинирования**
списка классов/выражений (можно вкладывать объекты внутрь массива).

## Подводные камни (gotchas)

- **Числовые значения в `:style`** требуют единиц: `fontSize: 18` не сработает, нужно `'18px'`.
- **kebab-case без кавычек** в объекте стилей — синтаксическая ошибка: `font-size` читается как
  вычитание. Только `'font-size'` или `fontSize`.
- **Несколько корней у компонента** ломают авто-наследование `class`/`style` — нужен ручной
  `$attrs`.
- **Перезапись вместо слияния**: помните, что статический и динамический атрибут объединяются;
  не дублируйте классы.
- **Сложные выражения прямо в шаблоне** ухудшают читаемость — выносите в `computed`.
- **Реактивность объекта стилей**: чтобы изменения подхватывались, объект должен быть
  реактивным (`reactive`/`ref`), а не обычным.

## Лучшие практики

- Для условных классов используйте **объектный** синтаксис, для наборов — **массивный**.
- Сложную логику классов/стилей выносите в `computed` — чисто и кешируется.
- Предпочитайте классы (CSS) инлайн-стилям; `:style` — для **динамических** значений
  (позиция, размер, цвет из данных).
- Всегда указывайте единицы измерения в `:style` (`'px'`, `'%'`, `'rem'`).
- Используйте `<style scoped>` или CSS-модули, а классы переключайте через `:class`.
- Для компонентов с несколькими корнями осознанно управляйте `inheritAttrs` и `$attrs`.

## Шпаргалка

```vue
<!-- :class -->
<div :class="{ active: isActive, 'is-error': hasError }" />     <!-- объект -->
<div :class="[classA, classB]" />                               <!-- массив строк -->
<div :class="[isActive ? 'on' : 'off', { warn: hasWarn }]" />   <!-- смешанный -->
<div class="base" :class="dyn" />                               <!-- слияние со static -->

<!-- :style -->
<div :style="{ color: 'red', fontSize: '16px' }" />             <!-- объект camelCase -->
<div :style="{ 'font-size': '16px' }" />                        <!-- kebab в кавычках -->
<div :style="[baseStyles, overrideStyles]" />                   <!-- массив (слияние) -->
<div :style="{ display: ['-webkit-box', 'flex'] }" />           <!-- множественные значения -->
```

| Синтаксис | `:class`                       | `:style`                        |
|-----------|--------------------------------|---------------------------------|
| Строка    | `"a b"`                        | —                               |
| Объект    | `{ имя: булево }`              | `{ свойство: значение }`        |
| Массив    | `[строки, объекты, выражения]` | `[объект1, объект2]` (слияние)  |
```
