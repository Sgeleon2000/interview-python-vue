# Vue 3 — Слоты (Slots) — конспект и вопросы

## О чём раздел

Слоты — это механизм передачи **контента** (фрагментов шаблона) от родительского компонента внутрь дочернего. Если props передают данные «сверху вниз» как значения, то слоты передают целые куски разметки, которые дочерний компонент сам решает, где и как отрендерить.

Слоты — основа композиции компонентов во Vue. Без них любой переиспользуемый компонент (модалка, карточка, таблица, список) был бы жёстко зашит и не настраивался бы под конкретное использование. Скоупные слоты (scoped slots) идут ещё дальше и позволяют дочернему компоненту **отдавать данные обратно** в разметку, которую предоставляет родитель — это фундамент для renderless-компонентов и «headless»-библиотек.

В этом разделе: слот по умолчанию, fallback-содержимое, именованные слоты, область видимости, scoped slots (с упором), динамические имена слотов, условный рендеринг через `$slots` и паттерн renderless-компонентов.

## Ключевые концепции

- **Слот (slot)** — «дырка» в шаблоне дочернего компонента (`<slot>`), куда вставляется контент родителя.
- **Слот по умолчанию (default slot)** — безымянный слот, куда попадает весь контент, не привязанный к именованному слоту.
- **Fallback-содержимое** — контент внутри `<slot>...</slot>`, который рендерится, если родитель ничего не передал.
- **Именованные слоты (named slots)** — несколько точек вставки, различаемых по `name`; используются с `v-slot:имя` или сокращением `#имя`.
- **Область видимости (scope)** — содержимое слота компилируется в области видимости **родителя**, а не дочернего компонента. Слот имеет доступ к данным родителя и НЕ имеет доступа к данным дочернего (если только это не scoped slot).
- **Scoped slots (скоупные слоты)** — слот «прокидывает» данные дочернего компонента наружу через props слота: `<slot :item="item" />`, а родитель принимает их: `v-slot="{ item }"`.
- **`$slots`** — объект с переданными слотами; позволяет проверить наличие слота и условно что-то рендерить.
- **Renderless-компонент** — компонент без собственной разметки: вся логика инкапсулирована, а отрисовка целиком делегируется через scoped slot.

## Подробный разбор с примерами кода

### 1. Слот по умолчанию и fallback

Дочерний компонент `BaseButton.vue`:

```vue
<script setup>
// Никакой логики не требуется — компонент просто оборачивает контент
</script>

<template>
  <button class="btn">
    <!-- Слот по умолчанию. Текст внутри — это fallback-содержимое,
         оно покажется, только если родитель НЕ передал контент -->
    <slot>Нажми меня</slot>
  </button>
</template>
```

Использование:

```vue
<template>
  <!-- Передаём свой контент в слот по умолчанию -->
  <BaseButton>Сохранить</BaseButton>

  <!-- Ничего не передаём — увидим fallback "Нажми меня" -->
  <BaseButton />
</template>
```

### 2. Именованные слоты

Когда точек вставки несколько, им дают имена. Слот без имени — это слот с именем `default`.

`BaseCard.vue`:

```vue
<template>
  <div class="card">
    <header class="card__header">
      <!-- Именованный слот "header" -->
      <slot name="header">Заголовок по умолчанию</slot>
    </header>

    <main class="card__body">
      <!-- Слот по умолчанию (name="default") -->
      <slot />
    </main>

    <footer class="card__footer">
      <slot name="footer" />
    </footer>
  </div>
</template>
```

Использование с `v-slot` и сокращением `#`:

```vue
<template>
  <BaseCard>
    <!-- Полная форма -->
    <template v-slot:header>
      <h2>Профиль пользователя</h2>
    </template>

    <!-- Контент без template попадает в слот по умолчанию -->
    <p>Имя: Андрей</p>
    <p>Email: shichkoandrei@gmail.com</p>

    <!-- Сокращённая форма #footer -->
    <template #footer>
      <button>Закрыть</button>
    </template>
  </BaseCard>
</template>
```

