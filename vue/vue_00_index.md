# Vue 3 — Индекс материалов для подготовки к собеседованию

Конспекты по официальному руководству Vue 3 (https://vuejs.org/guide/).
Современный стиль: **Vue 3, Composition API, `<script setup>`, TypeScript**.
Уровень — middle/senior.

Каждый файл-тема построен по единой структуре: о чём раздел → ключевые концепции →
подробный разбор с примерами → полный рабочий пример → частые вопросы на собеседовании
(Q/A) → подводные камни → лучшие практики → шпаргалка.

---

## Основы (Essentials)

| Файл | Тема |
|------|------|
| [vue_01_introduction.md](vue_01_introduction.md) | Введение: что такое Vue, прогрессивный фреймворк, SFC, Options vs Composition API |
| [vue_02_quick_start.md](vue_02_quick_start.md) | Быстрый старт: create-vue, Vite, структура проекта |
| [vue_03_template_syntax.md](vue_03_template_syntax.md) | Синтаксис шаблонов: интерполяция, директивы, биндинги, модификаторы |
| [vue_04_reactivity_fundamentals.md](vue_04_reactivity_fundamentals.md) | Основы реактивности: `ref`, `reactive`, `<script setup>` |
| [vue_05_computed.md](vue_05_computed.md) | Вычисляемые свойства: `computed`, кэширование, writable computed |
| [vue_06_class_style_bindings.md](vue_06_class_style_bindings.md) | Привязка class и style: объектный и массивный синтаксис |
| [vue_07_conditional_list_rendering.md](vue_07_conditional_list_rendering.md) | Условный рендеринг и списки: `v-if`/`v-show`, `v-for`, `key` |
| [vue_08_event_handling.md](vue_08_event_handling.md) | Обработка событий: `v-on`, модификаторы, аргументы |
| [vue_09_form_bindings.md](vue_09_form_bindings.md) | Формы: `v-model`, модификаторы `.lazy/.number/.trim` |
| [vue_10_lifecycle.md](vue_10_lifecycle.md) | Хуки жизненного цикла: `onMounted`, `onUnmounted` и др. |
| [vue_11_watchers.md](vue_11_watchers.md) | Наблюдатели: `watch`, `watchEffect`, опции `deep/immediate/flush` |
| [vue_12_template_refs.md](vue_12_template_refs.md) | Template refs: `useTemplateRef`, доступ к DOM/компонентам |

## Компоненты подробно (Components In-Depth)

| Файл | Тема |
|------|------|
| [vue_13_components_basics.md](vue_13_components_basics.md) | Основы компонентов: регистрация, props/events, динамические компоненты |
| [vue_14_props.md](vue_14_props.md) | Props: объявление, валидация, one-way flow, реактивность |
| [vue_15_component_events.md](vue_15_component_events.md) | События компонентов: `defineEmits`, валидация событий |
| [vue_16_component_v_model.md](vue_16_component_v_model.md) | `v-model` на компонентах: `defineModel`, несколько моделей |
| [vue_17_fallthrough_attributes.md](vue_17_fallthrough_attributes.md) | Fallthrough-атрибуты: `$attrs`, `inheritAttrs` |
| [vue_18_slots.md](vue_18_slots.md) | Слоты: дефолтные, именованные, scoped slots |
| [vue_19_provide_inject.md](vue_19_provide_inject.md) | Provide/Inject: передача данных вглубь дерева, `InjectionKey` |
| [vue_20_async_components.md](vue_20_async_components.md) | Асинхронные компоненты: `defineAsyncComponent`, ленивая загрузка |

## Переиспользование (Reusability)

| Файл | Тема |
|------|------|
| [vue_21_composables.md](vue_21_composables.md) | Composables: переиспользуемая логика, конвенции, vs mixins |
| [vue_22_custom_directives.md](vue_22_custom_directives.md) | Пользовательские директивы: хуки директив |
| [vue_23_plugins.md](vue_23_plugins.md) | Плагины: `app.use`, регистрация глобального функционала |

## Встроенные компоненты (Built-in Components)

| Файл | Тема |
|------|------|
| [vue_24_transitions.md](vue_24_transitions.md) | Анимации: `<Transition>`, `<TransitionGroup>` |
| [vue_25_keepalive_teleport_suspense.md](vue_25_keepalive_teleport_suspense.md) | `<KeepAlive>`, `<Teleport>`, `<Suspense>` |

## Масштабирование (Scaling Up)

| Файл | Тема |
|------|------|
| [vue_26_sfc_tooling.md](vue_26_sfc_tooling.md) | SFC и инструменты: Vite, `vue-tsc`, тулинг |
| [vue_27_routing.md](vue_27_routing.md) | Маршрутизация: Vue Router, динамические маршруты, navigation guards |
| [vue_28_state_management_pinia.md](vue_28_state_management_pinia.md) | Управление состоянием: Pinia, `storeToRefs` |
| [vue_29_testing.md](vue_29_testing.md) | Тестирование: Vitest, Vue Test Utils, e2e |
| [vue_30_ssr.md](vue_30_ssr.md) | SSR: серверный рендеринг, гидрация, Nuxt |

## Лучшие практики (Best Practices)

| Файл | Тема |
|------|------|
| [vue_31_performance.md](vue_31_performance.md) | Производительность: оптимизация рендеринга, ленивая загрузка |
| [vue_32_security.md](vue_32_security.md) | Безопасность: XSS, `v-html`, доверенный контент |
| [vue_33_production_accessibility.md](vue_33_production_accessibility.md) | Продакшн и доступность (a11y): деплой, семантика |

## TypeScript

| Файл | Тема |
|------|------|
| [vue_34_typescript.md](vue_34_typescript.md) | TypeScript во Vue: `defineProps<T>`, `defineEmits<T>`, `InjectionKey`, `InstanceType`, generic-компоненты, типизация Pinia |

## Глубокое погружение (Extra Topics / In-Depth)

| Файл | Тема |
|------|------|
| [vue_35_reactivity_in_depth.md](vue_35_reactivity_in_depth.md) | Реактивность изнутри: Proxy, track/trigger, dep, ReactiveEffect, `shallowRef`/`markRaw`/`customRef`, `nextTick`, `effectScope` |
| [vue_36_rendering_mechanism.md](vue_36_rendering_mechanism.md) | Механизм рендеринга: Virtual DOM, compile→mount→patch, оптимизации компилятора (hoisting, patch flags, blocks), `h()`/JSX, runtime vs compiler сборки |

---

## Топ-темы для собеседования

Темы, которые спрашивают чаще всего. Должны быть отскоком от зубов.

| Тема | Где смотреть | Суть в одной фразе |
|------|--------------|--------------------|
| **`ref` vs `reactive`** | 04, 35 | `ref` — для любых значений (примитивы) через `.value`; `reactive` — только объекты (Proxy), нельзя для примитивов |
| **Потеря реактивности при деструктуризации** | 04, 28, 35 | Деструктуризация `reactive` рвёт связь — нужны `toRefs`/`storeToRefs` |
| **`computed` vs `watch`** | 05, 11 | `computed` — производное кэшируемое значение без сайд-эффектов; `watch` — реакция/сайд-эффекты на изменения |
| **`v-if` vs `v-show`** | 07 | `v-if` создаёт/удаляет DOM (ленив); `v-show` переключает `display` (дешевле переключать) |
| **`key` в `v-for`** | 07, 36 | Стабильный уникальный key нужен для корректного и эффективного patch'а списков |
| **One-way data flow у props** | 14 | Props идут сверху вниз, мутировать props в потомке нельзя — `emit` наверх |
| **Composables vs mixins** | 21 | Composables — явные источники, без коллизий имён и неявной магии mixins |
| **Scoped slots** | 18 | Слоты, которым потомок передаёт данные через `v-slot="{...}"` |
| **Provide/Inject** | 19, 34 | Передача зависимостей вглубь дерева минуя props; типизация через `InjectionKey` |
| **Pinia / `storeToRefs`** | 28, 34 | Стор на Composition API; `storeToRefs` сохраняет реактивность при деструктуризации state/getters |
| **Navigation guards** | 27 | `beforeEach`/`beforeEnter`/`beforeRouteLeave` — защита и контроль навигации |
| **Lifecycle** | 10 | `onMounted` (DOM готов), `onUnmounted` (очистка), порядок выполнения |
| **Virtual DOM / реактивность изнутри** | 35, 36 | Proxy + track/trigger; compiler-informed VDOM (patch flags, hoisting, blocks) |

---

## Рекомендованный порядок изучения

1. **Основы** (01 → 12): шаблоны, реактивность, computed, условия/списки, события,
   формы, lifecycle, watchers, refs. Фундамент — без него дальше тяжело.
2. **Компоненты** (13 → 20): props, события, `v-model`, слоты, provide/inject,
   fallthrough, async-компоненты. Это ядро повседневной работы.
3. **Переиспользование** (21 → 23): composables (must-have для senior), директивы,
   плагины.
4. **Встроенные компоненты** (24 → 25): анимации, `KeepAlive`/`Teleport`/`Suspense`.
5. **Масштабирование** (26 → 30): тулинг, роутинг, Pinia, тестирование, SSR.
6. **Лучшие практики** (31 → 33): производительность, безопасность, продакшн/a11y.
7. **TypeScript** (34): параллельно со 2-3 этапами, если проект на TS.
8. **Глубокое погружение** (35 → 36): реактивность изнутри и механизм рендеринга —
   именно эти темы отделяют middle от senior на собеседовании.

> Совет: после каждого раздела прорешивайте блок "Частые вопросы на собеседовании" и
> "Подводные камни" — они смоделированы под реальные интервью.
