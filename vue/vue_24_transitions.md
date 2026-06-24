# Vue 3 — Анимации переходов (Transition, TransitionGroup) — конспект и вопросы

## О чём раздел

Этот раздел посвящён встроенным механизмам анимации Vue 3:

- **`<Transition>`** — обёртка для анимации появления/исчезновения **одного** элемента или компонента (или переключения между двумя элементами через `v-if`/`v-else`, динамический `:is`).
- **`<TransitionGroup>`** — анимация **списков** элементов (`v-for`): добавление, удаление и перемещение (FLIP).

Vue не навязывает анимационную библиотеку: он лишь автоматически навешивает/снимает CSS-классы в нужные моменты жизненного цикла и предоставляет JS-хуки. Дальше можно использовать чистый CSS (`transition`/`animation`), либо стороннюю библиотеку (GSAP, anime.js, Web Animations API).

## Ключевые концепции

- `<Transition>` анимирует **enter** (вход) и **leave** (выход) одного узла.
- Срабатывает на: `v-if`, `v-show`, динамические компоненты `<component :is>`, изменение `key`.
- Внутри должен быть **ровно один** корневой элемент/компонент (для нескольких — `<TransitionGroup>`).
- Vue добавляет 6 классов в течение перехода: `*-enter-from`, `*-enter-active`, `*-enter-to`, `*-leave-from`, `*-leave-active`, `*-leave-to`.
- Префикс по умолчанию — `v-`. С атрибутом `name="fade"` префикс становится `fade-`.
- JS-хуки (`@before-enter`, `@enter`, `@after-enter`, `@enter-cancelled` и аналоги для leave) для императивных/JS-анимаций.
- `appear` — анимировать при **первом** рендере.
- `mode="out-in"` / `"in-out"` — порядок при переходе между двумя элементами.
- `<TransitionGroup>` рендерит реальный обёрточный тег (`tag="ul"`), требует `key` у каждого элемента и анимирует перемещение через `*-move`.

## Подробный разбор с примерами кода

### 1. Базовый CSS-переход

```vue
<script setup>
import { ref } from 'vue'

const show = ref(true)
</script>

<template>
  <button @click="show = !show">Переключить</button>

  <!-- name="fade" => классы будут fade-enter-from и т.д. -->
  <Transition name="fade">
    <p v-if="show">Привет, Vue!</p>
  </Transition>
</template>

<style scoped>
/* enter-active и leave-active задают характер анимации (длительность, easing) */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

/* начальное состояние входа и конечное состояние выхода */
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
/* конечное состояние входа (.fade-enter-to) и начальное выхода (.fade-leave-from)
   опущены, потому что opacity: 1 — это значение по умолчанию */
</style>
```

### 2. Жизненный цикл классов (важно понимать на собеседовании)

**Enter (появление):**

1. `v-enter-from` добавляется до вставки в DOM (начальное состояние).
2. На следующий кадр (next frame) добавляется `v-enter-active`, а `v-enter-from` снимается и добавляется `v-enter-to`.
3. По окончании transition/animation оба `*-active` и `*-to` снимаются.

**Leave (исчезновение):**

1. `v-leave-from` + `v-leave-active` добавляются в момент старта выхода.
2. На следующий кадр `v-leave-from` снимается, добавляется `v-leave-to`.
3. По окончании анимации элемент удаляется из DOM, классы снимаются.

```
              from              active (всё время)            to
enter:   v-enter-from   →   v-enter-active   →   v-enter-to
leave:   v-leave-from   →   v-leave-active   →   v-leave-to
```

`*-active` — единственное место, где задаётся `transition`/`animation` и длительность.

### 3. CSS-анимации (@keyframes) вместо transition

CSS-анимации работают так же, но начальное состояние можно описать прямо в keyframes, поэтому `*-from`/`*-to` часто не нужны.

```vue
<template>
  <Transition name="bounce">
    <p v-if="show">Текст с keyframe-анимацией</p>
  </Transition>
</template>

<style scoped>
.bounce-enter-active {
  animation: bounce-in 0.5s;
}
.bounce-leave-active {
  /* проигрываем ту же анимацию в обратном направлении */
  animation: bounce-in 0.5s reverse;
}

@keyframes bounce-in {
  0%   { transform: scale(0); }
  50%  { transform: scale(1.25); }
  100% { transform: scale(1); }
}
</style>
```

### 4. Кастомные классы переходов

Полезно при интеграции со сторонними CSS-библиотеками (например, Animate.css), у которых свои имена классов.

