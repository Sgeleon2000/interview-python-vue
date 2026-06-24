# Vue 3 — Маршрутизация (Vue Router) — конспект и вопросы

## О чём раздел

Vue Router — официальная библиотека маршрутизации для Vue. Она превращает приложение в **SPA** (Single-Page Application): при переходах между «страницами» браузер не перезагружает документ — Vue Router сопоставляет URL с компонентом и подменяет его внутри `<router-view>`.

Это **топовая тема** собеседований по фронтенду. Разбираем: установку и создание роутера, режимы истории, определение маршрутов, навигацию (декларативную и программную), динамические сегменты, вложенные/именованные маршруты, query/params, **navigation guards** (защиту маршрутов), **ленивую загрузку** (code splitting), meta-поля, редиректы, alias и scroll-поведение.

## Ключевые концепции

- **SPA-роутер** — управляет URL и рендером компонентов на клиенте без полной перезагрузки страницы.
- **`createRouter`** — создаёт экземпляр роутера; принимает `history` и массив `routes`.
- **`createWebHistory`** — режим HTML5 History API: «красивые» URL (`/about`), требует настройки сервера на fallback. **`createWebHashHistory`** — URL с `#` (`/#/about`), работает без серверной настройки.
- **`<router-link>`** — декларативная навигация (рендерит `<a>`, перехватывает клик).
- **`<router-view>`** — точка вывода компонента текущего маршрута.
- **`useRoute()`** — доступ к текущему маршруту (params, query, meta...). **`useRouter()`** — доступ к экземпляру роутера для программной навигации.
- **Динамические сегменты** — `/user/:id`, доступ через `route.params.id`.
- **Вложенные маршруты** — `children`, рендерятся во вложенном `<router-view>`.
- **Navigation guards** — хуки `beforeEach`/`beforeEnter`/`beforeRouteEnter` и др. для проверки доступа/редиректов.
- **Ленивая загрузка** — `component: () => import('...')` для разбиения бандла (code splitting).
- **meta** — произвольные данные на маршруте (`requiresAuth`, заголовок и т.д.).
- **redirect / alias** — перенаправление и альтернативные пути.
- **scrollBehavior** — управление прокруткой при навигации.

## Подробный разбор с примерами кода

### 1. Установка и создание роутера

```bash
npm install vue-router
```

```js
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'
import About from '../views/About.vue'

const routes = [
  { path: '/', name: 'home', component: Home },
  { path: '/about', name: 'about', component: About },
]

const router = createRouter({
  // Режим истории браузера (HTML5 History API)
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
})

export default router
```

```js
// src/main.js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App)
  .use(router) // подключаем роутер как плагин
  .mount('#app')
```

### 2. createWebHistory vs createWebHashHistory

```js
import { createWebHistory, createWebHashHistory, createMemoryHistory } from 'vue-router'

createWebHistory()       // /about      — нужен fallback на сервере (SPA rewrite)
createWebHashHistory()   // /#/about    — работает без настройки сервера, хуже для SEO
createMemoryHistory()    // без URL     — для SSR / тестов
```

- **`createWebHistory`** — рекомендуется. «Чистые» URL. Сервер должен отдавать `index.html` на любой путь (иначе 404 при прямом заходе/обновлении).
- **`createWebHashHistory`** — когда нет контроля над сервером (статический хостинг без rewrite-правил). URL содержит `#`, не отправляется на сервер.

### 3. router-view и router-link

```vue
<!-- App.vue -->
<template>
  <nav>
    <!-- Декларативная навигация. active-class навешивается автоматически -->
    <router-link to="/">Главная</router-link>
    <router-link :to="{ name: 'about' }">О нас</router-link>
    <router-link :to="{ name: 'user', params: { id: 42 } }">Профиль 42</router-link>
  </nav>

  <!-- Сюда рендерится компонент текущего маршрута -->
  <router-view />
</template>
```

`<router-link>` поддерживает `replace`, `active-class`, `exact-active-class`, кастомный рендер через `v-slot` (`custom`).

### 4. Динамические сегменты и useRoute()

