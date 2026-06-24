# FastAPI — Background Tasks (фоновые задачи) — конспект и вопросы

## О чём раздел

Фоновые задачи (`BackgroundTasks`) — это встроенный механизм FastAPI, позволяющий
запустить какую-то работу **после того, как ответ уже отправлен клиенту**. Клиент
не ждёт завершения этой работы — он получает HTTP-ответ сразу, а задача выполняется
«в фоне» в том же процессе приложения.

Типичные сценарии:
- отправка письма (email) после регистрации;
- запись логов / аудита;
- инвалидация кеша;
- отправка уведомлений (push, webhook);
- лёгкая постобработка загруженного файла.

Важно понимать границу применимости: `BackgroundTasks` — это **простой** механизм
«сделай после ответа в том же процессе». Для тяжёлой, длительной, надёжной или
распределённой обработки нужен полноценный брокер задач (Celery / ARQ / RQ / Dramatiq).

## Ключевые концепции

- **`BackgroundTasks`** — класс из `fastapi` (реэкспорт из Starlette). Объявляется как
  параметр функции-обработчика; FastAPI сам внедрит (inject) экземпляр.
- **`add_task(func, *args, **kwargs)`** — регистрирует функцию для выполнения после ответа.
  Можно добавить несколько задач — они выполнятся **последовательно** в порядке добавления.
- **Когда выполняется** — после того как Starlette сформировал и отправил ответ
  (точнее, после того как тело ответа отдано, в рамках завершения обработки запроса).
- **Sync и async функции** — `add_task` принимает и обычные `def`, и `async def`.
  Обычные `def` выполняются в пуле потоков (threadpool), `async def` — в основном цикле.
- **Зависимости** — `BackgroundTasks` можно принимать не только в эндпоинте, но и
  в зависимостях (`Depends`); задачи, добавленные в зависимости, тоже выполнятся после ответа.
- **Тот же процесс** — задачи не переживают перезапуск/падение процесса, не масштабируются
  на другие воркеры, конкурируют за ресурсы (CPU, event loop) с обработкой запросов.

## Подробный разбор с примерами кода

### Базовый пример: запись лога после ответа

```python
from fastapi import BackgroundTasks, FastAPI
from typing import Annotated

app = FastAPI()


def write_log(message: str) -> None:
    # Обычная синхронная функция — выполнится в threadpool.
    # Пишем в файл уже ПОСЛЕ того, как клиент получил ответ.
    with open("log.txt", mode="a", encoding="utf-8") as log_file:
        log_file.write(message + "\n")


@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    # Регистрируем задачу: она НЕ выполнится прямо сейчас,
    # а будет запущена после отправки ответа клиенту.
    background_tasks.add_task(write_log, f"Уведомление отправлено на {email}")
    # Клиент мгновенно получит этот ответ, не дожидаясь записи в файл.
    return {"message": "Уведомление поставлено в очередь"}
```

### Несколько задач и порядок выполнения

```python
@app.post("/order/{order_id}")
async def create_order(order_id: int, background_tasks: BackgroundTasks):
    # Задачи выполнятся ПОСЛЕДОВАТЕЛЬНО в порядке add_task:
    background_tasks.add_task(write_log, f"Заказ {order_id} создан")      # 1
    background_tasks.add_task(write_log, f"Заказ {order_id}: уведомление")  # 2
    background_tasks.add_task(write_log, f"Заказ {order_id}: метрика")      # 3
    return {"order_id": order_id, "status": "created"}
```

### Async-задача: отправка email через aiosmtplib

```python
import aiosmtplib
from email.message import EmailMessage


async def send_email(to: str, subject: str, body: str) -> None:
    # async def задача выполнится в основном event loop ПОСЛЕ ответа.
    message = EmailMessage()
    message["From"] = "noreply@example.com"
    message["To"] = to
    message["Subject"] = subject
    message.set_content(body)
    # Реальная отправка по SMTP (может занять секунды — клиент этого не ждёт).
    await aiosmtplib.send(message, hostname="smtp.example.com", port=587)


@app.post("/register")
async def register(email: str, background_tasks: BackgroundTasks):
    # ... здесь логика создания пользователя в БД ...
    background_tasks.add_task(
        send_email,
        to=email,
        subject="Добро пожаловать!",
        body="Спасибо за регистрацию.",
    )
    return {"status": "ok", "email": email}
```

### Передача `BackgroundTasks` в зависимость

`BackgroundTasks` можно принимать прямо в зависимости. FastAPI «подхватит» один и
тот же объект задач для всего запроса — задачи из зависимостей и из эндпоинта
сложатся в общий список.

```python
from fastapi import Depends


def get_query(
    background_tasks: BackgroundTasks,
    q: str | None = None,
) -> str | None:
    # Если был передан параметр q — логируем его в фоне.
    if q:
        background_tasks.add_task(write_log, f"Поисковый запрос: {q}")
    return q


@app.get("/search")
async def search(
    background_tasks: BackgroundTasks,
    q: Annotated[str | None, Depends(get_query)] = None,
):
    # Эта задача добавится ПОСЛЕ задачи из зависимости.
    background_tasks.add_task(write_log, "Эндпоинт /search вызван")
    return {"q": q}
```

