# Vue 3 — Встроенные компоненты: KeepAlive, Teleport, Suspense — конспект и вопросы

## О чём раздел

Три встроенных компонента Vue 3, решающих частые архитектурные задачи:

- **`<KeepAlive>`** — кеширует экземпляры компонентов, чтобы при переключении не уничтожать и не пересоздавать их (сохраняется состояние, прокрутка, введённые данные). Кейсы: табы, многошаговые формы, вкладки в дашборде.
- **`<Teleport>`** — телепортирует часть шаблона в другое место DOM (например, в конец `<body>`), сохраняя логическую связь с родителем. Кейсы: модалки, тултипы, попапы, уведомления — всё, что должно вырваться из `overflow`/`z-index`-контекста родителя.
- **`<Suspense>`** — координирует ожидание асинхронных зависимостей (async `setup`, асинхронные компоненты), показывая запасной контент (`#fallback`) до готовности. **Статус: экспериментальный** — API может измениться.

## Ключевые концепции

### KeepAlive
- По умолчанию динамический `<component :is>` уничтожает старый компонент при переключении. `<KeepAlive>` его кеширует.
- Хуки жизненного цикла: `onActivated` (вход из кеша) и `onDeactivated` (уход в кеш) вместо/в дополнение к `onMounted`/`onUnmounted`.
- `include` / `exclude` (строка, RegExp, массив) — что кешировать; матчатся по `name` компонента.
- `max` — максимум кешируемых экземпляров (LRU-вытеснение).

### Teleport
- `to` — CSS-селектор или DOM-элемент-цель.
- Логически содержимое остаётся ребёнком исходного компонента (доступ к данным, событиям, provide/inject), но физически рендерится в цель.
- `disabled` — временно отключить телепортацию (рендер на месте).
- Несколько `<Teleport>` на одну цель добавляются по порядку (append).

### Suspense
- Слоты `#default` (основной контент) и `#fallback` (заглушка на время загрузки).
- Реагирует на: компоненты с `async setup()` и асинхронные компоненты (`defineAsyncComponent`).
- События: `@pending`, `@resolve`, `@fallback`.
- Только **первый** уровень async-зависимостей под `<Suspense>` ожидается совместно.

## Подробный разбор с примерами кода

### 1. KeepAlive — базовое кеширование

```vue
<script setup>
import { ref, shallowRef } from 'vue'
import TabPosts from './TabPosts.vue'
import TabArchive from './TabArchive.vue'

const tabs = { TabPosts, TabArchive }
const current = shallowRef(TabPosts)
</script>

<template>
  <button @click="current = tabs.TabPosts">Посты</button>
  <button @click="current = tabs.TabArchive">Архив</button>

  <!-- без KeepAlive каждый клик пересоздаёт компонент и теряет его состояние -->
  <KeepAlive>
    <component :is="current" />
  </KeepAlive>
</template>
```

### 2. include / exclude / max

`include`/`exclude` матчатся по опции `name` компонента. В `<script setup>` имя выводится из имени файла, но для надёжности задают явно через отдельный блок:

```vue
<!-- внутри кешируемого компонента -->
<script>
// явное имя для include/exclude и DevTools
export default { name: 'TabPosts' }
</script>

<script setup>
// логика компонента
</script>
```

```vue
<!-- Кешировать только указанные -->
<KeepAlive :include="['TabPosts', 'TabArchive']">
  <component :is="current" />
</KeepAlive>

<!-- Кешировать всё, кроме... (можно RegExp) -->
<KeepAlive :exclude="/Settings$/">
  <component :is="current" />
</KeepAlive>

<!-- Ограничить кеш (LRU): хранится максимум 5 экземпляров -->
<KeepAlive :max="5">
  <component :is="current" />
</KeepAlive>
```

### 3. Хуки onActivated / onDeactivated

При кешировании `onMounted`/`onUnmounted` НЕ вызываются на переключение (компонент не уничтожается). Используйте `onActivated`/`onDeactivated` для логики «при показе/скрытии».

```vue
<script setup>
import { onActivated, onDeactivated, onMounted } from 'vue'

onMounted(() => {
  // вызовется ОДИН раз при первом создании
  console.log('mounted')
})

onActivated(() => {
  // при каждом возврате компонента из кеша
  console.log('activated — например, обновить данные')
})

onDeactivated(() => {
  // при уходе в кеш — например, поставить опрос/таймер на паузу
  console.log('deactivated')
})
</script>
```

Важно: `onActivated` вызывается и при первом монтировании (после `onMounted`), и при каждом восстановлении из кеша.

