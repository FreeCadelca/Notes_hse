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

**Атомарная операция** - это операция, которую невозможно наблюдать в промежуточном состоянии, она либо выполнена либо нет. Атомарные операции могут состоять из нескольких операций. Если говорить про тип std::atomic, то он предоставляет ряд примитивных операций: `load`, `store`, `fetch_add`, `compare_exchange_*` и другие. Последние две операции - это read-modify-write операции, атомарность которых обеспечивается специальными инструкциями процессора.

Используется для **lock-free** программирования. Защищает примитивы без мьютексов.

```cpp
std::atomic<int> counter{0};
void foo() { counter.fetch_add(1); }
```

**Важно:** атомик не защищает составные операции (например, push в вектор), только **сам атомарный объект**.

Виды синхронизаторов в атомике:
- relaxed (гарантирует ТОЛЬКО атомарность, порядок выполнения строк может быть другой)
```cpp
std::string data;
std::atomic<bool> ready{ false };
 
void thread1() {
	data = "very important bytes"; // we have not warranty that this row will compile before next (cpu can change order)
	ready.store(true, std::memory_order_relaxed);
}
 
void thread2() {
	while (!ready.load(std::memory_order_relaxed));
	std::cout << "data is ready: " << data << "\n"; // potentially memory corruption is here
}
```
- release/acquire
что-то между relaxed и seq_cst, потому что используется в паре потоков. Release - там, где мы создаем данные и нам нужно, чтобы все данные перед release были подгруженны и все инструкции выполнены, acquire - там, где мы читаем эти данные.
- sequential consistency (seq_cst, гарантирует, что перед атомиком все строки выполнились, причем в правильном порядке)
```cpp
std::atomic<bool> x, y;
std::atomic<int> z;
 
void thread_write_x() {
	x.store(true, std::memory_order_seq_cst);
}
 
void thread_write_y() {
	y.store(true, std::memory_order_seq_cst);
}
 
void thread_read_x_then_y() {
	while (!x.load(std::memory_order_seq_cst));
	if (y.load(std::memory_order_seq_cst)) {
		++z;
	}
}
 
 
void thread_read_y_then_x() {
	while (!y.load(std::memory_order_seq_cst));
	if (x.load(std::memory_order_seq_cst)) {
		++z;
	}
}
```

### future/promise - работа с асинхронностью

Используются для обмена результатом между потоками.

**`std::future`** Это объект, используемый для получения результата асинхронной операции. Его можно рассматривать как заполнитель для «будущего значения», который позволяет получить результат задачи в нужный момент.

**`std::promise`** Это объект, используемый для установки «будущего значения». Он отвечает за передачу результата связанному с ним `std::future`.

Их рабочий механизм устроен следующим образом:
1. Вы создаёте объект `std::promise`
2. Вы получаете связанный объект `std::future` с помощью метода `std::promise::get_future()`.
3. В другом потоке `std::promise` устанавливает значение, а `std::future` отвечает за асинхронное получение этого значения.

```cpp
#include <iostream>
#include <thread>
#include <future> // Include std::future and std::promise

// Asynchronous task: calculate square
void calculateSquare(std::promise<int> promise, int value) {
    int result = value * value;
    std::this_thread::sleep_for(std::chrono::seconds(2)); // Simulate time-consuming operation
    promise.set_value(result); // Set the calculated result in promise
}

int main() {
    std::promise<int> promise; // Create promise
    std::future<int> future = promise.get_future(); // Get the associated future with promise

    int value = 5;
    std::thread t(calculateSquare, std::move(promise), value); // Start thread, passing promise

    std::cout << "Calculating the square of " << value << "..." << std::endl;

    int result = future.get(); // Get the result of the asynchronous operation (blocking wait)
    std::cout << "Result is: " << result << std::endl;

    t.join(); // Wait for thread to finish
    return 0;
}
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
#include <iostream> 
#include <thread> 

void hello() { 
	std::cout << "Hello from the thread!" << std::endl; 
} 

int main() { 
	std::thread t(hello); // Create and run the thread 
	t.join(); // Wait for the thread to finish 
	return 0; 
}
```

