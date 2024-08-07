[![latest](https://img.shields.io/github/v/release/GyverLibs/GyverDB.svg?color=brightgreen)](https://github.com/GyverLibs/GyverDB/releases/latest/download/GyverDB.zip)
[![PIO](https://badges.registry.platformio.org/packages/gyverlibs/library/GyverDB.svg)](https://registry.platformio.org/libraries/gyverlibs/GyverDB)
[![Foo](https://img.shields.io/badge/Website-AlexGyver.ru-blue.svg?style=flat-square)](https://alexgyver.ru/)
[![Foo](https://img.shields.io/badge/%E2%82%BD%24%E2%82%AC%20%D0%9F%D0%BE%D0%B4%D0%B4%D0%B5%D1%80%D0%B6%D0%B0%D1%82%D1%8C-%D0%B0%D0%B2%D1%82%D0%BE%D1%80%D0%B0-orange.svg?style=flat-square)](https://alexgyver.ru/support_alex/)
[![Foo](https://img.shields.io/badge/README-ENGLISH-blueviolet.svg?style=flat-square)](https://github-com.translate.goog/GyverLibs/GyverDB?_x_tr_sl=ru&_x_tr_tl=en)  

[![Foo](https://img.shields.io/badge/ПОДПИСАТЬСЯ-НА%20ОБНОВЛЕНИЯ-brightgreen.svg?style=social&logo=telegram&color=blue)](https://t.me/GyverLibs)

# GyverDB
Simple database for Arduino:
- Stores data in key-value pairs
- Supports all real numbers, float, string, and binary
- Dynamic recasting of data types
- Quick access due to hash keys and binary search - 10 times faster than [Pairs](https://github.com/GyverLibs/Pairs) library
- Efficient storage: 8 bytes per value
- Built-in auto-write to flash ESP8266/ESP32

### Compatibility
Compatible with all Arduino platforms (using native Arduino functions)

### Dependencies
- [GTL](https://github.com/GyverLibs/GTL) v1.0.6+
- [StringUtils](https://github.com/GyverLibs/StringUtils) v1.4.15+

## Table of Content
- [Usage](#usage)
- [Version](#versions)
- [Installation](#install)
- [Known Bugs and Feedback](#feedback)

<a id="usage"></a>

## Usage
Optional feature restriction definitions
```cpp
#define DB_NO_UPDATES  // disable refresh
#define DB_NO_FLOAT    // disable float support
#define DB_NO_INT64    // disable long long int (int64) support
#define DB_NO_CONVERT  // disable dynamic data recasting (in place of keepTypes)
```

### GyverDB
```cpp
// constructor
// reserve table space
GyverDB(uint16_t reserve = 0);


// do not change data types (default true)
void keepTypes(bool keep);

// enable updates (default false)
void useUpdates(bool use);

// data change flag. Resets to false after triggering
bool changed();

// display all database content
void dump(Print& p);

// database size
size_t size();

// database export size (for writeTo)
size_t writeSize();

// export database to Stream (e.g. file)
bool writeTo(Stream& stream);

// export database to buffer of size writeSize()
bool writeTo(uint8_t* buffer);

// import database from Stream (e.g. file)
bool readFrom(Stream& stream, size_t len);

// import database from buffer
bool readFrom(const uint8_t* buffer, size_t len);

// create a record. If exists, overwrite with a blank value
bool create(size_t hash, gdb::Type type, uint16_t reserve = 0);

// clear memory
void reset();

// clear all records (does not free up reserved space)
void clear();

// get record by hash
gdb::Entry get(size_t hash);

// get record by key
gdb::Entry get(const Text& key);

// delete record by hash
void remove(size_t hash);

// delete record by key
void remove(const Text& key);

// check if record exists by hash
bool has(size_t hash);

// check if record exists by key
bool has(const Text& key);

// set value for associated hash 
bool set(size_t hash, DATA data);

// set value for associated key name
bool set(const Text& key hash, DATA data);

// initialize record (creates a record if it does not exist)
bool init(size_t hash, DATA data);
bool init(const Text& key hash, DATA data);
```

### GyverDBFile
This class is inherited from GyverDB and can also self-write to a file in flash ESP on change and on timeout
```cpp
GyverDBFile(fs::FS* nfs = nullptr, const char* path = nullptr, uint32_t tout = 10000);

// set file system and file name
void setFS(fs::FS* nfs, const char* path);

// set record timeout in ms (default 10000)
void setTimeout(uint32_t tout = 10000);

// begin reading data
bool begin();

// refresh file content
bool update();

// heartbeat. Updates the data on change and/or on timeout ,can be used in a loop. Returns true
bool tick();
```

To use, launch the file system (FS) and use tick() in a loop. All changes will be recorded to a file on timeout:
```cpp
#include <LittleFS.h>
#include <GyverDBFile.h>
GyverDBFile db(&LittleFS, "db.bin");

void setup() {
    LittleFS.begin();
    db.begin(); // прочитать данные из файла

    // для работы в таком режиме очень пригодится метод init():
    // создаёт запись соответствующего типа и записывает "начальные" данные,
    // если такой записи ещё нет в БД
    db.init("key", 123);    // int
    db.init("uint", 123ul); // uint32
    db.init("str", "init"); // строка
}
void loop() {
    db.tick();
}
```

### Data types
```cpp
None
Int
Uint
Int64
Uint64
Float
String
Bin
```

### Entry
- Inherited from the `Text` class for optimized record processing

```cpp
// data type
gdb::Type type();

// output to buffer of size size(). Does not add a 0-terminator if it is a string
void writeBytes(void* buf);

// output to variable
bool writeTo(T& dest);

Value toText();
String toString();
bool toBool();
int toInt();
int32_t toInt();
int64_t toInt64();
double toFloat();
```

### Usage
#### Keys
БД хранит ключи в виде хэш-кодов, для доступа к БД нужно использовать непосредственно хэш или обычную строку, библиотека сама посчитает хэш:
```cpp
db["key1"] = 1234;      // строка
db[SH("key2")] = 1234;  // хэш
db["key3"_h] = 1234;    // хэш
```

> Здесь `SH()` - хэш-функция из библиотеки StringUtils, выполняемая на этапе компиляции. Переданная в неё строка не существует в программе - на этапе компиляции она превращается в число. Также можно использовать литерал `_h` - он делает же самое.

Так как хэш - это число, то с базой можно работать и просто обычными числами. Можно считать, что база данных - это массив на 2^29 ячейки:
```cpp
db[0] = 123;
db[2] = 456;
db[100] = "hello";
```

И поэтому вместо хэшей также можно использовать обычный enum:
```cpp
enum keys : size_t {
    key1,
    key2,
    mykey,
    lolkek,
};

db[keys::key1] = 123;
db[keys::key2] = 456;
db[keys::lolkek] = "hello";
```

Это очень удобно, потому что IDE подскажет имеющиеся ключи при вводе `keys::`.

#### Запись и чтение
```cpp
GyverDB db;

// эта ячейка у нас объявлена как int, текст корректно сконвертируется в число
db["key1"] = "123456";

// чтение. Библиотека сама конвертирует в нужный тип
int i = db["key1"];
float f = db[SH("key2")];

// любые данные "печатаются", даже бинарные
Serial.println(db["key3"_h]);

// можно указать конкретный тип при выводе
db["key3"_h].toInt32();

// можно сравнивать с целочисленными
int i = 123;
db["key1"] == i;
db["key1"] >= i;

// сравнение напрямую со строками работает только у записей с типом String
db["key1"] == "str";

// но можно и вот так, для любых типов записей
// toText() конвертирует все типы записей БД во временную строку
db["key1"].toText() == "12345";

// GyverDB может записать данные любого типа, даже составные (массивы, структуры)
uint8_t arr[5] = {1, 2, 3, 4, 5};
db["arr"] = arr;

// вывод обратно. Тип должен иметь такой же размер!
uint8_t arr2[5];
db["arr"].writeTo(arr2);

// посмотрим что записалось
db.dump(Serial);
```

### Примечания
- GyverDB хранит целые до 32 бит и float числа в памяти самой ячейки. 64-битные числа, строки и бинарные данные выделяются динамически
- Ради компактности используется 29-битное хэширование. Этого должно хватать более чем, шанс коллизий крайне мал
- Библиотека автоматически выбирает тип записи при записи в ячейку. Приводите тип вручную, если это нужно (например `db["key"] = 12345ull`)
- По умолчанию включен параметр `keepTypes()` - не изменять тип записи при перезаписи. Это означает, что если запись была int, то при записи в неё данных другого типа они будут автоматически конвертироваться в int, даже если это строка. И наоборот
- При создании пустой ячейки можно указать тип и зарезервировать место (только для строк и бинарных данных) `db.create("kek", gdb::Type::String, 100)`
- `Entry` имеет автоматический доступ к строке как оператор `String`, это означает что записи с текстовым типом (String) можно передавать в функции, которые принимают `String`, например `WiFi.begin(db["wifi_ssid"], db["wifi_pass"]);`
- Если нужно передать запись в функцию, принимающую `const char*` - используйте на ней `c_str()`. Это не продублирует строку в памяти, а даст к ней прямой доступ. Например `foo(db["str"].c_str())`

<a id="versions"></a>

## Версии
- v1.0
- v1.0.1 упразднены целые типы 8 и 16 бит, увеличено разрешение хэша

<a id="install"></a>
## Установка
- Библиотеку можно найти по названию **GyverDB** и установить через менеджер библиотек в:
    - Arduino IDE
    - Arduino IDE v2
    - PlatformIO
- [Скачать библиотеку](https://github.com/GyverLibs/GyverDB/archive/refs/heads/main.zip) .zip архивом для ручной установки:
    - Распаковать и положить в *C:\Program Files (x86)\Arduino\libraries* (Windows x64)
    - Распаковать и положить в *C:\Program Files\Arduino\libraries* (Windows x32)
    - Распаковать и положить в *Документы/Arduino/libraries/*
    - (Arduino IDE) автоматическая установка из .zip: *Скетч/Подключить библиотеку/Добавить .ZIP библиотеку…* и указать скачанный архив
- Читай более подробную инструкцию по установке библиотек [здесь](https://alexgyver.ru/arduino-first/#%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0_%D0%B1%D0%B8%D0%B1%D0%BB%D0%B8%D0%BE%D1%82%D0%B5%D0%BA)
### Обновление
- Рекомендую всегда обновлять библиотеку: в новых версиях исправляются ошибки и баги, а также проводится оптимизация и добавляются новые фичи
- Через менеджер библиотек IDE: найти библиотеку как при установке и нажать "Обновить"
- Вручную: **удалить папку со старой версией**, а затем положить на её место новую. "Замену" делать нельзя: иногда в новых версиях удаляются файлы, которые останутся при замене и могут привести к ошибкам!

<a id="feedback"></a>

## Баги и обратная связь
При нахождении багов создавайте **Issue**, а лучше сразу пишите на почту [alex@alexgyver.ru](mailto:alex@alexgyver.ru)  
Библиотека открыта для доработки и ваших **Pull Request**'ов!

При сообщении о багах или некорректной работе библиотеки нужно обязательно указывать:
- Версия библиотеки
- Какой используется МК
- Версия SDK (для ESP)
- Версия Arduino IDE
- Корректно ли работают ли встроенные примеры, в которых используются функции и конструкции, приводящие к багу в вашем коде
- Какой код загружался, какая работа от него ожидалась и как он работает в реальности
- В идеале приложить минимальный код, в котором наблюдается баг. Не полотно из тысячи строк, а минимальный код