```js
const routes = [
  // :id — динамический сегмент
  { path: '/user/:id', name: 'user', component: () => import('../views/User.vue') },
  // несколько сегментов и опциональный
  { path: '/posts/:category/:id?', component: Post },
  // повторяющийся параметр (массив)
  { path: '/files/:pathMatch(.*)*', component: NotFound }, // catch-all / 404
]
```

```vue
<!-- User.vue -->
<script setup>
import { useRoute, watch } from 'vue-router' // watch — из 'vue'
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
// Доступ к параметру
console.log(route.params.id)

// ВАЖНО: при переходе /user/1 -> /user/2 компонент НЕ пересоздаётся.
// Реагируем на смену параметра через watch:
watch(
  () => route.params.id,
  (newId, oldId) => {
    console.log(`id изменился: ${oldId} -> ${newId}`)
  }
)
</script>

<template>
  <p>Пользователь: {{ route.params.id }}</p>
</template>
```

### 5. Программная навигация useRouter()

```vue
<script setup>
import { useRouter } from 'vue-router'

const router = useRouter()

function goHome() {
  router.push('/')                                 // добавить запись в историю
}
function goUser() {
  router.push({ name: 'user', params: { id: 7 } }) // объект location
}
function search() {
  router.push({ path: '/search', query: { q: 'vue' } }) // /search?q=vue
}
function replaceRoute() {
  router.replace('/login')   // заменить текущую запись (без новой в истории)
}
function goBack() {
  router.go(-1)              // назад; router.go(1) — вперёд
}
</script>
```

- `push` — добавляет запись в стек истории (есть кнопка «назад»).
- `replace` — заменяет текущую запись (полезно после логина).
- `go(n)` — перемещение по истории на n шагов.

### 6. Передача params и props

Чтобы параметры приходили в компонент как props (а не через `route.params`):

```js
const routes = [
  // props: true -> route.params.id придёт как prop id
  { path: '/user/:id', component: User, props: true },

  // функция: гибкая трансформация
  { path: '/search', component: Search, props: (route) => ({ q: route.query.q }) },

  // статические props
  { path: '/promo', component: Promo, props: { theme: 'dark' } },
]
```

```vue
<!-- User.vue -->
<script setup>
defineProps({ id: String }) // получаем напрямую как prop
</script>
```

### 7. Вложенные маршруты

```js
const routes = [
  {
    path: '/user/:id',
    component: UserLayout,
    children: [
      // /user/:id -> рендерится в <router-view> внутри UserLayout
      { path: '', name: 'user-home', component: UserHome },
      // /user/:id/profile
      { path: 'profile', component: UserProfile },
      // /user/:id/posts
      { path: 'posts', component: UserPosts },
    ],
  },
]
```

```vue
<!-- UserLayout.vue -->
<template>
  <h2>Пользователь {{ $route.params.id }}</h2>
  <router-view /> <!-- сюда рендерятся дочерние маршруты -->
</template>
```

### 8. Именованные маршруты и query/params

```js
{ path: '/user/:id', name: 'user', component: User }
```

```js
// Навигация по имени (надёжнее, чем по строковому пути)
router.push({ name: 'user', params: { id: 5 }, query: { tab: 'info' } })
// URL: /user/5?tab=info
```

```js
// Чтение
route.params.id   // '5'
route.query.tab   // 'info'
route.hash        // '#section'
```

> Разница: **params** — часть пути (`:id`), **query** — строка запроса (`?x=y`), **hash** — якорь (`#...`).

### 9. Navigation Guards (защита маршрутов) — ключевая тема

Guards вызываются в определённом порядке и могут:
- разрешить переход (`return true` / ничего не возвращать / вызвать `next()`),
- отменить (`return false`),
- перенаправить (`return { name: 'login' }` / `next('/login')`).

Современный стиль — **возвращать значение** вместо `next` (хотя `next` тоже поддерживается).

#### 9.1. Глобальные guards

```js
// Перед каждым переходом — типичное место проверки авторизации
router.beforeEach((to, from) => {
  const isAuth = !!localStorage.getItem('token')

  // Защита по meta-полю
  if (to.meta.requiresAuth && !isAuth) {
    // Возвращаем location -> произойдёт редирект
    return { name: 'login', query: { redirect: to.fullPath } }
  }
  // ничего не возвращаем -> переход разрешён
})

// beforeResolve — после всех guards, перед подтверждением навигации
router.beforeResolve(async (to) => {
  if (to.meta.requiresData) {
    await fetchSomething()
  }
})

// afterEach — навигация уже произошла (нельзя отменить); удобно для аналитики/title
router.afterEach((to, from) => {
  document.title = to.meta.title ?? 'Моё приложение'
})
```

