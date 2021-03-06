---
title: API v1.0 - API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - java
  - python
  - c++

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Введение

This is the ExpressPay API documentation. Here you can find out about the functionality available through the API, and how you can integrate your system into ExpressPay system. This is designed to allow you to automate your business account operations.

To help you navigate, this page is split into 3 vertical sections:

- the navigation menu and search on the left,
- the content in the middle,
- and usage examples on the right.

## Требования к интерфейсу

- Интерфейс должен принимать запросы по протоколу HTTP или HTTPS с IP-адресов подсетей:
- IP ?
- Интерфейс должен обрабатывать параметры, передаваемые системой методом GET.
- Интерфейс должен формировать ответ системе в формате XML в кодировке UTF-8.
- Обмен информацией ведется в режиме запрос-ответ, при этом скорость ответа не должна превышать 60 секунд, в противном случае система разрывает соединение по таймауту.
- Если предполагаемое количество платежей за услуги подключаемого дилера, ожидается интенсивным (до 10 платежей в минуту и более), необходимо, чтобы интерфейс поддерживал многопоточную коммуникацию до 10-15 одновременных соединений.
- Интерфейс должен принимать запросы по протоколу HTTP или HTTPS на один из следующих TCP- порта: PORT. Использование иных портов не допускается.

<aside class="notice">
IP - 
</aside>

## Принципы работы интерфейса

1. Каждый платеж в системе ExpressPay имеет уникальный идентификатор, который передается провайдеру в переменной txn_id - целое число длиной до 20 знаков. По этому идентификатору производится дальнейшая сверка взаиморасчетов и решение спорных вопросов.

2. Сумма платежа в валюте, указанной в переменной ccy (код валюты Alpha-3 ISO 4217), передается провайдеру в переменной sum - дробное число с точностью до сотых, в качестве разделителя используется `«.» (точка)`. Если сумма представляет целое число, то оно все равно дополняется точкой и нулями, например - «152.00».

3. В запросе на добавление платежа (запрос «pay»), система передает дату платежа (под датой платежа в системе подразумевается дата получения запроса от клиента) в переменной `txn_date`  - дата в формате `ГГГГММДДЧЧММСС`. По этой дате производится дальнейшая сверка взаиморасчетов между  ExpressPay и дилером. Например, клиент отправил запрос в систему  ExpressPay  `31.12.2017 в 23:59:59`, и система  ExpressPay  отправила свой запрос провайдеру `01.01.2018 в 00:00:05`. Это может привести к проблеме сверки платежей, если система провайдера поместит транзакцию в следующий расчётный период. Чтобы избежать подобных проблем,  ExpressPay  предоставляет провайдеру оригинальную дату платежа. 

4. Провайдер идентифицирует своего абонента по уникальному идентификатору (номер лицевого счета, телефона, логин и т.д.). Перед отправкой провайдеру, идентификатор проходит проверку корректности в соответствии с регулярным выражением, которое должен предоставить провайдер. Идентификатор абонента передается в переменной account - строка, содержащая буквы, цифры и спецсимволы, длиной до 200 символов.

5. Передача информации о платеже провайдеру производится системой в 2 этапа - проверка статуса абонента и непосредственно проведение платежа. Также может добавляться этап получения дополнительных параметров платежа от провайдера, предоставляющего абоненту несколько сервисов, для информирования плательщика и добавления параметров платежа по выбору плательщика.
Тип запроса передается системой в переменной command - строка, принимающая значения «prvId», «getbalance», «check»,«pay» и «getlnfo».
  * При проверке (запрос «prvId»),  протокол должен возвращать доступные на тот момент услуги 
  * При проверке (запрос «getbalance»),  протокол должен возвращать баланс дилера
  * При проверке статуса (запрос «check»), протокол должен проверить наличие абонента с указанным  идентификатором, суммы платежа в соответствии с принятой логикой пополнения лицевых счетов через платежные систему
  * При проведении платежа (запрос «pay»), протокол должен произвести пополнение баланса абонента