### Инвалидация кеша после изменения данных

```python
# Условный клиент Redis (асинхронный).
from redis.asyncio import Redis

redis: Redis = Redis(host="localhost", port=6379, decode_responses=True)


async def invalidate_cache(key: str) -> None:
    # Сбрасываем кеш в фоне, чтобы не задерживать ответ на запись.
    await redis.delete(key)


@app.put("/products/{product_id}")
async def update_product(
    product_id: int,
    background_tasks: BackgroundTasks,
):
    # ... обновляем продукт в БД ...
    # Кеш списка/карточки сбросим уже после ответа.
    background_tasks.add_task(invalidate_cache, f"product:{product_id}")
    background_tasks.add_task(invalidate_cache, "products:list")
    return {"product_id": product_id, "updated": True}
```

### Возврат `BackgroundTask` через Response (низкоуровнево)

Иногда задачу привязывают напрямую к объекту `Response` (одна задача, без injection):

```python
from starlette.background import BackgroundTask
from fastapi.responses import JSONResponse


@app.post("/legacy-notify")
async def legacy_notify(email: str):
    task = BackgroundTask(write_log, f"legacy notify {email}")
    return JSONResponse({"ok": True}, background=task)
```

На практике в FastAPI предпочтительнее параметр `background_tasks: BackgroundTasks`.

## Полный рабочий пример

```python
from typing import Annotated

from fastapi import BackgroundTasks, Depends, FastAPI
from pydantic import BaseModel, EmailStr

app = FastAPI(title="Background Tasks Demo")


# --- Имитация сервисов (БД, почта, кеш) ---

async def fake_save_user(email: str) -> int:
    # Имитируем запись в БД и возвращаем id.
    return 42


def write_audit_log(message: str) -> None:
    # Синхронная задача (выполнится в threadpool).
    with open("audit.log", mode="a", encoding="utf-8") as f:
        f.write(message + "\n")


async def send_welcome_email(email: str) -> None:
    # Асинхронная задача (выполнится в event loop).
    # Здесь была бы реальная отправка письма.
    print(f"[email] Отправлено приветственное письмо на {email}")


async def warm_up_cache(user_id: int) -> None:
    # Фоновый прогрев кеша профиля.
    print(f"[cache] Прогрет кеш профиля user_id={user_id}")


# --- Зависимость, добавляющая фоновую задачу ---

def audit_dependency(background_tasks: BackgroundTasks) -> None:
    # Любой запрос, использующий эту зависимость, оставит запись в аудите.
    background_tasks.add_task(write_audit_log, "Запрос на регистрацию получен")


class RegisterIn(BaseModel):
    email: EmailStr


class RegisterOut(BaseModel):
    user_id: int
    email: EmailStr
    status: str


@app.post("/register", response_model=RegisterOut)
async def register(
    payload: RegisterIn,
    background_tasks: BackgroundTasks,
    _: Annotated[None, Depends(audit_dependency)],
) -> RegisterOut:
    # 1) Синхронная работа, которую клиент ДОЛЖЕН дождаться:
    user_id = await fake_save_user(payload.email)

    # 2) Фоновые задачи — выполнятся ПОСЛЕ ответа, в порядке добавления:
    background_tasks.add_task(write_audit_log, f"Создан пользователь {user_id}")
    background_tasks.add_task(send_welcome_email, payload.email)
    background_tasks.add_task(warm_up_cache, user_id)

    # Клиент получает ответ немедленно.
    return RegisterOut(user_id=user_id, email=payload.email, status="created")


if __name__ == "__main__":
    import uvicorn

    uvicorn.run("main:app", host="127.0.0.1", port=8000, reload=True)
```

## Частые вопросы на собеседовании (Q/A)

**1. Когда именно выполняются фоновые задачи?**
После того как ответ сформирован и отправлен клиенту (на этапе завершения обработки
запроса в Starlette). Клиент не ждёт их завершения.

**2. В каком порядке выполняются несколько задач?**
Последовательно, в порядке вызовов `add_task`. Параллелизма между ними нет.

**3. Можно ли использовать обычные `def` и `async def`?**
Да. `async def` выполняются в основном event loop, обычные `def` — в пуле потоков
(threadpool), чтобы не блокировать цикл.

**4. Чем `BackgroundTasks` отличается от Celery/ARQ/RQ?**
`BackgroundTasks` работает в том же процессе, без брокера, без персистентности,
без ретраев, без распределённого масштабирования. Celery/ARQ/RQ — это отдельные
воркеры, брокер сообщений (Redis/RabbitMQ), очереди, повторные попытки, расписания,
мониторинг и устойчивость к перезапускам.