Важно: контент, не обёрнутый в `<template #name>`, по умолчанию идёт в default-слот. Но если вы используете именованные слоты явно, лучше и default оборачивать в `<template #default>` для читаемости.

### 3. Область видимости слотов (scope)

Содержимое слота компилируется в области видимости **родителя**. Это значит:

```vue
<script setup>
// Это РОДИТЕЛЬ
const userName = 'Андрей'
</script>

<template>
  <BaseCard>
    <!-- userName доступен, т.к. слот живёт в области родителя -->
    <template #header>Привет, {{ userName }}</template>

    <!-- А вот данные ИЗ BaseCard здесь НЕ доступны напрямую.
         Чтобы их получить — нужны scoped slots (см. ниже) -->
  </BaseCard>
</template>
```

Правило: «контент слота имеет доступ к данным родителя и НЕ имеет доступа к данным дочернего компонента».

### 4. Scoped slots (скоупные слоты) — КЛЮЧЕВАЯ ТЕМА

Иногда содержимому слота нужны данные, которые есть только внутри дочернего компонента (например, элемент при итерации списка). Для этого дочерний компонент **передаёт данные как props на теге `<slot>`** — это называется «slot props». Родитель принимает их через `v-slot`.

Дочерний `MyList.vue`:

```vue
<script setup>
const props = defineProps({
  items: { type: Array, required: true }
})
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="item.id">
      <!--
        Передаём данные ИЗ дочернего НАРУЖУ через атрибуты слота.
        Это slot props: item, index.
        :item="item" — динамический slot prop.
        Атрибут name зарезервирован под имя слота и в slot props не попадает.
      -->
      <slot :item="item" :index="index">
        <!-- fallback на случай, если родитель не описал содержимое -->
        {{ item.title }}
      </slot>
    </li>
  </ul>
</template>
```

Родитель принимает slot props через деструктуризацию:

```vue
<script setup>
const products = [
  { id: 1, title: 'Клавиатура', price: 3000 },
  { id: 2, title: 'Мышь', price: 1500 }
]
</script>

<template>
  <MyList :items="products">
    <!-- v-slot="..." принимает объект всех slot props default-слота.
         Деструктурируем item и index -->
    <template #default="{ item, index }">
      <strong>{{ index + 1 }}.</strong>
      {{ item.title }} — {{ item.price }} ₽
    </template>
  </MyList>
</template>
```

Если используется только слот по умолчанию, можно писать `v-slot` прямо на компоненте без `<template>`:

```vue
<template>
  <MyList :items="products" v-slot="{ item }">
    {{ item.title }}
  </MyList>
</template>
```

Внимание: такая краткая запись работает **только когда есть единственный default-слот**. При наличии именованных слотов — обязательно `<template>`.

#### Как это работает «под капотом»

Скоупный слот фактически передаётся в дочерний компонент как **функция**. Дочерний компонент вызывает её, передавая данные (slot props), и получает обратно VNode'ы для рендера. Поэтому slot props реактивны: при изменении `item` дочерний перерендерит слот с новыми данными.

```js
// Концептуально (render-функция):
// в дочернем: this.$slots.default({ item, index })
// родитель передал: { default: ({ item, index }) => [/* vnodes */] }
```

### 5. Именованные scoped slots

Можно совмещать имя слота и slot props:

```vue
<!-- DataTable.vue (дочерний) -->
<template>
  <table>
    <thead>
      <tr>
        <th v-for="col in columns" :key="col.key">
          <!-- Скоупный слот для заголовка колонки -->
          <slot name="header" :column="col">{{ col.label }}</slot>
        </th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="row in rows" :key="row.id">
        <td v-for="col in columns" :key="col.key">
          <!-- Именованный скоупный слот для ячейки -->
          <slot name="cell" :row="row" :column="col">
            {{ row[col.key] }}
          </slot>
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script setup>
defineProps({ columns: Array, rows: Array })
</script>
```

Использование:

