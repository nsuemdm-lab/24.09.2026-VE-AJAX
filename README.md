Главная проблема при переходе к AJAX заключается в **смене парадигмы**. Они привыкли, что PHP генерирует HTML, а браузер перезагружает страницу. В AJAX бэкенд (PHP) и фронтенд (JS) общаются «в фоновом режиме» на языке JSON, а страница при этом не моргает. Вторая проблема — они не понимают, как связать клик по HTML-элементу с запуском JS-функции и последующим изменением верстки.

Ниже представлено **исчерпывающее пошаговое руководство**, которое закрывает все «слепые зоны» и дает готовые шаблоны для работы.

---

# 📘 ПОДРОБНОЕ РУКОВОДСТВО: Интерактивный UI и AJAX (Трек 2)

**Как это работает (Аналогия):**
Обычный сайт — это столовая: вы берете поднос, идете на раздачу (сервер), получаете еду (HTML) и садитесь за стол (браузер). За добавкой нужно вставать и идти заново (перезагрузка страницы).
**AJAX** — это ресторан: вы сидите за столом (страница не перезагружается), подзываете официанта (JavaScript Fetch), он идет на кухню (PHP Контроллер), забирает блюдо (JSON) и ставит вам на стол (JS обновляет HTML).

---

## ШАГ 1. Настройка Роутера (Связующее звено)
Чтобы JavaScript мог обратиться к PHP, у PHP должен быть адрес (URL).
Откройте ваш главный файл `public_html/index.php` и добавьте специальный маршрут для API:

```php
// public_html/index.php

// ... ваши обычные маршруты ...
$router->add('/kanban', 'KanbanController', 'index'); // Отдает HTML страницу

// 👇 МАРШРУТЫ ДЛЯ AJAX (API) 👇
// Обратите внимание: мы направляем запрос в отдельный ApiController
$router->add('/api/tasks/update', 'ApiController', 'updateTaskStatus');
```

---

## ШАГ 2. Бэкенд (PHP Контроллер, который отдает JSON)
Создайте файл `app/Controllers/ApiController.php`. 
**Важное правило API:** В этом файле **НЕ ДОЛЖНО БЫТЬ** `require 'view.php'`, `echo "Привет"` или HTML-тегов. Только чистые данные в формате JSON.

```php
<?php
// app/Controllers/ApiController.php

class ApiController {
    
    public function updateTaskStatus() {
        // 1. ПОЛУЧЕНИЕ ДАННЫХ ОТ JAVASCRIPT
        // Так как JS отправляет данные в формате JSON, стандартный $_POST работать НЕ БУДЕТ!
        // Мы читаем "сырой" поток данных:
        $json = file_get_contents('php://input');
        $data = json_decode($json, true); // Превращаем JSON в ассоциативный массив PHP

        // Извлекаем переменные
        $taskId = $data['task_id'] ?? null;
        $newStatus = $data['status'] ?? null;

        // 2. ПРОВЕРКА И РАБОТА С БД
        if ($taskId && $newStatus) {
            global $pdo;
            // Обновляем статус в базе
            $stmt = $pdo->prepare("UPDATE tasks SET status = ? WHERE id = ?");
            $stmt->execute([$newStatus, $taskId]);
            
            // 3. ОТВЕТ СЕРВЕРА (УСПЕХ)
            // Обязательно говорим браузеру, что возвращаем JSON, а не HTML
            header('Content-Type: application/json; charset=utf-8');
            echo json_encode([
                'success' => true, 
                'message' => 'Статус успешно обновлен'
            ]);
            exit; // Останавливаем скрипт
            
        } else {
            // 4. ОТВЕТ СЕРВЕРА (ОШИБКА)
            http_response_code(400); // Код 400 - Bad Request (Неверный запрос)
            header('Content-Type: application/json; charset=utf-8');
            echo json_encode([
                'success' => false, 
                'error' => 'Не переданы ID задачи или статус'
            ]);
            exit;
        }
    }
}
```

---

## ШАГ 3. Фронтенд (HTML + JavaScript)
Это то место, где у студентов больше всего проблем. Как связать кнопку с JS и как изменить HTML после ответа сервера?

Создайте файл представления `views/kanban/index.php`.

### Часть 3.1: HTML-разметка (Использование `data-` атрибутов)
Чтобы JS знал, какую задачу мы двигаем, мы прячем ID задачи прямо в HTML-тег с помощью атрибутов `data-id`.

```html
<!-- views/kanban/index.php -->
<div class="container d-flex">
    
    <!-- КОЛОНКА 1: Новые задачи -->
    <div class="column" id="col-new" style="width: 50%; border: 1px solid #ccc; padding: 10px;">
        <h3>Новые</h3>
        
        <!-- Карточка задачи -->
        <!-- data-id хранит ID из базы данных -->
        <div class="card mb-2 task-card" id="task-5" data-id="5">
            <div class="card-body">
                Починить принтер
                <!-- Кнопка, вызывающая JS функцию -->
                <button onclick="moveTask(5, 'done')" class="btn btn-sm btn-success float-end">
                    В готово ->
                </button>
            </div>
        </div>
        
    </div>

    <!-- КОЛОНКА 2: Готовые задачи -->
    <div class="column" id="col-done" style="width: 50%; border: 1px solid #ccc; padding: 10px;">
        <h3>Готово</h3>
        <!-- Сюда JS будет переносить карточки -->
    </div>

</div>
```

