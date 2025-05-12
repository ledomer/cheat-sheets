**[<<Оглавление](../../../TableOfContents.md)**

### Пример продюсера Kafka на Go с использованием Sarama

Вот полный пример продюсера Kafka с использованием библиотеки Sarama, включая обработку ошибок и корректное завершение:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/IBM/sarama"
)

type Producer struct {
	asyncProducer sarama.AsyncProducer
	syncProducer  sarama.SyncProducer
	topic         string
}

func NewProducer(brokers []string, topic string) (*Producer, error) {
	config := sarama.NewConfig()
	
	// Настройки для надежной доставки
	config.Producer.RequiredAcks = sarama.WaitForAll
	config.Producer.Retry.Max = 5
	config.Producer.Return.Successes = true
	config.Producer.Return.Errors = true
	config.Producer.Compression = sarama.CompressionSnappy
	config.Producer.Idempotent = true
	config.Net.MaxOpenRequests = 1

	// Создаем sync producer (для синхронной отправки)
	syncProducer, err := sarama.NewSyncProducer(brokers, config)
	if err != nil {
		return nil, fmt.Errorf("failed to create sync producer: %w", err)
	}

	// Создаем async producer (для асинхронной отправки)
	asyncProducer, err := sarama.NewAsyncProducer(brokers, config)
	if err != nil {
		syncProducer.Close()
		return nil, fmt.Errorf("failed to create async producer: %w", err)
	}

	return &Producer{
		asyncProducer: asyncProducer,
		syncProducer:  syncProducer,
		topic:        topic,
	}, nil
}

// SendMessageSync отправляет сообщение синхронно
func (p *Producer) SendMessageSync(key, value string) (partition int32, offset int64, err error) {
	msg := &sarama.ProducerMessage{
		Topic: p.topic,
		Key:   sarama.StringEncoder(key),
		Value: sarama.StringEncoder(value),
	}

	return p.syncProducer.SendMessage(msg)
}

// SendMessageAsync отправляет сообщение асинхронно
func (p *Producer) SendMessageAsync(key, value string) {
	msg := &sarama.ProducerMessage{
		Topic: p.topic,
		Key:   sarama.StringEncoder(key),
		Value: sarama.StringEncoder(value),
	}

	p.asyncProducer.Input() <- msg
}

func (p *Producer) Run(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case err := <-p.asyncProducer.Errors():
			log.Printf("Failed to produce message: %v", err)
		case success := <-p.asyncProducer.Successes():
			log.Printf("Produced message to topic=%s partition=%d offset=%d",
				success.Topic, success.Partition, success.Offset)
		}
	}
}

func (p *Producer) Close() error {
	var errs []error

	if err := p.asyncProducer.Close(); err != nil {
		errs = append(errs, fmt.Errorf("async producer close error: %w", err))
	}

	if err := p.syncProducer.Close(); err != nil {
		errs = append(errs, fmt.Errorf("sync producer close error: %w", err))
	}

	if len(errs) > 0 {
		return fmt.Errorf("errors while closing producer: %v", errs)
	}
	return nil
}

func main() {
	// Конфигурация
	brokers := []string{"localhost:9092"}
	topic := "test-topic"

	// Создаем продюсера
	producer, err := NewProducer(brokers, topic)
	if err != nil {
		log.Fatalf("Failed to create producer: %v", err)
	}
	defer producer.Close()

	// Настройка контекста для graceful shutdown
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Обработка сигналов для graceful shutdown
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		sig := <-sigChan
		log.Printf("Received signal %v, shutting down...", sig)
		cancel()
	}()

	// Запускаем обработчик асинхронных сообщений
	go producer.Run(ctx)

	// Пример отправки сообщений
	go func() {
		for i := 0; ; i++ {
			select {
			case <-ctx.Done():
				return
			default:
				// Синхронная отправка (блокирующая)
				_, _, err := producer.SendMessageSync(fmt.Sprintf("key-%d", i), 
					fmt.Sprintf("sync-message-%d", i))
				if err != nil {
					log.Printf("Sync send error: %v", err)
				}

				// Асинхронная отправка (неблокирующая)
				producer.SendMessageAsync(fmt.Sprintf("key-%d", i), 
					fmt.Sprintf("async-message-%d", i))

				time.Sleep(1 * time.Second)
			}
		}
	}()

	<-ctx.Done()
	log.Println("Producer stopped")
}
```

## Ключевые особенности:

1. **Два типа продюсеров**:
   - `SyncProducer` - для синхронной отправки с подтверждением
   - `AsyncProducer` - для асинхронной отправки с обработкой в фоне

2. **Надежная доставка**:
   - `RequiredAcks = WaitForAll` - ждем подтверждения от всех реплик
   - `Retry.Max = 5` - максимальное количество попыток
   - `Idempotent = true` - идемпотентный продюсер

3. **Graceful shutdown**:
   - Обработка сигналов SIGINT и SIGTERM
   - Корректное закрытие соединений

4. **Обработка ошибок**:
   - Отдельный канал для ошибок асинхронного продюсера
   - Логирование успешных отправок

5. **Оптимизации**:
   - Сжатие сообщений (Snappy)
   - Ограничение количества одновременных запросов

## Как использовать:

1. Замените `brokers` и `topic` на свои значения
2. Добавьте свою бизнес-логику генерации сообщений
3. Выберите подходящий тип отправки (sync/async) в зависимости от требований

Для production использования рекомендуется добавить:
- Мониторинг и метрики
- Логирование с структурой (например, через zap или logrus)
- Хэндлинг повторяемых/неповторяемых ошибок
- Механизм backoff при ретраях

**[<<Оглавление](../../../TableOfContents.md)**