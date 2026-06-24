# Vue 3 — Пользовательские директивы (custom directives) — конспект и вопросы

## О чём раздел

Помимо встроенных директив (`v-model`, `v-show`, `v-if` и т.д.), Vue позволяет создавать **пользовательские директивы**. Это инструмент для **переиспользования логики, работающей напрямую с DOM-элементом**.

Главный критерий «когда нужна директива»: когда требуется **низкоуровневый доступ к конкретному DOM-узлу**, которого нельзя достичь обычными средствами шаблона. Примеры: автофокус инпута при монтировании, отслеживание клика вне элемента, ленивая загрузка изображения через `IntersectionObserver`, интеграция с не-Vue библиотеками, манипулирующими DOM.

Если же логика связана с переиспользованием **состояния**, а не с прямой работой с DOM — это задача для **композабла**, а не директивы.

```vue
<script setup>
// Локальная директива: имя с префиксом v + camelCase
const vFocus = {
  mounted: (el) => el.focus() // фокус при вставке элемента в DOM
}
</script>

<template>
  <input v-focus />
</template>
```

## Ключевые концепции

- **Директива — это объект с хуками жизненного цикла** (или функция-сокращение).
- **Хуки**: `created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`.
- **Сокращённая форма (функция)**: если нужна одинаковая логика на `mounted` и `updated` — передаём функцию вместо объекта.
- **`binding`** — объект, передаваемый в хуки: содержит `value`, `oldValue`, `arg`, `modifiers`, `instance`, `dir`.
- **Регистрация**: локальная (в `<script setup>` переменная `vName`) или глобальная (`app.directive('name', def)`).
- **Именование в `<script setup>`**: переменная должна начинаться с `v` + PascalCase: `vFocus`, `vClickOutside`.
- **Аргумент (`arg`)**: часть после двоеточия `v-dir:arg`. Может быть динамическим: `v-dir:[argName]`.
- **Модификаторы (`modifiers`)**: после точки `v-dir.foo.bar`.
- **Значение (`value`)**: после `=`, например `v-dir="someValue"`.
- **На компонентах**: директивы применяются к **корневому элементу** компонента (с нюансами; не всегда рекомендуется).

## Подробный разбор с примерами кода

### 1. Полная форма: объект с хуками

```js
const myDirective = {
  // Перед привязкой атрибутов/слушателей элемента
  created(el, binding, vnode) {},
  // Перед вставкой элемента в DOM
  beforeMount(el, binding, vnode) {},
  // Элемент и его родитель вставлены в DOM
  mounted(el, binding, vnode) {},
  // Перед обновлением родительского компонента
  beforeUpdate(el, binding, vnode, prevVnode) {},
  // После обновления родителя и его детей
  updated(el, binding, vnode, prevVnode) {},
  // Перед удалением элемента
  beforeUnmount(el, binding, vnode) {},
  // Элемент удалён из DOM
  unmounted(el, binding, vnode) {}
}
```

Аргументы хуков:
- **`el`** — сам DOM-элемент, к которому привязана директива.
- **`binding`** — объект (см. ниже).
- **`vnode`** — виртуальный узел, представляющий элемент.
- **`prevVnode`** — предыдущий vnode (только в `beforeUpdate`/`updated`).

### 2. Объект binding

```vue
<template>
  <div v-example:foo.bar="baz"></div>
</template>
```

В хуке `binding` будет:

```js
{
  value: /* значение baz */,        // результат вычисления выражения
  oldValue: /* предыдущее значение */, // доступно в beforeUpdate/updated
  arg: 'foo',                       // аргумент (после ":")
  modifiers: { bar: true },         // модификаторы (после ".")
  instance: /* экземпляр компонента, использующего директиву */,
  dir: /* объект определения директивы */
}
```

### 3. Сокращённая форма (функция)

Частый случай — нужна одна и та же логика на `mounted` **и** `updated`, а другие хуки не важны. Тогда вместо объекта передают функцию:

```js
// Эта функция вызовется и на mounted, и на updated
app.directive('color', (el, binding) => {
  el.style.color = binding.value
})
```

```vue
<template>
  <!-- цвет применится при монтировании и обновится при изменении -->
  <p v-color="textColor">Текст</p>
</template>
```

### 4. Аргумент и модификаторы

```vue
<script setup>
const vPin = {
  mounted(el, binding) {
    el.style.position = 'fixed'
    // arg задаёт сторону: v-pin:top / v-pin:left
    const side = binding.arg || 'top'
    // value задаёт отступ
    el.style[side] = binding.value + 'px'
  }
}
</script>

<template>
  <!-- Прижать к низу на 20px -->
  <div v-pin:bottom="20">Закреплённый блок</div>
</template>
```

Динамический аргумент:

```vue
<template>
  <!-- сторона вычисляется реактивно -->
  <div v-pin:[direction]="200">...</div>
</template>
```

### 5. Передача нескольких значений через объект

У директивы только одно `value`, поэтому несколько параметров передают объектным литералом:

```vue
<template>
  <div v-demo="{ color: 'white', text: 'Привет!' }"></div>
</template>
```