```vue
<template>
  <DataTable :columns="columns" :rows="rows">
    <!-- Имя слота + деструктуризация slot props -->
    <template #header="{ column }">
      <span class="th">{{ column.label.toUpperCase() }}</span>
    </template>

    <template #cell="{ row, column }">
      <em v-if="column.key === 'price'">{{ row.price }} ₽</em>
      <span v-else>{{ row[column.key] }}</span>
    </template>
  </DataTable>
</template>
```

### 6. Динамические имена слотов

Имя слота можно вычислять во время выполнения:

```vue
<template>
  <BaseLayout>
    <!-- Динамическое имя слота через [выражение] -->
    <template #[dynamicSlotName]>
      Контент попадёт в слот, имя которого в переменной
    </template>
  </BaseLayout>
</template>

<script setup>
import { ref } from 'vue'
const dynamicSlotName = ref('header') // можно переключать
</script>
```

В дочернем компоненте имя слота тоже бывает динамическим:

```vue
<template>
  <component :is="tag">
    <slot :name="slotName" />
  </component>
</template>
```

### 7. Условные слоты через `$slots`

`$slots` (в template доступно как `$slots`, в `<script setup>` — через `useSlots()`) содержит переданные слоты. По наличию ключа можно условно рендерить обёртку, чтобы не было пустых элементов в DOM:

```vue
<script setup>
import { useSlots, computed } from 'vue'

const slots = useSlots()

// Реактивно проверяем, передал ли родитель слот "footer"
const hasFooter = computed(() => !!slots.footer)
</script>

<template>
  <div class="card">
    <div class="card__body"><slot /></div>

    <!-- Рендерим footer-обёртку, только если слот передан -->
    <footer v-if="hasFooter" class="card__footer">
      <slot name="footer" />
    </footer>

    <!-- В template можно и напрямую через $slots -->
    <header v-if="$slots.header">
      <slot name="header" />
    </header>
  </div>
</template>
```

### 8. Renderless-компоненты (паттерн)

Renderless-компонент не рендерит собственной разметки — он инкапсулирует логику (состояние, эффекты) и отдаёт её наружу через scoped slot. Вся отрисовка остаётся за потребителем.

`MouseTracker.vue` (renderless):

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const x = ref(0)
const y = ref(0)

function update(e) {
  x.value = e.clientX
  y.value = e.clientY
}

onMounted(() => window.addEventListener('mousemove', update))
onUnmounted(() => window.removeEventListener('mousemove', update))
</script>

<template>
  <!-- Никакой своей разметки: отдаём данные в default-слот.
       Контейнера тоже нет — слот рендерится «как есть» -->
  <slot :x="x" :y="y" />
</template>
```

Использование — отрисовку определяет потребитель:

```vue
<template>
  <MouseTracker v-slot="{ x, y }">
    <p>Координаты курсора: {{ x }}, {{ y }}</p>
  </MouseTracker>
</template>
```

Замечание для senior: в Vue 3 во многих случаях renderless-компоненты заменяются **composables** (`useMouse()`), потому что логику удобнее переиспользовать через функции, чем через компоненты. Renderless остаётся уместным, когда нужно жёстко связать логику с её представлением через шаблон или предоставить «слотовый» API библиотеки.

## Полный рабочий пример

Универсальный компонент «модальное окно» с default/header/footer слотами, fallback'ами, условным рендерингом footer и скоупным слотом, отдающим функцию закрытия.

```vue
<!-- BaseModal.vue -->
<script setup>
import { useSlots, computed } from 'vue'

const props = defineProps({
  modelValue: { type: Boolean, default: false },
  title: { type: String, default: '' }
})
const emit = defineEmits(['update:modelValue'])

const slots = useSlots()
const hasFooter = computed(() => !!slots.footer)

function close() {
  emit('update:modelValue', false)
}
</script>

<template>
  <Teleport to="body">
    <div v-if="modelValue" class="modal__overlay" @click.self="close">
      <div class="modal" role="dialog" aria-modal="true">
        <header class="modal__header">
          <!-- Именованный слот header со scoped-данными (отдаём close) -->
          <slot name="header" :close="close">
            <h3>{{ title || 'Без заголовка' }}</h3>
          </slot>
        </header>

        <section class="modal__body">
          <!-- Default-слот тоже скоупный: пробрасываем close внутрь -->
          <slot :close="close">Содержимое отсутствует</slot>
        </section>

        <!-- Footer рендерим только если он передан -->
        <footer v-if="hasFooter" class="modal__footer">
          <slot name="footer" :close="close" />
        </footer>
      </div>
    </div>
  </Teleport>
