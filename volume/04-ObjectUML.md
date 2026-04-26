# Лабораторная работа №4  
**Тема:** Объектно-ориентированный анализ с использованием UML для системы управления доступом и учёта действий пользователей

**Цель:** Освоить основы объектно-ориентированного моделирования.

## Задача 1. Выделение классов, атрибутов и методов

На основе анализа предметной области и результатов предыдущих работ (ЛР1–ЛР3) выделены следующие ключевые классы. Классы сгруппированы по слоям: **бизнес-сущности** и **управляющие сервисы**.

| Класс | Тип | Назначение | Ключевые атрибуты | Ключевые методы |
|-------|-----|------------|-------------------|-----------------|
| **User** | Сущность | Учётная запись пользователя | `int id`<br>`string username`<br>`-string passwordHash`<br>`string email`<br>`string status` | `+bool verifyPassword(string password)`<br>`+bool isActive()` |
| **Role** | Сущность | Роль, объединяющая разрешения на ресурсы | `int id`<br>`string name`<br>`string description` | `+void addPermission(Resource r, string op, string cond)`<br>`+void removePermission(Resource r, string op)` |
| **Resource** | Сущность | Защищаемый информационный ресурс | `int id`<br>`string name`<br>`string type`<br>`string location`<br>`bool isActive` | `+bool isAvailable()` |
| **Session** | Сущность | Активная сессия аутентифицированного пользователя | `int id`<br>`User user`<br>`string token`<br>`DateTime startTime`<br>`DateTime endTime`<br>`string ipAddress` | `+bool isValid()`<br>`+void terminate()` |
| **AuditEvent** | Сущность | Запись о событии безопасности | `int id`<br>`DateTime timestamp`<br>`User user`<br>`string action`<br>`Resource resource`<br>`string result`<br>`string details` | `+string toFormattedString()` |
| **AlertRule** | Сущность | Правило корреляции для выявления инцидентов | `int id`<br>`string name`<br>`string pattern`<br>`string severity`<br>`List<AlertRecord> alerts` (композиция) | `+bool matches(AuditEvent event)`<br>`+AlertRecord generateAlert(AuditEvent event)`<br>`+void acknowledge(int alertId)` |
| **AlertRecord** | Сущность (часть AlertRule) | Запись о сгенерированном оповещении | `DateTime timestamp`<br>`string message`<br>`string status` | – |
| **AccessManagementService** | Сервис | Управление доступом (аутентификация, авторизация, сессии) | – | `+Session authenticate(credentials)`<br>`+bool authorize(u, r, op)`<br>`+void logout(s)` |
| **AuditMonitoringService** | Сервис | Аудит событий и мониторинг инцидентов | – | `+void log(event)`<br>`+void processEvents()`<br>`+void checkRules()` |
| **AdminService** | Сервис | Администрирование учётных записей, ролей и отчётов | – | `+User createUser(data)`<br>`+void assignRole(u, r)`<br>`+Report generateReport(params)` |

### Пояснения к структуре

- **AlertRecord** является частью композиции `AlertRule` (сильная связь «целое-часть»). Не имеет собственных методов, управляется через методы `AlertRule`.
- **Сервисы** (`AccessManagementService`, `AuditMonitoringService`, `AdminService`) не обладают собственными атрибутами состояния, а реализуют бизнес-логику, взаимодействуя с сущностями.
- **Методы `addPermission`/`removePermission`** в классе `Role` принимают конкретные параметры (ресурс, операция, условия), а не объект `Permission`, что упрощает программный интерфейс и скрывает детали реализации связи «роль-ресурс» из ER-модели.
- Атрибут `isActive` и метод `isAvailable()` класса `Resource` позволяют динамически управлять доступностью ресурса, не удаляя его из системы.

## Задача 2. Диаграмма классов (Class Diagram)