### Часть 3.2: JavaScript (Магия без перезагрузки)
Добавьте этот скрипт в самый низ файла `views/kanban/index.php` (перед закрывающим `</body>`).

```javascript
<script>
// Функция принимает ID задачи и новый статус
function moveTask(taskId, newStatus) {
    
    // 1. Отправляем фоновый запрос на наш PHP-роутер
    fetch('/api/tasks/update', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json' // Говорим PHP, что шлем JSON
        },
        body: JSON.stringify({
            task_id: taskId,
            status: newStatus
        })
    })
    // 2. Ждем ответ от PHP и превращаем его обратно из JSON в JS-объект
    .then(response => response.json())
    // 3. Работаем с результатом
    .then(data => {
        if (data.success === true) {
            // УРА! База данных обновилась. Теперь обновляем HTML (DOM)
            
            // Находим саму карточку по её ID
            const taskCard = document.getElementById('task-' + taskId);
            
            // Находим колонку, куда нужно перенести карточку
            const targetColumn = document.getElementById('col-' + newStatus);
            
            // ПЕРЕНОСИМ: appendChild забирает элемент со старого места и ставит в новое
            targetColumn.appendChild(taskCard);
            
            // Опционально: прячем кнопку, так как задача уже выполнена
            taskCard.querySelector('button').style.display = 'none';
            
        } else {
            // Если PHP вернул success => false
            alert("Ошибка сервера: " + data.error);
        }
    })
    .catch(error => {
        // Если произошла фатальная ошибка (например, PHP упал с 500 ошибкой)
        console.error("Сетевая ошибка:", error);
        alert("Произошла ошибка при связи с сервером. Откройте консоль (F12).");
    });
}
</script>
```

---

## 🛠 ШАГ 4. Как отлаживать AJAX (Инструкция по спасению)
Если вы нажали на кнопку, и **ничего не произошло**, не паникуйте. AJAX работает скрыто, поэтому ошибки нужно искать в инструментах разработчика.

1. Нажмите **F12** в браузере (Инструменты разработчика).
2. Перейдите во вкладку **Network (Сеть)**.
3. Нажмите на вашу кнопку на сайте.
4. В списке появится запрос `update` (или ваш URL). Кликните по нему.
5. Справа откройте вкладку **Response (Ответ)**.
    * *Правильный ответ:* `{"success":true,"message":"..."}`
    * *Частая ошибка:* Вы увидите там кусок HTML-кода с ошибкой PHP (например, `Fatal error: Uncaught PDOException...`). Это значит, что ваш PHP-код сломался, и JS не смог прочитать JSON. Идите в `ApiController.php` и исправляйте PHP-код.

---
---

# 🔥 ЗАДАНИЕ ДЛЯ УРОВНЯ "А": Интеграция Chart.js (Дашборды)

Для тех, кто делает аналитику, графики или учет финансов. Задача: нарисовать график продаж по месяцам, запросив данные из БД через AJAX.

### 1. PHP Контроллер (Отдает данные для графика)
```php
// app/Controllers/ApiController.php
public function getChartData() {
    global $pdo;
    
    // Группируем продажи по месяцам (Пример для MySQL)
    $sql = "SELECT MONTHNAME(created_at) as month, SUM(total_price) as total 
            FROM orders 
            GROUP BY MONTH(created_at)";
    $stmt = $pdo->query($sql);
    $results = $stmt->fetchAll(PDO::FETCH_ASSOC);

    // Chart.js требует два отдельных массива: подписи (ось X) и данные (ось Y)
    $labels = [];
    $data = [];
    
    foreach ($results as $row) {
        $labels[] = $row['month'];
        $data[] = $row['total'];
    }

    header('Content-Type: application/json');
    echo json_encode([
        'labels' => $labels,
        'data' => $data
    ]);
}
```
*Не забудьте добавить роут:* `$router->add('/api/chart', 'ApiController', 'getChartData');`

### 2. HTML + JS (Отрисовка графика)
В вашем View подключите библиотеку Chart.js и создайте `canvas`.

```html
<!-- Подключаем Chart.js через CDN -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<div style="width: 600px; margin: auto;">
    <!-- Холст для графика -->
    <canvas id="myChart"></canvas>
</div>

<script>
// Как только страница загрузилась, запрашиваем данные
document.addEventListener("DOMContentLoaded", function() {
    
    fetch('/api/chart')
        .then(response => response.json())
        .then(chartData => {
            
            // Инициализируем график полученными данными
            const ctx = document.getElementById('myChart').getContext('2d');
            new Chart(ctx, {
                type: 'bar', // Тип графика: столбчатый (bar), линейный (line), круг (pie)
                data: {
                    labels: chartData.labels, // Подписи оси X (Месяца)
                    datasets: [{
                        label: 'Сумма продаж (руб)',
                        data: chartData.data, // Значения оси Y
                        backgroundColor: 'rgba(54, 162, 235, 0.5)',
                        borderColor: 'rgba(54, 162, 235, 1)',
                        borderWidth: 1
                    }]
                }
            });
            
        });
});
</script>
```