Сигнатура с `next` (классический вид):

```js
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !isAuth) {
    next({ name: 'login' }) // редирект
  } else {
    next()                  // продолжить (вызвать ОБЯЗАТЕЛЬНО ровно один раз)
  }
})
```

#### 9.2. Per-route guard: beforeEnter

```js
const routes = [
  {
    path: '/admin',
    component: Admin,
    // Срабатывает только при ВХОДЕ на этот маршрут
    // (не при смене params того же маршрута)
    beforeEnter: (to, from) => {
      if (!isAdmin()) return { name: 'home' }
    },
  },
  // Можно массив guards
  { path: '/secret', component: Secret, beforeEnter: [checkAuth, checkRole] },
]
```

#### 9.3. Per-component guards

В `<script setup>` доступны `onBeforeRouteUpdate` и `onBeforeRouteLeave`. Аналог `beforeRouteEnter` реализуется через глобальные guards или Options API.

```vue
<script setup>
import { onBeforeRouteUpdate, onBeforeRouteLeave } from 'vue-router'

// Вызывается при изменении маршрута, на котором рендерится этот компонент
// (например /user/1 -> /user/2). Компонент переиспользуется!
onBeforeRouteUpdate(async (to, from) => {
  // Здесь уже есть доступ к this/состоянию компонента
  await loadUser(to.params.id)
})

// Вызывается при уходе с маршрута — удобно спросить «есть несохранённые изменения?»
onBeforeRouteLeave((to, from) => {
  const ok = window.confirm('Покинуть страницу? Изменения не сохранены.')
  if (!ok) return false // отменить навигацию
})
</script>
```

В Options API доступен и `beforeRouteEnter`:

```js
export default {
  // ВНИМАНИЕ: компонент ещё НЕ создан -> нет доступа к this.
  // Доступ к instance — через колбэк next:
  beforeRouteEnter(to, from, next) {
    next((vm) => {
      // vm — инстанс компонента
      vm.userId = to.params.id
    })
  },
  beforeRouteUpdate(to, from) { /* this доступен */ },
  beforeRouteLeave(to, from) { /* this доступен */ },
}
```

#### 9.4. Полный порядок навигации (важно знать на собесе)

1. Навигация запущена.
2. `beforeRouteLeave` в деактивируемых компонентах.
3. Глобальные `beforeEach`.
4. `beforeRouteUpdate` в переиспользуемых компонентах.
5. `beforeEnter` в конфигурации маршрута.
6. Разрешение async-компонентов (ленивая загрузка).
7. `beforeRouteEnter` в активируемых компонентах.
8. Глобальные `beforeResolve`.
9. Навигация подтверждена.
10. Глобальные `afterEach`.
11. DOM обновлён.
12. Вызываются колбэки из `next(vm => ...)` в `beforeRouteEnter`.

### 10. Ленивая загрузка маршрутов (code splitting) — ключевая тема

Вместо статического импорта компонента используют **динамический import**. Сборщик (Vite/Webpack) выделит компонент в отдельный чанк, который загрузится только при переходе на маршрут — уменьшает первоначальный бандл.

```js
const routes = [
  // Загрузится только при переходе на /about
  {
    path: '/about',
    component: () => import('../views/About.vue'),
  },
  // Группировка нескольких маршрутов в один чанк (Webpack magic comment):
  {
    path: '/settings',
    component: () => import(/* webpackChunkName: "user" */ '../views/Settings.vue'),
  },
]
```

Можно комбинировать с `defineAsyncComponent` для тонкой настройки (loading/error компоненты), но для маршрутов достаточно функции-импорта.

```js
// Группировка через общую функцию (одинаковая строка пути -> один чанк в Vite)
const UserViews = () => import('../views/UserViews.vue')
```

> Почему важно: уменьшает Time-To-Interactive, грузит код «по требованию». На собеседовании частый вопрос «как оптимизировать загрузку SPA» — ответ: ленивые маршруты + code splitting.

### 11. meta-поля

