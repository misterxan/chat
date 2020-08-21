
## Тестовое задание

## Используемый стек:

* MySQL
* PHP, Go (в основном консьюмеры предпочтительнее использовать Golang)
* node.js (WS)
* Kafka
* Redis (в качестве кеша)

## Условности

* Изначально для простоты говорим что у зрителя и модели может быть только одна валюта и баланс.
* Так же мы считаем что у модели может быть один чат, и связующей сущностью будет выступать сама модель, а не чат.
* Авторизация по токенам. (jwt payload)
* Считаем что зритель зарегистрирован в системе либо берем его fingerprint в качестве uuid.


## API Endpoints:

Добавление сообщения

```
POST /chat/message
Request:
{
	"message": "hello"
	"is_public": "false"
}
Response:
201: {
	success: ok
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

Добавление доната:

```
POST /chat/donat
request:
{
	"sum": 15.5,
	"messgae": { //optional
		"message": "",
		"is_public": "false"
	}
}
response:
201: {
	success: ok
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

Регистрация коннекта:

```
POST /chat/connection
request:
{
	"connection_id": "uuid",
}
response:
201: {
	success: ok
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

Получение последних событий за 10 мин не более 20:
```
GET /chat/event/history 
response:
200: {
	"success": true,
	"events": [
		{
			type: message
		},
		{
			type: donate
		},
		{
			type: change_sponsor
		}
	]
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

Цели

```
POST /chat/goals
Request:
{
	"require_sum": "1000"
	"name": "Na Webku"
}
Response:
201: {
	success: ok
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"no more than one goal"
}
```

```
GET /chat/goals
Response:
200: {
	goals: [{
	   "name": "bla",
	   "sum" : "1000",
	   "require_sum": "2000"
	}]
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

```
PUT /chat/goals/{goal_id}
Response:
204: {
	{
	   "name": "blaBLA",
	   "require_sum": "2000"
	}
}
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```


```
DELETE /chat/goals/{goal_id}
Response:
204:
401: {
	"success": "false", "message":"anauthorized"
}
400: {
	"success": "false", "message":"bad params"
}
```

## Описание моделей и таблиц:

### Пользовательская часть

* User
table `users`
```
id (uint)
uuid
name (varchar)
login
password
balance
role
created_at
updated_at
```

* Connection
table `connections`

```
id
type (user or client)
entity_uuid
connection_uuid
```


### Часть связанная с сообщениями


* Donate
table `donations`

```
id
has_message (bool)
message_id (default null)
sum
created_at
updated_at
```

* Message 
table `messages`
```
id
is_public (bool)
receiver_id
sender_id
value (text)
created_at
updated_at
```

* Goal
table `goals`

```
id
user_id
sum
require_sum
name
created_at
updated_at
```


* Sponsor
table `sponsors`

```
id 
model_id
viwer_id
created_at
updated_at
```

## Схема событий и их консьюмеров:

```
[
"AddNewConnection" => 
	[
		"SaveConnection" => "SaveConnection"
	],
"AddNewMessage" => 
	[
		"SaveMessage" => SaveMessageListener,
		"SendToQueue" => SendMessageToQueeuListener,
	],
"AddNewDonate" => 
	[
		"SaveDonate" => SaveDonateListener,
		"CalculateNewSponsor" => CalculateNewSponsorListener,
		"SendToQueue" => SendDonateToQueeuListener,
		"TransferMoney" => TransferMoneyListener,
	],
"ReceiveMoney" => 
	[
		"UpdateGoal" => UpdateGoalListener
	],
"CreateGoal" => 
	[
		"SendCreateGoal" => SendCreateGoalListener,
	],
"UpdateGoal" => 
	[
		"SendUpdateGoal" => SendUpdateGoalListener
	],
"DeleteGoal" => 
	[
		"SendDeleteGoal" => SendDeleteGoalListener
	],
"EndGoal" => 
	[
		"SendEventToQueue" => SendEndGoalToQueue
	],
"NewSponsor" => 
	[
		"SaveNewSponsor" => SaveNewSponsor
		"SendEventToQueue" => SendNewSponsorToQueue
	],
]
```

## Описание работы сервиса:


Предлагается использовать событийно ориентированную архитектуру.

Существует общий брокер в виде Kafka и консьюмеры которые существуют в виде демонов (воркеров).

Каждый консьюмер слушает свой типа события и обрабатывает его. Связь событий и консьюмеров в структуре выше.

### Мотивация

Высокая степень независимости отдельных компонентов приложения, которые могут быть использованы параллельно и ассинхронно. Например отправка сообщений в вебсокет и сохранение данных в БД, или вычисление топ спонсора. Учитывая возможности kafka в распределеннии ресурсов, данная архитектура позволяет легко масштабироваться. Использование group_id при чтении из топика гарантирует очередности чтения сообщения (и защита повторго чтения) для разных копий одного и того же приложения. Так же возможность подключить обработчики для аналитических систем. Не мешая логике основного приложения.

### Риски

Может быть несогласованность между очередностью обработки слушателей.

### Описание консьюмеров:

#### Событие `AddNewConnection`
* SaveConnection - создает связку между uuid (пользователь или модель) с connection_id которая генерируется в node.js

#### Событие `AddNewMessage` 
* SaveMessageListener - оперирует с моделью Message сохраняет в нужные таблицы. Инвалидирует кеш последних событий.
* SendMessageToQueueListener - оперирует с моделью Message отправляет данные в очередь на отправку в ws

#### Событие `AddNewDonate`

* CalculateNewSponsorListener - оперирует с моделью Donate ассинхронно вычисляет топ донатера с помощью запросов к слейву. Новый спонсор создается по схеме many-to-many между моделью и пользователем. Таблица и структура Sponsor. Может породить событие NewSponsor.
* SaveDonateListener - оперирует с моделью Donate и Message сохраняет в нужные таблицы и если нужно сохраняет модель сообщения. Инвалидирует кеш последних событий.
* SendDonateToQueeuListener - оперирует с моделью Donate и Message отправляет данные в очередь на отправку в ws
* TransferMoneyListener - оперирует с моделью Donate осуществляет отправку денежных средств с аккаунта пользователя на аккаунт модели, необходимо при реализации учесть всю логику с недостаточным балансом и тд и тп. Возможно при усовершенствовании сервиса с конвертацией валют. Порождает событие получение денег моделью ReceiveMoney

#### Событие `ReceiveMoney`

* UpdateGoalListener - оперирует с моделью Donate осуществляет обновление целей в бд (если она имеется) (прибавляет сумму) у модели. Обычно генерирует событие `UpdateGoal`, если цель существует и она не завершена. При завершении цели отправляет событие `EndGoal`.

#### Событие `CreateGoal`

* SendCreateGoalListener - отправляет событие о создании цели, на фронт

#### Событие `DeleteGoal`

* SendDeleteGoalListener - отправляет событие о удалении цели, на фронт

#### Событие `UpdateGoal`

* SendUpdateGoalListener - отправляет событие о обновлении цели, на фронт

#### Событие `EndGoal`

* SendEndGoalToQueue - при достижении цели необходимо оповестить пользователей в чате.

#### Событие `NewSponsor`

* SaveNewSponsor - необходимо сохранить нового спонсора для определенной модели. Инвалидирует кеш последних событий.
* SendNewSponsorToQueue - при вычислении нового спонсора необходимо всех оповестить в чате.

### Описание ендпойнтов

* [POST] `/chat/message`:
	#### Основное действие:

	Происходит отправка сообщения в топик в кафке, консьюмеры парралельно начинают обработку сообщения. SaveMessageListener - сохраняет сообщение в бд, а SaveMessageToQueueListener записывает сообщение в топик на отправку сообщения в вебсокет.

	#### Логика:
	Сообщения бывают двух типов приватные и неприватные, логика доставки сообщения для модели и конкретного пользователя в следующем:

	* При создании чата в node.js создается комната с уникальным owner_id, необходимо зарегистрировать connection_id в ендпойнте `/chat/connection/add`
	* При подписке пользователя на вебсокет создается уникальный connection_id (это идентификатор пользователя в ws), необходимо так же зарегистрировать connection_id в ендпойнте `/chat/connection/add`
	* При отправке события непосредственно в вебсокет необходимо передавать флаг is_public = false, а так же передавать connection_id и owner_id
	* При получении события на стороне вебсокета, необходимо ориентироваться на connection_id (для модели он будет равносилен owner_id на ws) и флаг is_public и в зависимости от флага будет отсылать сообщение либо всем кто подписан на данную комнату, либо только определенным connection и owner.



* [POST] `/chat/donate`
	#### Основное действие:

	Порождает событие `AddNewDonate`. Существуют четыре обработчика событий. `SaveDonateListener` - сохраняет донат в бд и порождает событие `AddNewMessage` при необходимости (донат с комментарием). `CalculateNewSponsor` - рассчитывает нового спонсора и может порождать событие `NewSponsor`. `SendDonateToQueeuListener` - отправляет событие доната в очередь на отправку в вебсокет. И заключительное событие отправляет деньги с одного счета на другой.

* [POST] `/chat/connection`
	#### Основное действие:

	Порождает событие `AddNewConnection`. Составляет связь между connection_id owner_id в ws и uuid пользователя и модели. 

* [GET] `/chat/event/history`

	#### Основное действие:

	Получает историю всех событий за последние десять минут. Необходимо доставать данные из кеша. Если данных нет осуществлять запросы в бд, и записывать в кеш.  
	#### Логика
	
	При получении списка событий мы проверяеем наличие по ключу в редисе. Если ключа нет, мы создаем кеш по ключу. Логика добавления такая - делаем запрос с джойнами во все события, сортируем по времени, отсекаем по времени, ставим лимит. Записываем в кеш. Есть вариант унифицировать все события в одной таблице и ее кешировать при отсутствии ключа и инвалидировать при сохранении истории.

* [POST][PUT][DELETE][GET] ` /chat/goals...`
	#### Основная логика
	
	CRUD на цели модели. При создании действует ограничение на одну цель. При обновлении цели стоит учесть что сумма сбора не может быть меньше уже собранной суммы.
	Создание цели генерирует событие `CreateGoal` - для отправки уведомления о создании новой цели.
	Обновление цели генерирует событие `UpdateGoal` - для обновления цели на фронте, например увеличить или уменьшить сумму сбора.
	Удаление цели генерирует событие `DeleteGoal` - для удаления текущей цели на фронте
