# Vue 3 — Механизм рендеринга — конспект и вопросы

## О чём раздел

Как Vue превращает шаблон в реальный DOM и как обновляет его эффективно. Разберём
**Virtual DOM** (что это и зачем), этапы рендеринга (**compile → mount → patch**),
**компилятор шаблонов** и его оптимизации (**static hoisting, patch flags, tree
flattening**), за счёт которых Vue 3 обгоняет "наивный" VDOM diff, render-функции
`h()` и JSX, функциональные компоненты и разницу между сборками
**runtime+compiler** и **runtime-only**.

## Ключевые концепции

- **VNode (Virtual Node)** — лёгкий JS-объект, описывающий узел DOM (тег, props,
  children).
- **Virtual DOM** — дерево VNode в памяти; "виртуальное" представление UI.
- **render-функция** — функция, возвращающая VNode-дерево. Шаблон компилируется в неё.
- **mount** — первичное создание реального DOM из VNode.
- **patch (diff)** — сравнение старого и нового VNode-дерева и применение минимальных
  изменений к реальному DOM.
- **Compiler-informed VDOM** — главная фишка Vue 3: компилятор анализирует шаблон и
  оставляет подсказки (patch flags, hoisted-узлы), чтобы diff пропускал статику.

## Подробный разбор с примерами кода

### Virtual DOM — что это и зачем

VNode — это объект вроде:

```ts
const vnode = {
  type: 'div',
  props: { id: 'app', class: 'container' },
  children: [
    { type: 'h1', props: null, children: 'Hello' },
  ],
}
```

Зачем VDOM:
- **Декларативность**: пишем "как должно выглядеть", а не "как менять DOM руками".
- **Дешёвое сравнение**: операции с JS-объектами быстрее, чем прямые манипуляции DOM.
- **Платформонезависимость**: то же VNode-дерево можно рендерить в DOM, в строку
  (SSR), в нативные виджеты.

Минус "наивного" VDOM: чтобы найти изменения, нужно обойти **всё** дерево, включая
статические части. Vue 3 решает это компилятором.

### Этапы: compile → mount → patch

1. **Compile** — шаблон (`<template>`) компилируется в render-функцию. Может
   происходить заранее (build-time, через `@vue/compiler-sfc`) или в рантайме (для
   шаблонов-строк).
2. **Mount** — render-функция вызывается, возвращает VNode-дерево, из которого
   создаётся реальный DOM и вставляется на страницу. Тут же создаётся `ReactiveEffect`,
   связывающий рендер с реактивными зависимостями.
3. **Patch (update)** — при изменении реактивных данных эффект перезапускает
   render-функцию, получая *новое* VNode-дерево, которое сравнивается со старым; в
   реальный DOM применяются только различия.

```
template ──compile──> render() ──mount──> real DOM
                          ▲                   │
              реактивное изменение            │ patch (diff old vs new)
                          └───────────────────┘
```

### Компилятор шаблонов и его оптимизации