```js
const routes = [
  {
    path: '/dashboard',
    component: Dashboard,
    meta: { requiresAuth: true, title: 'Панель', roles: ['admin'] },
  },
]
```

```js
// Доступ в guard или компоненте
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isAuth()) return { name: 'login' }
})
```

`to.meta` агрегирует meta всех совпавших (включая родительские) маршрутов — удобно для вложенных лейаутов.

### 12. Редиректы и alias

```js
const routes = [
  // Простой редирект
  { path: '/home', redirect: '/' },

  // Редирект на именованный маршрут
  { path: '/old-user/:id', redirect: { name: 'user' } },

  // Динамический редирект (функция)
  { path: '/go', redirect: (to) => ({ path: '/search', query: { q: to.query.term } }) },

  // alias: один компонент доступен по нескольким путям БЕЗ смены URL
  { path: '/', component: Home, alias: ['/home', '/start'] },
]
```

> Разница: **redirect** меняет URL (история ведёт на целевой путь). **alias** оставляет URL как есть, но рендерит тот же маршрут.

### 13. scrollBehavior

```js
const router = createRouter({
  history: createWebHistory(),
  routes,
  // Управление прокруткой при навигации
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition          // при «назад» вернуть прежнюю позицию
    }
    if (to.hash) {
      return { el: to.hash, behavior: 'smooth' } // к якорю
    }
    return { top: 0 }               // иначе — наверх
  },
})
```

Можно вернуть промис для отложенной прокрутки (после анимации перехода).

## Полный рабочий пример

```js
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'

const routes = [
  { path: '/', name: 'home', component: Home, meta: { title: 'Главная' } },

  // Ленивая загрузка
  {
    path: '/about',
    name: 'about',
    component: () => import('../views/About.vue'),
    meta: { title: 'О нас' },
  },

  // Вложенные + динамический сегмент + props
  {
    path: '/user/:id',
    component: () => import('../views/UserLayout.vue'),
    props: true,
    meta: { requiresAuth: true },
    children: [
      { path: '', name: 'user', component: () => import('../views/UserHome.vue') },
      { path: 'profile', name: 'user-profile', component: () => import('../views/UserProfile.vue') },
    ],
  },

  {
    path: '/login',
    name: 'login',
    component: () => import('../views/Login.vue'),
  },

  // Редирект и alias
  { path: '/home', redirect: { name: 'home' } },

  // Catch-all 404
  {
    path: '/:pathMatch(.*)*',
    name: 'not-found',
    component: () => import('../views/NotFound.vue'),
  },
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    return savedPosition ?? { top: 0 }
  },
})

// Глобальная защита по авторизации
router.beforeEach((to) => {
  const isAuth = !!localStorage.getItem('token')
  if (to.meta.requiresAuth && !isAuth) {
    return { name: 'login', query: { redirect: to.fullPath } }
  }
})

// Обновление заголовка вкладки
router.afterEach((to) => {
  document.title = to.meta.title ?? 'Приложение'
})

export default router
```

```vue
<!-- views/Login.vue -->
<script setup>
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

function login() {
  localStorage.setItem('token', 'abc')
  // Возврат на исходную страницу после логина
  const redirect = route.query.redirect ?? '/'
  router.replace(redirect)
}
</script>

<template>
  <button @click="login">Войти</button>
</template>
```

```vue
<!-- views/UserLayout.vue -->
<script setup>
import { onBeforeRouteUpdate } from 'vue-router'

const props = defineProps({ id: String })

// Реагируем на смену /user/1 -> /user/2 (компонент переиспользуется)
onBeforeRouteUpdate((to) => {
  console.log('Загружаем пользователя', to.params.id)
})
</script>

<template>
  <h2>Пользователь {{ id }}</h2>
  <nav>
    <router-link :to="{ name: 'user', params: { id } }">Главная</router-link>
    <router-link :to="{ name: 'user-profile', params: { id } }">Профиль</router-link>
  </nav>
  <router-view />
</template>
```

## Частые вопросы на собеседовании (Q/A)

**Q1: Что делает Vue Router и зачем нужен SPA-роутер?**
A: Сопоставляет URL с компонентами и подменяет их в `<router-view>` без перезагрузки страницы. Даёт навигацию, историю, deep-linking, защиту маршрутов — всё на клиенте.

