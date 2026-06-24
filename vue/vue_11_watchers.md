# Vue 3 — Наблюдатели (watch / watchEffect) — конспект и вопросы

## О чём раздел

Наблюдатели — это API для запуска **побочных эффектов в ответ на изменения реактивного состояния**: запрос к API при смене id, валидация, синхронизация с localStorage, ручные операции с DOM. В отличие от `computed`, который вычисляет производное значение синхронно и без сайд-эффектов, наблюдатель предназначен именно для «делать что-то», когда данные изменились.

Во Vue 3 два основных API: `watch()` — следит за **явно указанным** источником и даёт старое/новое значение; `watchEffect()` — **автоматически** собирает зависимости из тела функции и сразу же запускается.

## Ключевые концепции

- **`watch(source, cb)`** — ленивый по умолчанию (срабатывает только при изменении), даёт `(newValue, oldValue)`, источник задаётся явно.
- **`watchEffect(fn)`** — запускается сразу и пересобирает зависимости при каждом проходе; нет старого значения.
- **Источники `watch`**: `ref`, геттер `() => ...`, реактивный объект (`reactive`), массив из нескольких источников.
- **`deep`** — глубокое наблюдение; для `reactive`-объекта включается автоматически.
- **`immediate`** — запустить колбэк сразу при создании.
- **`once`** (3.4+) — сработать ровно один раз.
- **`flush`** — момент срабатывания: `pre` (по умолчанию, до рендера), `post` (после обновления DOM), `sync` (синхронно).
- **Очистка эффектов** — `onCleanup` (аргумент колбэка) / `onWatcherCleanup` (импорт, 3.5+): отменяет устаревшие запросы.
- **Остановка**: `watch`/`watchEffect` возвращают функцию `stop()`.

## Подробный разбор с примерами кода

### `watch()` — источники

```vue
<script setup>
import { ref, reactive, watch } from 'vue'

const count = ref(0)
const user = reactive({ name: 'Аня', age: 30 })

// 1) Источник — ref
watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} -> ${newVal}`)
})

// 2) Источник — геттер (следим за вычисляемым выражением / отдельным свойством)
watch(
  () => user.age,
  (age) => console.log('возраст изменился:', age)
)

// 3) Источник — реактивный объект (deep включается АВТОМАТИЧЕСКИ)
watch(user, (newUser) => {
  // сработает при изменении любого свойства user
  console.log('user изменился', newUser)
})

// 4) Массив источников — несколько сразу
watch([count, () => user.name], ([c, name], [oldC, oldName]) => {
  console.log('что-то из [count, name] изменилось')
})
</script>
```

> ВАЖНО: нельзя следить напрямую за «голым» свойством реактивного объекта — `watch(user.age, ...)` передаст число, а не источник. Используйте геттер: `watch(() => user.age, ...)`.

### `watchEffect()` — автосбор зависимостей

```vue
<script setup>
import { ref, watchEffect } from 'vue'

const id = ref(1)
const data = ref(null)

// Запускается СРАЗУ. Vue сам видит, что внутри читается id.value,
// и перезапустит эффект при его изменении.
watchEffect(async () => {
  const res = await fetch(`/api/items/${id.value}`)
  data.value = await res.json()
})
// Менять id.value — эффект перезапустится автоматически.
</script>
```

Особенность: `watchEffect` отслеживает только те зависимости, что были **прочитаны синхронно** до первого `await`. Зависимости, прочитанные после `await`, не отслеживаются.

```js
watchEffect(async () => {
  const a = first.value          // отслеживается
  await someAsync()
  const b = second.value         // НЕ отслеживается (после await)
})
```

### Отличия `watch` vs `watchEffect` (ключевой вопрос)

| Критерий                | `watch`                                   | `watchEffect`                          |
|-------------------------|-------------------------------------------|----------------------------------------|
| Зависимости             | задаются **явно** (источник)              | **автоматически** из тела функции      |
| Старое значение         | да, `(new, old)`                          | нет                                    |
| Первый запуск           | ленивый (нужен `immediate: true`)         | **сразу** при создании                 |
| Что отслеживает         | только указанный источник                 | всё реактивное, прочитанное в функции  |
| Разделение «когда/что»  | разделено: следим за X, делаем Y          | слито: побочный эффект сам и зависимость|
| Точность                | выше (следишь за конкретным)              | ниже (легко собрать лишнее)            |

Когда что:
- Нужно старое значение, или следить за конкретным источником, или ленивый запуск → **`watch`**.
- Эффект использует несколько реактивных значений и должен реагировать на любое из них, без нужды в old value → **`watchEffect`** (компактнее).

### `deep` — глубокое наблюдение

```vue
<script setup>
import { ref, watch } from 'vue'

