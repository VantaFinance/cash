# Документация Абстрактного api банка

Данный документ описывает желаемый процесс получение кредита наличными.
Ниже представлен workflow процессы и сам api. 



## Workflow скоринга

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
    Bank->>Cash Bus: Выгружаем ЦПГ(ESIA) /{bank}/{$request.requestBody.externalApplicationId}/cpg
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



## Workflow авторизации договора займа


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
    loop Пока не получили конечный статус, каждые 30 секунд
        Note right of Cash Bus: Начинаем наблюдать за "печатью" черновика договора займа
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
    end
end


Cash Bus ->>+ Bank : Загружаем подписанный договор займа с помощью ЭЦП /api/v1/loan-agreement/{id}/upload
Bank ->>- Cash Bus : Возвращает идентификатор подписанного договора займа



rect rgb(210, 46,46)
    loop Пока не получили конечный статус авторизации, каждые 30 секунд
        Note right of Cash Bus: Начинаем наблюдать за авторизацией договора
        Cash Bus ->> Bank:  Получаем статус авторизации договора /api/v1/loan-agreement/{id}/status

        alt Если статус авторизации "WAITING"
            Note right of Cash Bus: Продолжаем наблюдать за авторизацией
        end

        rect rgb(0, 156, 65)
            alt Если статус печати "SUCCESS"
                Cash Bus->>Bank: Забираем подписанный договора займа /api/v1/loan-agreement/{id}/signed
                Cash Bus->>Bank: Забираем детали подписанного договора /api/v1/loan-agreement/{id}/details
            end
        end

        alt Если статус печати "ERROR"
            Note right of Cash Bus: Через N время пробуем снова запустить авторизацию договора<br/> Если через N попыток получаем ERROR, завершаем клиентский путь
        end
    end
end


alt Клиент захотел перевести денежные средства на ОФП счет 
    
Cash Bus ->> Bank : Запустили процесс финансирования в другой банк /api/v1/loan-agreement/{id}/financing


rect rgb(210, 46,46)
    loop Пока не получили конечный статус финансирования, каждые 30 секунд
        Note right of Cash Bus: Начинаем наблюдать за финансированием в другой банк
        Cash Bus ->> Bank:  Получаем статус финансирования договора /api/v1/loan-agreement/{id}/financing/status

        alt Если статус авторизации "WAITING"
            Note right of Cash Bus: Продолжаем наблюдать за финансированием
        end

        rect rgb(0, 156, 65)
            alt Если статус печати "SUCCESS"
                Note right of Cash Bus: Завершаем клиентский путь, сообщаем клиенту что успешно перевели средства
            end
        end

        alt Если статус печати "ERROR"
            Note right of Cash Bus: Через N время пробуем снова запустить финансирование<br/> Если через N попыток получаем ERROR, завершаем клиентский путь
        end
    end
end

end
```

