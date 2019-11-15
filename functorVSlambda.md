## Functor VS Lambda
Рассмотрим код:
```
std::unique_ptr<T, std::function<void(T*)>> scoped{new T, [](auto *obj) {...}};
```

[Так](https://godbolt.org/z/fm4sZj) выглядит код в ассемблере. С учетом оптимизации(-O3) получили 2041 строку. Многовато, не так ли?
В чем же проблема? - Рассмотрим отдельно `std::function`:
```
std::function<void(T*)> func = [](auto *obj) {...};
```
[Ассемблер](https://godbolt.org/z/dfGcZd). Уже 1698 строк. Т.е. основной вклад - это `std::function`.

Чем еще занимается `std::function`, кроме генерации такого количества кода:
* может аллоцировать память из heap'а.