# Документация абстрактного API МФО

Данный документ описывает целевой процесс получения кредита наличными.

Ниже представлены workflow-процессы и описание API.

## Глоссарий

| Название сервиса | Описание                                                                            |
|------------------|-------------------------------------------------------------------------------------|
| Marketplace      | Оператор финансовой платформы                                                       |
| Cash Bus         | Шина взаимодействия с финансовыми организациями, предоставляющими кредиты наличными |
| Cash Conveyor    | Конвейер кредитов наличными, являющийся точкой входа для оформления кредита         |
| MFO              | Микрофинансовая организация, предоставляющая кредиты наличными                      |

---

# Возможные способы доставки событий

> <span style="color:red">**ВНИМАНИЕ! Все способы доставки событий доступны исключительно через VPN-туннель между Marketplace и инфраструктурой партнера.**</span>

## Защитные механизмы

Реализация endpoint `GET /api/v1/application/{id}` является обязательной для всех вариантов интеграции.
Cash Bus использует данный endpoint в качестве резервного механизма получения статуса заявки.</br>
При недоступности основного канала доставки событий система автоматически переключается на периодический опрос (polling) для синхронизации состояния заявки.</br>
Если ни один из способов доставки событий не реализован, Cash Bus будет использовать исключительно polling-модель взаимодействия через endpoint `GET /api/v1/application/{id}`.</br>
При использовании только polling-модели необходимо учитывать, что синхронизация статусов происходит с задержкой, зависящей от настроек интервала опроса. В результате возможны временные расхождения между статусом заявки в системе партнера и в Marketplace. Например, заявка уже может быть авторизована в системе МФО, тогда как в Marketplace ее статус еще не обновлен.

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
    MFO[MFO]

subgraph Публичный Nexus
TN["Public Temporal Nexus"]
end

subgraph Внутренний Nexus
IT["Internal Temporal (cash-bus)"]
end

MFO -->|Вызов Nexus-операции| TN
TN -->|Запуск Workflow и передача сигнала| IT
```

---

## Kafka

Marketplace предоставляет партнеру доступ к Kafka-кластеру и выделенному топику для публикации событий.</br>
Партнерская система самостоятельно публикует события в указанный топик, после чего сервис Cash Bus считывает их и выполняет дальнейшую обработку.

### Схема взаимодействия

```mermaid
flowchart LR
    MFO[MFO] -->|Публикует событие| TOPIC[(Kafka Topic)]
    TOPIC -->|Читает событие из топика| CASHBUS[Cash Bus]

    subgraph KAFKA["Публичный Kafka Cluster"]
        TOPIC
    end
```

---

# Описание Workflow


## Скоринг заявки

```mermaid
sequenceDiagram

actor Client as Клиент

box rgb(166,83,266) Marketplace
    participant Cash Conveyor
    participant Cash Bus
end

box MFO
    participant MFO
end

Client ->> Cash Conveyor : Заполняет заявку на займ
Cash Conveyor ->> Cash Bus : Передача данных заявки

Cash Bus ->>+ MFO : Создание заявки на займ<br/>POST /api/v1/application<br/>(antifraudProviders, redirectUrls)
MFO -->>- Cash Bus : Идентификатор заявки (id)

MFO ->> MFO : Запуск скоринга (асинхронно)

rect rgb(210,46,46)
    loop До получения конечного статуса скоринга
        Note right of Cash Bus: Отслеживание статуса скоринга

        alt Событийный канал
            MFO ->> Cash Bus : Нотификация изменения скоринга<br/>UpdatedScoringNotification<br/>
        else Polling (резервный канал)
            Note right of Cash Bus: При отсутствии нотификаций
        end

        Cash Bus ->> MFO: Получение статуса заявки<br/>GET /api/v1/application/{id}

        alt Статус = SCORING_PENDING
            Note right of Cash Bus: Скоринг не завершён,<br/>ожидаем следующее изменение
        end

        rect rgb(0,156,65)
            break Статус = SCORING_APPROVED
                Note right of MFO: В ответе присутствует<br/>список предложений (offers)
                Cash Bus ->> Cash Conveyor : Передача одобренных предложений (offers)
                Cash Conveyor ->> Client : Отображение предложений<br/>(сумма, срок, ставка, платёж)
                Client ->> Cash Conveyor : Выбор предложения
                Cash Conveyor ->> Client : Редирект на экран МФО<br/>continueUrl выбранного offer
                Client ->> MFO : Переход на экран МФО<br/>для продолжения оформления
            end
        end

        rect rgb(60,115,168)
            break Статус = SCORING_REJECTED
                Cash Bus ->> Cash Conveyor : Скоринг отклонён
                Cash Conveyor ->> Client : Отказ в выдаче займа
            end
        end

        rect rgb(60,115,168)
            break Статус = SCORING_FAILED
                Cash Bus ->> Cash Conveyor : Ошибка при скоринге
                Cash Conveyor ->> Client : Сообщение об ошибке,<br/>предложение повторить позже
            end
        end
    end