Vue 3 — **compiler-informed Virtual DOM**: компилятор знает структуру шаблона на
этапе сборки и встраивает оптимизации. Посмотреть результат можно в
[Vue Template Explorer](https://template-explorer.vuejs.org/).

#### 1. Static Hoisting (подъём статики)

Статические VNode создаются **один раз** вне render-функции и переиспользуются при
каждом ре-рендере (не создаются заново).

```vue
<template>
  <div>
    <p class="static">Это статика</p>   <!-- никогда не меняется -->
    <p>{{ dynamic }}</p>                 <!-- меняется -->
  </div>
</template>
```

Компилируется примерно в:

```js
// hoisted — создаётся ОДИН раз при загрузке модуля
const _hoisted_1 = createElementVNode("p", { class: "static" }, "Это статика", -1 /* HOISTED */)

function render(_ctx) {
  return (openBlock(), createElementBlock("div", null, [
    _hoisted_1,                                   // переиспользуется, не пересоздаётся
    createElementVNode("p", null, toDisplayString(_ctx.dynamic), 1 /* TEXT */),
  ]))
}
```

#### 2. Patch Flags (флаги патчинга)

Каждому **динамическому** VNode компилятор присваивает числовой флаг (битовая маска),
указывающий, *что именно* в нём может измениться. При patch'е Vue проверяет только
помеченные аспекты, а не все props/children.

```vue
<template>
  <div :id="id" class="static">{{ text }}</div>
</template>
```

```js
createElementVNode("div", { id: _ctx.id, class: "static" },
  toDisplayString(_ctx.text),
  9 /* TEXT, PROPS */,        // 9 = 1(TEXT) | 8(PROPS)
  ["id"]                       // dynamicProps: меняться может только id
)
```

Основные patch flags (значения — степени двойки, комбинируются через ИЛИ):

| Флаг            | Значение | Что отслеживается                          |
|-----------------|----------|--------------------------------------------|
| `TEXT`          | 1        | динамический текстовый контент             |
| `CLASS`         | 2        | динамический `class`                       |
| `STYLE`         | 4        | динамический `style`                       |
| `PROPS`         | 8        | динамические props (список в `dynamicProps`) |
| `FULL_PROPS`    | 16       | props с динамическим ключом — полный diff  |
| `NEED_PATCH`    | 32       | ref/директивы — нужна обработка            |
| `STABLE_FRAGMENT` | 64     | фрагмент со стабильным порядком детей       |
| `HOISTED`       | -1       | статический узел, патч пропускается        |
| `BAIL`          | -2       | оптимизация отключена, полный diff          |

Смысл: благодаря флагам patch для `<div :id="id" class="static">{{ text }}</div>`
проверит только текст и `id`, проигнорировав `class` и структуру.

#### 3. Tree Flattening (уплощение дерева / Blocks)

Шаблон делится на **блоки** (block) — участки со стабильной структурой. Блок хранит
плоский массив **только динамических потомков** (`dynamicChildren`), даже если они
вложены глубоко. При обновлении Vue обходит этот плоский массив, а не рекурсивно всё
дерево.

```vue
<template>
  <div>            <!-- block root -->
    <section>
      <header>Заголовок</header>          <!-- статика, в dynamicChildren не попадёт -->
      <p>{{ msg }}</p>                     <!-- динамика -> в dynamicChildren -->
    </section>
  </div>
</template>
```

```js
function render(_ctx) {
  return (openBlock(), createElementBlock("div", null, [
    createElementVNode("section", null, [
      _hoisted_1 /* <header> */,
      createElementVNode("p", null, toDisplayString(_ctx.msg), 1 /* TEXT */),
    ])
  ]))
  // блок собирает [<p>] в dynamicChildren — обновляется ТОЛЬКО он
}
```

**Почему Vue быстрее "наивного" VDOM diff:** наивный diff обходит весь объём дерева
(O(n) по всем узлам). Vue благодаря blocks обходит только динамические узлы — объём
работы пропорционален количеству *динамического* контента, а не всему шаблону.
Статика, hoisted-узлы и неизменные атрибуты вообще не сравниваются.

> Нюанс: структуры, ломающие стабильность блока (`v-if`, `v-for`), создают
> **собственные блоки**, чтобы корректно отслеживать изменения структуры.

### Render-функции h() и JSX

Когда шаблона недостаточно, пишут render-функцию вручную через `h()` (hyperscript):

```ts
import { h, ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    // render-функция возвращается из setup
    return () =>
      h('div', { class: 'box' }, [
        h('span', `Счёт: ${count.value}`),
        h('button', { onClick: () => count.value++ }, '+'),
      ])
  },
}
```

`h(type, props?, children?)`:
- `type` — строка тега, компонент или Fragment.
- `props` — объект атрибутов/событий (`onClick`, `class`, `style`...).
- `children` — строка, массив VNode или объект слотов.

JSX/TSX (через `@vitejs/plugin-vue-jsx`):

```tsx
import { ref } from 'vue'

export default function Counter() {
  const count = ref(0)
  return () => (
    <div class="box">
      <span>Счёт: {count.value}</span>
      <button onClick={() => count.value++}>+</button>
    </div>
  )
}
```

> Важно: render-функции/JSX **не получают** compiler-оптимизаций (patch flags,
> hoisting) автоматически — они полнодиффятся. Для большинства UI шаблоны
> производительнее.

#### Когда нужны render-функции

- Программная генерация структуры (динамический тег/число детей).
- Высокая логическая гибкость, которую неудобно выразить в шаблоне
  (например, рендер по сложным правилам, `v-for` с ветвлениями).
- Библиотечные компоненты-обёртки, прозрачно проксирующие слоты/props.
- Компоненты вроде `<RouterView>`, render-props паттерны.

### Функциональные компоненты

Функциональный компонент — это просто функция `(props, context) => VNode`, без
собственного состояния и инстанса. Лёгкие, но в Vue 3 разница в производительности с
обычными невелика.

```ts
import { h, type FunctionalComponent } from 'vue'

// props и события объявляются через свойства функции
const Heading: FunctionalComponent<{ level: number }> = (props, { slots }) => {
  return h(`h${props.level}`, slots.default?.())
}
Heading.props = { level: { type: Number, required: true } }
```

Применение: презентационные обёртки, динамический выбор тега, мелкие утилитарные
компоненты.

### runtime+compiler vs runtime-only

Vue поставляется в двух основных вариантах сборки:

- **runtime + compiler** (`vue/dist/vue.esm-bundler.js`): содержит компилятор
  шаблонов, умеет компилировать строковые шаблоны **в браузере** (опция `template:`
  или монтирование в элемент с HTML). Больше по размеру (~+компилятор).
- **runtime-only** (`vue.runtime.esm-bundler.js`): только рантайм. Шаблоны должны быть
  скомпилированы **заранее** на этапе сборки (`vue-loader`/`@vitejs/plugin-vue`).
  Меньше размер, быстрее старт.

```ts
// runtime-only: НЕ сработает, нет компилятора в рантайме
createApp({ template: '<div>{{ msg }}</div>' })   // ⚠ ошибка/warning

// runtime-only: ОК — render-функция или предкомпилированный SFC
createApp({ render: () => h('div', 'ok') })
```

В большинстве проектов (Vite/CLI с `.vue`-файлами) используется **runtime-only** —
шаблоны компилируются в build-time. Если требуется монтировать строковые шаблоны на
лету, нужна сборка с компилятором (или alias `vue: 'vue/dist/vue.esm-bundler.js'`).

## Полный рабочий пример

Сравнение шаблона и его скомпилированной render-функции + ручной render-компонент:

```vue
<!-- DynamicGreeting.vue — обычный шаблон (получает все оптимизации) -->
<script setup lang="ts">
import { ref } from 'vue'
const name = ref('Andrei')
</script>

<template>
  <div class="greeting">
    <!-- статика поднимется (hoist) -->
    <h2 class="title">Привет</h2>
    <!-- динамика: patch flag TEXT, обновится только текст -->
    <p>{{ name }}</p>
    <input v-model="name" />
  </div>
</template>
```

```tsx
// TableCell.tsx — render-функция оправдана: тег зависит от props
import { h, type FunctionalComponent } from 'vue'

interface Props {
  tag?: 'td' | 'th'
  value: string | number
  align?: 'left' | 'right'
}

// Функциональный компонент: динамический тег + проксирование стилей
const TableCell: FunctionalComponent<Props> = (props) => {
  return h(
    props.tag ?? 'td',
    { style: { textAlign: props.align ?? 'left' } },
    String(props.value),
  )
}
TableCell.props = ['tag', 'value', 'align']

export default TableCell
```

```ts
// main.ts — runtime-only сборка: всё предкомпилировано плагином Vite
import { createApp } from 'vue'
import App from './App.vue'

createApp(App).mount('#app')   // никаких строковых шаблонов в рантайме
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое Virtual DOM и зачем он нужен?**
Дерево лёгких JS-объектов (VNode), описывающих UI. Позволяет декларативно описывать
интерфейс, дёшево сравнивать изменения в памяти и рендерить на разные платформы
(DOM/SSR).

**2. Какие этапы проходит шаблон до DOM?**
compile (шаблон → render-функция) → mount (VNode → реальный DOM) → patch (diff
старого и нового VNode при изменении данных).

**3. Что такое compiler-informed Virtual DOM?**
Подход Vue 3: компилятор анализирует шаблон на этапе сборки и встраивает подсказки
(patch flags, hoisting, blocks), позволяя пропускать статику при diff'е.

**4. Что такое static hoisting?**
Подъём статических VNode за пределы render-функции: они создаются один раз и
переиспользуются при каждом ре-рендере, не пересоздаваясь.

**5. Что такое patch flags?**
Числовые битовые флаги на динамических VNode, указывающие, что именно может измениться
(TEXT/CLASS/STYLE/PROPS...). Patch проверяет только помеченное, а не все аспекты узла.

**6. Что такое tree flattening / blocks?**
Шаблон делится на блоки, каждый хранит плоский массив только динамических потомков
(`dynamicChildren`). Обновление обходит этот массив, а не всё дерево рекурсивно.

**7. Почему Vue быстрее "наивного" VDOM diff?**
Наивный diff обходит все узлы; Vue благодаря blocks обходит только динамические —
работа пропорциональна объёму динамики, а не всему шаблону. Статика не сравнивается.

**8. Почему `v-if`/`v-for` создают отдельные блоки?**
Они меняют структуру дерева (число/наличие узлов), что ломает стабильность родительского
блока. Свой блок позволяет корректно отслеживать структурные изменения.

**9. Когда нужны render-функции вместо шаблона?**
Когда нужна программная гибкость: динамический тег/число детей, сложная логика
ветвления, библиотечные обёртки, render-props. Минус — нет compiler-оптимизаций.

**10. Получают ли render-функции/JSX patch flags?**
Нет (по умолчанию). Они полностью диффятся. Для типового UI шаблоны производительнее
из-за compiler-оптимизаций.

**11. Что такое функциональный компонент?**
Чистая функция `(props, ctx) => VNode` без состояния и инстанса. Лёгкая, для
презентационных/утилитарных случаев. В Vue 3 выигрыш в перфомансе минимален.

**12. Чем отличаются сборки runtime+compiler и runtime-only?**
runtime+compiler включает компилятор шаблонов (можно компилировать строковые шаблоны в
браузере, больше размер). runtime-only — только рантайм, шаблоны компилируются на
build-time (меньше, быстрее; стандарт для `.vue`-проектов).

**13. Где увидеть результат компиляции шаблона?**
В Vue Template Explorer (template-explorer.vuejs.org) — показывает render-функцию,
hoisted-узлы и patch flags.

**14. Что такое `h()`?**
Функция создания VNode: `h(type, props?, children?)`. Низкоуровневый аналог того, во
что компилируется шаблон.

## Подводные камни (gotchas)

- **render-функции теряют оптимизации** — patch flags и hoisting не применяются, diff
  полный. Не переписывайте шаблоны в `h()` без причины.
- **runtime-only + строковый `template`** не работает — нет компилятора; получите
  warning/ошибку.
- **`v-for` без `key`** мешает корректному и эффективному patch'у списков
  (переиспользование не тех узлов).
- **Большой объём статики в шаблоне** не вредит (она хойстится), но динамические
  обёртки вокруг статики могут снижать выигрыш от blocks.
- **JSX и `v-model`/директивы** — синтаксис отличается; не все шаблонные удобства
  переносятся 1:1.
- **Функциональные компоненты** не имеют `this`, инстанса, lifecycle-хуков и ref'ов на себя.

## Лучшие практики

- По умолчанию используйте шаблоны — они получают все compiler-оптимизации.
- Render-функции/JSX — только когда нужна программная гибкость.
- Всегда задавайте `key` в `v-for` для корректного patch'а.
- Для `.vue`-проектов используйте runtime-only сборку (предкомпиляция в build-time).
- Изучайте вывод Template Explorer, чтобы понимать, как пишется ваш шаблон.
- Не дробите шаблон на render-функции "ради скорости" — обычно это медленнее.

## Шпаргалка

```
VNode            { type, props, children }
Этапы            compile → mount → patch (diff)
Static hoisting  статика создаётся 1 раз, переиспользуется
Patch flags      TEXT=1, CLASS=2, STYLE=4, PROPS=8, FULL_PROPS=16,
                 HOISTED=-1, BAIL=-2 (битовая маска)
Tree flattening  block.dynamicChildren — плоский список только динамики
Почему быстро    diff пропорционален динамике, не всему дереву
h(type, props, children)   // создание VNode вручную
JSX              @vitejs/plugin-vue-jsx
FunctionalComponent (props, ctx) => VNode   // без состояния
Сборки           runtime+compiler (строковые шаблоны в браузере)
                 runtime-only   (шаблоны компилируются в build, стандарт)
Explorer         template-explorer.vuejs.org
```
