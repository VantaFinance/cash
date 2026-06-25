# Документация Абстрактного api банка

Данный документ описывает желаемый процесс получение кредита наличными.
Ниже представлен workflow процессы и сам api. 


## Глоссарий

| Название сервиса | Описание                                                                            |
|------------------|-------------------------------------------------------------------------------------|
| Marketplace      | Оператор финансовой платформы                                                       |
| Cash Bus         | Шина взаимодействия с финансовыми организациями, предоставляющими кредиты наличными |
| Cash Conveyor    | Конвейер кредитов наличными, являющийся точкой входа для оформления кредита         |
| Bank             | Финансовая организация                                                              |

---

# Возможные способы доставки событий

> <span style="color:red">**ВНИМАНИЕ! Все способы доставки событий доступны исключительно через VPN-туннель между Marketplace и инфраструктурой партнера.**</span>

## Защитные механизмы

Реализация endpoint's: 
- `GET /api/v1/application/{id}/offers` 
- `GET /api/v1/loan-agreement/{id}/draft/status`
- `GET /api/v1/loan-agreement/{id}/status`


является обязательной для всех вариантов интеграции.
Cash Bus использует данный endpoint в качестве резервного механизма получения статуса заявки.</br>
При недоступности основного канала доставки событий система автоматически переключается на периодический опрос (polling) для синхронизации состояния заявки.</br>
Если ни один из способов доставки событий не реализован, Cash Bus будет использовать исключительно polling-модель взаимодействия.</br>
При использовании только polling-модели необходимо учитывать, что синхронизация статусов происходит с задержкой, зависящей от настроек интервала опроса. 
<br>
В результате возможны временные расхождения между статусом заявки в системе партнера и в Marketplace. 
<br>
Например, заявка уже может быть авторизована в системе банка, тогда как в Marketplace ее статус еще не обновлен.

---

## Temporal Nexus

> <span style="color:red">**ВНИМАНИЕ! Приоритетный и рекомендуемый способ доставки событий.**</span>