6. В случае если любой из запросов протокола завершается ошибкой, то протокол возвращает код ошибки в соответствии с таблицей, приведенной ниже (Приложение Б). Все ошибки имеют признак фатальности. Для системы фатальная ошибка означает, что повторная отправка запроса с теми же параметрами приведет к 100% повторению той же ошибки - следовательно, система прекращает обработку клиентского запроса и завершает его с ошибкой. Нефатальная ошибка означает для системы, что повторение запроса с теми же параметрами через некоторый промежуток времени, возможно, приведет к успеху. Система будет повторять запросы, завершающиеся не фатальной ошибкой, постоянно увеличивая интервал, пока операция не завершится успехом или фатальной ошибкой. Отсутствие связи с сервером провайдера является нефатальной ошибкой. Отсутствие в ответе элемента <result> (некорректный XML, страница Service temporarily unavailable и т.д.) - также является нефатальной ошибкой.

7. В информационной системе провайдера не должно содержаться двух успешно проведенных платежей с одним и тем же номером txn_id. Если система повторно присылает запрос с уже существующим в информационной системе провайдера идентификатором txn_id, то провайдер должен вернуть результат обработки предыдущего запроса.

8. Провайдер возвращает ответ на запросы системе в формате XML.

> Ответ:

```xml
  <response>
    <ep_txn_id>123323498</ep_txn_id>
    <prv_txn>12369Bdkjh9</prv_txn>
    <sum>100.00</sum>
    <ccy>643</ccy>
    <fields>
      <field name="prv-date">2012-04-05T12:00:07</field>
    </fields>
    <type hasList="true" hasInfo="true" />
    <extra>
      <list>
        <field name=".."></field>
      </list>
      <info>
        <field name=".."></field>
      </info>
    </extra>
    <result>0</result>
    <comment></comment>
  </response>
```
 
**Ответ**

Parameter | Description
--------- | -----------
response | тело ответа
ep_txn_id | номер транзакции в системе ExpressPay , который передается провайдеру в переменной txn_id. Этот элемент должен возвращаться провайдером после запросов на проверку состояния абонента и пополнение баланса (запросы `«check»` и `«pay»`)
prv_txn | уникальный номер операции пополнения баланса абонента (в базе дилера). Формат номера - строка не более 32 символов. Этот элемент должен возвращаться провайдером после запросов на проверку состояния абонента и пополнение баланса (запросы `«check»` и `«pay»`)
sum | сумма платежа, передаваемая провайдеру, дробное число с точностью до сотых, в качестве разделителя используется «.» (точка). Если сумма представляет целое число, то оно все равно дополняется точкой и нулями, например - «152.00». Этот элемент должен возвращаться провайдером после запросов на проверку состояния абонента и пополнение баланса (запросы `«check»` и `«pay»`)
ccy | валюта суммы платежа (код Alpha-3 ISO 4217). Этот элемент должен возвращаться провайдером после запросов на проверку состояния абонента и пополнение баланса (запросы `«check»` и `«pay»`)
result | код результата завершения запроса
comment | необязательный (опциональный) комментарий о результате выполнения операции
type | `<type hasList="true" hasInfo="true" />` - признак присутствия/отсутствия секций info и list в ответе. Тег должен содержать обязательные атрибуты `hasList` и `hasInfo`. При этом, если атрибут `hasList=”true”`, обязательно наличие секции `<list>` внутри
extra | * Аналогично, если атрибут `hasInfo=”true”`, обязательно наличие секции `<info>` внутри `<extra>`. Этот элемент должен возвращаться провайдером после запроса на получение дополнительных данных (запрос `«getInfo»`); 
  | * необязательный (опциональный) тег, содержащий список данных для клиента. Этот элемент должен возвращаться провайдером после запроса на получение дополнительных данных (запрос `«getInfo»`)
field | `<field name="prv-date">` - дата и время принятия платежа у провайдера в формате `ГГГГ-ММ-ДД’Т’ЧЧ:ММ:СС` (например, `2011-02-03T00:00:07`). 
  | * Этот элемент должен возвращаться провайдером после запроса на пополнение (запроса `«pay»`). 
  | * Эта дата используется для бухгалтерских взаиморасчетов. Например, имеет место ситуация: клиент прислал в систему запрос `31.12.2010 в 23:59:59`. Учитывая задержку на обработку данных и пересылку информации по каналам связи, провайдером платеж будет получен 1.01.2011 00:00:05 и, соответственно, учтен в системе провайдера в другом отчетном периоде. Чтобы при проведении сверок проблем с разными отчетными периодами не возникало, необходимо, чтобы провайдер возвращал дату, по которой производится учет в его системе.


