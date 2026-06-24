# Vue 3 — Привязка форм (v-model) — конспект и вопросы

## О чём раздел

Работа с формами — это синхронизация состояния компонента с тем, что вводит пользователь. Вручную это означало бы для каждого поля писать `:value="x"` и `@input="x = $event.target.value"`. Vue даёт директиву **`v-model`**, которая делает двустороннюю привязку (two-way binding) одной строкой.

В этом разделе:
- что такое `v-model` и во что он **разворачивается** (синтаксический сахар);
- применение к `input`, `textarea`, `checkbox` (одиночный и массив), `radio`, `select` (одиночный и множественный);
- привязки значений: `true-value`/`false-value`, `:value` для опций/радио;
- модификаторы `.lazy`, `.number`, `.trim`;
- `v-model` на компонентах (кратко — подробнее в теме компонентов).

## Ключевые концепции

- **`v-model`** — двусторонняя привязка: «значение → элемент» и «ввод → значение».
- Это **синтаксический сахар**: Vue сам выбирает нужное свойство и событие в зависимости от типа элемента.
- На разных элементах `v-model` разворачивается **по-разному**:
  - текстовые поля → `:value` + `@input`;
  - чекбоксы/радио → `:checked` + `@change`;
  - select → `:value` + `@change`.
- `v-model` **игнорирует начальные `value`/`checked`/`selected` в HTML** — источником истины всегда является связанное состояние JS.
- Модификаторы: `.lazy` (синхронизация по `change`, а не `input`), `.number` (приведение к числу), `.trim` (обрезка пробелов).
- На компонентах `v-model` по умолчанию разворачивается в проп `modelValue` + событие `update:modelValue`.

## Подробный разбор с примерами кода

### 1. Во что разворачивается `v-model` (ОБЯЗАТЕЛЬНО к пониманию)

`v-model` — это сахар. Компилятор смотрит на тип элемента и подставляет нужную пару «свойство + событие».

**Текстовый input / textarea:**

```vue
<!-- Это: -->
<input v-model="text" />

<!-- Разворачивается примерно в: -->
<input
  :value="text"
  @input="event => text = event.target.value"
/>
```

**Checkbox / radio:**

```vue
<!-- Это: -->
<input type="checkbox" v-model="checked" />

<!-- Разворачивается в (используется checked и событие change): -->
<input
  type="checkbox"
  :checked="checked"
  @change="event => checked = event.target.checked"
/>
```

**Select:**

```vue
<!-- Это: -->
<select v-model="selected">...</select>

<!-- Разворачивается в (value + change): -->
<select
  :value="selected"
  @change="event => selected = event.target.value"
>...</select>
```

Сводная таблица «во что разворачивается»:

| Элемент | Привязываемое свойство | Слушаемое событие |
|---|---|---|
| `<input type="text">`, `<textarea>` | `value` | `input` |
| `<input type="checkbox">`, `<input type="radio">` | `checked` | `change` |
| `<select>` | `value` | `change` |

> Поэтому `v-model` всегда можно «расписать руками» через `:value`/`:checked` + соответствующее событие. Это полезно, когда нужна нестандартная логика (например, форматирование при вводе).

### 2. Текстовые поля: input и textarea

```vue
<script setup>
import { ref } from 'vue'
const text = ref('')
const message = ref('')
</script>

<template>
  <input v-model="text" placeholder="Введите текст" />
  <p>Значение: {{ text }}</p>

  <!-- В textarea НЕЛЬЗЯ использовать интерполяцию {{ }} как значение -->
  <!-- НЕВЕРНО: <textarea>{{ message }}</textarea> -->
  <!-- ВЕРНО: -->
  <textarea v-model="message" placeholder="Сообщение"></textarea>
</template>
```

### 3. Checkbox: одиночный и массив

**Одиночный** чекбокс привязывается к булеву значению:

```vue
<script setup>
import { ref } from 'vue'
const checked = ref(false)
</script>

<template>
  <input type="checkbox" id="agree" v-model="checked" />
  <label for="agree">Согласен: {{ checked }}</label>
</template>
```

**Несколько** чекбоксов, привязанных к одному **массиву** — в массив попадают `value` выбранных:

```vue
<script setup>
import { ref } from 'vue'
const checkedNames = ref([]) // массив выбранных значений
</script>

<template>
  <input type="checkbox" value="Иван" v-model="checkedNames" />
  <input type="checkbox" value="Пётр" v-model="checkedNames" />
  <input type="checkbox" value="Анна" v-model="checkedNames" />
  <p>Выбраны: {{ checkedNames }}</p>
  <!-- Например: ["Иван", "Анна"] -->
</template>
```

