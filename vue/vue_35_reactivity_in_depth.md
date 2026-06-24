# Vue 3 — Реактивность изнутри — конспект и вопросы

## О чём раздел

Реактивность — сердце Vue. Этот раздел объясняет, *как именно* Vue 3 отслеживает
зависимости и перерисовывает компоненты: на чём построена реактивность (`Proxy` vs
`Object.defineProperty`), что такое **track/trigger** и **dep**, что такое
**эффект** (`ReactiveEffect`), почему `reactive` не работает с примитивами и зачем
нужен `ref` с `.value`, какие есть ограничения, продвинутые API
(`shallowRef`, `markRaw`, `customRef`, `effectScope`) и как реактивность связана с
асинхронной очередью обновлений и `nextTick`.

## Ключевые концепции

- **Сигнал-подобная модель**: данные "знают", кто их читает (эффекты), и оповещают
  при изменении.
- **`reactive()`** оборачивает объект в `Proxy`, перехватывая `get`/`set`/`deleteProperty` и т.д.
- **`ref()`** оборачивает значение в объект с аксессором `.value` (геттер/сеттер).
- **track** — при чтении реактивного свойства внутри активного эффекта связь
  "свойство → эффект" записывается в **dep** (dependency).
- **trigger** — при изменении свойства все эффекты из его dep ставятся в очередь на
  повторный запуск.
- **`ReactiveEffect`** — обёртка над функцией (например, render-функцией компонента),
  которая запускается при изменении её зависимостей.
- **Асинхронная очередь**: эффекты не выполняются синхронно — они батчатся и
  выполняются в микротаске; `nextTick` ждёт завершения этой очереди.

## Подробный разбор с примерами кода

### Proxy (Vue 3) vs Object.defineProperty (Vue 2)

**Vue 2** использовал `Object.defineProperty` для каждого свойства каждого объекта при
инициализации. Ограничения:

- Не отслеживалось **добавление/удаление** свойств (нужны были `Vue.set`/`Vue.delete`).
- Не отслеживались изменения **по индексу массива** и `length` напрямую (метод массива
  патчили вручную).
- Рекурсивный обход всего объекта при инициализации — дорого для больших структур.

**Vue 3** использует **`Proxy`** — обёртку, перехватывающую любые операции с объектом:

```ts
// Упрощённая модель reactive
function reactive(target) {
  return new Proxy(target, {
    get(obj, key, receiver) {
      track(obj, key)                       // фиксируем зависимость
      const res = Reflect.get(obj, key, receiver)
      // ленивая, рекурсивная реактивность для вложенных объектов
      return typeof res === 'object' && res !== null ? reactive(res) : res
    },
    set(obj, key, value, receiver) {
      const old = obj[key]
      const res = Reflect.set(obj, key, value, receiver)
      if (old !== value) trigger(obj, key)  // оповещаем зависимости
      return res
    },
    deleteProperty(obj, key) {
      const had = key in obj
      const res = Reflect.deleteProperty(obj, key)
      if (had) trigger(obj, key)
      return res
    },
  })
}
```

Преимущества Proxy:
- Перехватывает добавление/удаление свойств, операции с массивами, `in`, итерацию.
- Реактивность вложенных объектов **ленивая** — оборачиваются только при доступе.
- Не нужны `Vue.set` / `Vue.delete`.

### track / trigger и dep

```ts
// Упрощённо: глобально храним "текущий выполняемый эффект"
let activeEffect = null

// targetMap: WeakMap<target, Map<key, Set<effect>>>
const targetMap = new WeakMap()

function track(target, key) {
  if (!activeEffect) return
  let depsMap = targetMap.get(target)
  if (!depsMap) targetMap.set(target, (depsMap = new Map()))
  let dep = depsMap.get(key)
  if (!dep) depsMap.set(key, (dep = new Set()))
  dep.add(activeEffect)              // свойство запоминает эффект
}

function trigger(target, key) {
  const depsMap = targetMap.get(target)
  const dep = depsMap?.get(key)
  dep?.forEach(effect => effect.run())  // перезапускаем зависимые эффекты
}
```

**dep** (dependency) — это набор (`Set`) эффектов, зависящих от конкретного
свойства. Структура: `target → key → dep(Set эффектов)`. Используется `WeakMap` на
верхнем уровне, чтобы объекты могли собираться сборщиком мусора.

### Что такое эффект (ReactiveEffect)

Эффект — функция, которая *читает* реактивные данные и должна *перезапускаться* при
их изменении. Примеры эффектов: render-функция компонента, `computed`, `watch`,
`watchEffect`.

