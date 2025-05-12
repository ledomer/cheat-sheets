# Пример консьюмера на Go с использованием Sarama и ручным управлением оффсетами

Вот пример консьюмера Kafka на Go с использованием библиотеки [Sarama](https://github.com/IBM/sarama), который вручную управляет оффсетами:

**[<<Оглавление](../../../TableOfContents.md)**

```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync"
	"time"

	"github.com/IBM/sarama"
)

// Consumer представляет собой простого Kafka-консьюмера с ручным управлением оффсетами
type Consumer struct {
	ready    chan bool
	consumer sarama.Consumer
	offset   int64
	topic    string
	partition int32
	mu       sync.Mutex
}

func NewConsumer(brokers []string, topic string, partition int32, offset int64) (*Consumer, error) {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	
	// Создаем consumer
	consumer, err := sarama.NewConsumer(brokers, config)
	if err != nil {
		return nil, fmt.Errorf("failed to create consumer: %w", err)
	}

	return &Consumer{
		ready:    make(chan bool),
		consumer: consumer,
		offset:   offset,
		topic:    topic,
		partition: partition,
	}, nil
}

func (c *Consumer) Start() error {
	// Создаем partition consumer с указанного оффсета
	pc, err := c.consumer.ConsumePartition(c.topic, c.partition, c.offset)
	if err != nil {
		return fmt.Errorf("failed to start partition consumer: %w", err)
	}

	defer func() {
		if err := pc.Close(); err != nil {
			log.Printf("Error closing partition consumer: %v", err)
		}
	}()

	// Сигнализируем, что consumer готов
	close(c.ready)

	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)

	// Счетчик для демонстрации коммита оффсетов
	messageCount := 0
	batchSize := 5 // Коммитим оффсет каждые 5 сообщений

ConsumerLoop:
	for {
		select {
		case msg := <-pc.Messages():
			c.mu.Lock()
			c.offset = msg.Offset + 1 // Сохраняем следующий ожидаемый оффсет
			c.mu.Unlock()

			log.Printf("Received message: Topic=%s, Partition=%d, Offset=%d, Key=%s, Value=%s\n",
				msg.Topic, msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))

			messageCount++
			if messageCount%batchSize == 0 {
				log.Printf("Processed %d messages, current offset: %d", messageCount, c.offset)
				// В реальном приложении здесь можно сохранять оффсет во внешнее хранилище
			}

		case err := <-pc.Errors():
			log.Printf("Error: %v\n", err)

		case <-signals:
			log.Println("Received interrupt signal, shutting down...")
			break ConsumerLoop
		}
	}

	log.Printf("Last processed offset: %d", c.offset)
	return nil
}

func (c *Consumer) Close() error {
	if err := c.consumer.Close(); err != nil {
		return fmt.Errorf("error closing consumer: %w", err)
	}
	return nil
}

func main() {
	// Конфигурация
	brokers := []string{"localhost:9092"} // Адреса брокеров Kafka
	topic := "test-topic"                 // Топик для потребления
	partition := int32(0)                 // Партиция
	initialOffset := sarama.OffsetOldest   // Начинаем с самого старого сообщения (можно указать конкретный оффсет)

	// Создаем consumer
	consumer, err := NewConsumer(brokers, topic, partition, initialOffset)
	if err != nil {
		log.Fatalf("Failed to create consumer: %v", err)
	}
	defer consumer.Close()

	// Запускаем consumer в горутине
	go func() {
		if err := consumer.Start(); err != nil {
			log.Fatalf("Error in consumer: %v", err)
		}
	}()

	// Ждем, пока consumer будет готов
	<-consumer.ready
	log.Println("Consumer is ready. Press Ctrl+C to exit.")

	// Ожидаем сигнал завершения
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, os.Interrupt)
	<-sigchan

	log.Println("Shutting down consumer...")
}
```

## Ключевые моменты:

1. **Ручное управление оффсетами**: В этом примере оффсеты не комитятся автоматически в Kafka, а сохраняются в памяти. В реальном приложении вы должны сохранять их в постоянное хранилище (например, базу данных).

2. **Начальный оффсет**: Вы можете начать чтение с:
   - `sarama.OffsetOldest` (самое старое сообщение)
   - `sarama.OffsetNewest` (самое новое сообщение)
   - Конкретного числового оффсета

3. **Сохранение оффсета**: После обработки каждого сообщения оффсет сохраняется в переменной `c.offset`. В реальном приложении этот оффсет следует периодически сохранять в надежное хранилище.

4. **Обработка ошибок**: Пример включает обработку ошибок из канала ошибок.

5. **Грациозное завершение**: Приложение корректно обрабатывает сигнал прерывания (Ctrl+C).

## Улучшения для production:

1. **Сохранение оффсетов**: Реализуйте сохранение оффсетов в базу данных или другое надежное хранилище.

2. **Реконнект**: Добавьте логику переподключения при разрыве соединения.

3. **Конфигурирование**: Вынесите настройки (batch size, таймауты и т.д.) в конфигурационный файл.

4. **Метрики**: Добавьте сбор метрик для мониторинга.

5. **Обработка паники**: Добавьте recover в горутины для обработки паник.

**[<<Оглавление](../../../TableOfContents.md)**
