PVS - V837

Идиома **try-emplace** применяется для вставки в ассоциативные контейнеры (`std::map`, `std::unordered_map`), когда нужно вставить элемент только если ключ отсутствует, и при этом избежать создания временного объекта `std::pair`, которое происходит при использовании `insert` или `emplace`.

Неправильно:

```cpp
class Worker {
    std::string name;
    std::string surname;
    std::string position;

public:
    Worker(std::string name, std::string surname, std::string position)
        : name(name), surname(surname), position(position) {}
};

std::map<int, Worker> workers;

bool add_worker(int id,
                const std::string& name,
                const std::string& surname,
                const std::string& position)
{
    // emplace конструирует Worker, даже если id уже есть
    workers.emplace(id, Worker(name, surname, position));
    return workers.count(id) == 1;
}
```

**Проблема**: eсли `id` уже существует:

   - `Worker` конструируется (копируются/перемещаются `std::string`);
   - затем конструируется `pair<const int, Worker>`;
   - и только потом проверяется наличие ключа — вставка не происходит, освобождается pair.

Это лишние копирования/перемещения и трата ресурсов.

Для решения проблемы до С++17 использовалась идиома lower_bound-emplace:

Правильно:

```cpp
bool add_worker(int id,
                const std::string& name,
                const std::string& surname,
                const std::string& position)
{
	auto it = workers.lower_bound(id);
	if (it == workers.end() || it->first != id) {
		// ключа нет — вставляем перед it, без повторного поиска
		workers.emplace(it, id, name, surname, position);
		return true;
	}
	// ключ есть — не вставляем
	return false;
}
```

В С++17 появилась функция `try_emplace`, которая сперва проверяет наличие ключа, и только потом конструирует объект `std::pair`.

Правильно:

```cpp
bool add_worker(int id,
                const std::string& name,
                const std::string& surname,
                const std::string& position)
{
    // Конструируем Worker напрямую внутри мапы
    auto [it, inserted] = workers.try_emplace(id, name, surname, position);
    return inserted;
}
```

Если id уже есть, то объект `Worker` и `std::pair` не создаются.

Если уже есть созданный объект, который нужно поместить в контейнер, то используйте `std::move`:

```cpp
bool add_worker(int key, Worker worker) {
	auto [it, inserted] = entries.try_emplace(key, std::move(worker));
	if (!inserted) {
		// worker гарантированно не был перемещён, его можно использовать
		// dereferencing ptr here
	}
	return inserted;
}
```

Внимание! Избегайте вставки с конструированием временного объекта:

```cpp
auto [it, inserted] = workers.try_emplace(
	id,
	Worker(name, surname, position)  // ← здесь конструируется Worker
);
```

В этом случае Worker в любом случае будет создан, но std::par будет создаваться только если id не найден.  