### 4. Teleport — базовый телепорт

```vue
<template>
  <button @click="open = true">Открыть</button>

  <!-- Содержимое физически окажется в конце <body>,
       но логически остаётся в этом компоненте -->
  <Teleport to="body">
    <div v-if="open" class="overlay">
      <div class="dialog">
        <p>Я отрендерен в body, но управляюсь отсюда: {{ message }}</p>
        <button @click="open = false">Закрыть</button>
      </div>
    </div>
  </Teleport>
</template>
```

Зачем телепортировать в `body`: модалка с вложенностью внутри карточки с `overflow: hidden`, `transform` или низким `z-index` будет обрезана/перекрыта. Телепорт в `body` выносит её из этих контекстов наложения.

### 5. Teleport: `disabled` и несколько на одну цель

```vue
<script setup>
import { ref } from 'vue'
const fullscreen = ref(false)
</script>

<template>
  <!-- disabled=true => рендер на месте; false => в body.
       Удобно для видеоплеера: на месте / в полноэкранный контейнер -->
  <Teleport to="#fs-target" :disabled="!fullscreen">
    <video-player />
  </Teleport>
</template>
```

```vue
<!-- Несколько Teleport в одну цель добавляются по порядку (append) -->
<Teleport to="#modals"><ModalA /></Teleport>
<Teleport to="#modals"><ModalB /></Teleport>
<!-- В #modals будет: ModalA, затем ModalB -->
```

### 6. Suspense + async setup

Компонент с `async setup()` (или `<script setup>` с `await` на верхнем уровне) приостанавливает рендер, пока промис не разрешится.

```vue
<!-- AsyncProfile.vue -->
<script setup>
// await на верхнем уровне делает setup асинхронным
const res = await fetch('/api/profile')
const profile = await res.json()
</script>

<template>
  <div>{{ profile.name }}</div>
</template>
```

```vue
<!-- Родитель -->
<template>
  <Suspense>
    <template #default>
      <AsyncProfile />
    </template>

    <template #fallback>
      <div>Загрузка профиля…</div>
    </template>
  </Suspense>
</template>
```

### 7. Suspense + асинхронные компоненты

```vue
<script setup>
import { defineAsyncComponent } from 'vue'

const Dashboard = defineAsyncComponent(() =>
  import('./Dashboard.vue')
)
</script>

<template>
  <Suspense>
    <Dashboard />
    <template #fallback>Загрузка дашборда…</template>
  </Suspense>
</template>
```

### 8. События Suspense и обработка ошибок

```vue
<script setup>
import { ref, onErrorCaptured } from 'vue'

const error = ref(null)

// Suspense НЕ ловит ошибки сам — используем onErrorCaptured
onErrorCaptured((e) => {
  error.value = e
  return false // остановить всплытие
})
</script>

<template>
  <div v-if="error">Ошибка загрузки: {{ error.message }}</div>
  <Suspense
    v-else
    @pending="/* начало ожидания */"
    @resolve="/* всё разрешилось */"
    @fallback="/* показан fallback */"
  >
    <AsyncView />
    <template #fallback>Загрузка…</template>
  </Suspense>
</template>
```

### 9. Комбинация Suspense + Transition + KeepAlive (порядок вложенности)

Рекомендуемый порядок при роутинге с RouterView:

```vue
<Router-View v-slot="{ Component }">
  <template v-if="Component">
    <Transition mode="out-in">
      <KeepAlive>
        <Suspense>
          <component :is="Component" />
          <template #fallback>Загрузка…</template>
        </Suspense>
      </KeepAlive>
    </Transition>
  </template>
</Router-View>
```

## Полный рабочий пример

### Модалка через Teleport (с фокус-ловушкой эскейпа) + табы через KeepAlive