```mermaid
---
config:
  theme: neutral
---
classDiagram

class User {
  +int id
  +string username
  -string passwordHash
  +string email
  +string status
  +bool verifyPassword(string password)
  +bool isActive()
}
class Role {
  +int id
  +string name
  +string description
  +addPermission(Resource r, string op, string cond) void
  +removePermission(Resource r, string op) void
}
class Resource {
  +int id
  +string name
  +string type
  +string location
  +bool isActive
  +bool isAvailable()
}
class Session {
  +int id
  +User user
  +string token
  +DateTime startTime
  +DateTime endTime
  +string ipAddress
  +bool isValid()
  +void terminate()
}
class AuditEvent {
  +int id
  +DateTime timestamp
  +User user
  +string action
  +Resource resource
  +string result
  +string details
  +string toFormattedString()
}
class AlertRule {
  +int id
  +string name
  +string pattern
  +string severity
  +List~AlertRecord~ alerts
  +bool matches(AuditEvent event)
  +AlertRecord generateAlert(AuditEvent event)
  +void acknowledge(int alertId)
}
class AlertRecord {
  +DateTime timestamp
  +string message
  +string status
}

class AccessManagementService {
  +authenticate(credentials) Session
  +authorize(u, r, op) bool
  +logout(s) void
}
class AuditMonitoringService {
  +log(event) void
  +processEvents() void
  +checkRules() void
}
class AdminService {
  +createUser(data) User
  +assignRole(u, r) void
  +generateReport(params) Report
}

%% Структурные связи между сущностями
User "1" -- "0..*" Session : has
User "0..*" -- "0..*" Role : assigned to
Role "0..*" -- "0..*" Resource : permissions on
User "1" -- "0..*" AuditEvent : generates
Resource "1" -- "0..*" AuditEvent : is object of
AlertRule "1" *-- "0..*" AlertRecord : contains

%% Связи-зависимости от сервисов к сущностям
AccessManagementService ..> User : <<use>>
AccessManagementService ..> Session : <<use>>
AccessManagementService ..> Role : <<use>>
AccessManagementService ..> Resource : <<use>>

AuditMonitoringService ..> AuditEvent : <<use>>
AuditMonitoringService ..> AlertRule : <<use>>

AdminService ..> User : <<use>>
AdminService ..> Role : <<use>>
AdminService ..> Resource : <<use>>
```

*Примечание:* для связи «многие ко многим» между User и Role подразумевается ассоциативный класс UserRoleAssignment (на диаграмме не детализирован отдельно для упрощения, но подразумевается через связь с `UserRoleAssignment`).

## Задача 3. Диаграмма прецедентов (Use Case Diagram)

**Актёры:**
- **Пользователь (User)** – штатный сотрудник, запрашивающий доступ к ресурсам.
- **Администратор (Admin)** – управляет ролями, учётными записями и просматривает отчёты.
- **Кадровая система (HR System)** – предоставляет данные о сотрудниках.
- **Сервис оповещения (Notification Service)** – внешняя система для отправки уведомлений (актёр, взаимодействующий с системой).

```mermaid
---
config:
  theme: neutral
---
graph TD

subgraph "Система управления доступом и учёта действий"
    UC_Auth(Аутентификация пользователя)
    UC_Access(Запрос доступа к ресурсу)
    UC_Logout(Завершение сессии)
    UC_ViewOwn(Просмотр своих действий)
    UC_ManageUsers(Управление учётными записями)
    UC_ManageRoles(Управление ролями и правами)
    UC_ViewLogs(Просмотр журнала аудита)
    UC_GenerateReports(Генерация отчётов)
    UC_RuleMgmt(Управление правилами алертов)
    UC_HRsync(Синхронизация данных из кадровой системы)
    UC_SendAlert(Отправка уведомлений об инцидентах)
end

User((Пользователь))
Admin((Администратор))
HR((Кадровая система))
Notif((Сервис оповещений))

User --> UC_Auth
User --> UC_Access
User --> UC_Logout
User --> UC_ViewOwn
Admin --> UC_ManageUsers
Admin --> UC_ManageRoles
Admin --> UC_ViewLogs
Admin --> UC_GenerateReports
Admin --> UC_RuleMgmt
HR --> UC_HRsync
UC_HRsync -.->|include| UC_ManageUsers
UC_SendAlert --> Notif
UC_ViewLogs -.->|extension point| UC_SendAlert
Admin --> UC_SendAlert
```