end
```


## Workflow авторизации договора займа

```mermaid
sequenceDiagram

    actor Client as Клиент

    box rgb(166,83,266) Marketplace
        participant Cash Bus
        participant Cash Conveyor
    end

    box MFO
        participant MFO
    end

    MFO ->> MFO : Клиент подписывает договор займа
    MFO ->>+ Cash Bus : Событие старта авторизации<br/>onAuthorizeLoanAgreementStarted<br/>(AuthorizeLoanAgreementStarted, applicationId)
    Cash Bus ->> MFO : Получение деталей договора займа<br/>GET /api/v1/loan-agreement/{id}/details
    MFO ->>+ Cash Conveyor : МФО возвращает клиента в конвейер кредитов

    rect rgb(210,46,46)
        loop До получения конечного статуса авторизации
            Note right of Cash Bus: Начинаем отслеживание авторизации договора займа

            alt Событийный канал (приоритетный)
                MFO ->> Cash Bus : Событие авторизации договора<br/>onAuthorizationLoanAgreement<br/>(status, message)
            else Polling (резервный канал)
                Cash Bus ->> MFO: Получение статуса авторизации договора<br/>GET /api/v1/application/{id}
            end

            alt Статус = AUTHORIZE_PENDING
                Note right of Cash Bus: Продолжаем отслеживание авторизации договора займа
            end

            rect rgb(0,156,65)
                break Статус = AUTHORIZED
                    Cash Bus->>MFO: Получение подписанного договора займа<br/>GET /api/v1/loan-agreement/{id}/signed
                    Cash Bus->>MFO: Получение черновика договора займа<br/>GET /api/v1/loan-agreement/{id}/draft

                    Cash Bus->>Cash Conveyor: Сообщение об успешной авторизации займа
                    Cash Conveyor->>+Cash Conveyor: Внутренняя логика обработки
                    Cash Conveyor-->>Client: Редирект на success-экран<br/>(договор авторизован)
                end
            end

            rect rgb(60,115,168)
                break Статус = AUTHORIZED_FAILED
                    Cash Bus->>Cash Conveyor: Сообщение об ошибке авторизации займа<br/>(договор не авторизирован)
                    Cash Conveyor->>+Cash Conveyor: Внутренняя логика обработки
                end
            end

            rect rgb(60,115,168)
                break Статус = AUTHORIZED_REJECTED
                    Cash Bus->>Cash Conveyor: Сообщение об отказе в авторизации займа<br/>(МФО отклонила заявку на финальном этапе)
                    Cash Conveyor->>+Cash Conveyor: Внутренняя логика обработки
                    Cash Conveyor-->>Client: Редирект на экран отказа<br/>(redirectUrl: AUTHORIZED_REJECTED)
                end
            end

            rect rgb(60,115,168)
                break Статус = AUTHORIZED_CANCELLED
                    Client--)MFO: Клиент отказался от получения займа<br/>на экране MFO
                    Cash Bus->>Cash Conveyor: Сообщение об отмене авторизации займа<br/>(клиент отказался от получения займа)
                    Cash Conveyor->>+Cash Conveyor: Внутренняя логика обработки
                    Cash Conveyor-->>Client: Редирект на экран отмены<br/>(redirectUrl: AUTHORIZED_CANCELLED)
                end
            end

        end
    end
```