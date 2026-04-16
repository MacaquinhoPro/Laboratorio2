# Respuestas — Preguntas Prácticas FreeRTOS

---

## 1. ¿Cómo se ejecutan tareas de FreeRTOS con la misma función pero distintos parámetros?

`xTaskCreate()` acepta un puntero genérico `void *pvParameters` que se pasa a la función de tarea al momento de crearla. Basta con crear una estructura con los parámetros distintos y pasar su dirección:

```cpp
// Estructura de parámetros
typedef struct {
  uint8_t       sensorPin;
  uint8_t       sensorId;
  QueueHandle_t queue;
} SensorTaskParams_t;

static SensorTaskParams_t p1 = { 4,  1, xQueue1 };
static SensorTaskParams_t p2 = { 15, 2, xQueue2 };

// Ambas tareas ejecutan TaskSensor() con datos distintos
xTaskCreate(TaskSensor, "Sensor1", 2048, &p1, 2, nullptr);
xTaskCreate(TaskSensor, "Sensor2", 2048, &p2, 2, nullptr);
```

Las estructuras deben ser `static` (o globales) para que no sean destruidas al salir de `setup()`.

---

## 2. ¿Cuál es el tipo de dato que recibe una tarea de FreeRTOS? ¿Cómo se convierte?

La firma obligatoria de toda tarea es:

```cpp
void MiTarea(void *pvParameters);
```

El parámetro es `void *` — un puntero genérico sin tipo. Para usar los datos concretos se hace un *cast* al tipo de puntero correcto:

```cpp
void TaskSensor(void *pvParameters) {
  // Cast explícito de void* al tipo concreto
  SensorTaskParams_t *params = (SensorTaskParams_t *) pvParameters;

  // A partir de aquí se puede usar params->sensorPin, params->queue, etc.
  uint16_t val = touchRead(params->sensorPin);
}
```

El programador es responsable de garantizar que el puntero apunta realmente al tipo esperado; FreeRTOS no hace verificación de tipos en tiempo de ejecución.

---

## 3. ¿Qué pasa cuando una cola se llena y una tarea quiere insertar nuevos elementos?

El comportamiento depende del parámetro `xTicksToWait` de `xQueueSend()`:

| xTicksToWait | Comportamiento cuando la cola está llena |
|---|---|
| `0` | Retorna inmediatamente con `errQUEUE_FULL` |
| `N ticks` | Bloquea la tarea hasta N ticks o hasta que haya espacio |
| `portMAX_DELAY` | Bloquea indefinidamente hasta que haya espacio |

En la práctica se usa un tiempo de espera corto para no perder datos por mucho tiempo pero sí detectar el problema:

```cpp
BaseType_t ok = xQueueSend(queue, &data, pdMS_TO_TICKS(10));
if (ok == errQUEUE_FULL) {
  // Cola llena — el dato se descarta en este ciclo
  overflowCount++;
}
```

---

## 4. ¿Es posible que varias tareas lean y escriban a la misma cola?

**Sí.** Las colas de FreeRTOS son thread-safe por diseño — internamente usan secciones críticas para proteger el acceso. Es perfectamente válido:

- **Múltiples productores → una cola → un consumidor** (fan-in)
- **Un productor → una cola → múltiples consumidores** (fan-out)
- **Múltiples productores → una cola → múltiples consumidores**

Cada `xQueueReceive()` extrae exactamente un elemento; si dos tareas consumidoras esperan en la misma cola, solo una recibirá cada mensaje. El orden es FIFO dentro de la cola.

---

## 5. ¿Qué es un deadlock? Ejemplo y cómo evitarlo

Un **deadlock** (interbloqueo) ocurre cuando dos o más tareas se bloquean mutuamente esperando un recurso que la otra ya tiene, sin que ninguna pueda avanzar.

### Ejemplo de código que genera deadlock

```cpp
SemaphoreHandle_t mutexA;
SemaphoreHandle_t mutexB;

// Tarea 1: toma A, luego intenta tomar B
void Task1(void *pvParams) {
  for (;;) {
    xSemaphoreTake(mutexA, portMAX_DELAY);  // ① Toma A
    vTaskDelay(pdMS_TO_TICKS(10));           // Da tiempo a Task2 para tomar B
    xSemaphoreTake(mutexB, portMAX_DELAY);  // ② Bloqueada: Task2 tiene B
    // ──► DEADLOCK: Task1 espera B, Task2 espera A

    xSemaphoreGive(mutexB);
    xSemaphoreGive(mutexA);
  }
}

// Tarea 2: toma B, luego intenta tomar A
void Task2(void *pvParams) {
  for (;;) {
    xSemaphoreTake(mutexB, portMAX_DELAY);  // ① Toma B
    vTaskDelay(pdMS_TO_TICKS(10));
    xSemaphoreTake(mutexA, portMAX_DELAY);  // ② Bloqueada: Task1 tiene A
    // ──► DEADLOCK: Task2 espera A, Task1 espera B

    xSemaphoreGive(mutexA);
    xSemaphoreGive(mutexB);
  }
}
```