**Q2: В чём разница `createWebHistory` и `createWebHashHistory`?**
A: `createWebHistory` использует HTML5 History API — «чистые» URL (`/about`), но сервер должен отдавать `index.html` на любой путь (иначе 404 при обновлении). `createWebHashHistory` кладёт маршрут после `#` (`/#/about`) — работает без серверной настройки, но хуже для SEO.

**Q3: Чем `router-link` лучше обычного `<a>`?**
A: `router-link` перехватывает клик, делает навигацию без перезагрузки, добавляет active-классы, поддерживает объектные `:to`. Обычный `<a href>` вызвал бы полную перезагрузку.

**Q4: Как получить параметр маршрута внутри компонента?**
A: Через `const route = useRoute(); route.params.id`. Либо включить `props: true` и принимать `id` как prop.

**Q5: В чём разница `useRoute()` и `useRouter()`?**
A: `useRoute()` — реактивный объект ТЕКУЩЕГО маршрута (params, query, meta). `useRouter()` — экземпляр роутера для навигации (`push`, `replace`, `go`).

**Q6: Разница `router.push` и `router.replace`?**
A: `push` добавляет запись в историю (работает «назад»). `replace` заменяет текущую запись — полезно после логина, чтобы кнопка «назад» не вернула на форму входа.

**Q7: Почему при переходе `/user/1 → /user/2` не срабатывают `created`/`onMounted` заново?**
A: Это один и тот же маршрут — компонент переиспользуется ради производительности. Реагировать на смену параметра нужно через `watch(() => route.params.id, ...)` или `onBeforeRouteUpdate`.

**Q8: Что такое navigation guards и какие бывают?**
A: Хуки для контроля переходов. Глобальные: `beforeEach`, `beforeResolve`, `afterEach`. Per-route: `beforeEnter`. Per-component: `beforeRouteEnter`, `beforeRouteUpdate`, `beforeRouteLeave` (в setup — `onBeforeRouteUpdate`/`onBeforeRouteLeave`).

**Q9: Как guard разрешает/отменяет/перенаправляет переход?**
A: Современно — возвратом значения: `true`/ничего = разрешить, `false` = отменить, объект location = редирект. Классически — через `next()`, `next(false)`, `next('/login')`, причём `next` нужно вызвать ровно один раз.

**Q10: Каков порядок срабатывания guards?**
A: `beforeRouteLeave` → глобальный `beforeEach` → `beforeRouteUpdate` → `beforeEnter` → резолв async-компонентов → `beforeRouteEnter` → `beforeResolve` → навигация подтверждена → `afterEach` → DOM обновлён → колбэки `next(vm=>...)`.

**Q11: Почему в `beforeRouteEnter` нет `this`?**
A: Компонент ещё не создан на момент входа. Доступ к инстансу получают через колбэк: `next(vm => { /* vm — инстанс */ })`.

**Q12: Как реализовать защиту авторизации?**
A: Помечают маршруты `meta.requiresAuth: true` и в `beforeEach` проверяют токен; при отсутствии — редирект на `/login` с `query.redirect` для возврата.

**Q13: Что такое ленивая загрузка маршрутов и зачем она нужна?**
A: `component: () => import('...')` — динамический импорт, который сборщик выделяет в отдельный чанк. Чанк грузится только при переходе на маршрут, уменьшая первоначальный бандл и ускоряя загрузку приложения.

**Q14: Как сгруппировать несколько маршрутов в один чанк?**
A: Через общую функцию-импорт или Webpack magic comment `/* webpackChunkName: "group" */`. В Vite одинаковая строка импорта попадает в общий чанк автоматически.

**Q15: Чем отличаются `redirect` и `alias`?**
A: `redirect` меняет URL — пользователь оказывается на целевом пути. `alias` оставляет URL как есть, но рендерит тот же маршрут (один компонент доступен по нескольким путям).

**Q16: Как настроить прокрутку наверх при переходах?**
A: Опция `scrollBehavior(to, from, savedPosition)` в `createRouter`. Возврат `savedPosition` восстанавливает позицию при «назад», `{ top: 0 }` прокручивает наверх, `{ el: to.hash }` — к якорю.