- [Что такое Temporal?](https://docs.temporal.io/temporal)
- [Что такое Temporal Nexus?](https://docs.temporal.io/nexus)

Marketplace предоставляет партнеру доступ к выделенному Nexus Endpoint в Temporal.</br>
Партнерская система инициирует вызов Nexus-операции, после чего событие маршрутизируется во внутренний контур и запускает соответствующий Workflow в системе Cash Bus.</br>
Такой подход обеспечивает надежную доставку событий, встроенные механизмы повторных попыток, трассировку выполнения и централизованное управление бизнес-процессами.

### Схема взаимодействия

```mermaid
flowchart LR
    Bank[Bank]

subgraph Публичный Nexus
TN["Public Temporal Nexus"]
end

subgraph Внутренний Nexus
IT["Internal Temporal (cash-bus)"]
end

Bank -->|Вызов Nexus-операции| TN
TN -->|Запуск Workflow и передача сигнала| IT
```

---

## Kafka

Marketplace предоставляет партнеру доступ к Kafka-кластеру и выделенному топику для публикации событий.</br>
Партнерская система самостоятельно публикует события в указанный топик, после чего сервис Cash Bus считывает их и выполняет дальнейшую обработку.

### Схема взаимодействия

```mermaid
flowchart LR
    Bank[Bank] -->|Публикует событие| TOPIC[(Kafka Topic)]
    TOPIC -->|Читает событие из топика| CASHBUS[Cash Bus]

    subgraph KAFKA["Публичный Kafka Cluster"]
        TOPIC
    end
```





# Workflow взаимодействия с API



## Скоринг заявки

```mermaid
sequenceDiagram
    
box rgb(166,83, 266) Marketplace
    participant Cash Bus
end

box Bank
    participant Bank
end



Cash Bus->>+ Bank : Создает заявку /api/v1/application
alt Если заявка соотвествует доменным правилам
    Bank->>Cash Bus: Возвращаем Идентификатор заявки
else Если с заявкой что-то не так
    rect rgb(210, 46,46)
        Bank->>-Cash Bus: Возвращаем ошибку 
    end
end



rect rgb(0, 156, 65)
    par Если найдено банковское предложение
        Bank->>+Cash Bus: Нотифицируем событие onUpdatedScoring
        Cash Bus->>-Bank: Получаем результат ручки /api/v1/application/{id}/offers
    end
end


rect rgb(210, 46,46)
    alt Если отказано в банковских предложениях
        Bank->>+Cash Bus: Нотифицируем событие onUpdatedScoring
        Cash Bus->>-Bank: Получаем результат ручки /api/v1/application/{id}/offers
    end
end



rect rgb(210, 46,46)
    loop Пока не истек ttl, каждые 30 секунд
        Cash Bus->>Bank: Получаем результат ручки /api/v1/application/{id}/offers
        Note right of Cash Bus: Наблюдение начинается спустя определенное время,<br/> если заявка не в конечном статусе
    end    
end



```



## Авторизации договора займа


```mermaid
sequenceDiagram

box rgb(166,83, 266) Marketplace
    participant Cash Bus
end

box Bank
    participant Bank
end


Cash Bus ->>+ Bank : Запрашиваем договор займа /api/v1/offer/{id}/print-draft-agreement
Bank ->>- Cash Bus : Возвращаем  идентификатор черновика договора займа

rect rgb(210, 46,46)
    Note right of Cash Bus: Начинаем наблюдать за "печатью" черновика договора займа
    opt Bank присылает событие при смене статуса печати 
        Bank --) Cash Bus: onUpdatedDraftStatus — статус печати изменился
        Note right of Cash Bus: Можно сразу запросить статус, не дожидаясь следующего опроса
    end
    loop Пока не получили конечный статус, каждые 30 секунд
        Cash Bus ->> Bank:  Получаем статус "печати" /api/v1/loan-agreement/{id}/draft/status

        alt Если статус печати "WAITING"
            Note right of Cash Bus: Продолжаем наблюдать за "печатью"
        end     

        rect rgb(0, 156, 65)
            alt Если статус печати "SUCCESS"
                Cash Bus->>Bank: Забираем черновик договора займа /api/v1/loan-agreement/{id}/draft
            end
        end

        alt Если статус печати "ERROR"
            Note right of Cash Bus: Через N время пробуем снова запустить "печать" договора<br/> Если через N попыток получаем ERROR, завершаем клиентский путь
        end

        alt Если статус печати "REJECTED"
            Note right of Cash Bus: Банк отказал в получения черновика. Дальнейшая обработка на стороне Cash Bus
        end
    end
end


Cash Bus ->>+ Bank : Загружаем подписанный договор займа с помощью ЭЦП /api/v1/loan-agreement/{id}/upload
Bank ->>- Cash Bus : Возвращает идентификатор подписанного договора займа

Cash Bus ->> Bank: Забираем детали подписанного договора /api/v1/loan-agreement/{id}/details


rect rgb(210, 46,46)
    Note right of Cash Bus: Начинаем наблюдать за авторизацией договора
    opt Bank присылает событие при смене статуса договора 
        Bank --) Cash Bus: onUpdatedLoanAgreementStatus — статус договора изменился
        Note right of Cash Bus: Можно сразу запросить статус, не дожидаясь следующего опроса
    end
    loop Пока не получили конечный статус авторизации, каждые 30 секунд
        Cash Bus ->> Bank:  Получаем статус авторизации договора /api/v1/loan-agreement/{id}/status

        alt Если статус авторизации "WAITING"
            Note right of Cash Bus: Продолжаем наблюдать за авторизацией
        end

        rect rgb(0, 156, 65)
            alt Если статус авторизации "SUCCESS"
                Cash Bus->>Bank: Забираем подписанный договора займа /api/v1/loan-agreement/{id}/signed
            end
        end

        alt Если статус авторизации "ERROR"
            Note right of Cash Bus: Через N время пробуем снова запустить авторизацию договора<br/> Если через N попыток получаем ERROR, завершаем клиентский путь
        end

        alt Если статус авторизации "REJECTED"
            Note right of Cash Bus: Отказано в авторизации договора<br/> Дальнейшая обработка на стороне Cash Bus
        end

        alt Если статус авторизации "CANCELED"
            Note right of Cash Bus: Клиент отказался от авторизации договора<br/> Дальнейшая обработка на стороне Cash Bus
        end
    end
end


alt Клиент захотел перевести денежные средства на ОФП счет 
    
Cash Bus ->> Bank : Запустили процесс финансирования в другой банк /api/v1/loan-agreement/{id}/financing


rect rgb(210, 46,46)
    Note right of Cash Bus: Начинаем наблюдать за финансированием в другой банк
    opt Bank присылает событие при смене статуса финансирования (webhook, опционально)
        Bank --) Cash Bus: onUpdatedFinancingStatus — статус финансирования изменился
        Note right of Cash Bus: Можно сразу запросить статус, не дожидаясь следующего опроса
    end
    loop Пока не получили конечный статус финансирования, каждые 30 секунд
        Cash Bus ->> Bank:  Получаем статус финансирования договора /api/v1/loan-agreement/{id}/financing/status

        alt Если статус финансирования "WAITING"
            Note right of Cash Bus: Продолжаем наблюдать за финансированием
        end

        rect rgb(0, 156, 65)
            alt Если статус финансирования "SUCCESS"
                Note right of Cash Bus: Завершаем клиентский путь, сообщаем клиенту что успешно перевели средства
            end
        end

        alt Если статус финансирования "ERROR"
            Note right of Cash Bus: Через N время пробуем снова запустить финансирование<br/> Если через N попыток получаем ERROR, завершаем клиентский путь
        end


        alt Если статус финансирования "REJECTED"
            Note right of Cash Bus: Банк отказал в переводе денежных средств<br/> Дальнейшая обработка на стороне Cash Bus
        end
    end
end

end
```