(9) Протокол (запросы `«check»` и `«pay»`)  предусматривает возможность передачи дополнительных реквизитов платежа (экстра полей) провайдеру в формате `extra\parameter_name]=parameter_value`. Данная возможность может быть использована в случае, если платёж невозможно провести без дополнительных данных (одного идентификатора пользователя в системе провайдера недостаточно). Например, идентификатор пользователя -
номер кредитной карты, но для проведения платежа также нужно указать срок действия карты. На имена параметров налагаются следующее ограничения: допустимыми являются лишь цифры (0-9), подчёркивание `(_)` и строчные буквы латинского алфавита `(a-z)`. Перечень необходимых полей для передачи провайдеру необходимо указывать в заявке на подключение (Приложение А). 

(10) Для поддержки расширяемости и сохранения работоспособности сервиса провайдера в период включения различных функций, предусмотренных протоколом (например, включение передачи новых реквизитов платежа), предполагается, что провайдер не препятствует появлению в запросе новых HTTP- параметров. Гарантируется, что появление новых параметров в запросе не приведет к необходимости изменения обработки запросов со стороны провайдера, за исключением случаев, когда такое изменение логики было согласовано с провайдером.

# API

## Проверки Баланса Дилера 

`GET http://IP:PORT/test.asp?command=getbalance&login=login&ccy=USD&sign=md5(login+password)`

**URL Parameters**

Имя элемента | Обязательно | Пример / Описание
-------------|-------------|-------------------
Command | ДА | `getbalance` - идентификатор запроса на проверку баланса дилера
Login | ДА | Ваш логин
Prvid | ДА | `id`  провайдера, полученное из списка провайдеров (Глава 4). `prvid` уникален в нашей системе
Sign | ДА | Хеш результат `md5(login+password)`

> Ответ:
 
```xml
<response>
  <result>0</result>
  <action>Get balance</action>
  <date>12.01.2018 13:38:29</date>
  <comment>OK</comment>
  <result>0</result>
  <balance>-1</balance>
  <limit>0</limit>
</response>
```

**Ответ**

Имя элемента | Обязательно | Тип | Описание
-------------|-------------|-----|---------
ep_txn_id | ДА | String | внутренний номер платежа в системе ExpressPay
prv_txn | ДА | String | Уникальный код транзакции
Sum | ДА | Float | сумма к зачислению на лицевой счет абонента;
Ccy | ДА | String | Валюта 
Result | ДА | Integer | Код результата 
Comment | ДА | String | Результат обработки запроса

## Проверка доступных операторов

`GET http://IP:PORT/test.asp?command=prvid&login=login&prvid=&sign=md5(login+password)`

**URL Parameters**
 
Имя элемента  | Обязательно | Пример / Описание 
--------------|-------------| -----------------
Command | ДА | prvid - идентификатор запроса на доступных операторов;
Login | ДА | Ваш логин
Prvid | ДА | prvid  провайдера, полученное из списка провайдеров (Глава 4) .prvid уникален в нашей системе
Sign | ДА | Хеш результат md5(login+password)


> Ответ:

```xml
<response>
  <result>0</result>
  <action>Providers</action>
  <date>11.01.2018 22:42:47</date>
  <comment>OK</comment>
  <providers>
    <provider>
      <articul>62</articul>
      <typeEnter>0</typeEnter>
      <prefix>918,98</prefix>
      <name>Вавилон-М</name>
      <digits>9</digits>
      <info>YES</info>
      <category>1</category>
        Упешный ответ
      <result>0</result>
      <maxsum>0</maxsum>
      <minsum>0</minsum>
    </provider>
  </providers>
</response>
```

**Ответ**

