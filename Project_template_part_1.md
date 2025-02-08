# Задание 1. Анализ и планирование

### 1. Описание функциональности монолитного приложения

**Управление отоплением:**

- Пользователи могут включать/выключать отопление в своих домах.
- Система поддерживает удалённое управление
- Система монтируется только сотрудниками «Тёплый дом»

**Мониторинг температуры:**

- Пользователи могут проверять температуру в своих домах.
- Система поддерживает удалённый мониторинг

### 2. Анализ архитектуры монолитного приложения

Перечислите здесь основные особенности текущего приложения: какой язык программирования используется, какая база данных,
как организовано взаимодействие между компонентами и так далее.

Используется язык программирования Java,     
БД - PostgreSQL  
Приложение синхронное, все взаимодействия последовательные     
Приложение монолитное, поэтому сложно масштабируется


### 3. Определение доменов и границы контекстов

Выделил такие домены:
1. Сбор данных с датчиков, сохранение их в БД
1. Аутентификация
1. Предоставление данных пользователю

### **4. Проблемы монолитного решения**

- Затруднительно масштабировать
- Увеличенное время запросов при высокой нагрузке
- Обновления приложения требую остановки всего приложения
- Над приложением трудно совместно работать нескольким командам
- Даже небольшие изменения приложения требут большого внимания над всей кодовой базой

Пока приложение было небольшим и пользователей было не много, все эти проблемы были не актуальными.  
Возможно уже пора задуматься над более гибким, надежными и maintainability решением. 

### 5. Визуализация контекста системы — диаграмма С4
```plantuml
@startuml
!includeurl https://raw.githubusercontent.com/RicardoNiepel/C4-PlantUML/master/C4_Component.puml
HIDE_STEREOTYPE()

title Контекстная диаграмма C4

Person(user, "Пользователь")
System(app, "Монолитное приложение")
System_Ext(sensors, "Датчики температуры", "Внешние датчики, которые отправляют данные о температуре")
ContainerDb(db, "База данных", "PostgreSQL", "хранит данные о температуре, о пользователях")

Rel(user, app, "Запрашивает данные о температуре", "REST")
Rel(app, db, "Сохраняет и извлекает данные о температуре")
Rel(sensors, app, "Отправляет данные о температуре", "REST")

@enduml
```


# Задание 2. Проектирование микросервисной архитектуры

**Диаграмма контейнеров (Containers)**
```plantuml
@startuml
!include  https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
HIDE_STEREOTYPE()

title Диаграмма контенеров C4

Person(user, "Пользователь", "Владелец дома")
Container(front, "Frontend App", "React", "SPA-приложение")
Container(gateway_1, "API Gateway", "APISIX Gateway", "Для пользователей")


System_Boundary("domen_4", "Домен Хранения данных") {
  ContainerDb(database, "База данных показаний датчиков", "PostgreSQL")
}

System_Boundary("domen_3", "Домен Чтения данных") {
  Container(backend, "Чтение данных", "java", "Возвращает пользователям показания счетчиков")
  ContainerDb(cache, "Cache", "Redis",)
}

System_Boundary("domen_2", "Домен Аутентификация") {
  ContainerDb(database_users, "База данных пользователей", "PostgreSQL", "")
  ContainerDb(cache_auth, "Cache", "Redis", "хранит черный\n(или белый)\nсписок токенов")
  Container(auth, "Аутентификация\nавторизация", "java", "Выдает токены пользователям")
}

System_Boundary("domen_1", "Домен Сбор данных") {
  Container(etl, "Обработка\nданных", "java", "Сервис приводит данные к нужному виду и укладывает в БД")
  Container(gateway_2, "API Gateway", "APISIX Gateway", "Для датчиков")
  Container(datastorage, "Сбор данных", "java", "Собирает данные со всех датчиков и\nприводит к удобному виду для дальнейшей обработки")
  System_Ext(sensors, "Датчики") {
    Container(sensor_1, "Датчик_1")
    Container(sensor_2, "Датчик_2")
    Container(sensor_3, "Датчик_3")
    Container(sensor_4, "Датчик_4")
  }
ContainerQueue(databus, "Шина данных", "Kafka", "Шина для всех даных с датчиков")
}

Rel(sensor_1, gateway_2,)
Rel(sensor_2, gateway_2,)
Rel(sensor_3, gateway_2,)
Rel(sensor_4, gateway_2,)
Rel(gateway_2, datastorage,)
Rel(gateway_1, auth,)
Rel(gateway_1, backend,)

Rel(user, front,)
Rel(front, gateway_1,)

Rel(backend, database, "читает\nданные")
Rel(backend, cache, "cache")
Rel(auth, database_users,)
Rel(auth, cache_auth,)
Rel(datastorage, databus,)

Rel(etl, databus, )
Rel(etl, database, "записывает\nданные")

@enduml
```
**Диаграмма компонентов (Components)**
```plantuml
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml
HIDE_STEREOTYPE()

title Диаграмма компонентов C4


Person(user, "Пользователь", "Владелец дома")

Container_Boundary(backend, "Приложение аутентификации и авторизации") {
  Component(signup, "Регистрация пользователя", "", "Обрабатывает запросы на регистацию")
  Component(signin, "Аутентификация пользователя", "", "Аутентифицирует и авторизует")
}
ComponentDb(database, "База данных", "PostgreSQL",  "Хранит учетные данные пользователей")
ComponentDb(cache, "База данных токенов", "Redis",  "Хранит белый (или черный) список токенов")

Rel(user, signup, "Отправляет логин, пароль и тд.", "https")
Rel(signup, user, "Возвращает JWT-токен", "https")

Rel(user, signin, "Отправляет логин и пароль", "https")
Rel(signin, user, "Возвращает JWT-токен", "https")
Rel(signin, cache, "добавляет токены\nв список",)
Rel(signin, cache, "сверяется\nсо списоком",)
Rel(signin, database, "получает данные пользователя",)

Rel(signup, database, "Запись учетных данных",)

@enduml
```
Добавьте диаграмму для каждого из выделенных микросервисов.






**Диаграмма кода (Code)**

Добавьте одну диаграмму или несколько.

# Задание 3. Разработка ER-диаграммы

Добавьте сюда ER-диаграмму. Она должна отражать ключевые сущности системы, их атрибуты и тип связей между ними.
