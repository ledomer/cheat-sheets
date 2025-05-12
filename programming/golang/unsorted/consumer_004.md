```go
package main

import (
	"fmt"
	"log"
	"os"
	"os/signal"
	"sync"

	"github.com/IBM/sarama"
)

func Consumer() {
	// Конфигурация консьюмера
	config := sarama.NewConfig()
	config.Version = sarama.V2_8_0_0 // Указываем версию Kafka
	config.Consumer.Return.Errors = true
	config.Consumer.Offsets.AutoCommit.Enable = false // Отключаем авто-коммит

	// Адреса брокеров Kafka
	brokers := []string{"localhost:9092"}

	// Создаем клиент Kafka
	client, err := sarama.NewClient(brokers, config)
	if err != nil {
		log.Fatalf("Ошибка создания клиента Kafka: %v", err)
	}
	defer client.Close()

	// Создаем менеджер смещений
	offsetManager, err := sarama.NewOffsetManagerFromClient("my-consumer-group", client)
	if err != nil {
		log.Fatalf("Ошибка создания менеджера смещений: %v", err)
	}
	defer offsetManager.Close()

	// Топик и партиция для чтения
	topic := "test-topic"
	partition := int32(0)

	// Создаем PartitionOffsetManager для нашей партиции
	pom, err := offsetManager.ManagePartition(topic, partition)
	if err != nil {
		log.Fatalf("Ошибка создания менеджера смещений для партиции: %v", err)
	}
	defer pom.Close()

	// Получаем текущее смещение
	nextOffset, _ := pom.NextOffset()
	if nextOffset == -1 {
		// Если смещение не задано, начинаем с начала
		nextOffset = sarama.OffsetOldest
	}

	// Создаем партишен-консьюмер
	consumer, err := sarama.NewConsumerFromClient(client)
	if err != nil {
		log.Fatalf("Ошибка создания консьюмера: %v", err)
	}
	defer consumer.Close()

	pc, err := consumer.ConsumePartition(topic, partition, nextOffset)
	if err != nil {
		log.Fatalf("Ошибка создания партишен-консьюмера: %v", err)
	}
	defer pc.Close()

	// Каналы для обработки сигналов и ошибок
	signals := make(chan os.Signal, 1)
	signal.Notify(signals, os.Interrupt)

	var wg sync.WaitGroup
	wg.Add(1)

	// Горутина для чтения сообщений
	go func() {
		defer wg.Done()
		for {
			select {
			case msg := <-pc.Messages():
				// Обработка сообщения
				if err := processMessage(msg); err != nil {
					log.Printf("Ошибка обработки сообщения (offset %d): %v", msg.Offset, err)
					/*pc.Pause()
					time.Sleep(3 * time.Second)
					pc.Resume()*/
					continue
				}

				// В случае успешной обработки - коммитим смещение
				// Коммитим следующее смещение (текущее + 1)
				pom.MarkOffset(msg.Offset+1, "metadata")

				fmt.Printf("Обработано сообщение: offset %d, key: %s, value: %s\n",
					msg.Offset, string(msg.Key), string(msg.Value))

			case err := <-pc.Errors():
				log.Printf("Ошибка чтения: %v", err)

			case <-signals:
				return
			}
		}
	}()

	fmt.Println("Консьюмер запущен. Нажмите Ctrl+C для остановки.")
	wg.Wait()
	fmt.Println("Консьюмер остановлен.")
}

// processMessage - пример функции обработки сообщения
func processMessage(msg *sarama.ConsumerMessage) error {
	// Здесь ваша логика обработки сообщения
	// Возвращаем ошибку, если обработка не удалась
	return nil
}
```