Имя элемента | Тип | Описание | Обязательно 
-------------|-----|----------|------------
Result | Integer | Код результата | ДА
Date | Date | Дата и время обработки запроса | ДА
Articul | String | id  провайдера, полученное из списка провайдеров. `prvid` уникален в нашей системе | ДА
typeEnter | Integer | 0-цифры, 1- цифры и  буквы | ДА
Prefix | String | Префиксы  провайдера (разделенные запятыми) Например: Вавилон-М имеет префиксы 918 и 98. Это значит передаваемый параметр number в action 1 должен начинаться с номером 918 или 98. Если значение тега `<prefix></prefix>` пустой, то у этого провайдера нет префикса. | ДА. Вы должны контролировать
Comment | String | Результат обработки запроса | ДА
Maxsum | Money | Максимальная сумма платежа. Если возвращает пустое значение, то нет ограничения на пополняемую сумму | ДА
Minsum | Money | Минимальная сумма платежа | ДА

## Возврат курса

`GET http://IP:PORT/test.asp?command=getcurrency&login=login&account=992928901106&prvid=70&sum=1&ccy=USD&sign=md5(login+password)`

**URL Parameters**

Имя элемента | Обязательно | Пример / Описание
-------------|-------------|------------------
Command | ДА | getcurrency - идентификатор запроса на проверку баланса дилера
Login | ДА | Ваш логин
Prvid | ДА | id  провайдера, полученное из списка провайдеров (Глава 4). `prvid` уникален в нашей системе
Sign | ДА | Хеш результат `md5(login+password)`

> Ответ:

```xml
<response>
  <result>0</result>
  <command>getcurrency</command>
  <date>25.05.2018 14:53:02</date>
  <comment>OK</comment>
  <balance>9,364152</balance>
</response>
```

**Ответ**

Имя элемента | Тип | Описание
-------------|-----|---------
result | | 
command | | 
date | | 
comment | | 
balance | | 

## Проверка существование счета абонента

Условия примера: Платежное приложение провайдера test.asp, располагается по адресу IP. Для проверки состояния абонента, система генерирует запрос следующего вида:

`GET http://IP:PORT/test.asp?command=check&login=login&txn_id=1234567&account=918401468&prvid=62&ccy=USD&sum=10.45`

**URL Parameters**

Имя элемента | Обязательно | Пример / Описание
-------------|-------------|-------------------
command | ДА | check - идентификатор запроса на проверку существование абонента
login | ДА | Ваш логин
account | ДА | Номер абонента 
txn_id | ДА | Ваш уникальный номер транзакции
prvid | ДА | prvid  провайдера, полученное из списка провайдеров (Глава 4) .prvid уникален в нашей системе
sum | ДА | Сумма пополнения
sign | ДА | Хеш результат `md5(login+txn_id+account+password)`

> Ответ:

```xml
<response>
  <ep_txn_id>login_1234567</ep_txn_id>
  <result>0</result>
  <prv_txn>1234567</prv_txn>
  <sum>10.45</sum>
  <ccy/>
  <result>0</result>
  <comment>OK</comment>
</response>
```

**Ответ**

Имя элемента | Тип | Описание | Обязательно 
-------------|-----|----------|-------------
ep_txn_id | String | внутренний номер платежа в системе ExpressPay | ДА
prv_txn | String | Уникальный код транзакции | ДА
sum | Float | сумма к зачислению на лицевой счет абонента | ДА
ccy | String | Валюта | ДА
result | Integer | Код результата | ДА
comment | String | Результат обработки запроса | ДА

<aside class="notice">
Возвращение <code>result=0</code> на запрос <code>«check»</code> свидетельствует о том, что лицевой счет абонента с соответствующим ему номером в поле account может быть пополнен на сумму, указанную в запросе в поле sum. После успешной проверки состояния счета абонента система переходит к формированию и отправке запроса на пополнение баланса (запрос <code>«pay»</code>).

В необязательном поле <code>comment</code> содержится служебный комментарий.
</aside>

## Пополнение лицевого счета

Условия примера: Платежное приложение провайдера test.asp, располагается по адресу IP. Для подтверждения платежа, система генерирует запрос следующего вида: 