</template>
```

```vue
<!-- App.vue (потребитель) -->
<script setup>
import { ref } from 'vue'
import BaseModal from './BaseModal.vue'

const open = ref(false)
</script>

<template>
  <button @click="open = true">Открыть модалку</button>

  <BaseModal v-model="open" title="Подтверждение">
    <!-- header переопределяем и используем slot prop close -->
    <template #header="{ close }">
      <h3>Удалить элемент?</h3>
      <button class="modal__x" @click="close">×</button>
    </template>

    <!-- default-слот -->
    <template #default>
      <p>Это действие нельзя отменить.</p>
    </template>

    <!-- footer со scoped close -->
    <template #footer="{ close }">
      <button @click="close">Отмена</button>
      <button class="danger" @click="close">Удалить</button>
    </template>
  </BaseModal>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. В какой области видимости компилируется содержимое слота — родителя или дочернего?**
В области видимости **родителя**. Контент слота имеет доступ к данным/методам родителя, но не к внутреннему состоянию дочернего компонента — за исключением данных, которые дочерний явно отдаёт через scoped slot props.

**2. Что такое scoped slots и зачем они нужны?**
Это слоты, через которые дочерний компонент передаёт свои данные наружу — в содержимое слота, определённое родителем. Дочерний делает `<slot :item="item" />`, родитель принимает `v-slot="{ item }"`. Нужны, когда родителю нужно отрисовать что-то, используя данные, которые есть только внутри дочернего (например, текущий элемент списка, состояние загрузки и т. п.).

**3. Как технически реализованы scoped slots?**
Скоупный слот передаётся в дочерний компонент как **функция**, принимающая объект slot props и возвращающая VNode'ы. Дочерний вызывает её с актуальными данными в нужном месте рендера. Отсюда реактивность: при изменении данных функция вызывается заново.

**4. Чем `#name` отличается от `v-slot:name`?**
Ничем по смыслу — `#` это сокращённый синтаксис `v-slot:`. `#default` === `v-slot:default`.

**5. Можно ли использовать `v-slot` без `<template>`?**
Да, но только если используется единственный слот по умолчанию: `<Comp v-slot="{ x }">...</Comp>`. При наличии именованных слотов краткая форма для default запрещена — нужно оборачивать каждый слот в `<template>`.

**6. Как условно отрендерить обёртку, только если слот передан?**
Через `$slots` (в template) или `useSlots()` (в setup): `v-if="$slots.footer"`. Это позволяет не рендерить пустые DOM-узлы.

**7. Что такое fallback-содержимое слота?**
Контент между `<slot>` и `</slot>`. Он рендерится, если родитель не передал ничего в этот слот. Если передал — fallback заменяется.

**8. Что такое renderless-компонент?**
Компонент без собственной разметки: вся логика инкапсулирована внутри, а отрисовка делегируется наружу через scoped slot (`<slot :data="..." />`). В Vue 3 часто заменяется composables.

**9. Куда попадает контент, не обёрнутый в `<template #name>`?**
В слот по умолчанию (`default`). Можно смешивать неименованный контент и именованные `<template>`, но для ясности default тоже лучше оборачивать.

**10. Как сделать динамическое имя слота?**
Через синтаксис `#[выражение]`: `<template #[name]>`, где `name` — реактивная переменная.

**11. Можно ли передавать в слот несколько именованных scoped-слотов одновременно?**
Да. Каждый именованный слот имеет свой набор slot props: `<slot name="cell" :row="row" />`, родитель: `<template #cell="{ row }">`.

**12. Реактивны ли slot props?**
Да. Поскольку слот — это функция, вызываемая дочерним компонентом при рендере, при изменении переданных значений слот перерисовывается с новыми данными.