const state = ref({ profile: { name: 'Аня' } })

// По умолчанию watch за ref-объектом следит за ЗАМЕНОЙ ссылки .value,
// но НЕ за изменением вложенных свойств.
watch(state, () => console.log('сменилась ссылка')) // не среагирует на state.value.profile.name = ...

// deep: true — следим за вложенными изменениями
watch(
  state,
  () => console.log('изменилось что-то внутри'),
  { deep: true }
)

// Можно задать глубину (3.5+): deep: 1 — только первый уровень
watch(state, () => {}, { deep: 1 })
</script>
```

Нюанс: при `deep` для объекта `newValue` и `oldValue` будут **одним и тем же объектом** (ссылка не менялась), сравнить «было/стало» по полям не получится.

> Для `reactive`-объекта `deep` всегда включён неявно — `watch(reactiveObj, cb)` уже глубокий.

### `immediate` и `once`

```vue
<script setup>
import { ref, watch } from 'vue'

const query = ref('')

// immediate — выполнить колбэк сразу при создании (oldValue будет undefined)
watch(query, (q) => fetchResults(q), { immediate: true })

// once (3.4+) — сработать ровно один раз, затем авто-стоп
watch(query, (q) => console.log('первое изменение:', q), { once: true })

function fetchResults(q) {}
</script>
```

### `flush` timing: pre / post / sync

```vue
<script setup>
import { ref, watch, watchPostEffect, watchSyncEffect } from 'vue'

const count = ref(0)
const el = ref(null)

// pre (по умолчанию): колбэк перед ре-рендером компонента — DOM ещё СТАРЫЙ
watch(count, () => {
  console.log('pre: el.textContent =', el.value?.textContent) // старое значение
})

// post: после обновления DOM — можно читать актуальный DOM
watch(count, () => {
  console.log('post: el.textContent =', el.value?.textContent) // новое
}, { flush: 'post' })

// sync: синхронно, сразу при изменении (без батчинга) — осторожно, может быть много вызовов
watch(count, () => console.log('sync'), { flush: 'sync' })

// Сахар для flush: 'post' и flush: 'sync' у watchEffect:
watchPostEffect(() => { /* DOM уже обновлён */ })
watchSyncEffect(() => { /* синхронно */ })
</script>

<template>
  <p ref="el">{{ count }}</p>
  <button @click="count++">+1</button>
</template>
```

- `pre` — до обновления DOM (по умолчанию), несколько изменений батчатся.
- `post` — после патчинга DOM (нужно, когда эффект читает обновлённый DOM/`$refs`).
- `sync` — немедленно и синхронно (для редких случаев; легко получить лишние срабатывания).

### Остановка наблюдателя

```vue
<script setup>
import { ref, watch, watchEffect } from 'vue'

const x = ref(0)

const stop1 = watch(x, () => {})
const stop2 = watchEffect(() => console.log(x.value))

// Останавливаем вручную, когда наблюдатель больше не нужен
stop1()
stop2()
</script>
```

Наблюдатели, созданные **синхронно в `setup`**, автоматически останавливаются при размонтировании компонента. Но если создать наблюдатель асинхронно (после `await`, в `setTimeout`), его нужно останавливать вручную через `stop()`, иначе утечка.

### Очистка побочных эффектов: `onCleanup` / `onWatcherCleanup`

Защита от «гонки» (race condition): пока летит старый запрос, источник меняется, и нужно отменить устаревший эффект.

```vue
<script setup>
import { ref, watch, watchEffect, onWatcherCleanup } from 'vue'

const id = ref(1)
const data = ref(null)

// Способ 1: третий аргумент колбэка watch — onCleanup
watch(id, async (newId, oldId, onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort()) // вызовется ПЕРЕД следующим запуском и при остановке
  const res = await fetch(`/api/${newId}`, { signal: controller.signal })
  data.value = await res.json()
})