**Q17 (бонус): Как сделать страницу 404?**
A: Catch-all маршрут `path: '/:pathMatch(.*)*'` с компонентом NotFound в конце списка маршрутов.

## Подводные камни (gotchas)

- **`createWebHistory` без серверного fallback** → 404 при прямом заходе/обновлении страницы. Настройте rewrite на `index.html`.
- **Компонент не пересоздаётся** при смене динамического сегмента — используйте `watch` на `route.params` или `onBeforeRouteUpdate`, иначе данные «застрянут».
- **`next` нужно вызвать ровно один раз** в классическом guard — двойной вызов или его отсутствие ломает навигацию (зависает). Современный возврат значения безопаснее.
- **`beforeRouteEnter` без `this`** — частая ловушка; используйте `next(vm => ...)`.
- **`params` без совпадающего сегмента в пути игнорируются** при `push({ name, params })` — параметр должен присутствовать в `path` (`/user/:id`). Для произвольных данных используйте `query` или state.
- **Реактивность `route`**: деструктуризация `const { params } = useRoute()` теряет реактивность — обращайтесь как `route.params`.
- **Порядок маршрутов** для catch-all/404: должен идти последним; более специфичные пути — выше.
- **alias и SEO**: дублирование контента по нескольким URL может вредить SEO (нужен canonical).
- **Ленивая загрузка ломает SSR-чанки** при неверной настройке — проверяйте, что сборщик корректно разбивает чанки.

## Лучшие практики

- Используйте `createWebHistory` + серверный fallback для «чистых» URL и SEO.
- Применяйте **ленивую загрузку** для всех неглавных маршрутов (особенно тяжёлых).
- Навигируйте по **именованным маршрутам** (`{ name }`) — устойчиво к изменению путей.
- Авторизацию и доступ выносите в `beforeEach` + `meta.requiresAuth`/`roles`.
- Для реакции на смену параметров используйте `onBeforeRouteUpdate`/`watch`, а не повторный mount.
- Используйте `props: true`, чтобы компоненты не зависели жёстко от `route` (легче тестировать).
- Заголовок страницы и аналитику ставьте в `afterEach`.
- 404 — через catch-all `/:pathMatch(.*)*` в конце.
- Сохраняйте/восстанавливайте прокрутку через `scrollBehavior`.

## Шпаргалка

```js
// Создание
import { createRouter, createWebHistory } from 'vue-router'
const router = createRouter({ history: createWebHistory(), routes })

// Маршрут
{ path: '/user/:id', name: 'user', component: () => import('./User.vue'),
  props: true, meta: { requiresAuth: true }, redirect, alias, beforeEnter,
  children: [ /* вложенные */ ] }

// Навигация
router.push('/x'); router.push({ name, params, query })
router.replace('/login'); router.go(-1)

// В компоненте
const route  = useRoute()    // route.params / route.query / route.meta / route.hash
const router = useRouter()   // push / replace / go

// Guards
router.beforeEach((to, from) => true | false | { name: 'login' })
router.beforeResolve(async to => {})
router.afterEach((to, from) => {})            // нельзя отменить
beforeEnter: (to, from) => {}                 // per-route
onBeforeRouteUpdate((to, from) => {})         // смена params
onBeforeRouteLeave((to, from) => false)       // уход со страницы
```

| Тема              | Суть                                              |
|-------------------|---------------------------------------------------|
| history           | `createWebHistory` (чистые URL) / `WebHash` (#)   |
| router-link/view  | декларативная навигация / точка вывода            |
| useRoute          | текущий маршрут (params, query, meta)             |
| useRouter         | навигация (push/replace/go)                       |
| params vs query   | часть пути (`:id`) vs `?x=y`                       |
| guards            | beforeEach/beforeEnter/beforeRouteUpdate/Leave    |
| lazy loading      | `() => import('...')` → отдельный чанк            |
| meta              | произвольные данные маршрута                       |
| redirect vs alias | меняет URL / оставляет URL                        |
| scrollBehavior    | управление прокруткой при переходах               |

| Порядок guards (кратко)                                              |
|---------------------------------------------------------------------|
| Leave → beforeEach → Update → beforeEnter → async → Enter →         |
| beforeResolve → подтверждено → afterEach → DOM → next(vm=>)         |
```