`v-model` может также привязываться к **`Set`** для коллекции выбранных значений.

### 4. true-value / false-value

Для одиночного чекбокса можно задать произвольные значения для состояний «вкл/выкл»:

```vue
<script setup>
import { ref } from 'vue'
const toggle = ref('yes')
</script>

<template>
  <input
    type="checkbox"
    v-model="toggle"
    true-value="yes"
    false-value="no"
  />
  <!-- toggle станет 'yes' при выборе и 'no' при снятии -->
  <p>{{ toggle }}</p>
</template>
```

Если нужно динамическое значение (не строка), используйте `:true-value`/`:false-value`:

```vue
<input
  type="checkbox"
  v-model="toggle"
  :true-value="dynamicTrue"
  :false-value="dynamicFalse"
/>
```

> `true-value`/`false-value` работают только на **одиночном** чекбокле и **не влияют** на режим массива. Эти атрибуты — особенность Vue, в нативный input они не попадают.

### 5. Radio

Радиокнопки привязываются к одному значению; выбранное `value` становится значением `v-model`:

```vue
<script setup>
import { ref } from 'vue'
const picked = ref('') // например, 'one'
</script>

<template>
  <input type="radio" value="one" v-model="picked" />
  <input type="radio" value="two" v-model="picked" />
  <p>Выбрано: {{ picked }}</p>
</template>
```

Для нестроковых значений используйте `:value`:

```vue
<input type="radio" :value="optionObject" v-model="picked" />
```

### 6. Select: одиночный и множественный

**Одиночный** select:

```vue
<script setup>
import { ref } from 'vue'
const selected = ref('') // одно значение
</script>

<template>
  <select v-model="selected">
    <!-- disabled-опция с пустым value на iOS помогает показать placeholder -->
    <option disabled value="">Выберите...</option>
    <option>A</option>
    <option>B</option>
    <option>C</option>
  </select>
  <p>Выбрано: {{ selected }}</p>
</template>
```

**Множественный** select (`multiple`) привязывается к **массиву**:

```vue
<script setup>
import { ref } from 'vue'
const selected = ref([]) // массив значений
</script>

<template>
  <select v-model="selected" multiple>
    <option>A</option>
    <option>B</option>
    <option>C</option>
  </select>
  <p>Выбраны: {{ selected }}</p>
  <!-- Например: ["A", "C"] -->
</template>
```

Опции можно рендерить через `v-for`, а нестроковые значения привязывать через `:value`:

```vue
<script setup>
import { ref } from 'vue'
const selected = ref(null)
const options = [
  { text: 'Один', value: { id: 1 } },
  { text: 'Два', value: { id: 2 } },
]
</script>

<template>
  <select v-model="selected">
    <!-- :value позволяет хранить в selected объект, а не строку -->
    <option v-for="o in options" :key="o.text" :value="o.value">
      {{ o.text }}
    </option>
  </select>
  <p>{{ selected }}</p> <!-- { "id": 1 } -->
</template>
```

### 7. Модификаторы v-model

**`.lazy`** — синхронизировать по событию `change` (потеря фокуса / Enter), а не по каждому `input`:

```vue
<!-- Обновится не на каждое нажатие, а по событию change -->
<input v-model.lazy="msg" />
```

**`.number`** — автоматически приводить ввод к числу через `parseFloat`. Если значение не парсится — вернётся исходная строка:

```vue
<script setup>
import { ref } from 'vue'
const age = ref(0)
</script>

<template>
  <!-- typeof age === 'number' -->
  <input v-model.number="age" type="number" />
</template>
```

> `.number` применяется автоматически, если у input стоит `type="number"`, но явное указание делает намерение очевидным и работает для других типов.

**`.trim`** — обрезать начальные и конечные пробелы:

```vue
<input v-model.trim="msg" />
<!-- '  привет  ' → 'привет' -->
```

Модификаторы можно комбинировать: `v-model.lazy.trim="msg"`, `v-model.number.lazy="price"`.

### 8. v-model на компонентах (кратко)

На своём компоненте `v-model` по умолчанию разворачивается в проп `modelValue` + событие `update:modelValue`. В Vue 3.4+ внутри компонента удобно использовать макрос `defineModel()`:

```vue
<!-- Родитель -->
<MyInput v-model="text" />
<!-- ≡ -->
<MyInput :model-value="text" @update:model-value="text = $event" />
```

