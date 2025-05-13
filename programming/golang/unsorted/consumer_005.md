
### Способы, как в случае ошибки, не продолжить читать сообщение, а повторить попытку чтения сообщения на котором произошла ошибка

**[<<Оглавление](../../../TableOfContents.md)**

Нет, создавать новый `PartitionConsumer` не требуется! В Kafka/Sarama есть несколько способов повторно прочитать то же сообщение при ошибке обработки без пересоздания потребителя:

#### 1. Самый простой способ - не коммитить смещение и продолжить цикл

```go
case msg := <-pc.Messages():
    if err := processMessage(msg); err != nil {
        log.Printf("Ошибка обработки offset %d: %v", msg.Offset, err)
        // Не коммитим смещение и продолжаем цикл
        continue
    }
    pom.MarkOffset(msg.Offset+1, "")
```

**Как работает**:
- При следующей итерации цикла потребитель попытается прочитать следующее сообщение
- Но так как смещение не сдвинулось, Kafka вернет то же самое сообщение снова
- *Однако это поведение зависит от настроек потребителя и версии Kafka*

#### 2. Более надежный способ - использовать Pause/Resume

```go
case msg := <-pc.Messages():
    if err := processMessage(msg); err != nil {
        log.Printf("Ошибка обработки offset %d: %v", msg.Offset, err)
        
        // Приостанавливаем потребление
        pc.Pause()
        
        // Подождем перед повторной попыткой
        time.Sleep(5 * time.Second)
        
        // Возобновляем потребление (сообщение будет прочитано снова)
        pc.Resume()
        continue
    }
```

#### 3. Точечный перезапуск потребления для конкретной партиции

```go
case msg := <-pc.Messages():
    if err := processMessage(msg); err != nil {
        log.Printf("Ошибка обработки offset %d: %v", msg.Offset, err)
        
        // Закрываем текущий потребитель
        pc.Close()
        
        // Создаем нового потребителя для той же партиции с ТЕМ ЖЕ смещением
        newOffset := pom.NextOffset() // Получаем текущее (не сдвинутое) смещение
        pc, err = consumer.ConsumePartition(topic, partition, newOffset)
        if err != nil {
            log.Fatalf("Ошибка пересоздания потребителя: %v", err)
        }
        continue
    }
```

#### 4. Оптимальный способ - ручное управление позицией

```go
case msg := <-pc.Messages():
    if err := processMessage(msg); err != nil {
        log.Printf("Ошибка обработки offset %d: %v", msg.Offset, err)
        
        // Устанавливаем позицию чтения на текущий offset
        pc.ResetOffset(msg.Offset, "")
        continue
    }
```

**Важные нюансы**:
1. Метод `ResetOffset()` доступен в последних версиях Sarama
2. Для гарантированного повторного чтения нужно:
   - Отключить авто-коммит (`config.Consumer.Offsets.AutoCommit.Enable = false`)
   - Не вызывать `MarkOffset()` для ошибочных сообщений
3. При использовании consumer groups логика немного сложнее

#### Полный пример с обработкой ошибок:

```go
for {
    select {
    case msg := <-pc.Messages():
        retries := 0
        maxRetries := 3
        
        for retries < maxRetries {
            if err := processMessage(msg); err != nil {
                retries++
                log.Printf("Попытка %d для offset %d: %v", retries, msg.Offset, err)
                
                if retries < maxRetries {
                    // Для Sarama >= 1.27
                    if resetter, ok := pc.(sarama.PartitionOffsetResetter); ok {
                        resetter.ResetOffset(msg.Offset, "")
                    }
                    
                    time.Sleep(time.Second * time.Duration(retries))
                    continue
                }
                
                // После всех попыток - сохраняем в DLQ и двигаемся дальше
                sendToDLQ(msg)
                pom.MarkOffset(msg.Offset+1, "dlq")
                break
            }
            
            // Успешная обработка
            pom.MarkOffset(msg.Offset+1, "")
            break
        }

    case err := <-pc.Errors():
        log.Printf("Ошибка потребителя: %v", err)
    case <-signals:
        return
    }
}
```

Выбор метода зависит от ваших требований:
- Для простых случаев достаточно `continue` без коммита
- Для надежности лучше использовать `ResetOffset()`
- Для сложных сценариев - комбинация попыток + DLQ

**[<<Оглавление](../../../TableOfContents.md)**