Идиома **IILE** (Immediately Invoked Lambda Expression) — это лямбда, которую определяют и тут же вызывают. Она позволяет внедрить сложную логику инициализации прямо в месте объявления переменной, делая её `const`.

Неправильно:

```cpp
std::vector<int> data = {1, 2, 3, 4, 5};

// Хотим: вектор квадратов чётных чисел
std::vector<int> result;          // Не const
for (int x : data) {
    if (x % 2 == 0)
        result.push_back(x * x);  // Промежуточные мутации
}
// result можно изменить дальше, а мы этого не хотели
```

Правильно:

```cpp
const std::vector<int> data = {1, 2, 3, 4, 5};

const auto result = [&] {
    std::vector<int> res;
    for (int x : data) {
        if (x % 2 == 0)
            res.push_back(x * x);
    }
    return res;
}(); // Вызов на месте
// result = {4, 16} — const, неизменяем
```