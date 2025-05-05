
**[<<Оглавление](../../TableOfContents.md)**

LLM Studio, Ollama позволяет работать с локальными языковыми моделями через API.

GUI для Ollama:
[ollama-gui](https://chyok.github.io/ollama-gui/)
```bash
pip install ollama-gui
ollama-gui
```

### **1. Установка и запуск локальной модели**  
Перед использованием API убедитесь, что у вас:  
- Локальная LLM (например, **Llama 3, Mistral, Phi-3, Gemma**) загружена и запущена через **Ollama, LM Studio, Text Generation Inference (TGI), vLLM** или аналогичный инструмент.  
- Сервер API доступен (обычно `http://localhost:PORT`).  

#### **Пример с LM Studio**  
1. Скачайте [LM Studio](https://lmstudio.ai/) и загрузите нужную GGUF-модель.  
2. Запустите встроенный сервер (в настройках включите **"Local Server"**).  
   - По умолчанию API доступен на `http://localhost:1234/v1/completions`.  

#### **Пример с Ollama**  
1. Установите [Ollama](https://ollama.ai/) и скачайте модель:  
   ```bash
   ollama pull llama3
   ollama run llama3
   ```  
2. API будет доступен на `http://localhost:11434/api/generate`.  

---  

### **2. Отправка запроса через API (без токена)**  
Если сервер локальный, аутентификация обычно не требуется. Примеры запросов:  

#### **Через `curl`**  
```bash
curl -X POST "http://localhost:1234/v1/completions" \
-H "Content-Type: application/json" \
-d '{
  "model": "local-model",  # Имя модели (если требуется)
  "prompt": "Как работает API?",
  "max_tokens": 100
}'
```

#### **Через Python (`requests`)**  
```python
import requests

url = "http://localhost:1234/v1/completions"
payload = {
    "model": "local-model",
    "prompt": "Привет! Как дела?",
    "max_tokens": 50
}

response = requests.post(url, json=payload)
print(response.json())
```

---  

### **3. Параметры API (в зависимости от сервера)**  
- **LM Studio** использует OpenAI-совместимый API (`/v1/completions` или `/v1/chat/completions`).  
- **Ollama** имеет свой формат (`/api/generate`).  
- **Text Generation Inference (TGI)** от Hugging Face поддерживает `/generate` и `/generate_stream`.  

---  

### **4. Пример для Ollama API**  
```python
import requests

url = "http://localhost:11434/api/generate"
data = {
    "model": "llama3",
    "prompt": "Расскажи о квантовой физике",
    "stream": False
}

response = requests.post(url, json=data)
print(response.json()["response"])
```

---  

### **5. Если сервер требует токен**  
Некоторые локальные серверы (например, **FastChat**) могут использовать фиктивный токен. В таком случае добавьте заголовок:  
```python
headers = {"Authorization": "Bearer no-token"}  # Фиктивный токен
response = requests.post(url, json=payload, headers=headers)
```

---  

### **Вывод**  
1. Убедитесь, что локальная модель запущена.  
2. Проверьте URL API (зависит от инструмента).  
3. Отправляйте запросы без токена (или с фиктивным, если требуется).  

Если вы используете конкретный инструмент (например, **Ollama, LM Studio, TGI**), уточните его API-документацию.

**[<<Оглавление](../../TableOfContents.md)**