# Микросервис сообщений в мессенджере

## Структура
* взаимодействие сервер-клиент происходит с помощью сообщений, с каждым действием связано соответствующее сообщение(log in, log out...). Сообщение инкапсулирует в себе все необходимы данные для обработки.
* мессенджер поддерживает обработку сообщений от пользователя в специальном формате, начинающемся с "/"
* в случае ошибки обработки сообщения, сервер возвращает сервисное сообщение со статусом ошибки

## Архитектура мессенджера
![alt text](https://github.com/MikhailMe/mes_db/blob/master/architecture.png)

## Типы сообщений, которыми общаются клиент и сервер
* MSG_LOGIN - залогиниться (если логин не указан, то авторизоваться)
* MSG_INFO - получить всю информацию о пользователе
* MSG_CHAT_LIST - получить список чатов пользователя
* MSG_CHAT_CREATE - создать новый чат
* MSG_CHAT_HIST - список сообщений из указанного чата
* MSG_TEXT - отправить текстовое сообщение в заданный чат

### В данном случае мы будем рассматриваем только текстовые сообщения

## Возможности, которые предоставляем
* обеспечить общение между двумя клиентами или группами клиентов(макс. 50 человек)
* отправка сообщений от клиента другим клиентам
* пересылка сообщений из одного диалога в другой
* удаление сообщений из диалога (у другого клиента все сообщения сохраняются - не удаляем из бд, а просто ставим флаг)
* удаление целого диалога (у другого клиента диалог не пропадает - не удаляем из бд, а просто ставим флаг)
* обеспечить возможность прикреплять различные файлы
* хранить все сообщения в каждом диалоге
* отображать последние 15 сообщений в каждом диалоге(остальные подгружать по мере надобности)
* возможность закреплять сверху наиболее "важные" диалоги

## Требования к сообщениям
* максимальная длина сообщения: 4096 символов
* максимальное количество собеседников в диалогах – 50 собеседников
* максимальное количество вложений – 10 вложений

## Хранимые данные
![alt text](https://github.com/MikhailMe/mes_db/blob/master/diagram.png)

## Консистентное хеширование
если ресайзим таблицу с помощью консистентного хеширования, то должны отремапить только часть ключей (K/n). K - число ключей, n - число слотов
![alt text](https://github.com/MikhailMe/mes_db/blob/master/riak-data-distribution.png)
У нас 2^160 - слотов, node (1-4) - физические ноды = машины
каждая физическая машина будет держать несколько виртуальных нод
партиция = слот, и к какой машине этот слот привязан
у каждой машины по 8 партиций
если какая-то машина переполнена, то можно перемапить какаую-то партицию и она будет на другой машине
![alt text](https://github.com/MikhailMe/mes_db/blob/master/replication.png)
пишем на 3 последовательные партиции и это будут реплики
мы знаем, что ключ попадает в первую партицию, но запишем ещё в 2 следующие партиции
так как машины чередуются, то информация будет на разных машинах
когда ищем ключ, идём в первую партицию, эта машина оффлайн, но знаем, что было 3 реплики, идём на следующую машину по кольцу и вынимаем значение

## CAP
### Основные требования
#### C - consistency
* на всех репликах должна храниться одна и та же информация
* сообщение отправленное от клиента1 клиенту2 должно быть отмечено у клиента2, как "принятое", а клиента1, как "отправленное"
#### A - availability
* каждый клиент имеет доступ к отправленным/полученным сообщениям от других клиентов
* запись/изменение данных либо во все реплики, либо ни в одну
#### P - partition tolerance
* расщепление распределённой системы на несколько изолированных секций не приводит к некорректности отклика от каждой из секций

### В данном случае мы жертвуем доступностью, то есть если у нас что-то не отвечает, то ждём пока не придёт ответ. Если всё хорошо, то пишем во все реплики.

## Методы API
* Метод отправки сообщения
#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер конкретного диалога  |
| textMessage   | текст сообщения     |
| stringInfo    | информация об отправке|

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-доставлено, false-нет|

* Метод удаления сообщения из диалога
#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| messageID     | номер сообщения, которое удаляем  |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-удалено, false-нет|

* Метод создания нового диалога

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| listClientsID | список ID участников  |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-создан, false-нет|

* Метод удаления определённого диалога

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер диалога, который удаляем  |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-удалено, false-нет|

* Метод добавления клиента(ов) в определённый диалог

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер диалога, в который добавляем клиентов |
| listClientsID | список ID клиентов, которых добавляем в диалог |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-добалены, false-нет|

* Метод удаления клиента(ов) из оперделённого диалога

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер диалога, из которого удаляем клиентов |
| listClientsID | список ID клиентов, которых удаляем из диалога |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-удалены, false-нет|

* Метод добавления вложения(документа) в диалог

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер диалога, в который добавляем клиентов |
| document      | документ, которых добавляем в диалог |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-добален, false-нет|


* Метод удалиения вложения в диалог

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер диалога, из которого удаляем документ |
| document      | документ, который удаляем из диалога |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-удалён, false-нет|

* Метод регистрации клиента

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| login         | логин клиента      |
| password      | пароль клиента     |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-зарегистрирован, false-нет|


## API, которое реализовали

* Метод отправки сообщения
#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| chatID        | номер конкретного диалога  |
| textMessage   | текст сообщения     |
| stringInfo    | информация об отправке|

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-доставлено, false-нет|

* Метод создания нового диалога

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| listClientsID | список ID участников  |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-создан, false-нет|

* Метод удаления сообщения по ID из всех диалогов

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| messageID     | номер сообщения, которое удаляем|

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-удалено, false-нет|

* Метод регистрации клиента

#### Запрос

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| login         | логин клиента      |
| password      | пароль клиента     |

#### Ответ

|**Параметр**   |**Описание**        |
| --------------|--------------------|
| statusMessage | true-зарегистрирован, false-нет|
