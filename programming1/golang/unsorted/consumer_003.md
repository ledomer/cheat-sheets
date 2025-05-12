**[<<Оглавление](../../../TableOfContents.md)**

### Пример консьюмера Kafka на Go с использованием Sarama

Вот полный пример консьюмера Kafka с использованием библиотеки Sarama, включая обработку ошибок и корректное завершение:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"

	"github.com/IBM/sarama"
)

type Consumer struct {
	client   sarama.Consumer
	handlers map[string]func(*sarama.ConsumerMessage) error
	mu       sync.Mutex
}

func NewConsumer(brokers []string) (*Consumer, error) {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	config.Consumer.Offsets.AutoCommit.Enable = false // Отключаем авто-коммит

	client, err := sarama.NewConsumer(brokers, config)
	if err != nil {
		return nil, fmt.Errorf("failed to create consumer: %w", err)
	}

	return &Consumer{
		client:   client,
		handlers: make(map[string]func(*sarama.ConsumerMessage) error),
	}, nil
}

func (c *Consumer) RegisterHandler(topic string, handler func(*sarama.ConsumerMessage) error) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.handlers[topic] = handler
}

func (c *Consumer) Start(ctx context.Context, topics []string) error {
	// Создаем partition consumers для каждой темы
	var partitions []sarama.PartitionConsumer
	defer func() {
		for _, pc := range partitions {
			pc.Close()
		}
	}()

	for _, topic := range topics {
		partitionsList, err := c.client.Partitions(topic)
		if err != nil {
			return fmt.Errorf("failed to get partitions for topic %s: %w", topic, err)
		}

		for _, partition := range partitionsList {
			// Начинаем чтение с последнего коммитнутого оффсета
			pc, err := c.client.ConsumePartition(topic, partition, sarama.OffsetNewest)
			if err != nil {
				return fmt.Errorf("failed to start consumer for partition %d: %w", partition, err)
			}
			partitions = append(partitions, pc)

			go func(pc sarama.PartitionConsumer, topic string) {
				for {
					select {
					case msg := <-pc.Messages():
						c.mu.Lock()
						handler, ok := c.handlers[topic]
						c.mu.Unlock()

						if ok {
							if err := handler(msg); err != nil {
								log.Printf("Error processing message (topic %s, partition %d, offset %d): %v",
									topic, msg.Partition, msg.Offset, err)
							} else {
								log.Printf("Processed message (topic %s, partition %d, offset %d)",
									topic, msg.Partition, msg.Offset)
							}
						}
					case err := <-pc.Errors():
						log.Printf("Consumer error: %v", err)
					case <-ctx.Done():
						return
					}
				}
			}(pc, topic)
		}
	}

	<-ctx.Done()
	return nil
}

func (c *Consumer) Close() error {
	if err := c.client.Close(); err != nil {
		return fmt.Errorf("error closing consumer: %w", err)
	}
	return nil
}

func main() {
	// Конфигурация
	brokers := []string{"localhost:9092"}
	topics := []string{"test-topic"}
	
	// Создаем consumer
	consumer, err := NewConsumer(brokers)
	if err != nil {
		log.Fatalf("Failed to create consumer: %v", err)
	}
	defer consumer.Close()

	// Регистрируем обработчики для тем
	consumer.RegisterHandler("test-topic", func(msg *sarama.ConsumerMessage) error {
		fmt.Printf("Received message: %s\n", string(msg.Value))
		// Здесь можно добавить бизнес-логику обработки сообщения
		return nil
	})

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

	// Запускаем consumer
	log.Println("Starting consumer...")
	if err := consumer.Start(ctx, topics); err != nil {
		log.Fatalf("Consumer error: %v", err)
	}

	log.Println("Consumer stopped")
}
```

## Ключевые особенности:

1. **Отключение авто-коммита**: `config.Consumer.Offsets.AutoCommit.Enable = false` - оффсеты не комитятся автоматически.

2. **Graceful shutdown**: Обработка сигналов SIGINT и SIGTERM для корректного завершения.

3. **Обработка ошибок**: Отдельный канал для ошибок потребителя.

4. **Гибкая система обработчиков**: Можно регистрировать разные обработчики для разных топиков.

5. **Потокобезопасность**: Использование sync.Mutex для защиты handlers.

6. **Контекст**: Использование context.Context для управления жизненным циклом.

## Как использовать:

1. Замените `brokers` и `topics` на свои значения
2. Добавьте свою бизнес-логику в обработчик сообщений
3. При необходимости реализуйте сохранение оффсетов (в примере они не сохраняются)

Для production использования рекомендуется добавить:
- Сохранение оффсетов в постоянное хранилище
- Мониторинг и логирование
- Хэндлинг повторяемых/неповторяемых ошибок
- Механизм retry для обработки сообщений

**[<<Оглавление](../../../TableOfContents.md)**