```vue
<template>
  <Transition
    enter-active-class="animate__animated animate__tada"
    leave-active-class="animate__animated animate__bounceOutRight"
  >
    <p v-if="show">Animate.css</p>
  </Transition>
</template>
```

### 5. Одновременные transition и animation — атрибут `type`

Если на элементе есть и `transition`, и `animation`, Vue не знает, по которому считать длительность. Указываем явно:

```vue
<Transition type="animation">
  <!-- Vue будет ориентироваться на animationend, а не transitionend -->
</Transition>
```

### 6. Явная длительность `:duration`

Vue автоматически определяет окончание по событиям `transitionend`/`animationend`. Но при вложенных переходах (родитель + дети) это может работать неверно — тогда задаём длительность явно (в мс):

```vue
<!-- одинаковая длительность входа и выхода -->
<Transition :duration="550">...</Transition>

<!-- раздельно -->
<Transition :duration="{ enter: 500, leave: 800 }">...</Transition>
```

### 7. JS-хуки переходов

Хуки дают полный программный контроль. Критично: при JS-анимации ставьте `:css="false"`, чтобы Vue не ждал CSS-события, а полностью полагался на ваш вызов `done`.

```vue
<script setup>
// Пример с GSAP (gsap import опущен)
function onBeforeEnter(el) {
  el.style.opacity = 0
  el.style.height = 0
}

// done ОБЯЗАТЕЛЬНО вызвать, иначе переход «зависнет»
function onEnter(el, done) {
  gsap.to(el, {
    opacity: 1,
    height: '1.6em',
    delay: el.dataset.index * 0.15,
    onComplete: done, // сигнал Vue, что enter завершён
  })
}

function onLeave(el, done) {
  gsap.to(el, {
    opacity: 0,
    height: 0,
    delay: el.dataset.index * 0.15,
    onComplete: done,
  })
}
</script>

<template>
  <Transition
    :css="false"
    @before-enter="onBeforeEnter"
    @enter="onEnter"
    @leave="onLeave"
  >
    <div v-if="show">Анимация через JS</div>
  </Transition>
</template>
```

Полный список хуков: `@before-enter`, `@enter(el, done)`, `@after-enter`, `@enter-cancelled`, `@before-leave`, `@leave(el, done)`, `@after-leave`, `@leave-cancelled`.

### 8. Появление при первом рендере — `appear`

По умолчанию `<Transition>` НЕ анимирует элемент, который виден при начальном рендере. Атрибут `appear` включает анимацию входа на старте.

```vue
<Transition appear>
  <p>Анимируется сразу при загрузке</p>
</Transition>
```

Можно задать отдельные классы появления: `appear-class`, `appear-active-class`, `appear-to-class` и хуки `@appear`, `@before-appear`, `@after-appear`.

### 9. Переход между элементами и режимы (`mode`)

Когда два элемента переключаются через `v-if`/`v-else`, по умолчанию входной и выходной элементы анимируются **одновременно**, что вызывает «скачок» (оба занимают место). Решение — `mode`:

- `mode="out-in"` — сначала уходит старый, потом появляется новый (самый частый).
- `mode="in-out"` — сначала появляется новый, потом уходит старый.

```vue
<Transition name="slide" mode="out-in">
  <button v-if="isEditing" key="save" @click="isEditing = false">
    Сохранить
  </button>
  <button v-else key="edit" @click="isEditing = true">
    Редактировать
  </button>
</Transition>
```

Переход между **динамическими компонентами** работает так же — по смене `:is` (рекомендуется уникальный `key`):

```vue
<Transition name="fade" mode="out-in">
  <component :is="currentTab" :key="currentTab" />
</Transition>
```

### 10. `<TransitionGroup>` для списков

Отличия от `<Transition>`:

- Рендерит **реальный обёрточный элемент** (по умолчанию ничего не рендерит как тег, нужно указать `tag`, например `tag="ul"`).
- Не поддерживает `mode`.
- Каждый ребёнок **обязан** иметь уникальный `key`.
- CSS-классы переходов вешаются на **дочерние элементы**, а не на контейнер.
- Поддерживает **перемещение** через класс `*-move` (FLIP-анимация).

