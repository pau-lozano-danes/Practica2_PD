# Informe - Práctica 2:

En esta práctica trabajaremos con interrupciones (ISR), que son eventos que detienen la ejecución normal del procesador para realizar otra tarea, y luego reanudar la actividad original.

Usaremos dos tipos de eventos que ejecutan funciones ISR:

- Evento hardware (Botón).
- Evento programado (Timer).

---

## Interrupción por hardware - Parte A

Primero declaramos el evento por hardware: un botón. Definimos una estructura que contiene el pin asignado a la interrupción, un contador de pulsaciones y un booleano para indicar si el botón está activo.

```cpp
struct Button {
  const uint8_t PIN;
  uint32_t numberKeyPresses;
  bool pressed;
};
```
Para los dos primeros usamos tipos sin signo uintX_t, que son enteros de X bits.

Asignamos el pin GPIO 18 como entrada para el botón:
```cpp
Button button1 = {18, 0, false};
```
La función que se ejecutará cuando se produzca la interrupción suma 1 al contador y marca el botón como presionado:
```cpp
void IRAM_ATTR isr() {
  button1.numberKeyPresses += 1;
  button1.pressed = true;
}
```
En el setup() inicializamos el puerto serie y configuramos el pin como INPUT_PULLUP para evitar rebotes y asegurar un nivel alto cuando el botón no se pulsa. Además, vinculamos la interrupción con la función ISR en modo FALLING (cuando el pin pasa de HIGH a LOW):
```cpp
void setup() {
  Serial.begin(115200);
  pinMode(button1.PIN, INPUT_PULLUP);
  attachInterrupt(button1.PIN, isr, FALLING);
}
```
En el loop() comprobamos si el botón fue presionado y, en ese caso, imprimimos el número de veces que se activó. Además, tras 1 minuto desactivamos la interrupción para dejar de monitorizar el pin:
```cpp
static uint32_t lastMillis = 0;

void loop() {
  if (button1.pressed) {
    Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
    button1.pressed = false;
  }

  if (millis() - lastMillis > 60000) {
    lastMillis = millis();
    detachInterrupt(button1.PIN);
    Serial.println("Interrupt Detached!");
  }
}
```
---
### Salida ejemplo
```Serial Monitor
Button 1 has been pressed 21 times
Button 1 has been pressed 80 times
Button 1 has been pressed 813 times
Button 1 has been pressed 871 times
Button 1 has been pressed 8160 times
Interrupt Detached!
```

---

##Interrupción por Timer - Parte B

Declaramos una variable contador volatile int interruptCounter; para evitar que el compilador la optimice fuera, ya que será accedida desde el ISR y el loop().

También declaramos un puntero al timer hardware y un mutex para proteger el acceso concurrente a interruptCounter:
```cpp
volatile int interruptCounter;
int totalInterruptCounter;

hw_timer_t * timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;
```
La función ISR para el timer incrementa el contador dentro de una sección crítica para evitar condiciones de carrera:
```cpp
void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}
```
En el setup() inicializamos el timer 0 con un prescaler de 80 (frecuencia base 80MHz / 80 = 1MHz), configuramos la interrupción para que se active cada segundo (1.000.000 ticks), y habilitamos la alarma:
```cpp
void setup() {
  Serial.begin(115200);

  timer = timerBegin(0, 80, true);
  timerAttachInterrupt(timer, &onTimer, true);
  timerAlarmWrite(timer, 1000000, true);
  timerAlarmEnable(timer);
}
```
En el loop() hacemos polling para comprobar si el contador es mayor que cero, y en ese caso decrementamos el contador dentro de una sección crítica y mostramos el total de interrupciones recibidas:
```cpp
void loop() {
  if (interruptCounter > 0) {
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);
    totalInterruptCounter++;
    Serial.print("An interrupt has occurred. Total number: ");
    Serial.println(totalInterruptCounter);
  }
}
```
### Salida ejemplo
```Srial Monitor
An interrupt has occurred. Total number: 1
An interrupt has occurred. Total number: 2
An interrupt has occurred. Total number: 3
An interrupt has occurred. Total number: 4
An interrupt has occurred. Total number: 5
...

```