Пояснения:
- «Синхронизация данных о сотрудниках» инициируется кадровой системой и включает создание/обновление учётных записей (UC4).
- «Отправка уведомлений об инцидентах» включает просмотр журнала аудита (UC6) для администратора, а также взаимодействие с внешним сервисом оповещения.

## Задача 4. Диаграмма последовательности (Sequence Diagram)

**Сценарий:** успешный доступ пользователя к защищённому ресурсу с аутентификацией и авторизацией.

**Участники:**  
- Пользователь,  
- клиентское приложение (PEP),  
- AuthService,  
- AccessControlService,  
- AuditService,  
- Ресурс (целевая система),  
- MonitoringService (фоновый сервис).

```mermaid
---
config:
  theme: neutral
---
sequenceDiagram

actor U as Пользователь
participant PEP as PEP (Клиент)
participant AMS as AccessManagementService
participant AMon as AuditMonitoringService
participant R as Ресурс

U->>PEP: запрос доступа (ресурс, операция)
PEP->>AMS: authenticate(credentials) -> сессия
AMS->>AMS: проверка пароля и isActive()
alt неудача
    AMS-->>PEP: исключение
    PEP-->>U: отказ в доступе
else успех
    AMS-->>PEP: сессия
    PEP->>AMS: authorize(user, resource, operation)
    Note over AMS: проверяет роли и разрешения (Resource.isAvailable())
    AMS-->>PEP: true
    PEP->>R: запрос операции
    R-->>PEP: результат
    PEP->>AMon: log(AuditEvent)
    AMon->>AMon: добавить в журнал, processEvents()
    Note over AMon: внутри проверяет правила (AlertRule.matches) и генерирует алерты
end
PEP-->>U: доступ предоставлен (или ошибка)
```

**Сценарий «Отправка алерта при обнаружении аномалии» (дополнительный):**

```mermaid
---
config:
  theme: neutral
---
sequenceDiagram

participant Mon as MonitoringService
participant Audit as AuditService (журнал)
participant Rule as AlertRule
participant Alert as Alert
participant Notif as Сервис оповещения
participant Admin as Администратор

Mon->>Audit: получить свежие события
Audit-->>Mon: список событий
loop для каждого события
    Mon->>Rule: matches(event)
    alt правило сработало
        Rule-->>Mon: true
        Mon->>Alert: создать new Alert(event, rule)
        Alert->>Notif: отправить уведомление
        Notif-->>Admin: СМС/email: "Обнаружена подозрительная активность"
    end
end
```

## Описание классов и сценариев

**Классы-сущности** (User, Role, Resource, Permission, Session, AuditEvent, Alert, AlertRule) являются основой доменной модели. Они инкапсулируют данные и минимальное поведение: проверка пароля, проверка активности сессии, сопоставление правила событию.  
**Сервисные классы** реализуют основную бизнес-логику:
- **AuthService** отвечает за идентификацию и аутентификацию, создание и завершение сессий.
- **AccessControlService** принимает решение об авторизации на основе назначенных пользователю ролей и разрешений.
- **AuditService** централизованно собирает события от разных компонентов и сохраняет их.
- **MonitoringService** в асинхронном режиме анализирует журнал событий и по правилам (AlertRule) генерирует оповещения.
- **AdminService** предоставляет интерфейс для управления учётными записями, ролями, разрешениями и формирования отчётов.

Взаимодействие объектов проиллюстрировано диаграммой последовательности, где чётко разделены этапы аутентификации и авторизации, а также асинхронная обработка событий мониторингом.
