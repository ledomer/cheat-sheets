**[<<Оглавление](../../TableOfContents.md)**

Открытие проекта в Docker-контейнере для разработки и отладки в **VS Code** – удобный способ обеспечить изолированное и воспроизводимое окружение. Вот пошаговая инструкция для **Go**.

---

### **1. Установка необходимых инструментов**
- Убедитесь, что у вас установлены:
  - [Docker](https://www.docker.com/get-started/)
  - [VS Code](https://code.visualstudio.com/)
  - Расширение **Remote - Containers** (официальное от Microsoft).

---

### **2. Подготовка проекта**
Структура проекта (пример для Go):
```
my_project/
├── .devcontainer/          # Конфигурация для Docker
│   ├── devcontainer.json   # Настройки VS Code
│   └── Dockerfile          # Образ контейнера
├── src/                    # Исходный код
│   └── main.go (или main.py)
└── docker-compose.yml      # Опционально (для сложных контейнеров)
```

---

### **3. Настройка Docker-контейнера**
#### **Для Go**
1. Создайте `.devcontainer/Dockerfile`:
   ```dockerfile
   FROM golang:1.21  # или другой версии
   WORKDIR /workspace
   COPY . .
   RUN go mod download
   ```

2. Создайте `.devcontainer/devcontainer.json`:
   ```json
   {
     "name": "Go Dev Container",
     "dockerFile": "Dockerfile",
     "settings": {
       "go.gopath": "/workspace",
       "go.useLanguageServer": true
     },
     "extensions": ["golang.Go"]
   }
   ```

#### **Для Python**
1. Создайте `.devcontainer/Dockerfile`:
   ```dockerfile
   FROM python:3.11  # или другой версии
   WORKDIR /workspace
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   ```

2. Создайте `.devcontainer/devcontainer.json`:
   ```json
   {
     "name": "Python Dev Container",
     "dockerFile": "Dockerfile",
     "settings": {
       "python.pythonPath": "/usr/local/bin/python",
       "python.linting.enabled": true
     },
     "extensions": ["ms-python.python"]
   }
   ```

---

### **4. Запуск проекта в контейнере**
1. Откройте проект в VS Code.
2. Нажмите `F1` → **Remote-Containers: Reopen in Container**.
3. VS Code соберёт образ (если нужно) и запустит контейнер.
4. Теперь вы можете редактировать код, а окружение будет внутри контейнера.

---

### **5. Отладка**
- **Go**: Установите расширение **Go**, затем используйте стандартные `launch.json` конфигурации.
- **Python**: Выберите интерпретатор внутри контейнера (`/usr/local/bin/python`) и настройте `launch.json`.

Пример `launch.json` для Python:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Current File",
      "type": "python",
      "request": "launch",
      "program": "${file}",
      "console": "integratedTerminal"
    }
  ]
}
```

---

### **6. Дополнительные возможности**
- **Docker Compose**: Если нужны несколько контейнеров (например, БД), используйте `docker-compose.yml` и укажите его в `devcontainer.json`:
  ```json
  {
    "dockerComposeFile": "docker-compose.yml",
    "service": "app",
    "workspaceFolder": "/workspace"
  }
  ```
- **Горячая перезагрузка**: Для Go/Python можно настроить авто-перезапуск при изменениях (например, с `air` для Go или `uvicorn --reload` для Python).

---

### **Итог**
Теперь у вас есть:
- Изолированное окружение в Docker.
- Полная интеграция с VS Code (редактирование, отладка, терминал).
- Возможность делиться конфигурацией через `.devcontainer`.

Для более сложных сценариев см. [официальную документацию](https://code.visualstudio.com/docs/remote/containers).

**[<<Оглавление](../TableOfContents.md)**