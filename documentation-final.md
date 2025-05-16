## **1. Введение**  
### **Цели проекта**  
Разработка системы управления задачами (TODO) с возможностью отправки уведомлений в Telegram.  
- Предоставление удобного интерфейса для управления задачами.  
- Обеспечение мгновенных уведомлений о новых и завершённых задачах.  
- Создание масштабируемой микросервисной архитектуры.  

### **Функциональность**  
- **Сервер TODO:**  
  - Регистрация и аутентификация пользователей.  
  - CRUD-операции с задачами.  
  - Отправка событий в RabbitMQ.  
- **Сервер уведомлений:**  
  - Обработка событий из RabbitMQ.  
  - Отправка сообщений в Telegram.  
- **Клиентская часть (React):**  
  - Интерфейс для работы с задачами.  

---

## **2. Архитектура**  
### **Общая схема системы**  
```
[React Frontend] → [TODO API (C#)] → [RabbitMQ] → [Notification Service (C#)] → 
					(PostgreSQL)                    (Telegram API)

[Telegram Bot]
```  
![[c4-for-final-proj.jpg]]
### **Микросервисы**  
1. **TODO API**  
   - **Функции:**  
     - Управление пользователями (`/api/Auth`).  
     - Управление задачами (`/api/Todo`).  
   - **Технологии:** ASP.NET Core, PostgreSQL, RabbitMQ.  

2. **Notification Service**  
   - **Функции:**  
     - Получение событий из RabbitMQ.  
     - Отправка уведомлений (`/api/notifications/send`).  
   - **Технологии:** ASP.NET Core, Telegram.Bot.  

### **Взаимодействие сервисов**  
- **TODO API** публикует события (например, `TaskCreated`) в RabbitMQ.  
- **Notification Service** подписывается на очередь и обрабатывает события.  

---

## **3. Реализация**  
### **Примеры кода**  
#### **TODO API (Создание задачи + отправка в RabbitMQ)**  
```csharp
[HttpPost]
public async Task<IActionResult> CreateTodo([FromBody] TodoItem item)
{
    _dbContext.Todos.Add(item);
    await _dbContext.SaveChangesAsync();

    // Отправка события в RabbitMQ
    var message = new { TaskId = item.Id, Title = item.Title, UserId = item.UserId };
    _rabbitMqService.Publish("todo.created", message);

    return Ok(item);
}
```  

#### **Notification Service (Обработка события)**  
```csharp
public void SubscribeToEvents()
{
    _rabbitMqService.Subscribe("todo.created", async (message) => 
    {
        var task = JsonConvert.DeserializeObject<TodoCreatedEvent>(message);
        await _botClient.SendTextMessageAsync(
            chatId: task.UserTelegramId,
            text: $"Новая задача: {task.Title}");
    });
}
```  

---

## **4. Установка и настройка**  
### **Системные требования**  
- .NET 6+  
- PostgreSQL 14+  
- RabbitMQ 3.9+  
- Node.js (для клиентской части)  

### **Инструкция по развертыванию**  
1. **Запуск TODO API:**  
   ```bash
   cd TodoApi
   dotnet restore
   dotnet run
   ```  

2. **Запуск Notification Service:**  
   ```bash
   cd NotificationService
   dotnet restore
   dotnet run
   ```  

3. **Настройка RabbitMQ:**  
   - Указать `ConnectionString` в `appsettings.json` обоих сервисов.  

4. **Настройка Telegram Bot:**  
   - Получить токен бота у `@BotFather`.  
   - Указать в `appsettings.json` сервиса уведомлений.  

5. **Клиентская часть (React):**  
   ```bash
   npm install
   npm start
   ```  

---

## **5. Тестирование**  
### **Методы тестирования**  
- **Юнит-тесты:**  
  - Проверка бизнес-логики (например, валидация задач).  
- **Интеграционные тесты:**  
  - Проверка API (`/api/Todo`, `/api/Auth`).  
  - Тестирование взаимодействия с RabbitMQ.  
- **Ручное тестирование**
	- Авторизованная страница пользователя ![[Pasted image 20250502175814.png]]
	- Новая созданная задача ![[sasfsdfaoh.png]]
	- Выполненная задача  ![[Pasted image 20250502175832.png]]
	- После выполнения получаем уведомление в TG бота ![[Pasted image 20250502175837.png]]
### **Пример теста (xUnit)**  
```csharp
[Fact]
public async Task CreateTodo_ReturnsOk()
{
    var todo = new TodoItem { Title = "Test Task" };
    var response = await _client.PostAsJsonAsync("/api/Todo", todo);
    response.EnsureSuccessStatusCode();
}
```  

---

## **6. Мониторинг**  
### **Сбор метрик**  
- **RabbitMQ Management UI:** отслеживание очередей.  

---

## **7. Заключение**  
### **Оценка успешности**  
- Реализованы все запланированные функции.  
- Система устойчива к нагрузкам благодаря RabbitMQ.  

### **Планы развития**  
- Добавить WebSocket для real-time уведомлений в UI.  
- Поддержка нескольких мессенджеров (WhatsApp, Email).  
- Внедрение Kubernetes для оркестрации микросервисов.  

--- 

**Автор:** Плехов Андрей Дмитриевич