```vue
<script setup>
import { ref } from 'vue'

const items = ref([1, 2, 3, 4, 5])
let next = 6

function shuffle() {
  items.value = items.value
    .map(v => ({ v, r: Math.random() }))
    .sort((a, b) => a.r - b.r)
    .map(o => o.v)
}
function add() {
  const i = Math.floor(Math.random() * (items.value.length + 1))
  items.value.splice(i, 0, next++)
}
function remove(item) {
  items.value = items.value.filter(v => v !== item)
}
</script>

<template>
  <button @click="add">Добавить</button>
  <button @click="shuffle">Перемешать</button>

  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item" @click="remove(item)">
      {{ item }}
    </li>
  </TransitionGroup>
</template>

<style scoped>
.list-enter-active,
.list-leave-active {
  transition: all 0.5s ease;
}
.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(30px);
}

/* Класс move — плавное перемещение элементов при сортировке/вставке (FLIP) */
.list-move {
  transition: transform 0.5s ease;
}

/* ВАЖНО: чтобы уходящий элемент не «толкал» соседей,
   выводим его из потока на время выхода */
.list-leave-active {
  position: absolute;
}
</style>
```

### Что такое FLIP?

FLIP = **F**irst, **L**ast, **I**nvert, **P**lay. Vue замеряет позицию элемента до (First) и после (Last) изменения списка, применяет `transform`, который мгновенно возвращает элемент в старую позицию (Invert), затем убирает transform с анимацией (Play, через класс `*-move`). За счёт `transform` (а не `top`/`left`) анимация GPU-ускоренная и плавная.

### 11. Анимация одиночных элементов внутри группы по `data-index` (стаггер)

```vue
<TransitionGroup
  tag="ul"
  :css="false"
  @before-enter="onBeforeEnter"
  @enter="onEnter"
  @leave="onLeave"
>
  <li v-for="(item, index) in list" :key="item.id" :data-index="index">
    {{ item.text }}
  </li>
</TransitionGroup>
```

## Полный рабочий пример

Уведомления-«тосты»: появляются справа, исчезают, переупорядочиваются.

