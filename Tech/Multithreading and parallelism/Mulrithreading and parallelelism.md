https://habr.com/ru/articles/517918/

### Механизмы синхронизации (мьютексы, атомики, condition_variable)

#### std::mutex

Базовый блокировщик. Используется для защиты общих данных от одновременного доступа.
```cpp
std::mutex m;
int counter = 0;

void foo() {
    std::lock_guard<std::mutex> lock(m); // RAII блокировка
    ++counter;
}
```

**Подводные камни:**
- Deadlock — взаимная блокировка потоков.
- Livelock — потоки заняты обходом deadlock'а, но не прогрессируют.
- Используй `std::lock` для безопасной фиксации нескольких мьютексов.

#### std::recursive_mutex

Позволяет одному потоку захватывать один и тот же mutex несколько раз. **Старайся избегать**, часто признак плохого дизайна.

#### std::shared_mutex

Дает **разрешение на несколько читателей** и **одного писателя**:

```cpp
std::shared_mutex shm;

void reader() {
    std::shared_lock lock(shm);
}

void writer() {
    std::unique_lock lock(shm);
}

```

#### std::condition_variable

Используется для ожидания определенного состояния (например, пока очередь не станет непустой). Работает **только с mutex + unique_lock**.

```cpp
std::condition_variable cv;
std::mutex m;
bool ready = false;

void waitThread() {
    std::unique_lock<std::mutex> lock(m);
    cv.wait(lock, []{ return ready; }); // атомарно отпускает mutex и засыпает
}

void signalThread() {
    {
        std::lock_guard<std::mutex> lock(m);
        ready = true;
    }
    cv.notify_one();
}

```

#### std::atomic

Используется для **lock-free** программирования. Защищает примитивы без мьютексов.

```cpp
std::atomic<int> counter{0};
void foo() { counter.fetch_add(1); }
```

**Важно:** атомик не защищает составные операции (например, push в вектор), только **сам атомарный объект**.

### future/promise - работа с асинхронностью

Используются для обмена результатом между потоками.

`std::promise` + `std::future`

```cpp
std::promise<int> p;
auto f = p.get_future();

std::thread t([&p]{
    p.set_value(42);
});
t.join();

int result = f.get(); // 42
```

std::async

Упрощает асинхронный вызов без ручного thread:

```cpp
auto f = std::async(std::launch::async, []{
    return 42;
});
cout << f.get();

```

### гонки данных (Data Race, Race Condition)

Data Race - **Незаконный одновременный доступ к одной памяти без синхронизации.** Undefined behavior.

```cpp
int x = 0;
void f() { x++; }  // потенциальный data race при многопоточности
```

Race Condition - **Логическая ошибка**, когда результат зависит от порядка выполнения потоков. Например: проверка + действие не атомарны. 

### std::thread, корутины (stackless/stackfull), fibers - модели исполнения

#### std::thread

Запуск потока:
```cpp
std::thread t([]{ /* работа */ });
t.join();
```
Плюсы – гибко, минусы – вручную управлять.

#### Корутины (C++20)

https://habr.com/ru/articles/850970/

-Лёгкие кооперативные задачи, **stackless**, без отдельного стека. Используют co_await, co_yield.

```cpp
task foo() {
    co_await something();
}
```

#### Fibers (Boost.Fiber, Windows Fibers)

**Stackful** кооперативные задачи: у каждой есть свой стек, в отличие от корутин. Переключение быстрее потока, но медленнее корутин.