// Способ 2 (3.5+): onWatcherCleanup — импортируемая функция, работает и в watchEffect
watchEffect(() => {
  const controller = new AbortController()
  onWatcherCleanup(() => controller.abort()) // вызвать СИНХРОННО до первого await
  fetch(`/api/${id.value}`, { signal: controller.signal })
})
</script>
```

Когда срабатывает cleanup: перед каждым повторным запуском эффекта и при остановке наблюдателя/размонтировании. `onWatcherCleanup` нужно регистрировать **синхронно** (до первого `await`), иначе он не привяжется к текущему запуску.

### `watch` vs `computed`

```vue
<script setup>
import { ref, computed, watch } from 'vue'

const first = ref('Иван')
const last = ref('Петров')

// computed — производное ЗНАЧЕНИЕ, без сайд-эффектов, кешируется, синхронно
const fullName = computed(() => `${first.value} ${last.value}`)

// watch — ПОБОЧНЫЙ ЭФФЕКТ в ответ на изменение (запрос, лог, запись в storage)
watch(fullName, (name) => localStorage.setItem('name', name))
</script>
```

Правило: нужно **получить значение** из других → `computed`. Нужно **сделать действие** при изменении → `watch`/`watchEffect`. Не имитируйте `computed` через `watch`, записывая результат в отдельный `ref` — это лишний код и источник багов.

## Полный рабочий пример

```vue
<script setup>
import { ref, watch, onWatcherCleanup } from 'vue'

const query = ref('')
const results = ref([])
const loading = ref(false)

// Живой поиск с debounce, отменой устаревших запросов и immediate
watch(
  query,
  async (q, _old, onCleanup) => {
    if (!q) {
      results.value = []
      return
    }
    loading.value = true

    // debounce: отменяем таймер при новом вводе
    let timer
    const debounced = new Promise((resolve) => {
      timer = setTimeout(resolve, 300)
    })
    onCleanup(() => clearTimeout(timer))
    await debounced

    // отмена устаревшего запроса
    const controller = new AbortController()
    onCleanup(() => controller.abort())

    try {
      const res = await fetch(`/api/search?q=${encodeURIComponent(q)}`, {
        signal: controller.signal,
      })
      results.value = await res.json()
    } catch (e) {
      if (e.name !== 'AbortError') throw e
    } finally {
      loading.value = false
    }
  },
  { immediate: false } // ленивый: не искать на пустой старт
)
</script>

<template>
  <input v-model="query" placeholder="Поиск…" />
  <p v-if="loading">Идёт поиск…</p>
  <ul v-else>
    <li v-for="r in results" :key="r.id">{{ r.title }}</li>
  </ul>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем `watch` отличается от `watchEffect`?**
`watch` следит за явно заданным источником, даёт старое и новое значение, ленив по умолчанию. `watchEffect` сам собирает зависимости из тела функции, запускается сразу и не даёт старого значения. `watch` — когда важны «что именно» и «было/стало»; `watchEffect` — компактный эффект на любые использованные реактивные значения.

**2. Чем `watch` отличается от `computed`?**
`computed` возвращает кешированное производное значение без побочных эффектов и вычисляется лениво. `watch` — для побочных эффектов (запросы, логи, запись в storage), значения не возвращает. Производные данные → `computed`, действия по изменению → `watch`.

**3. Почему `watch(reactiveObj, cb)` срабатывает на вложенные изменения, а `watch(refObj, cb)` — нет?**
Для `reactive`-объекта `deep` включается автоматически. Для `ref` на объект без `deep: true` отслеживается только замена `.value` (ссылки), но не мутации вложенных полей.

**4. Что такое `flush` и зачем `post`?**
`flush` задаёт момент срабатывания относительно рендера: `pre` (до обновления DOM, по умолчанию), `post` (после), `sync` (синхронно). `post` нужен, когда колбэк читает обновлённый DOM или template refs.

**5. Как избежать гонки запросов в наблюдателе?**
Через очистку: `onCleanup` (3-й аргумент колбэка `watch`) или `onWatcherCleanup` (3.5+). В колбэке создаём `AbortController` и в cleanup вызываем `abort()` — устаревший запрос отменяется при следующем срабатывании.

**6. Можно ли получить старое значение в `watchEffect`?**
Нет. Если нужно `oldValue`, используйте `watch`.

**7. Как остановить наблюдатель?**
`watch`/`watchEffect` возвращают функцию `stop`; вызвать её. Наблюдатели, созданные синхронно в `setup`, останавливаются автоматически при размонтировании; асинхронно созданные надо останавливать вручную.