```js
app.directive('demo', (el, binding) => {
  console.log(binding.value.color) // 'white'
  console.log(binding.value.text)  // 'Привет!'
})
```

### 6. Регистрация: локальная и глобальная

**Глобальная** (доступна во всём приложении):

```js
// main.js
const app = createApp(App)

app.directive('focus', {
  mounted: (el) => el.focus()
})
```

**Локальная** в Composition API (`<script setup>`):

```vue
<script setup>
// Переменная vFocus автоматически доступна как директива v-focus
const vFocus = {
  mounted: (el) => el.focus()
}
</script>
```

**Локальная** в Options API:

```js
export default {
  directives: {
    focus: {
      mounted: (el) => el.focus()
    }
  }
}
```

### 7. Директивы на компонентах (нюанс)

Когда директиву применяют к компоненту, она навешивается на **корневой элемент** этого компонента (как и неунаследованные атрибуты — fallthrough):

```vue
<MyComponent v-demo="test" />
```

```vue
<!-- MyComponent имеет один корневой элемент -->
<template>
  <div>Корень — сюда применится v-demo</div>
</template>
```

**Нюансы:**
- Если у компонента **несколько корневых элементов** (фрагмент), Vue не знает, куда применить директиву — будет предупреждение, директива проигнорируется.
- В отличие от атрибутов, директиву **нельзя** «прокинуть» в нужный элемент через `v-bind="$attrs"`.
- Поэтому применять директивы к компонентам в общем случае **не рекомендуется**.

## Полный рабочий пример

Директива `v-click-outside` — закрытие по клику вне элемента, с очисткой слушателя:

```js
// directives/clickOutside.js
export const vClickOutside = {
  // beforeMount/mounted — настраиваем слушатель
  mounted(el, binding) {
    // Сохраняем обработчик на самом элементе, чтобы убрать его позже
    el.__clickOutsideHandler__ = (event) => {
      // Клик был НЕ внутри элемента?
      if (!(el === event.target || el.contains(event.target))) {
        // binding.value — переданный колбэк
        binding.value(event)
      }
    }
    document.addEventListener('click', el.__clickOutsideHandler__)
  },
  // unmounted — обязательно чистим за собой
  unmounted(el) {
    document.removeEventListener('click', el.__clickOutsideHandler__)
    delete el.__clickOutsideHandler__
  }
}
```

```vue
<!-- Dropdown.vue -->
<script setup>
import { ref } from 'vue'
import { vClickOutside } from './directives/clickOutside'

const open = ref(false)

function close() {
  open.value = false
}
</script>

<template>
  <div class="dropdown">
    <button @click="open = !open">Меню</button>

    <!-- При клике вне блока вызовется close -->
    <ul v-if="open" v-click-outside="close" class="menu">
      <li>Профиль</li>
      <li>Настройки</li>
      <li>Выход</li>
    </ul>
  </div>
</template>
```

Бонус — ленивая загрузка изображений через `IntersectionObserver`:

```js
// directives/lazyLoad.js
export const vLazy = {
  mounted(el, binding) {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        el.src = binding.value      // реальный URL из value
        observer.unobserve(el)      // загрузили — больше не следим
      }
    })
    observer.observe(el)
    el.__observer__ = observer
  },
  unmounted(el) {
    el.__observer__?.disconnect()  // чистим observer
  }
}
```

