Идиома "Copy And Swap" пользовалась большой популярностью до выхода C++11. В чем ее суть?

Рассмотрим некий класс, владеющий ресурсом (в нашем случае нужно выделять память). С использованием стандарта С++98/03, а также с учетом правила трех, его код мог бы выглядеть следующим образом:

```cpp
// myclass.h

#pragma once

class myclass {
public:
	myclass();
	myclass(const myclass& rhs);
	myclass& operator=(const myclass& rhs);
	~myclass();
private:
	char* buffer;
};

// myclass.cpp

#include "myclass.h"

#include <cstring>
#include <utility>

myclass::myclass() : buffer(0) {
}

myclass::myclass(const myclass& rhs)
{
	char* tmp = 0;
	if (rhs.buffer) {
		size_t len = std::strlen(rhs.buffer);
		tmp = new char[len + 1];
		std::memcpy(tmp, rhs.buffer, len + 1);
	}
	buffer = tmp;
}

myclass& myclass::operator=(const myclass& rhs) {
	if (this != &rhs) {
		char* tmp = 0;
		if (rhs.buffer) {
			size_t len = std::strlen(rhs.buffer);
			tmp = new char[len + 1];
			std::memcpy(tmp, rhs.buffer, len + 1);
		}
		delete[] buffer;
		buffer = tmp;
	}
	return *this;
}

myclass::~myclass() {
	delete[] buffer;
}
```

Заметьте, в этом коде соблюдена строгая гарантия безопасности исключений: если при копировании что-то пойдёт не так, исходный объект останется нетронутым.

Но дублирование кода выглядит печально. Логика копирования повторяется и в конструкторе копирования, и в операторе присваивания. Можно, конечно, вынести её в отдельную вспомогательную функцию и вызывать оттуда.

Однако ещё в конце 90-х был придуман более элегантный подход, который не только убирает дублирование, но и автоматически обеспечивает строгую гарантию безопасности исключений. В класс добавляется функция `swap`, которая быстро и дёшево обменивает значения членов класса. Стандартная функция `std::swap`  из-за копирования здесь неприменима из-за возникающей рекурсии.

Кроме того добавляется еще свободная функция `swap`, объявленная в том же пространстве имён, что и класс. Благодаря механизму ADL (поиск имён по типам аргументов) именно эта эффективная версия будет вызываться для нашего класса вместо медленного `std::swap` (до C++11 выполняла три операции копирования, вместо дешевого перемещения).

```cpp
// myclass.h

#pragma once

class myclass {
public:
	myclass();
	myclass(const myclass& rhs);
	myclass& operator=(const myclass& rhs);
	~myclass();
	void swap(myclass& other);
private:
	char* buffer;
};

void swap(myclass& a, myclass& b);

// myclass.cpp

#include "myclass.h"

#include <cstring>
#include <utility>

myclass::myclass() : buffer(0) {
}

myclass::myclass(const myclass& rhs) : buffer(0)
{
	if (rhs.buffer) {
		size_t len = std::strlen(rhs.buffer);
		buffer = new char[len + 1];
		std::memcpy(buffer, rhs.buffer, len + 1);
	}
}

myclass& myclass::operator=(const myclass& rhs) {
	myclass tmp(rhs);
	swap(tmp);
	return *this;
}

void myclass::swap(myclass& other) {
	char* tmp = buffer;
	buffer = other.buffer;
	other.buffer = tmp;
}

myclass::~myclass() {
	delete[] buffer;
}

void swap(myclass& a, myclass& b) {
	a.swap(b);
}
```

Эта идиома получила название "Copy And Swap" и решала три проблемы:

- Контроль самоприсваивания (легко забыть)
- Легко реализуемая строгая гарантия безопасности исключений
- Дублирование кода

Минусы:

- Больше операций (Конструктор = выделение + копирование; Обмен через swap; Деструктор), чем в ручной версии (выделение и освобождение). Впрочем разница получается незначительная, а также здесь возможна оптимизация компилятора.
- При копировании самого в себя, происходит безопасное создание копии и обмен с самим собой. Это крайне редкий случай, который нужно пресекать не внутри функции, а до ее вызова.
- Часто новички пытаются использовать std::swap, что приводит к бесконечной рекурсии и  stack overflow

С приходом C++11, move-семантики и новой версии std::swap, популярность идиомы сошла на нет. Обновленная версия идиомы выглядит теперь так:

```cpp
// myclass.h

#pragma once

class myclass final {
public:
	myclass() noexcept;
	myclass(const myclass& rhs);
	myclass(myclass&& rhs) noexcept;
	myclass& operator=(myclass rhs) noexcept;
	~myclass();

	void swap(myclass& other) noexcept;

private:
	char* buffer{ nullptr };
};

void swap(myclass& a, myclass& b) noexcept;

// myclass.cpp

#include "myclass.h"

#include <cstring>    // std::strlen, std::memcpy
#include <utility>    // std::exchange

myclass::myclass(const myclass& rhs)
{
	if (rhs.buffer) {
		auto len = std::strlen(rhs.buffer);
		buffer = new char[len + 1];
		std::memcpy(buffer, rhs.buffer, len + 1);
	}
}

myclass::myclass(myclass&& rhs) noexcept
	: buffer(std::exchange(rhs.buffer, nullptr))
{
}

myclass& myclass::operator=(myclass rhs) noexcept {
	swap(rhs);
	return *this;
}

void myclass::swap(myclass& other) noexcept {
	using std::swap;
	swap(buffer, other.buffer);
}

myclass::~myclass() {
	delete[] buffer;
}

void swap(myclass& a, myclass& b) noexcept {
	a.swap(b);
}
```

Обратите внимание, что оператор присваивания принимает аргумент по значению! Нет необходимости писать два раздельных оператора:

```cpp
myclass& operator=(const myclass& rhs);
myclass& operator=(myclass&& rhs) noexcept;
```

Когда параметр передаётся по значению, компилятор сам выбирает, как его создать:

- Если передаётся **lvalue** (`a = b`) → вызывается **конструктор копирования** → внутри функции оказывается копия
- Если передаётся **rvalue** (`a = std::move(b)`) → вызывается **move-конструктор** → внутри функции оказывается перемещённый объект

Затем мы просто вызываем `swap` с этим параметром, и он забирает его содержимое, а старые данные объекта умирают вместе с параметром при выходе из функции.

**Плюсы**:

- **Меньше кода** - одна функция вместо двух
- **Нет дублирования** - логика обмена пишется один раз
- **Автоматическая безопасность** - строгая гарантия исключений для копирования, корректная обработка самоприсваивания
- **Не нужно писать `if (this != &rhs)`**- даже для перемещающего присваивания

**Минусы**:

- **Всегда выделяется новая память при копировании** - даже если старый буфер достаточно велик и можно было переиспользовать его
- **Лишний move при перемещающем присваивании** - вместо прямого обмена создаётся промежуточный объект, который сразу уничтожается

Когда имеет смысл применять:

Подход хорош, когда память при присваивании **всё равно перевыделяется** (как в примере выше) и корректность важнее микрооптимизаций. Не стоит использовать, когда переиспользование существующего буфера даёт значительный выигрыш в производительности.

http://gotw.ca/gotw/059.htm
https://www.modernescpp.com/index.php/the-copy-and-swap-idiom/
https://mropert.github.io/2019/01/07/copy_swap_20_years/
https://metanit.com/cpp/tutorial/13.1.php