```vue
<!-- BaseModal.vue -->
<script setup>
import { watch, onUnmounted } from 'vue'

const props = defineProps({
  modelValue: { type: Boolean, default: false },
})
const emit = defineEmits(['update:modelValue'])

function close() {
  emit('update:modelValue', false)
}

function onKeydown(e) {
  if (e.key === 'Escape') close()
}

// блокируем прокрутку body и слушаем Escape пока модалка открыта
watch(
  () => props.modelValue,
  (open) => {
    document.body.style.overflow = open ? 'hidden' : ''
    if (open) document.addEventListener('keydown', onKeydown)
    else document.removeEventListener('keydown', onKeydown)
  }
)

onUnmounted(() => {
  document.body.style.overflow = ''
  document.removeEventListener('keydown', onKeydown)
})
</script>

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="modelValue" class="modal-overlay" @click.self="close">
        <div class="modal-box" role="dialog" aria-modal="true">
          <header class="modal-header">
            <slot name="header">Заголовок</slot>
            <button class="modal-close" @click="close">×</button>
          </header>
          <div class="modal-body">
            <slot />
          </div>
          <footer class="modal-footer">
            <slot name="footer">
              <button @click="close">Закрыть</button>
            </slot>
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal-box {
  background: #fff;
  border-radius: 8px;
  width: min(480px, 90vw);
  padding: 20px;
}
.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.modal-close {
  border: none;
  background: none;
  font-size: 24px;
  cursor: pointer;
}
/* анимация появления */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.25s ease;
}
.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}
</style>
```

```vue
<!-- App.vue: табы с KeepAlive + модалка -->
<script setup>
import { ref, shallowRef } from 'vue'
import BaseModal from './BaseModal.vue'
import TabForm from './TabForm.vue'
import TabStats from './TabStats.vue'

const tabs = { TabForm, TabStats }
const currentName = ref('TabForm')
const current = shallowRef(TabForm)

function switchTab(name) {
  currentName.value = name
  current.value = tabs[name]
}

const showModal = ref(false)
</script>

<template>
  <nav>
    <button
      :class="{ active: currentName === 'TabForm' }"
      @click="switchTab('TabForm')"
    >
      Форма
    </button>
    <button
      :class="{ active: currentName === 'TabStats' }"
      @click="switchTab('TabStats')"
    >
      Статистика
    </button>
  </nav>

  <!-- KeepAlive сохраняет введённые в форму данные при переключении вкладок -->
  <KeepAlive :max="10">
    <component :is="current" />
  </KeepAlive>

  <button @click="showModal = true">Открыть модалку</button>

  <BaseModal v-model="showModal">
    <template #header>Подтверждение</template>
    <p>Содержимое модального окна (отрендерено в body через Teleport).</p>
  </BaseModal>
</template>
```

```vue
<!-- TabForm.vue: благодаря KeepAlive введённый текст не теряется -->
<script>
export default { name: 'TabForm' } // имя для include/exclude и DevTools
</script>

<script setup>
import { ref, onActivated, onDeactivated } from 'vue'

const text = ref('')

onActivated(() => console.log('Форма показана из кеша'))
onDeactivated(() => console.log('Форма ушла в кеш, данные сохранены'))
</script>

<template>
  <form>
    <label>Имя: <input v-model="text" /></label>
    <p>Текущее значение: {{ text }}</p>
  </form>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Зачем нужен `<KeepAlive>` и что он сохраняет?**
Кеширует экземпляр компонента вместо уничтожения при переключении (динамический `:is`, RouterView). Сохраняются реактивное состояние, введённые данные, позиция прокрутки. Полезно для табов, форм, тяжёлых компонентов.

**2. Какие хуки жизненного цикла появляются с KeepAlive и почему нельзя полагаться на `onUnmounted`?**
`onActivated` (показ из кеша) и `onDeactivated` (уход в кеш). При кешировании компонент не уничтожается, поэтому `onMounted`/`onUnmounted` срабатывают только на реальное создание/удаление, а не на переключение.

**3. По чему матчатся `include`/`exclude`?**
По опции `name` компонента. В `<script setup>` имя выводится из файла, но для надёжности задают через дополнительный `<script>` с `export default { name: '...' }`.

**4. Что делает `max` в KeepAlive?**
Ограничивает число кешированных экземпляров; при превышении вытесняется наименее недавно использованный (LRU).

**5. Зачем `<Teleport>`, если можно просто положить модалку в шаблон?**
Чтобы вырваться из контекста наложения родителя: `overflow: hidden`, `transform`, `z-index`, `position`. Модалки/тултипы телепортируют в `body`, сохраняя при этом логическую связь (props, события, provide/inject) с исходным компонентом.

**6. Сохраняется ли реактивность и provide/inject после Teleport?**
Да. Телепортируется только физическое положение в DOM. Компонентная иерархия (родитель/потомок, контекст inject, события) остаётся прежней.

**7. Что делает `disabled` у Teleport и зачем `to` может быть элементом, а не селектором?**
`disabled` рендерит содержимое на месте (полезно для полноэкранного режима плеера). `to` принимает CSS-селектор или прямую ссылку на DOM-элемент.

**8. Несколько `<Teleport>` в одну цель — что произойдёт?**
Их содержимое добавляется в цель по порядку объявления (append), не перезаписывая друг друга.

**9. Что решает `<Suspense>` и на какие зависимости реагирует?**
Координирует ожидание async-зависимостей: компонентов с `async setup()`/верхнеуровневым `await` и асинхронных компонентов (`defineAsyncComponent`). До готовности показывает `#fallback`, потом `#default`.