```vue
<!-- Дочерний компонент MyInput.vue (Vue 3.4+) -->
<script setup>
const model = defineModel() // двусторонняя привязка к v-model родителя
</script>

<template>
  <input v-model="model" />
</template>
```

Можно делать **именованные** `v-model` (несколько на компоненте): `v-model:title="..."`, `v-model:content="..."`. Подробнее — в разделе о компонентах и `defineModel`.

## Полный рабочий пример

```vue
<script setup>
import { reactive } from 'vue'

// Единый объект состояния формы
const form = reactive({
  name: '',
  bio: '',
  subscribe: 'no',     // true-value/false-value
  hobbies: [],         // массив чекбоксов
  gender: '',          // radio
  country: '',         // одиночный select
  langs: [],           // множественный select
  age: null,           // .number
})

function submit() {
  console.log('Отправка формы:', JSON.parse(JSON.stringify(form)))
}
</script>

<template>
  <form @submit.prevent="submit">
    <!-- Текст с .trim: уберём лишние пробелы -->
    <label>Имя:
      <input v-model.trim="form.name" />
    </label>

    <!-- textarea + .lazy: обновление по потере фокуса -->
    <label>О себе:
      <textarea v-model.lazy="form.bio"></textarea>
    </label>

    <!-- Одиночный чекбокс с произвольными значениями -->
    <label>
      <input type="checkbox" v-model="form.subscribe"
             true-value="yes" false-value="no" />
      Подписка ({{ form.subscribe }})
    </label>

    <!-- Несколько чекбоксов → массив -->
    <fieldset>
      <legend>Хобби:</legend>
      <label><input type="checkbox" value="спорт" v-model="form.hobbies" /> спорт</label>
      <label><input type="checkbox" value="музыка" v-model="form.hobbies" /> музыка</label>
      <label><input type="checkbox" value="код" v-model="form.hobbies" /> код</label>
    </fieldset>

    <!-- Radio -->
    <fieldset>
      <legend>Пол:</legend>
      <label><input type="radio" value="m" v-model="form.gender" /> М</label>
      <label><input type="radio" value="f" v-model="form.gender" /> Ж</label>
    </fieldset>

    <!-- Одиночный select -->
    <label>Страна:
      <select v-model="form.country">
        <option disabled value="">Выберите...</option>
        <option>Россия</option>
        <option>Казахстан</option>
        <option>Беларусь</option>
      </select>
    </label>

    <!-- Множественный select → массив -->
    <label>Языки:
      <select v-model="form.langs" multiple>
        <option>JS</option>
        <option>Python</option>
        <option>Go</option>
      </select>
    </label>

    <!-- Число через .number -->
    <label>Возраст:
      <input v-model.number="form.age" type="number" />
    </label>

    <button type="submit">Отправить</button>

    <pre>{{ form }}</pre>
  </form>
</template>
```

## Частые вопросы на собеседовании (Q/A)

**1. Что такое `v-model` и во что он разворачивается?**
Это синтаксический сахар для двусторонней привязки. На текстовом input разворачивается в `:value` + `@input`, на checkbox/radio — в `:checked` + `@change`, на select — в `:value` + `@change`. Vue сам выбирает свойство и событие по типу элемента.

**2. Почему нельзя писать `<textarea>{{ msg }}</textarea>`?**
Интерполяция внутри textarea не сработает для двусторонней привязки. Нужно использовать `v-model`: `<textarea v-model="msg"></textarea>`.

**3. Что станет значением `v-model`, если связать несколько чекбоксов с одним массивом?**
Массив `value` всех отмеченных чекбоксов, например `["спорт", "код"]`. Можно привязывать и к `Set`.

**4. Зачем нужны `true-value` и `false-value`?**
Чтобы одиночный чекбокс хранил не `true`/`false`, а произвольные значения (например, `'yes'`/`'no'` или числа/объекты через `:true-value`). Работают только на одиночном чекбоксе.

**5. Чем отличается одиночный select от множественного по части привязки?**
Одиночный привязывается к скаляру (одно значение). Множественный (`multiple`) — к массиву выбранных значений.

**6. Что делает модификатор `.lazy`?**
Меняет событие синхронизации с `input` на `change` — значение обновляется не на каждый ввод, а по потере фокуса / Enter. Уменьшает частоту обновлений.

**7. Что делает `.number` и что будет, если ввод не число?**
Приводит значение к числу через `parseFloat`. Если строка не парсится в число — возвращается исходная строка. Часто используется с `type="number"`.