```ts
class ReactiveEffect {
  constructor(public fn) {}
  run() {
    activeEffect = this        // делаем себя активным
    try {
      return this.fn()         // во время вызова все track() свяжут свойства с this
    } finally {
      activeEffect = null
    }
  }
}

// watchEffect по сути:
function watchEffect(fn) {
  const effect = new ReactiveEffect(fn)
  effect.run()                 // первый запуск собирает зависимости
}
```

Когда `fn` читает `state.count`, срабатывает `track`, и эффект попадает в dep
свойства `count`. При `state.count++` срабатывает `trigger`, который вызывает
`effect.run()` заново.

### Почему `reactive` не работает с примитивами и зачем `ref` (.value)

`Proxy` можно создать **только вокруг объекта**. Примитивы (`number`, `string`,
`boolean`) проксировать нельзя, а главное — они передаются **по значению**: если
вернуть число из функции, потеряется связь с источником.

```ts
let count = reactive(0)   // не сработает — Proxy не оборачивает примитив
```

Решение — **`ref`**: контейнер-объект с аксессором `.value`. Объект можно
проксировать/отслеживать, а доступ к `.value` запускает track/trigger:

```ts
// Упрощённая модель ref
function ref(value) {
  const r = {
    _value: value,
    get value() {
      track(r, 'value')         // чтение .value -> зависимость
      return this._value
    },
    set value(newVal) {
      this._value = newVal
      trigger(r, 'value')       // запись .value -> оповещение
    },
  }
  return r
}
```

Поэтому `.value` нужен в JS-коде. В шаблоне Vue делает авто-unwrap для ref'ов
верхнего уровня, поэтому там `.value` не пишут.

### Ограничения реактивности

**1. Замена всего reactive-объекта рвёт реактивность:**

```ts
let state = reactive({ count: 0 })
// ПЛОХО: переприсваивание переменной — старый Proxy теряется, связи рвутся
state = reactive({ count: 1 })   // эффекты подписаны на старый объект
```

Правильно — мутировать свойства существующего объекта или использовать `ref`:

```ts
const state = ref({ count: 0 })
state.value = { count: 1 }       // ок: .value-сеттер запускает trigger
```

**2. Деструктуризация рвёт связь:**

```ts
const state = reactive({ count: 0 })
let { count } = state            // count — обычное число, не реактивно
count++                          // state.count не изменится, эффекты не сработают
```

Решение — `toRefs` / `toRef`:

```ts
import { toRefs } from 'vue'
const { count } = toRefs(state)  // count: Ref<number>, реактивность сохранена
count.value++                    // обновит state.count
```

**3. Передача свойства в функцию** теряет реактивность по той же причине (передаётся
значение). Передавайте сам reactive-объект или ref.

### toRaw / markRaw / shallowRef / shallowReactive / customRef

```ts
import {
  toRaw, markRaw, shallowRef, shallowReactive, customRef, triggerRef,
} from 'vue'

// toRaw — получить исходный объект без Proxy (для интеропа / производительности)
const obj = reactive({ a: 1 })
const raw = toRaw(obj)           // оригинал, мутации НЕ триггерят эффекты

// markRaw — пометить объект как НЕ реактивный навсегда
// (полезно для тяжёлых внешних инстансов: карты, графики, классы)
const heavy = markRaw(new ThirdPartyLib())
const state = reactive({ lib: heavy })   // lib не оборачивается в Proxy

// shallowRef — реактивна только сама .value, но не её содержимое
const sr = shallowRef({ count: 0 })
sr.value.count++                 // НЕ триггерит (вложенное не отслеживается)
sr.value = { count: 1 }          // триггерит (замена .value)
triggerRef(sr)                   // принудительно оповестить после мутации

// shallowReactive — реактивны только свойства верхнего уровня
const ss = shallowReactive({ nested: { x: 1 } })
ss.nested.x++                    // НЕ триггерит
ss.nested = { x: 2 }             // триггерит

// customRef — полный контроль над track/trigger (напр., debounce)
function useDebouncedRef(value, delay = 300) {
  return customRef((track, trigger) => {
    let timeout
    return {
      get() { track(); return value },
      set(newVal) {
        clearTimeout(timeout)
        timeout = setTimeout(() => { value = newVal; trigger() }, delay)
      },
    }
  })
}
```

Когда что использовать:
- `shallowRef`/`shallowReactive` — большие неизменяемые структуры или интеграция с
  иммутабельными данными; снижают накладные расходы.
- `markRaw` — внешние тяжёлые объекты, которые не должны быть реактивными.
- `toRaw` — оптимизация горячих участков, интероп с библиотеками.
- `customRef` — кастомная логика track/trigger (debounce, throttle, localStorage).

