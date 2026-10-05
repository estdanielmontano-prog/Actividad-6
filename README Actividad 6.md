# Teclado con brazo robótico dibujando y reconocimiento de dígitos con SPI (ESP32 + Python)

Práctica de Micros con dos puntos:

1. **Teclado 4×4 + LCD I²C + brazo robótico en PyBullet** que dibuja el número oprimido.
2. **Reconocimiento de dígitos escritos a mano** con OpenCV y una CNN; el resultado viaja por serial a una ESP32 **maestra SPI**, que lo envía a una ESP32 **esclava SPI** que lo muestra en una **OLED I²C**.

Se usó como base el brazo `brazo.urdf` y los ejemplos de OpenCV del repositorio del curso ([dialejobv/U_Militar](https://github.com/dialejobv/U_Militar)).

| Enunciado Punto 1 | Enunciado Punto 2 |
|---|---|
| ![Punto 1](docs/enunciado_punto1.png) | ![Punto 2](docs/enunciado_punto2_a.png) |

---

## 1. Objetivo

**Punto 1**
- Leer un teclado matricial con la ESP32 y mostrar la tecla en un LCD por I²C.
- Enviar la tecla al PC por serial y hacer que el brazo del URDF dibuje el dígito en PyBullet.

**Punto 2**
- Capturar dígitos escritos a mano con la cámara y reconocerlos con una red neuronal convolucional.
- Transmitir el dígito por serial → SPI (maestro/esclavo) → pantalla OLED por I²C.

## 2. Descripción del sistema

### Punto 1 – Brazo dibujando
- Teclas **0–9**: el brazo dibuja el dígito en una pizarra virtual.
- Tecla **#**: borra el dibujo. Las demás teclas se ignoran.
- El LCD muestra en la línea 1 la tecla pulsada y en la línea 2 el estado (`Dibujando 5`, `Listo`).

El brazo se mueve con tres articulaciones del URDF:

| Articulación | Tipo | Papel al dibujar |
|---|---|---|
| `joint_1` | Revoluta (Z) | Desplazamiento horizontal sobre la pizarra |
| `joint_2` | Revoluta (Y) | Inclinación del brazo (altura del trazo) |
| `joint_gripper` | Prismática | Extiende la punta hasta la pizarra; al retraerla 3 cm la "pluma" se levanta |

### Punto 2 – Dígitos con SPI
- La CNN (2 capas convolucionales + capa densa) está entrenada con MNIST.
- El dígito se envía solo cuando es estable y distinto del anterior.
- La OLED muestra el dígito grande y una barra con el porcentaje de confianza.

## 3. Hardware

### Punto 1
- 1 × ESP32 DevKit, 1 × teclado matricial 4×4, 1 × LCD 16×2 con módulo I²C (0x27)
- Cables Dupont y cable USB

| Elemento | Pines ESP32 |
|---|---|
| Teclado – filas R1…R4 | GPIO32, GPIO33, GPIO25, GPIO26 |
| Teclado – columnas C1…C4 | GPIO27, GPIO14, GPIO13, GPIO4 |
| LCD I²C | SDA = GPIO21, SCL = GPIO22, VCC, GND |

### Punto 2
- 2 × ESP32 DevKit (ESP-A maestro, ESP-B esclavo), 1 × OLED SSD1306 128×64 I²C (0x3C)
- Cámara web, cables Dupont y cable USB para la ESP-A

**Bus SPI entre las dos ESP32 (VSPI)**

| Señal | ESP-A (maestro) | ESP-B (esclavo) |
|---|---|---|
| SCK | GPIO18 | GPIO18 |
| MISO | GPIO19 | GPIO19 |
| MOSI | GPIO23 | GPIO23 |
| CS | GPIO5 | GPIO5 |
| GND | GND | GND (común) |

**OLED en la ESP-B:** SDA = GPIO21, SCL = GPIO22, VCC = 3.3 V, GND.

📷 *Coloca aquí las fotos de los montajes:* `evidencias/punto1/montaje.jpg` y `evidencias/punto2/montaje.jpg`

## 4. Arquitectura

**Punto 1**
```
Teclado 4x4 ──► ESP32 ──USB serial "K5\n"──► Python (dibujar_brazo.py) ──► PyBullet (brazo.urdf)
                  ▲  │                                  │
              LCD I²C └────── "Dibujando 5" / "Listo" ◄─┘
```

**Punto 2**
```
Cámara PC → Preproceso OpenCV → CNN (MNIST) → Serial "7:98\n" → ESP-A (maestro SPI)
                                                                      │ SPI (paquete de 4 bytes)
                                                                      ▼
                                                      ESP-B (esclavo SPI) → OLED I²C
```

**Protocolo SPI (4 bytes):** `[dígito] [confianza] [checksum = dígito ^ confianza ^ 0x3C] [0xC3]`. La ESP-B descarta cualquier paquete cuyo checksum o marca de fin no coincidan.

## 5. Estructura del proyecto

```
Actividad6_Teclado_Digitos/
├── docs/                                  # Enunciados
├── evidencias/
│   ├── punto1/
│   └── punto2/
├── punto1_teclado_brazo/
│   ├── firmware/teclado_lcd/teclado_lcd.ino
│   └── python/
│       ├── dibujar_brazo.py               # Serial + PyBullet
│       ├── cinematica.py                  # Cinemática inversa/directa
│       ├── trazos.py                      # Dígitos 0-9 como trazos
│       ├── brazo.urdf
│       └── requirements.txt
├── punto2_digitos_spi/
│   ├── firmware/
│   │   ├── esp_a_maestro/esp_a_maestro.ino
│   │   └── esp_b_esclavo/esp_b_esclavo.ino
│   └── python/
│       ├── reconocer_digitos.py           # Cámara + CNN + serial
│       ├── preproceso.py                  # Preprocesamiento tipo MNIST
│       ├── entrenar_cnn.py                # Entrenamiento (una sola vez)
│       ├── modelo_mnist_cnn.h5
│       └── requirements.txt
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Punto 1 – Firmware (`teclado_lcd.ino`)
1. Lee el teclado con la librería `Keypad`.
2. Muestra la tecla en la línea 1 del LCD y envía `K<tecla>` por serial.
3. Todo texto recibido del PC se muestra en la línea 2 del LCD.

### 6.2 Punto 1 – Python
- **`trazos.py`:** cada dígito es una lista de trazos (rectas y arcos de elipse) en un cuadro unitario.
- **`cinematica.py`:** la punta queda en `hombro + (0.42 + ext) · dirección(q1, q2)`, por lo que la inversa tiene solución cerrada. Para un punto `(y, z)` de la pizarra a una distancia `D`:
  - `q1 = atan2(y, D)`
  - `q2 = atan2(√(D² + y²), z − z_hombro)`
  - `ext = √(D² + y² + (z − z_hombro)²) − 0.42`
  
  Se verificó numéricamente con la cinemática directa: la punta cae sobre la pizarra con error nulo y la pinza se mantiene dentro de su recorrido (0 – 0.15 m).
- **`dibujar_brazo.py`:**
  1. Lee `K<tecla>` del serial (o del teclado del PC con `--sin-serial`).
  2. Para cada trazo, acerca la pluma retraída, la apoya y recorre los puntos subdivididos cada 5 mm a velocidad constante.
  3. Pinta la tinta con `addUserDebugLine` siguiendo la posición real de la punta.
  4. Al terminar vuelve a reposo y envía `Listo` al LCD.

### 6.3 Punto 2 – Python
- **`entrenar_cnn.py`:** entrena la CNN sobre MNIST con aumento de datos (rotaciones y traslaciones leves) y guarda `modelo_mnist_cnn.h5`.
- **`preproceso.py`:** escala de grises → desenfoque gaussiano → umbral adaptativo → contorno mayor → recorte → lado mayor a 20 px → lienzo de 28×28 → traslación para centrar el centro de masa (como MNIST).
- **`reconocer_digitos.py`:**
  1. Recorta una zona central de 300×300 px **antes** de dibujar el marco verde, para que el borde no se confunda con un dígito.
  2. La CNN predice el dígito y la confianza.
  3. Votación mayoritaria sobre los últimos 6 cuadros: se envía solo si hay ≥ 4 votos, confianza ≥ 60 % y el dígito es distinto del último enviado.
  4. Si la zona queda vacía unos cuadros, se permite volver a enviar el mismo dígito.
  5. Envía `dígito:confianza` por serial a la ESP-A.

### 6.4 Punto 2 – Firmware
- **ESP-A (maestro):** valida la línea recibida y arma el paquete de 4 bytes con checksum; lo envía por SPI a 1 MHz, modo 0.
- **ESP-B (esclavo):** usa el driver `spi_slave` del ESP-IDF, valida el paquete y dibuja en la OLED el dígito y la barra de confianza.

### Dependencias
- **Python:** `pip install -r requirements.txt` en cada carpeta `python/` (se recomienda Python 3.11 para TensorFlow).
- **Arduino:** Keypad, LiquidCrystal_I2C, Adafruit SSD1306, Adafruit GFX (core ESP32 v3.x).

## 7. Evidencias

🎥 **Punto 1 – video de funcionamiento:** *(pega aquí el enlace del video)*

🎥 **Punto 2 – video de funcionamiento:** *(pega aquí el enlace del video)*

## Referencias
- [Repositorio del curso (U_Militar)](https://github.com/dialejobv/U_Militar)
- [ESP-IDF – SPI Slave Driver](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/peripherals/spi_slave.html)