```vue
<script setup>
import { ref } from 'vue'

let uid = 0
const toasts = ref([])

function notify(text) {
  const id = uid++
  toasts.value.push({ id, text })
  // автозакрытие через 3 секунды
  setTimeout(() => dismiss(id), 3000)
}

function dismiss(id) {
  toasts.value = toasts.value.filter(t => t.id !== id)
}
</script>

<template>
  <button @click="notify('Сообщение ' + (uid + 1))">
    Показать уведомление
  </button>

  <TransitionGroup name="toast" tag="div" class="toast-stack">
    <div
      v-for="toast in toasts"
      :key="toast.id"
      class="toast"
      @click="dismiss(toast.id)"
    >
      {{ toast.text }}
    </div>
  </TransitionGroup>
</template>

<style scoped>
.toast-stack {
  position: fixed;
  top: 16px;
  right: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.toast {
  padding: 12px 16px;
  background: #323232;
  color: #fff;
  border-radius: 6px;
  cursor: pointer;
  min-width: 220px;
}

/* появление/исчезновение */
.toast-enter-active,
.toast-leave-active {
  transition: all 0.4s ease;
}
.toast-enter-from {
  opacity: 0;
  transform: translateX(100%);
}
.toast-leave-to {
  opacity: 0;
  transform: translateX(100%);
}
/* плавный сдвиг оставшихся тостов вверх */
.toast-move {
  transition: transform 0.4s ease;
}
/* уходящий тост убираем из потока, чтобы не «прыгали» соседи */
.toast-leave-active {
  position: absolute;
  width: calc(100% - 32px);
}
</style>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `<Transition>` отличается от `<TransitionGroup>`?**
`<Transition>` оборачивает один элемент/компонент и не рендерит DOM-обёртку. `<TransitionGroup>` анимирует список (`v-for`), рендерит реальный тег (через `tag`), требует `key` у детей, поддерживает анимацию перемещения (`*-move`), но не поддерживает `mode`.

**2. Какие 6 классов навешивает Vue и когда?**
`*-enter-from` (старт входа), `*-enter-active` (всё время входа — здесь transition), `*-enter-to` (конец входа); аналогично `*-leave-from`, `*-leave-active`, `*-leave-to`. `from`→`to` переключаются на следующем кадре.

**3. Как Vue определяет, что анимация закончилась?**
Слушает события `transitionend`/`animationend` на `*-active`-элементе и берёт максимальную длительность. Можно переопределить через `:duration` или указать `type="transition|animation"`.

**4. Зачем `mode="out-in"`?**
При переключении двух элементов по умолчанию они анимируются одновременно и «скачут». `out-in` сначала убирает старый, затем показывает новый; `in-out` — наоборот.

**5. Почему элемент не анимируется при первой загрузке страницы?**
По умолчанию вход при первом рендере не анимируется. Нужен атрибут `appear`.

**6. Зачем нужен `:css="false"` в JS-хуках?**
Чтобы Vue не пытался определять окончание по CSS-событиям, а полностью полагался на ваш вызов `done`. Также это небольшая оптимизация (пропускает работу с CSS-классами).

**7. Что произойдёт, если в `@enter`/`@leave` не вызвать `done`?**
Переход «зависнет»: Vue будет считать его незавершённым, элемент не вставится/не удалится корректно (при `:css="false"`).

**8. Что такое FLIP и зачем класс `*-move`?**
FLIP (First-Last-Invert-Play) — техника плавного перемещения элементов списка через `transform`. Класс `*-move` задаёт `transition` для этого перемещения при сортировке/вставке/удалении.

**9. Почему при удалении элемента из `<TransitionGroup>` соседи «прыгают»?**
Уходящий элемент всё ещё занимает место в потоке во время leave-анимации. Решение — `position: absolute` в `*-leave-active`, чтобы вывести его из потока.

**10. Можно ли использовать `<Transition>` с `v-show`?**
Да. Vue анимирует переключение `display`. Отличие от `v-if`: элемент не удаляется из DOM, лишь скрывается.

**11. Как анимировать переключение динамических компонентов?**
Обернуть `<component :is="cur" :key="cur" />` в `<Transition mode="out-in">`. Уникальный `key` гарантирует пересоздание и срабатывание перехода.

**12. Можно ли подключить GSAP / Web Animations API?**
Да — через JS-хуки (`@enter`, `@leave`) с `:css="false"`. Vue лишь даёт точки входа в жизненный цикл, сама анимация делается библиотекой.

## Подводные камни (gotchas)

- **Несколько корневых узлов в `<Transition>`** — переход не сработает. Должен быть ровно один элемент/компонент; для нескольких используйте `<TransitionGroup>` или объедините в один контейнер.
- **Отсутствие `key`** в `<TransitionGroup>` ломает анимацию входа/выхода и перемещения.
- **Скачки соседей** при удалении — забыли `position: absolute` в `*-leave-active`.
- **`*-move` не работает**, если у уходящих элементов нет `*-leave-active` или они в потоке — Vue не сможет корректно посчитать FLIP.
- **Анимация при первом рендере не идёт** — забыли `appear`.
- **transition + animation одновременно** без `type` — Vue может неверно определить длительность.
- **JS-хуки без `done`** или без `:css="false"` — переход подвисает или дублируется.
- **`mode` в `<TransitionGroup>`** не поддерживается — не пытайтесь его задавать.
- **Условие должно меняться** реактивно: `v-if` на статичном значении ничего не анимирует.
- **transition на `display`** не работает напрямую — поэтому Vue и нужен для управления вставкой/удалением; для `v-show` Vue откладывает смену `display` до конца leave-анимации.

## Лучшие практики

- Задавайте `transition`/`animation` **только** в `*-active`-классах, а начальные/конечные состояния — в `*-from`/`*-to`.
- Анимируйте `opacity` и `transform` — они GPU-ускоряются и не вызывают reflow.
- Для смены двух элементов почти всегда используйте `mode="out-in"`.
- В `<TransitionGroup>` всегда задавайте стабильный `key` (id, а не index).
- Для уходящих из потока элементов — `position: absolute` в leave-active.
- Для сложных JS-анимаций используйте `:css="false"` и не забывайте `done`.
- Выносите длительность в CSS-переменные для единообразия.
- Уважайте `prefers-reduced-motion` — отключайте/упрощайте анимации для пользователей с такой настройкой.

## Шпаргалка

```text
<Transition>            один элемент/компонент, без DOM-обёртки
<TransitionGroup>       список v-for, реальный тег (tag="ul"), нужен key

Классы (префикс v- или name-):
  *-enter-from   старт входа
  *-enter-active всё время входа  ← сюда transition/animation + duration
  *-enter-to     конец входа
  *-leave-from   старт выхода
  *-leave-active всё время выхода
  *-leave-to     конец выхода
  *-move         перемещение в TransitionGroup (FLIP)

Атрибуты <Transition>:
  name           префикс классов
  appear         анимировать при первом рендере
  mode="out-in"  сначала уход, потом вход (для смены элементов)
  type           "transition" | "animation" (приоритет события)
  :duration      число | { enter, leave } в мс
  :css="false"   только JS-хуки

JS-хуки:
  @before-enter, @enter(el, done), @after-enter, @enter-cancelled
  @before-leave, @leave(el, done), @after-leave, @leave-cancelled
  (+ appear-аналоги)

Триггеры: v-if, v-show, <component :is>, смена :key

TransitionGroup leave без прыжков:
  .x-leave-active { position: absolute; }
  .x-move        { transition: transform .4s; }
```