### ¿Por qué sucede?

- Task1 tiene A y espera B.
- Task2 tiene B y espera A.
- Ninguna puede liberar lo que la otra necesita → ciclo de espera infinita.

### Cómo evitarlo

**Regla de orden consistente:** todas las tareas adquieren los mutex siempre en el mismo orden.

```cpp
// SOLUCIÓN: ambas tareas toman A primero, luego B
void Task1_fixed(void *pvParams) {
  for (;;) {
    xSemaphoreTake(mutexA, portMAX_DELAY);  // Orden: A → B
    xSemaphoreTake(mutexB, portMAX_DELAY);
    // trabajo...
    xSemaphoreGive(mutexB);
    xSemaphoreGive(mutexA);
  }
}

void Task2_fixed(void *pvParams) {
  for (;;) {
    xSemaphoreTake(mutexA, portMAX_DELAY);  // Mismo orden: A → B
    xSemaphoreTake(mutexB, portMAX_DELAY);
    // trabajo...
    xSemaphoreGive(mutexB);
    xSemaphoreGive(mutexA);
  }
}
```

Otras estrategias:
- Usar `xSemaphoreTake()` con timeout finito y reintentar si falla.
- Diseñar el sistema para que cada tarea necesite un solo mutex (preferido).
- Usar FreeRTOS **mutex con herencia de prioridad** (`xSemaphoreCreateMutex`) para mitigar *priority inversion*, que puede empeorar situaciones de deadlock.

---

## 6. Acceso concurrente sin y con semáforo

### Problema: datos inconsistentes sin mutex

```cpp
// Variable compartida entre las dos tareas
volatile int32_t sharedCounter = 0;

void TaskIncrement(void *pvParams) {
  for (;;) {
    // PROBLEMA: read-modify-write NO es atómica en la mayoría de CPUs
    sharedCounter++;   // Se compila en: LOAD, ADD, STORE (3 instrucciones)
    vTaskDelay(1);
  }
}

void TaskRead(void *pvParams) {
  for (;;) {
    // Puede leer un valor intermedio (e.g. después de LOAD pero antes de STORE)
    Serial.println(sharedCounter);
    vTaskDelay(1);
  }
}
// Resultado: valores perdidos, duplicados o corruptos bajo preemption
```

### Solución: acceso protegido con mutex

```cpp
SemaphoreHandle_t xCounterMutex;
volatile int32_t sharedCounter = 0;

void setup_mutex_example() {
  xCounterMutex = xSemaphoreCreateMutex();
}

void TaskIncrement_safe(void *pvParams) {
  for (;;) {
    if (xSemaphoreTake(xCounterMutex, portMAX_DELAY) == pdTRUE) {
      sharedCounter++;            // Sección crítica protegida
      xSemaphoreGive(xCounterMutex);
    }
    vTaskDelay(1);
  }
}

void TaskRead_safe(void *pvParams) {
  for (;;) {
    int32_t snapshot;
    if (xSemaphoreTake(xCounterMutex, portMAX_DELAY) == pdTRUE) {
      snapshot = sharedCounter;   // Lectura atómica dentro de la sección crítica
      xSemaphoreGive(xCounterMutex);
    }
    Serial.println(snapshot);
    vTaskDelay(1);
  }
}
// Resultado: sharedCounter siempre refleja un estado consistente
```

**Clave:** el mutex garantiza que solo una tarea ejecuta la sección crítica (lectura o escritura) en cualquier momento. La otra tarea queda bloqueada hasta que el mutex es liberado.

---

## Diagrama de deadlock

Ver archivo `Deadlock_Diagram.png` / diagrama en el repositorio (generado con el código de la práctica).

```
Estado de deadlock:

  Task1 ──tiene──► [Mutex A]
    │
    └──espera──► [Mutex B] ◄──tiene── Task2
                                          │
                  [Mutex A] ──espera──────┘

  Ninguna puede avanzar. Sistema bloqueado indefinidamente.

Estado resuelto (orden consistente A → B):

  Task1: toma A → toma B → libera B → libera A
  Task2: toma A → toma B → libera B → libera A
         (Task2 bloquea solo en A, nunca crea ciclo)
```