Когда вы создаёте поток, он сразу же начинает выполнять указанную функцию. Однако важно управлять его жизненным циклом, в частности дождаться его завершения. Именно здесь в игру вступает метод `join()`.

Если вы вызовете функцию join(), основной поток будет заблокирован до тех пор, пока не завершится поток, представленный объектом std::thread. С другой стороны, если вы хотите, чтобы поток продолжал работать независимо, вы можете использовать функцию detach().

**Поток** — это легковесный процесс. У каждого процесса может быть один или несколько потоков — это минимальная единица выполнения внутри процесса.

Основные элементы потока:
- **Стек** (место, где хранятся временные данные),
- **Регистры** процессора (для хранения текущих вычислений),
- **ID потока** (идентификатор),
- **Указатель на инструкцию**, которая должна выполниться,
- **Состояние** потока (например, "готов к выполнению" или "ожидание").

#### Корутины (C++20)

https://habr.com/ru/articles/850970/

https://habr.com/ru/companies/wunderfund/articles/582000/

Обычно потоки реализуются на уровне операционной системы (такие потоки называются **потоками ядра**), но у этого подхода есть недостатки:
1. **Высокая стоимость переключения контекста**:  
    - Переключение между потоками требует сложных операций и занимает много ресурсов, что замедляет выполнение программы.
2. **Зависимость от ядра**:  
    - Если поток блокируется (например, ожидает завершения ввода-вывода), то ядро передаёт управление другому потоку. Эффективность работы зависит от того, насколько хорошо ядро управляет потоками.
3. **Ограниченная масштабируемость**:  
    - Если процесс создаёт много потоков, это увеличивает нагрузку на операционную систему и её планировщик, что может привести к снижению производительности.

Для решения этих проблем были созданы потоки в пользовательском пространстве
В этом подходе управление потоками выполняется не операционной системой, а на уровне пользовательских библиотек. Это позволяет переключаться между потоками без участия ядра ОС, что делает процесс более быстрым.
Есть разные виды потоков в пользовательском пространстве, такие как **fibers**, **green threads** и **корутины** (именно о них пойдёт речь в этой статье).

В общих чертах, **корутины** — это специальные функции, которые позволяют приостановить выполнение в одном месте и продолжить его с того же места позже

Есть разные модели мультиплексирования корутин

![](./Materials/Mulrithreading%20and%20parallelelism-1760951017889.jpeg)

Существуют два основных подхода для реализации корутин: stackful и stackless.
1. **Stackful корутины**:
    Эти корутины имеют собственный стек. Это значит, что сами корутины управляют своим выполнением, а не зависят от операционной системы. Благодаря наличию собственного стека, такие корутины могут хранить больше информации: они запоминают не только текущее состояние функции, но и иерархию вложенных вызовов других функций, которые были вызваны внутри этой корутины. Это позволяет, например, приостановить выполнение одной корутины на любом уровне вызова функций и передать управление другой корутине.
2. **Stackless корутины**:
    В отличие от stackful корутин, stackless корутины не имеют собственного стека. Их выполнение управляется как **машина состояний** (state machine), которую генерирует компилятор. Это выглядит так, как будто выполнение функции просто переключается между разными состояниями — например, между ожиданием данных и продолжением выполнения после их получения.
    
На практике stackless корутины чаще всего реализуются через ключевые слова вроде `async`/`await`. Когда функция встречает `await`, она приостанавливается до тех пор, пока не будет готов результат, после чего продолжает выполнение.

В C++ корутины - лёгкие кооперативные задачи, **stackless**, без отдельного стека. Используют co_await, co_yield, co_return.

```cpp
#include <coroutine>

X coroutine(){
    co_yield "Hello ";
    co_yield "world";
    co_return "!";
}

task foo() {
    co_await something();
}

int main() {
	auto x = coroutine();
	std::cout << x.next();
	std::cout << x.next();
	std::cout << x.next();
	std::cout << std::endl;
}
```

#### Fibers (Boost.Fiber, Windows Fibers)

**Stackful** корутина - кооперативные задачи: у каждой есть свой стек, в отличие от корутин. Переключение быстрее потока, но медленнее корутин.