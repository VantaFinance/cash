# Документация Абстрактного API МФО

Данный документ описывает желаемый процесс получение кредита наличными.
Ниже представлен workflow процессы и сам api.


#### Глосарий


| Название сервиса | Описание                                                                   |
|------------------|----------------------------------------------------------------------------|
| Cash Bus         | Шина для работы с фин.организациями, которые предоставляют кредит наличных |
| Cash Conveyor    | Ковеер кредита наличных, точка входа для взятие кредита                    |
| MFO              | Микрофинаносвая организация, которая предоставляет кредит наличных         |   



## Возможные транспорты доставки событий

**<span style="color:red">ВНИМАНИЕ! ВСЕ Доступные транспорты доступны ТОЛЬКО через VPN тунель!</span>**


### Temporal Nexus

**<span style="color:red">ВНИМАНИЕ! ПРИОРИТЕТНЫЙ И ЖЕЛАЕМЫЙ метод доставки, событий</span>**

- [Что такое Temporal?](https://docs.temporal.io/temporal)
- [Что такое Temporal Nexus?](https://docs.temporal.io/nexus)


Marketplace предоставляет партнеру доступ к выделенному Nexus Endpoint в Temporal. 
Партнерская система инициирует вызов Nexus-операции, после чего событие маршрутизируется во внутренний контур и запускает соответствующий Workflow в системе Cash Bus.
</br>
Такой подход обеспечивает надежную доставку событий, встроенные механизмы повторных попыток, трассировку выполнения и централизованное управление процессами.


#### Cхема взаимодействия

```mermaid
flowchart LR
    MFO[MFO]

subgraph Публичный Nexus
TN["Public Temporal Nexus"]
end

subgraph Внутренний Nexus
IT["Internal Temporal (cash-bus)"]
end

MFO -->|Вызов nexus операции| TN
TN -->|Запуск worfklow c сигналом| IT
```



### Kafka

Marketplace предоставляет партнеру доступ к Kafka-кластеру и выделенному топику для публикации событий.</br>
Партнерская система самостоятельно публикует события в указанный топик, после чего сервис Cash Bus считывает их и выполняет дальнейшую обработку.



#### Cхема взаимодействия


```mermaid
flowchart LR
    MFO[MFO] -->|Публикует событие| TOPIC[(Kafka Topic)]
    TOPIC -->|Читает событие из топика| CASHBUS[Cash Bus]

    subgraph KAFKA["Публичная Kafka Cluster"]
        TOPIC
    end
```





## Описание Workflows



### Workflow авторизации договора займа

```mermaid
sequenceDiagram

box rgb(166,83, 266) Marketplace
    participant Cash Bus
    participant Cash Conveyor
end

box MFO
    participant MFO
end

MFO ->> MFO : Клиент подписывает договор займа
MFO ->>+ Cash Bus : МФО сообщает об начале авторизации займа
MFO ->>+ Cash Conveyor : МФО возвращает клиента на конвеер кэшей


rect rgb(210, 46,46)
    loop Пока не получили конечный статус авторизации
        Note right of Cash Bus: Начинаем наблюдать за авторизацией договора
        Cash Bus ->> MFO:  Получаем статус авторизации договора /api/v1/application/{id}

        alt Если статус авторизации договора "AUTHORIZE_PENDING"
            Note right of Cash Bus: Продолжаем наблюдать за авторизацией договора
        end

        rect rgb(0, 156, 65)
            break Если статус авторизации "AUTHORIZED"
                Cash Bus->>MFO: Забираем подписанный договора займа /api/v1/loan-agreement/{id}/signed
                Cash Bus->>MFO: Забираем черновик договора займа /api/v1/loan-agreement/{id}/draft
                Cash Bus->>MFO: Забираем детали авторизиованного договора /api/v1/loan-agreement/{id}/details
            end
        end


        rect rgb(60, 115, 168)
            break Если статус авторизации "AUTHORIZED_REJECTED"
                Cash Bus->>Cash Conveyor: Сообщаем об отказе авторизации займа (МФО решил сделать отказ на последнем шаге)
                Cash Conveyor->>+Cash Conveyor: Своя логика обработки
            end
        end



        rect rgb(60, 115, 168)
            break Если статус авторизации "AUTHORIZED_CANCELLED"
                Cash Bus->>Cash Conveyor: Сообщаем об отмене авторизации займа (клиент отказался)
                Cash Conveyor->>+Cash Conveyor: Своя логика обработки
            end
        end
        
    end
end
```







