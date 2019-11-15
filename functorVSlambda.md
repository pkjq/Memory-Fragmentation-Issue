## Functor VS Lambda
Рассмотрим код:
```
std::unique_ptr<T, std::function<void(T*)>> scoped{new T, [](auto *obj) {...}};
// sizeof(scoped) в 32-битной версии == 48 bytes
```

[Так](https://godbolt.org/z/fm4sZj) выглядит код в ассемблере. С учетом оптимизации(-O3) получили 2041 строку. Многовато, не так ли?
В чем же проблема? - Рассмотрим отдельно `std::function`:
```
std::function<void(T*)> func = [](auto *obj) {...};
```
[Ассемблер](https://godbolt.org/z/dfGcZd). Уже 1698 строк. Т.е. основной вклад - это `std::function`.

В MSVC `std::function` устроен примерно так:
```
template <typename ...>
class _Func_class
{
    union _Storage
    {
        ...
        _Ptrt *_Ptrs[_Small_object_num_ptrs];	// _Ptrs[_Small_object_num_ptrs - 1] is reserved and contains pointer to _Func_impl_XXX<...>
    };
};

template <typename Callable, ...>
class function: public _Func_class<Callable, ...> {...};
```

`_Ptrs[_Small_object_num_ptrs - 1]` содержит указатель на `std::_Func_impl_XXX<...>`, который может быть либо указатель на часть памяти, принадлежащей самому инстансу `std::function`, если объект захвата уместился в `_Storage` (*Small Object Optimization*), либо указатель на память, выделенную в heap'е.


Более оптимальным кодом, как с точки зрения ассемблерных инструкций и потребления памяти, так и удобства использования, будет использование функтора:
```
struct ScopedDeleterFunctor
{
    auto operator() (T *obj) { return ...; }
};

std::unique_ptr<T, ScopedDeleterFunctor> scoped { new T };
// sizeof(scoped) в 32-битной версии == 4 bytes
```

[Ассемблер](https://godbolt.org/z/EAu6wg). **205 строк против 2041 строки!**


Если объект deleter должен иметь состояние, то это будет выглядеть так:
```
struct ScopedDeleterFunctor
{
    ScopedDeleterFunctor(...): a(... {}

    auto operator() (T *obj) { return ...; }
    
    int a, b, c;
};

std::unique_ptr<T, ScopedDeleterFunctor> scoped {new T, 1, 2, 3 };
```


**Итого:** 
Использование функтора приносит сплошные плюсы:
* размер объекта будет значительно меньше.
* точно не будет дополнительных аллокаций в heap'е.
* количество ассемблерных инструкций значительно меньше.