**13. Чем слоты отличаются от props?**
Props передают **значения данных** сверху вниз. Слоты передают **фрагменты разметки** (VNode-фабрики). Scoped slots при этом ещё и возвращают данные снизу вверх. Это разные оси композиции.

**14. Почему предпочитают composables вместо renderless-компонентов в Vue 3?**
Composables (`useX()`) дают переиспользуемую логику без накладных расходов на лишний компонент в дереве, без лишних слотов и проще типизируются. Renderless оставляют там, где нужен слотовый API или жёсткая связка логики с шаблоном.

## Подводные камни (gotchas)

- **Доступ к данным дочернего без scoped slot.** Частая ошибка — ожидать, что в слоте доступны внутренние переменные дочернего. Нет: нужны slot props.
- **Краткий `v-slot` на компоненте при наличии именованных слотов.** Vue выдаст ошибку: смешивать default-сокращение и именованные слоты нельзя — оборачивайте всё в `<template>`.
- **`$slots.foo` это функция, а не булево «есть/нет».** Проверка `!!$slots.foo` проверяет наличие функции слота, а не наличие реального контента. Если слот передан, но рендерит пустоту (например, `v-if` внутри), `$slots.foo` всё равно truthy.
- **Деструктуризация slot props ломает реактивность только при неправильном использовании в setup.** В шаблоне `v-slot="{ item }"` деструктуризация безопасна — это аргументы функции рендера, вызываемой заново.
- **Имя `default`.** Безымянный слот — это `default`. `$slots.default` всегда существует, если хоть какой-то контент передан в тело.
- **Зарезервированные атрибуты.** Атрибут `name` на `<slot>` — это имя слота, он не попадает в slot props. Все остальные атрибуты становятся slot props.
- **Слоты и `v-for`.** Если генерируете слоты в цикле, следите за ключами и динамическими именами, иначе возможна путаница в сопоставлении.

## Лучшие практики

- Давайте слотам **осмысленные имена** (`header`, `footer`, `actions`), а не `slot1`.
- Всегда продумывайте **fallback-содержимое** для гибкости и graceful degradation.
- Для условных обёрток используйте `$slots` / `useSlots()` + `computed`, чтобы не плодить пустые DOM-узлы.
- Документируйте **контракт slot props** — какие данные дочерний отдаёт наружу (особенно важно для библиотечных компонентов).
- Для переиспользуемой **логики** предпочитайте composables; renderless-компоненты — для слотового API.
- Используйте сокращение `#name` для читаемости; для default явно пишите `#default`, когда есть и другие слоты.
- Прокидывайте через scoped slots не только данные, но и **действия** (функции вроде `close`, `submit`) — это делает компонент по-настоящему гибким.

## Шпаргалка

```vue
<!-- ДОЧЕРНИЙ -->
<slot />                          <!-- default слот -->
<slot>fallback</slot>             <!-- fallback-содержимое -->
<slot name="header" />            <!-- именованный слот -->
<slot :item="item" :i="i" />      <!-- scoped: отдаём данные наружу -->
<slot name="cell" :row="row" />   <!-- именованный + scoped -->

<!-- РОДИТЕЛЬ -->
<Comp>контент</Comp>                          <!-- в default -->
<template #header>...</template>              <!-- именованный (# = v-slot:) -->
<template #default="{ item, i }">...</template> <!-- scoped, деструктуризация -->
<Comp v-slot="{ item }">...</Comp>            <!-- краткий default scoped -->
<template #[name]>...</template>              <!-- динамическое имя -->

<!-- ПРОВЕРКА НАЛИЧИЯ -->
v-if="$slots.footer"                          <!-- в template -->
const slots = useSlots(); !!slots.footer      <!-- в setup -->
```

Краткие правила:
- Слот компилируется в области **родителя**.
- Scoped slot = слот передаёт данные **из дочернего наружу** (через slot props).
- `#name` = `v-slot:name`. Краткий `v-slot` на компоненте — только для одиночного default.
- `$slots` — для условного рендеринга обёрток.
- Renderless = логика внутри, рендер снаружи через scoped slot; часто заменяется composables.