**5. Когда нужен полноценный брокер задач, а не BackgroundTasks?**
Когда задача: длительная (минуты/часы), ресурсоёмкая (CPU), требует гарантий доставки
и ретраев, должна переживать перезапуск приложения, должна масштабироваться на отдельные
воркеры, требует расписания (cron) или приоритетов, может «положить» веб-процесс.

**6. Переживут ли задачи перезапуск или падение процесса?**
Нет. Они в памяти текущего процесса. Перезапуск/краш — задачи теряются. Нет персистентности.

**7. Можно ли добавлять задачи внутри зависимостей?**
Да. Достаточно принять `background_tasks: BackgroundTasks` в зависимости — это тот же
объект, что и в эндпоинте; задачи объединяются в общий список.

**8. Будут ли задачи блокировать обработку других запросов?**
Async-задачи делят event loop с обработкой запросов; тяжёлая CPU-работа в них может
тормозить весь воркер. Sync-задачи занимают потоки из пула — их тоже ограниченное число.

**9. Что произойдёт с исключением внутри фоновой задачи?**
Ответ клиенту уже отправлен, поэтому клиент об ошибке не узнает. Исключение всплывёт
в логах сервера. Поэтому в фоновых задачах нужен собственный try/except и логирование.

**10. Можно ли вернуть результат фоновой задачи клиенту?**
Нет. Ответ уже ушёл. Если клиенту нужен результат — это не фоновая задача (используйте
обычную обработку либо очередь + отдельный эндпоинт статуса/вебхук).

**11. Работает ли это с несколькими воркерами Uvicorn/Gunicorn?**
Задача выполнится в том воркере, который обработал запрос. Между воркерами нет общей
очереди. Для распределения нагрузки нужен брокер.

**12. Как протестировать фоновую задачу?**
`TestClient` выполняет фоновые задачи синхронно после ответа, поэтому в тестах можно
проверить их эффект (например, что файл/моки были вызваны). Часто задачу мокают.

**13. Можно ли передавать в задачу объекты, зависящие от жизненного цикла запроса
(например, сессию БД из Depends)?**
Осторожно: сессия БД может быть закрыта после ответа (по выходу из зависимости).
Лучше передавать примитивы/идентификаторы, а ресурсы открывать заново внутри задачи.

**14. Есть ли ограничение на число фоновых задач?**
Жёсткого лимита нет, но все они выполняются в одном процессе, поэтому большое число
тяжёлых задач деградирует производительность.

## Подводные камни (gotchas)

- **Нет персистентности и ретраев** — задача потеряется при падении/перезапуске; повторов нет.
- **Тот же процесс** — задачи конкурируют с обработкой запросов за CPU/loop/threadpool.
- **Закрытые ресурсы запроса** — сессия БД/файлы из зависимостей могут быть уже закрыты
  к моменту выполнения задачи. Передавайте id, а ресурс открывайте заново.
- **Тяжёлый CPU в async-задаче** блокирует event loop и тормозит весь воркер.
- **Молчаливые ошибки** — исключение в задаче не дойдёт до клиента; добавляйте логирование.
- **Несколько воркеров** — нет общей очереди между процессами.
- **Не для долгих задач** — для длительных/важных операций берите брокер.

## Лучшие практики

- Используйте `BackgroundTasks` только для **лёгкой** работы «после ответа»: логи, email,
  инвалидация кеша, простые уведомления.
- Передавайте в задачи **примитивы** (id, строки), а не объекты, привязанные к запросу.
- Оборачивайте тело задачи в `try/except` и логируйте ошибки.
- Для CPU-bound и долгих задач — выносите в **очередь** (Celery/ARQ/RQ/Dramatiq) с брокером.
- В тестах проверяйте эффект задачи или мокайте её через `dependency_overrides`/`monkeypatch`.
- Не полагайтесь на фоновые задачи для критичных операций, требующих гарантий доставки.

## Шпаргалка

```python
from fastapi import BackgroundTasks, FastAPI, Depends
from typing import Annotated

app = FastAPI()

# Объявление в эндпоинте
@app.post("/x")
async def handler(bt: BackgroundTasks):
    bt.add_task(func, arg1, kw=val)   # def или async def
    return {"ok": True}               # ответ уходит сразу, задача — после

# В зависимости
def dep(bt: BackgroundTasks): bt.add_task(func, ...)

@app.get("/y")
async def y(_: Annotated[None, Depends(dep)]): ...
```

| Свойство            | BackgroundTasks      | Celery / ARQ / RQ        |
|---------------------|----------------------|--------------------------|
| Процесс             | тот же, что и веб    | отдельные воркеры        |
| Брокер              | не нужен             | нужен (Redis/RabbitMQ)   |
| Персистентность     | нет                  | да                       |
| Ретраи / расписание | нет                  | да                       |
| Масштабирование     | в пределах воркера   | горизонтально            |
| Сложность           | минимальная          | выше                     |
| Для чего            | лёгкие задачи        | тяжёлые/важные задачи     |

Главное правило: **`BackgroundTasks` = «сделай после ответа в том же процессе».**
Нужны гарантии, ретраи, масштаб или долгая работа — берите брокер задач.