```vue
<template>
  <!-- картинка загрузится только когда попадёт во вьюпорт -->
  <img v-lazy="'/images/big-photo.jpg'" alt="" />
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Когда стоит использовать пользовательскую директиву, а не компонент/композабл?**
Когда нужен **прямой низкоуровневый доступ к DOM-элементу**: фокус, скролл, наблюдатели (Intersection/Resize), интеграция со сторонними DOM-библиотеками. Если логика про переиспользование состояния — это композабл; если про разметку — компонент.

**2. Какие хуки жизненного цикла есть у директивы?**
`created`, `beforeMount`, `mounted`, `beforeUpdate`, `updated`, `beforeUnmount`, `unmounted`. (В Vue 2 были другие имена: `bind`, `inserted`, `update`, `componentUpdated`, `unbind` — частый вопрос на знание миграции.)

**3. Что такое сокращённая форма директивы?**
Если нужно выполнять одну и ту же логику на `mounted` и `updated`, можно передать **функцию** вместо объекта с хуками. Она будет вызвана в обоих этих случаях.

**4. Что содержит объект binding?**
`value` (текущее значение выражения), `oldValue` (предыдущее, в update-хуках), `arg` (аргумент после `:`), `modifiers` (объект модификаторов после `.`), `instance` (экземпляр компонента), `dir` (определение директивы).

**5. Как передать несколько параметров в директиву?**
Через объектный литерал в `value`: `v-demo="{ color: 'red', text: 'hi' }"`, поскольку у директивы только одно значение.

**6. Как сделать аргумент динамическим?**
Использовать квадратные скобки: `v-demo:[argName]="value"`, где `argName` — реактивная переменная компонента.

**7. Как зарегистрировать директиву глобально и локально?**
Глобально: `app.directive('name', definition)`. Локально в `<script setup>`: объявить переменную `vName` (она автоматически станет `v-name`). В Options API — через опцию `directives`.

**8. Как назвать локальную директиву в `<script setup>`?**
Переменная должна начинаться с `v` и далее PascalCase: `const vMyDirective = {...}` → используется как `v-my-directive`.

**9. Что происходит при применении директивы к компоненту?**
Директива навешивается на **корневой элемент** компонента (механизм похож на fallthrough-атрибуты). Если корней несколько — директива игнорируется с предупреждением. В целом применять директивы к компонентам не рекомендуется.

**10. Почему важно чистить ресурсы в директивах и как?**
Если в `mounted` добавлен слушатель события, observer или таймер — без очистки будет утечка памяти. Очистку делают в хуке `unmounted` (удаляют слушатель/`disconnect()` observer). Состояние удобно хранить прямо на элементе (`el.__handler__`).

**11. В чём разница между директивой и компонентом для переиспользования?**
Компонент инкапсулирует разметку + логику + состояние и рендерит свой шаблон. Директива не рендерит ничего — она лишь модифицирует поведение/DOM существующего элемента. Для переиспользования UI берут компонент, для «поведения над элементом» — директиву.

**12. Приведите типичные кейсы директив.**
`v-focus` (автофокус), `v-click-outside` (клик вне), `v-lazy` (ленивая загрузка изображений), `v-tooltip`, `v-resize`/`v-intersect`, интеграция с drag-and-drop библиотеками, маски ввода.

## Подводные камни (gotchas)

- **Забытая очистка**: слушатели/observers/таймеры из `mounted` без удаления в `unmounted` → утечки памяти.
- **Несколько корневых элементов** у компонента → директива на компоненте игнорируется с предупреждением.
- **Имена хуков из Vue 2**: `bind`/`inserted`/`unbind` в Vue 3 не работают — нужны `mounted`/`unmounted` и т.д.
- **`value` обновляется реактивно, но логика — нет**: если применять значение только в `mounted`, при изменении `value` ничего не произойдёт. Нужно обрабатывать `updated` (или использовать функцию-сокращение).
- **`updated` срабатывает на любой ререндер родителя**, даже если `value` директивы не менялось — внутри стоит сравнивать `binding.value` с `binding.oldValue`, чтобы не делать лишнюю работу.
- **Хранение состояния на `el`**: удобно, но используйте уникальные ключи (`el.__myDirHandler__`), чтобы не конфликтовать.
- **SSR**: директивы выполняют DOM-логику только в браузере; на сервере хуки вроде `mounted` не вызываются.
- **Применение к компонентам** — антипаттерн в большинстве случаев; предпочитайте обычный элемент.

## Лучшие практики

- **Используйте директивы только для DOM-манипуляций**, для которых нет декларативной альтернативы.
- **Всегда очищайте ресурсы** в `unmounted` (или `beforeUnmount`).
- **Обрабатывайте обновление значения**: применяйте логику и в `mounted`, и в `updated` (или используйте функцию-сокращение), если `value` может меняться.
- **Сравнивайте `value`/`oldValue`** в `updated`, чтобы избегать лишних операций.
- **Передавайте несколько параметров объектом** через `value`.
- **Именуйте осмысленно** и с префиксом по соглашению (`vFocus`, `v-click-outside`).
- **Избегайте директив на компонентах**, кроме случаев с одним корневым элементом и осознанной необходимости.
- **Не дублируйте задачу композаблов/компонентов** — директива не для stateful-логики и не для переиспользования разметки.

## Шпаргалка

```js
// Полная форма
const vDir = {
  created(el, binding, vnode) {},
  beforeMount(el, binding, vnode) {},
  mounted(el, binding, vnode) {},          // вставлен в DOM — настройка
  beforeUpdate(el, binding, vnode, prev) {},
  updated(el, binding, vnode, prev) {},    // value мог измениться
  beforeUnmount(el, binding, vnode) {},
  unmounted(el, binding, vnode) {}         // удалён — очистка
}

// Сокращённая форма (mounted + updated)
const vColor = (el, binding) => { el.style.color = binding.value }

// binding: { value, oldValue, arg, modifiers, instance, dir }
```

```vue
<!-- Синтаксис применения -->
<div v-dir:arg.mod1.mod2="value" />
<div v-dir:[dynamicArg]="value" />
<div v-dir="{ a: 1, b: 2 }" />     <!-- несколько параметров -->
```

| Что | Где |
|---|---|
| Локально (script setup) | `const vName = {...}` → `v-name` |
| Локально (Options API) | `directives: { name: {...} }` |
| Глобально | `app.directive('name', {...})` |

| Vue 2 хук | Vue 3 хук |
|---|---|
| `bind` | `beforeMount` |
| `inserted` | `mounted` |
| `update` | `beforeUpdate` |
| `componentUpdated` | `updated` |
| `unbind` | `unmounted` |

| Нужно... | Использовать |
|---|---|
| Прямой доступ к DOM-узлу | Директиву |
| Переиспользовать stateful-логику | Композабл |
| Переиспользовать разметку + логику | Компонент |
