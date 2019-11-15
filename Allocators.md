## Allocators
Для управления памятью в C++ STL используется специальный шаблонный класс - `std::allocator`. До С++11 аллокаторы не имели состояния (`stateless allocators`).
Т.е. всегда выполнялось:
```
    CustomAllocator<T> a;
    CustomAllocator<T> b;
    
    assert(a == b); // - always true
```


В **C++11** аллокаторы могут иметь состояние (`stateful allocators`). И появились `scoped_allocators` (`std::scoped_allocator_adaptor`), которые позволяют задать аллокаторы для нижележащих контейнеров.

Но стоит понимать, что не всегда контейнеру нужен тот аллокатор, который он принимает в конструкторе :). Например, `std::list`.
Для хранения `list<T, Allocator<T>>` использует некую структуру `List_node`, которая включает в себя:
```
struct List_node
{
    List_node *next;
    List_node *prev;
    T value;
};
```
Соответственно для выделения памяти в данном случае требуется аллокатор типа `allocator<List_node>`. Для этого стандарт предусматривает два пути:
1. Шаблонная функцию `rebind` у реализации самого аллокатора:
```
template< class U > struct rebind { typedef allocator<U> other; };
```
2. allocator_traits:
```
using MyAllocator = Allocator<T>;
using NewTypeAllocator = typename allocator_traits<MyAllocator>::rebind_alloc<NewRequiredType>;
```

Также аллокаторы с состоянием могут иметь проблемы при использовании нескольких состояний в контейнере, как, например, рассказывается по этой [ссылке](https://youtu.be/DIH-9ae_3DE?t=618).


В **С++17** появились Polymorphic Allocators - `std::pmr::polymorphic_allocator`, который может принимать в конструкторе голый указатель на  `memory_resource`.
`memory_resource` (расположены в `std::pmr namespace`) могут быть:

##### *new_delete_resource*
Для работы с памятью используется `operator new` \ `operator delete`.

##### *null_memory_resource*
Попытка выделить память приводит к генерации исключения `std::bad_alloc`.

##### *synchronized_pool_resource* \ *unsynchronized_pool_resource*
Pool объектов. Как видно из названия есть вариант с синхронизацией и без.

Pool'ы запрашивают память под объекты `chunk`'ами, размеры которых растут в геометрической прогрессии.
Если для `chunk`'а не удастся получить память непрерывным куском, то будет сгенерировано исключение, которое сам pool получил от своего `upstream_allocator`'а. В дефолтной реализации - это будет `std::bad_alloc`.

Подробнее про это можно почитать в статье о [разбухании Heap'а](BubblesOfUnusedMemoryInHeap.md).

##### *monotonic_buffer_resource*
Ресурс, который принимает некий буфер и выдает память из него.
Важной особенностью является то, что память никогда не деаллоцируется. Т.е. при новых запросах на память происходит движение вперед по буферу.
Данный ресурс позволяет выставить upstream, у которого будет запрашиваться память, в случае её нехватки в буфере. По умолчанию - это `std::pmr::get_default_resource`.

`Memory resource` по умолчанию можно выставить функцией `set_default_resource`.