**8. Почему мой `watchEffect` не реагирует на значение, прочитанное после `await`?**
Отслеживаются только зависимости, прочитанные синхронно до первого `await`. Считывайте нужные реактивные значения в переменные до `await`.

**9. Как сделать так, чтобы колбэк сработал сразу при создании?**
Опция `{ immediate: true }` у `watch`. У `watchEffect` это поведение по умолчанию.

**10. Что делает опция `once`?**
(3.4+) Наблюдатель сработает один раз и автоматически остановится.

**11. Можно ли следить за несколькими источниками одним `watch`?**
Да: `watch([a, () => b.x], ([newA, newBx], [oldA, oldBx]) => ...)`. Колбэк получает массивы новых и старых значений.

**12. Почему `oldValue` равен `newValue` при `deep`-наблюдении за объектом?**
Потому что мутируется тот же объект — ссылка не меняется, и Vue передаёт один и тот же объект как old и new. Для сравнения «было/стало» делайте `deep`-копию вручную либо следите за конкретными примитивными полями через геттер.

**13. Когда использовать `flush: 'sync'`?**
Редко: когда нужна немедленная синхронная реакция без батчинга (например, интеграция с императивным API). Минус — может срабатывать много раз и бить по производительности.

**14. Что произойдёт, если в `watch` следить за `props.someObject` без `deep`?**
Сработает только при замене самой ссылки на объект (родитель передал новый объект), но не при мутации его полей. Нужен `deep: true` или геттер на конкретное поле.

## Подводные камни (gotchas)

- **`watch(reactiveObj.prop, cb)`** не работает — передаётся значение, не источник. Нужен геттер: `watch(() => reactiveObj.prop, cb)`.
- **`ref` на объект без `deep`** не ловит вложенные мутации.
- **`oldValue === newValue`** при `deep`-наблюдении за объектом (одна ссылка).
- **Зависимости после `await` в `watchEffect`** не отслеживаются.
- **`onWatcherCleanup` после `await`** не привяжется — регистрируйте синхронно.
- **`flush: 'pre'` и чтение DOM** — увидите старый DOM; для нового нужен `flush: 'post'`.
- **Имитация `computed` через `watch` + `ref`** — антипаттерн, плодит баги и лишние ре-рендеры.
- **Асинхронно созданные наблюдатели** не чистятся автоматически — храните `stop` и вызывайте при размонтировании.
- **`watchEffect` с тяжёлой логикой** может собрать лишние зависимости и срабатывать чаще, чем нужно — иногда точнее `watch`.

## Лучшие практики

- Производные значения — через `computed`, побочные эффекты — через `watch`/`watchEffect`.
- Для конкретного поля реактивного объекта используйте геттер-источник.
- Защищайтесь от гонок через `onCleanup`/`onWatcherCleanup` + `AbortController`.
- `flush: 'post'` (или `watchPostEffect`) когда нужен актуальный DOM.
- `immediate: true` вместо дублирования логики «запустить сразу + по изменению».
- Останавливайте наблюдатели, созданные вне синхронного `setup`.
- Не складывайте в один `watchEffect` слишком много — это снижает предсказуемость; разбивайте по смыслу.
- Выносите сложные наблюдатели в composables.

## Шпаргалка

```text
watch(src, (new, old, onCleanup) => {})   // явный источник, old value, ленивый
watchEffect(() => { ... })                // автозависимости, сразу, без old
watchPostEffect(fn)                       // = watchEffect(fn, { flush: 'post' })
watchSyncEffect(fn)                        // = watchEffect(fn, { flush: 'sync' })

Источники watch: ref | () => expr | reactiveObj | [a, () => b]
Опции: { deep, immediate, once, flush: 'pre'|'post'|'sync' }

deep:        для reactive — авто; для ref-объекта — нужен deep: true
immediate:   запустить колбэк сразу (old = undefined)
once:        сработать один раз (3.4+)
flush pre:   до рендера (DOM старый)  ← по умолчанию
flush post:  после рендера (DOM новый)
flush sync:  синхронно, без батчинга

Стоп:        const stop = watch(...); stop()
Очистка:     onCleanup(() => ctrl.abort())  /  onWatcherCleanup(...) // синхронно!
Авто-стоп:   только если создан синхронно в setup

watch   → нужен источник/old/ленивость
computed→ нужно производное ЗНАЧЕНИЕ (кеш, без side-effect)
```
