### [Основы языка и STL](Basics/Basics.md) +
- Базовые контейнеры STL
- Как утроен вектор и как работает push_back
- map vs unordered map
- reserve vs resize
- Конкатенация строк и sting bulder
- Безопасная работа с подстроками без копирования - string_view
- Доступ к несуществующему индексу
- Шаблоны - основы generic-программирования


### [Memory](Memory/Memory.md) +-
- Из чего состоит память программы (stack, heap)
- Выравнивание памяти *
- Умные указатели: unique_ptr, shared_ptr, weak_ptr *
- Идиомы RAII, PIMPL (pointer to implementation) *-
- Forward declaration *

### [ООП](OOP/OOP.md)  
- Таблица виртуальных функций*-
- Виртуальный деструктор
- Ромбовидное наследование и решение
- Константные методы

### [Многопоточность и параллелелизм](Multithreading%20and%20parallelism/Mulrithreading%20and%20parallelelism.md)
- Механизмы синхронизации (мьютексы, атомики, condition_variable)
- future/promise - работа с асинхронностью
- гонки данных (Data Race, Race Condition)
- std::thread, корутины (stackless/stackfull), fibers - модели исполнения

### [Эффективная работа с языком](Tech/Effective%20work%20with%20lang/Effective%20work%20with%20lang.md)
- Передача аргументов (по значению, по ссылке, константной ссылке)
- Оптимальная передача объектов (move semantics, RVO/NRVO)
- Исключения
- Работа с сырыми указателями
- Виды конструкторов (default, copy, move, explicit)
- Когда вызываются специальные (магические) методы класса - жизненный цикл объекта