**10. Ловит ли Suspense ошибки загрузки?**
Нет, сам не ловит. Ошибки обрабатываются через `onErrorCaptured` в родителе или `errorComponent` в `defineAsyncComponent`. Suspense лишь управляет fallback-состоянием.

**11. Почему важен статус Suspense как «экспериментального»?**
API может измениться между версиями. В продакшене стоит закладывать риск изменений; часто async-данные грузят и без Suspense (через локальное состояние loading).

**12. Какой рекомендуемый порядок вложенности Transition / KeepAlive / Suspense?**
`Transition` снаружи, затем `KeepAlive`, затем `Suspense`, внутри — `component`. Так переход анимирует, кеш сохраняет состояние, а Suspense управляет загрузкой.

**13. Когда `onActivated` вызывается в первый раз?**
При первом монтировании (сразу после `onMounted`) и далее при каждом восстановлении из кеша.

## Подводные камни (gotchas)

### KeepAlive
- `include`/`exclude` не работают без заданного `name` компонента — типичная ловушка с `<script setup>`.
- В кешированных компонентах таймеры/подписки/опрос нужно ставить на паузу в `onDeactivated` и возобновлять в `onActivated`, иначе они продолжают работать в фоне.
- `KeepAlive` ожидает **один** прямой дочерний компонент. Несколько детей одновременно не кешируются.
- Память: без `max` кеш может неограниченно расти.

### Teleport
- **Цель должна существовать в DOM до монтирования** `<Teleport>`. Если цель рендерится самим Vue позже, телепорт не найдёт её. Цель вроде `body` или статичного `#app`-соседа безопасна.
- Стили `scoped` родителя применяются к телепортированному содержимому (атрибуты data-v- сохраняются), но контекстные CSS-селекторы родителя — нет.
- SSR: телепорт в произвольный элемент при гидрации требует, чтобы цель присутствовала и на сервере.

### Suspense
- Экспериментальный API.
- Только **первый** async-узел под Suspense ожидается; вложенные async-компоненты глубже создают собственные границы.
- Без обработки ошибок зависший/упавший промис оставит fallback навсегда.
- `async setup()` означает, что компонент ДОЛЖЕН быть под `<Suspense>` (иначе предупреждение/не отрендерится корректно).

## Лучшие практики

- **KeepAlive**: задавайте явный `name`, используйте `onActivated`/`onDeactivated` для управления ресурсами, ставьте `max` для контроля памяти.
- **Teleport**: для модалок/тултипов телепортируйте в `body` или выделенный контейнер `#modals`; сохраняйте доступность (`role="dialog"`, `aria-modal`, ловушка фокуса, Escape).
- Оборачивайте Teleport-содержимое в `<Transition>` для плавного появления.
- **Suspense**: предусматривайте обработку ошибок (`onErrorCaptured` / `errorComponent`) и осмысленный fallback (скелетоны вместо «Загрузка…»).
- Соблюдайте порядок Transition > KeepAlive > Suspense в роутинге.
- Блокируйте прокрутку `body` при открытой модалке и восстанавливайте при закрытии/размонтировании.

## Шпаргалка

```text
<KeepAlive>
  кеширует экземпляр (не уничтожает)
  props: include / exclude (string | RegExp | array, по name),
         max (LRU)
  хуки:  onActivated()   — показ из кеша (и первый mount)
         onDeactivated() — уход в кеш
  один прямой дочерний компонент

<Teleport to="body" :disabled="bool">
  переносит DOM физически, сохраняет логическую иерархию
  to: CSS-селектор | DOM-элемент (должен существовать заранее)
  несколько на одну цель => append по порядку
  кейсы: модалки, тултипы, попапы, уведомления

<Suspense>  (ЭКСПЕРИМЕНТАЛЬНЫЙ)
  слоты: #default, #fallback
  ждёт: async setup() / await на верхнем уровне,
        defineAsyncComponent
  события: @pending, @resolve, @fallback
  ошибки: НЕ ловит сам => onErrorCaptured / errorComponent
  ждёт только первый уровень async-зависимостей

Порядок при роутинге:
  Transition > KeepAlive > Suspense > component
```