**8. Что делает `.trim`?**
Обрезает начальные и конечные пробелы у введённой строки перед записью в состояние.

**9. Учитывает ли `v-model` начальные `value`/`checked`/`selected` в HTML?**
Нет. `v-model` игнорирует начальные атрибуты разметки — источник истины всегда связанное JS-состояние. Начальное значение задаётся в `ref`/`reactive`.

**10. Как привязать select к объекту, а не к строке?**
Через `:value` на `<option>`: `<option :value="obj">`. Тогда в `v-model` попадёт сам объект (по ссылке), а не строка.

**11. Можно ли расписать `v-model` вручную и зачем?**
Да: `:value` + `@input` (или нужная пара). Это нужно для нестандартной логики — например, форматировать ввод (телефон, маска), нормализовать значение, или когда нужен полный контроль над событием.

**12. Как работает `v-model` на пользовательском компоненте?**
По умолчанию разворачивается в проп `modelValue` + событие `update:modelValue`. В Vue 3.4+ внутри компонента используют `defineModel()`. Можно делать именованные: `v-model:title`.

**13. Можно ли комбинировать модификаторы `v-model`?**
Да: `v-model.lazy.trim`, `v-model.number.lazy` и т.п.

**14. В чём разница между `@change` и `@input` для текстового поля?**
`input` срабатывает на каждое изменение значения (каждое нажатие), `change` — при подтверждении (потеря фокуса/Enter). Обычный `v-model` использует `input`, а `v-model.lazy` — `change`.

## Подводные камни (gotchas)

- **`v-model` игнорирует начальные атрибуты разметки** (`value`, `checked`, `selected`). Задавайте начальное значение в JS-состоянии.
- **`textarea` с интерполяцией не работает** — только `v-model`.
- **`.number` при непарсящемся вводе возвращает строку**, а не `NaN` — будьте внимательны при валидации.
- **`true-value`/`false-value` только для одиночного чекбокса** и не влияют на массивы.
- **Множественный select требует массива** в качестве модели; если дать скаляр — поведение будет некорректным.
- **iOS и select с placeholder**: иногда первая `disabled value=""` опция нужна, чтобы Vue корректно показал плейсхолдер при пустом значении.
- **Объекты в `:value`** сравниваются по ссылке — для radio/select с объектами выбранное значение должно быть тем же объектом (по ссылке), иначе предвыбор не отобразится.
- **IME-ввод** (китайский/японский и т.п.): `v-model` не обновляется во время композиции IME — для таких случаев иногда нужен ручной `@input`.
- **`.lazy` меняет момент обновления**, что может ломать ожидания валидации «на лету».

## Лучшие практики

- Используйте `v-model` для стандартных случаев; расписывайте вручную `:value`+событие только когда нужна особая логика (маски, форматирование).
- Для одного логического значения чекбокса — булев `ref`; для группы — массив или `Set`.
- Задавайте `:key` при рендере опций через `v-for`.
- Для числовых полей применяйте `.number` и валидируйте результат (возможна строка при невалидном вводе).
- `.trim` — хорошая практика для имён, email и прочих текстовых полей, где пробелы по краям нежелательны.
- Храните состояние формы в `reactive`-объекте для удобной отправки и валидации.
- На компонентах используйте `defineModel()` (Vue 3.4+) — это короче и понятнее, чем ручные `modelValue` + `update:modelValue`.
- Для placeholder в select добавляйте первую `disabled` опцию с пустым `value`.

## Шпаргалка

```text
v-model на элементах разворачивается в:
  input[text], textarea  →  :value   + @input
  checkbox, radio        →  :checked + @change
  select                 →  :value   + @change

Checkbox:
  одиночный → boolean
  несколько с массивом → массив value выбранных
  true-value / false-value → произвольные вкл/выкл значения (одиночный)

Radio:
  v-model = value выбранной кнопки
  :value для нестроковых значений

Select:
  одиночный  → скаляр
  multiple   → массив
  :value на <option> для объектов

Модификаторы:
  .lazy    синхронизация по change (а не input)
  .number  parseFloat (иначе исходная строка)
  .trim    обрезать пробелы по краям
  комбинируются: v-model.number.lazy, v-model.lazy.trim

Компоненты:
  v-model → :model-value + @update:model-value
  внутри: const model = defineModel()  // Vue 3.4+
  именованные: v-model:title="..."

Важно:
  v-model игнорирует начальные value/checked/selected в HTML
  источник истины — JS-состояние
```