`GET http://IP:PORT/test.asp?command=pay&login=login&txn_id=1234567&prvid=70&account=927700463&ccy=USD&sum=10.00&sign=<sign>)`

**URL Parameters**

Имя элемента | Обязательно | Пример / Описание
-------------|-----|----------|------------
command | ДА | pay - идентификатор запроса на пополнение баланса абонента
login | ДА | Ваш логин 
account | ДА | Номер абонента 
txn_id | ДА | Ваш уникальный номер транзакции
prvid | ДА | id  провайдера, полученное из списка провайдеров (Глава 4). `prvid` уникален в нашей системе
sum | ДА | Сумма пополнения
sign | ДА | Хеш результат `md5(login+txn_id+account+password)`

> Ответ:

```xml
<response>
  <ep_txn_id>login_1234567</ep _txn_id>
  <prv_txn>1234567</prv_txn>
  <sum>10.45</sum>
  <ccy>RUB</ccy>
  <result>0</result>
  <comment>OK</comment>
  <fields>
    <field name="prv-date">2011-08-15T12:06:45</field>
  </fields>
</response>
```

**Ответ**

Имя элемента | Тип | Описание | Обязательно 
-------------|-----|----------|------------
ep_txn_id | String | внутренний номер платежа в системе ExpressPay | ДА
prv_txn | String | Уникальный код транзакции | ДА
sum | Float | сумма к зачислению на лицевой счет абонента | ДА
ccy | String | Валюта | ДА
result | Integer | Код результата | ДА
comment | String | Результат обработки запроса | ДА

<aside class="notice">
Возвращая `result=0` на запрос «pay», провайдер сообщает об успешном завершении операции пополнения баланса. Система полностью завершает обработку данной транзакции. 

В необязательном поле `comment` содержится служебный комментарий.
</aside>


## Отмена и корректировка платежа
Условия примера: Платежное приложение провайдера test.asp, располагается по адресу IP. Для отмены или корректировка платежа, система генерирует запрос следующего вида: 

`GET http://IP:PORT/test.asp?command=cancel&login=login&txn_id=1234567&newaccount=xxx&ccy=&sign=<sign>`

**URL Parameters**

Имя элемента | Обязательно | Пример / Описание
-------------|-------------|------------------
command | ДА | cancel - идентификатор запроса на отмена корректировка платежа
login | ДА | Ваш логин
txn_id | ДА | Ваш уникальный номер транзакции
account | ДА, если корректировка | Правильный номер абонента для корректировки
sign | ДА | Хеш результат `md5(login+txn_id+password)`

> Ответ:

```xml
<response>
  <osmp_txn_id>1234567</osmp_txn_id>
  <prv_txn>1234567</prv_txn>
  <result>31</result>
  <cancel_amount>10</cancel_amount>
  <comment>
    The application was successfully corrected/canceled. Заявка успешно корректирован/аннулирован.
  </comment>
  <fields>
    <field name="prv-date">08.03.2018 2:06:04</field>
  </fields>
</response>
```


**Ответ**

Имя элемента | Тип | Описание | Обязательно 
-------------|-----|----------|-------------
osmp_txn_id | String | внутренний номер платежа в системе ExpressPay | ДА 
prv_txn | String | Уникальный код транзакции | ДА 
cancel_amount | Float | Отменённая или корректированная сумма, только при успешном результате | ДА 
result | Integer | Код результата | ДА
comment | String | Результат обработки запроса | ДА


**Возвращаемые коды результатов по отмене/корректировке платежа:**

Код | Комментарий (en) | Комментарий (ру)| Фатальность
----|------------------|-----------------|------------
31 | The application was successfully corrected/canceled | Заявка успешно корректирован/аннулирован | +
30 | The application was accepted. In the process | Заявка принято. В процессе | 
17 | Cancel payment isn't provided | Отмена платежа не поддерживается | +
18 | Payment in the process. Cancel payment isn't provided | Платеж в очереде. Невозможно отменить. +99244630999 | +
19 | Provide only edit. Please enter new account for edit | Поддерживается только корректировка. Введите правильный счет | +
21 | The application was rejected by the provider | Заявка отклонена поставщиком | +
22 | Sum too few | Сумма слишком маленькая | +