### Реактивность и асинхронность (nextTick и очередь обновлений)

Изменения реактивного состояния **не вызывают DOM-обновление синхронно**. Vue
батчит все изменения в текущем "тике" и обновляет DOM один раз в **микротаске**.

```ts
import { ref, nextTick } from 'vue'

const count = ref(0)

count.value++
count.value++
count.value++
// DOM ещё показывает старое значение — обновление отложено

await nextTick()
// теперь DOM обновлён, можно читать актуальные размеры/значения из DOM
```

Зачем батчинг: если в одном обработчике поменять 100 свойств, компонент
перерисуется **один раз**, а не 100. Эффекты дедуплицируются (один эффект в очереди
не дублируется). `nextTick()` возвращает Promise, резолвящийся после применения
всех обновлений DOM.

```ts
// Типичный кейс: получить актуальный DOM после изменения данных
items.value.push(newItem)
await nextTick()
listEl.value.scrollTop = listEl.value.scrollHeight   // прокрутка к новому элементу
```

### Эффект-скоупы (effectScope)

`effectScope` группирует эффекты (computed/watch/watchEffect), чтобы остановить их
все разом — основа для библиотек и переиспользуемой логики вне компонента.

```ts
import { effectScope, ref, watch, computed } from 'vue'

const scope = effectScope()

scope.run(() => {
  const a = ref(1)
  const double = computed(() => a.value * 2)        // эффект внутри скоупа
  watch(a, () => console.log('changed'))            // и этот тоже
})

// Останавливаем ВСЕ эффекты внутри скоупа одним вызовом
scope.stop()
```

Это механизм, на котором Pinia и VueUse реализуют автоматическую очистку. Внутри
компонента Vue создаёт скоуп автоматически и останавливает его при размонтировании
(поэтому `watch`/`computed`, созданные в `setup`, убираются сами).

## Полный рабочий пример

`useThrottledRef.ts` — кастомный ref с дросселированием через `customRef` +
`effectScope` для группировки:

```ts
import { customRef, effectScope, watch, type Ref } from 'vue'

// customRef: ограничиваем частоту обновлений значения
export function useThrottledRef<T>(initial: T, ms = 500): Ref<T> {
  let value = initial
  let last = 0
  let timer: ReturnType<typeof setTimeout> | undefined

  return customRef<T>((track, trigger) => ({
    get() {
      track()                      // регистрируем зависимость
      return value
    },
    set(newVal) {
      const now = Date.now()
      const remaining = ms - (now - last)
      clearTimeout(timer)
      if (remaining <= 0) {
        last = now
        value = newVal
        trigger()                  // оповещаем эффекты
      } else {
        timer = setTimeout(() => {
          last = Date.now()
          value = newVal
          trigger()
        }, remaining)
      }
    },
  }))
}
```

```vue
<script setup lang="ts">
import { effectScope, watchEffect } from 'vue'
import { useThrottledRef } from './useThrottledRef'

const search = useThrottledRef('', 400)

// эффекты можно сгруппировать скоупом, если нужно ручное управление жизненным циклом
const scope = effectScope()
scope.run(() => {
  watchEffect(() => {
    console.log('throttled search:', search.value)
  })
})
// scope.stop() — когда логика больше не нужна
</script>

<template>
  <input v-model="search" placeholder="Поиск (throttled)" />
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Чем реактивность Vue 3 отличается от Vue 2?**
Vue 3 использует `Proxy` (перехват любых операций, включая add/delete/array),
Vue 2 — `Object.defineProperty` (геттеры/сеттеры по каждому свойству при инициализации).
Vue 3 ленив, не требует `Vue.set`/`Vue.delete`, отслеживает индексы массивов.

**2. Что такое track и trigger?**
`track` вызывается при *чтении* реактивного свойства и связывает свойство с активным
эффектом (записывает в dep). `trigger` вызывается при *изменении* и перезапускает все
эффекты из dep этого свойства.

**3. Что такое dep?**
Dependency — `Set` эффектов, зависящих от конкретного свойства. Хранится в структуре
`WeakMap<target, Map<key, dep>>` (`targetMap`).

**4. Что такое ReactiveEffect / эффект?**
Обёртка над функцией, которая читает реактивные данные и перезапускается при их
изменении. Render-функция компонента, `computed`, `watch` — всё это эффекты.

**5. Почему `reactive` не работает с примитивами?**
`Proxy` оборачивает только объекты; примитивы нельзя проксировать и они передаются по
значению, теряя связь. Поэтому для примитивов используется `ref` (объект-контейнер с
`.value`).

**6. Зачем `.value` у ref?**
`.value` — это геттер/сеттер, через который проходят track/trigger. Без обёртки в
объект нельзя было бы перехватить чтение/запись примитива. В шаблоне `.value`
авто-разворачивается.

**7. Почему теряется реактивность при деструктуризации `reactive`?**
Деструктуризация копирует *значение* свойства в новую переменную, разрывая связь с
Proxy. Решение — `toRefs`/`toRef` (создаёт ref'ы, синхронизированные с источником).

**8. Почему нельзя переприсваивать reactive-объект?**
Эффекты подписаны на конкретный Proxy. Замена переменной создаёт новый Proxy, на
который никто не подписан. Используйте `ref` (замена через `.value`) или мутируйте
свойства.

**9. В чём разница `shallowRef` и `ref`?**
`ref` делает значение глубоко реактивным (вложенные объекты тоже). `shallowRef`
отслеживает только замену `.value`; мутации внутри не триггерят. Полезно для больших
структур и оптимизации.

**10. Что делают `markRaw` и `toRaw`?**
`markRaw` помечает объект как никогда не реактивный (не оборачивается в Proxy даже
внутри reactive). `toRaw` возвращает исходный объект из Proxy. Применяют для тяжёлых
внешних инстансов и оптимизации.

**11. Зачем нужен `customRef`?**
Для полного контроля над track/trigger — например, debounce/throttle значения или
синхронизация с внешним источником (localStorage).

**12. Что такое очередь обновлений и `nextTick`?**
Vue батчит изменения и обновляет DOM раз за микротаск, дедуплицируя эффекты.
`nextTick()` возвращает Promise, резолвящийся после применения обновлений DOM —
нужен, чтобы читать актуальный DOM после изменения данных.

**13. Что такое `effectScope` и где применяется?**
Группа эффектов, которую можно остановить одним вызовом `scope.stop()`. Основа
автоматической очистки в Pinia/VueUse и логики, живущей вне компонента.

**14. Почему `watch`/`computed` в `setup` не нужно вручную останавливать?**
Vue создаёт effectScope для компонента и останавливает его при размонтировании, очищая
все созданные эффекты автоматически.

## Подводные камни (gotchas)

- **Деструктуризация `reactive`** рвёт реактивность — используйте `toRefs`.
- **Переприсваивание `reactive`-переменной** теряет подписки — мутируйте или берите `ref`.
- **`shallowRef` + мутация содержимого** не триггерит — нужен `triggerRef` или замена `.value`.
- **Ожидание синхронного DOM** после изменения данных — неверно; используйте `nextTick`.
- **`reactive` для примитива** молча не работает — ошибки нет, но реактивности нет.
- **`toRaw`-мутации** не вызывают обновлений (это и есть цель) — легко выстрелить себе в ногу.
- **Реактивность ref в массиве/Map** требует доступа через `.value` (не авто-unwrap в коллекциях).

## Лучшие практики

- Для примитивов — `ref`; для связанных групп состояния — `reactive` или объект в `ref`.
- Не деструктурируйте `reactive` без `toRefs`.
- Не переприсваивайте reactive-объект; держите состояние в `ref` если нужна замена целиком.
- `markRaw` для тяжёлых внешних инстансов (карты, чарты, WebSocket-клиенты).
- `shallowRef`/`shallowReactive` — для больших неизменяемых структур.
- `customRef` — для debounce/throttle и интеграций.
- Помните о батчинге: после мутации данных DOM читайте только после `nextTick`.

## Шпаргалка

```ts
reactive(obj)          // Proxy, глубокая реактивность объектов
ref(val)               // .value-контейнер, работает с примитивами
shallowRef(val)        // реактивна только замена .value
shallowReactive(obj)   // реактивны только свойства верхнего уровня
toRef(obj, 'key')      // один реактивный ref на свойство
toRefs(obj)            // объект ref'ов (для деструктуризации reactive)
toRaw(proxy)           // исходный объект без Proxy
markRaw(obj)           // объект никогда не станет реактивным
customRef((track, trigger) => ({ get, set }))   // свой track/trigger
triggerRef(shallowRef) // принудительный trigger
await nextTick()       // дождаться применения DOM-обновлений
effectScope() / scope.run(fn) / scope.stop()    // группировка эффектов
```

Структуры под капотом: `targetMap: WeakMap<target, Map<key, dep:Set<effect>>>`,
поток — `get → track`, `set → trigger → effect.run()`, обновления DOM — батч в микротаске.
