# **Репозиторий для приема пакетов протокола CRSF , чтения, распаковки, обработки, запаковки и эха пакетов.**

## 1.  Протокол CRSF.
https://github.com/tbs-fpv/tbs-crsf-spec/blob/main/crsf.md#frame-construction исходник на спецификацию протокола.

CRSF (Crossfire Radio System Protocol) — это цифровой протокол связи, для передачи данных между RC-приемником и полетным контроллером (flight controller) в моделях самолетов, дронах и других радиоуправляемых устройствах. 

### 1.1. Cтруктура пакета.
Все пакеты имеют следующий формат : **[Adress] [len] [type] [payload] [crc8].**

- ### **Adress(1 byte).**

https://github.com/crsf-wg/crsf/wiki/CRSF-Addresses исходник со всеми адресами.

Оновные:

<table>
    <tr>
        <th>Address (Hex / Decimal)</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>0x00 / 0</td>
        <td>Broadcast (all devices process packet)</td>
    </tr>
    <tr>
        <td>0xC8 / 200</td>
        <td>Flight Controller (Betaflight / iNav)</td>
    </tr>
    <tr>
        <td>0xEA / 234</td>
        <td>Handset (EdgeTX), not transmitter</td>
    </tr>
    <tr>
        <td>0xEE / 238</td>
        <td>Transmitter module, not handset</td>
    </tr>
</table>
 

 - ### **Len(1 byte).**

 Общая длина пакета составляет: [len]+2(Adress+crc).

- ### **Type(1 byte).**

https://github.com/crsf-wg/crsf/wiki/Packet-Types исходник со всеми типами пакетов, которые могут встретиться.

Основные:
<table>
    <tr>
        <th>Type (Hex / Decimal)</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>0x16 / 22</td>
        <td>Channels data (both handset to TX and RX to flight controller)</td>
    </tr>
    <tr>
        <td>0x3A / 58</td>
        <td>Extended type used for OPENTX_SYNC</td>
    </tr>
    <tr>
        <td>0x08 / 8</td>
        <td>Battery voltage, current, mAh, remaining percent</td>
    </tr>
    <tr>
        <td>0x32 / 50</td>
        <td>CRSF command execute</td>
    </tr>
        <tr>
        <td>0x29 / 41</td>
        <td>Device name, firmware version, hardware version, serial number (PING response)</td>
    </tr>
    <tr>
        <td>0x28 / 40</td>
        <td>Sender requesting DEVICE_INFO from all destination devices</td>
    </tr>
    <tr>
        <td>0x2E / 46</td>
        <td>!!Non Standard!! ExpressLRS good/bad packet count, status flags</td>
    </tr>
    <tr>
        <td>0x2B / 43</td>
        <td>Configuration item data chunk</td>
    </tr>
    <tr>
        <td>0x16 / 22</td>
        <td>Channels data (both handset to TX and RX to flight controller)</td>
    </tr>
</table>

- ### **Payload.**

Упаковка полезной нагрузки (payload) в протоколе CRSF зависит от типа пакета. Разные типы пакетов имеют разные форматы полезной нагрузки. Рассмотрим наиболее распространенный тип: *RC Channels Packed (0x16).*

**Упаковка RC Channels Packed (0x16):**

Этот тип пакета содержит данные о каналах управления (стиках, переключателях и т. д.). CRSF использует 11 бит для представления значения каждого канала (от 0 до 2047). Поскольку 11 бит не умещаются в один байт (8 бит), приходится “упаковывать” данные нескольких каналов в несколько байтов. CRSF делает это довольно хитрым способом для максимальной эффективности.

1. **Разбиение на 11-битные значения:** Предположим, у вас есть 16 значений каналов, каждое из которых находится в диапазоне от 0 до 2047. Каждое из этих значений занимает 11 бит.

2. **Упаковка в байты:** CRSF упаковывает эти 11-битные значения в последовательность байтов. Для 16 каналов требуется 22 байта. Упаковка происходит следующим образом:
- Байт 4: CH1[0..7] | CH2[0..2]
- Байт 5: CH2[3..7] | CH3[0..5]
- Байт 6: CH3[6..7]
- Байт 7: CH3[8..10] | CH4[0..7]
- Байт 8: CH4[8..10] | CH5[0..4]
- Байт 9: CH5[5..7] | CH6[0..6]
- Байт 10: CH6[7..10]
- Байт 11: CH7[0..7] | CH8[0..1]
- Байт 12: CH8[2..7] | CH9[0..3]
- Байт 13: CH9[4..7] | CH10[0..6]
- Байт 14: CH10[7..10] | CH11[0..2]
- Байт 15: CH11[3..7] | CH12[0..4]
- Байт 16: CH12[5..7] | CH13[0..5]
- Байт 17: CH13[6..10]
- Байт 18: CH14[0..7] | CH15[0..1]
- Байт 19: CH15[2..7] | CH16[0..3]
- Байт 20: CH16[4..7]
- Байт 21: CH16[8..10] | CH17[0..7]
- Байт 22: CH17[8..10] | CH18[0..4]
- Байт 23: CH18[5..7] | CH19[0..6]
- Байт 24: CH19[7..10]
- Байт 25: CH20[0..7]

3. **Порядок байтов:** Обычно используется порядок от младшего к старшему (little-endian), но это нужно проверять в вашей конкретной реализации.


Пример кода для запаковки пакета:

``` cpp
void crsfPrepareDataPacket(byte packet[], uint16_t channels[]) {

    packet[0] = Adress; 
    packet[1] = Len;          
    packet[2] = Type;
    packet[3] = (uint8_t)(channels[0] & 0x07FF);
    packet[4] = (uint8_t)((channels[0] & 0x07FF) >> 8 | (channels[1] & 0x07FF) << 3);
    packet[5] = (uint8_t)((channels[1] & 0x07FF) >> 5 | (channels[2] & 0x07FF) << 6);
    packet[6] = (uint8_t)((channels[2] & 0x07FF) >> 2);
    packet[7] = (uint8_t)((channels[2] & 0x07FF) >> 10 | (channels[3] & 0x07FF) << 1);
    packet[8] = (uint8_t)((channels[3] & 0x07FF) >> 7 | (channels[4] & 0x07FF) << 4);
    packet[9] = (uint8_t)((channels[4] & 0x07FF) >> 4 | (channels[5] & 0x07FF) << 7);
    packet[10] = (uint8_t)((channels[5] & 0x07FF) >> 1);
    packet[11] = (uint8_t)((channels[5] & 0x07FF) >> 9 | (channels[6] & 0x07FF) << 2);
    packet[12] = (uint8_t)((channels[6] & 0x07FF) >> 6 | (channels[7] & 0x07FF) << 5);
    packet[13] = (uint8_t)((channels[7] & 0x07FF) >> 3);
    packet[14] = (uint8_t)((channels[8] & 0x07FF));
    packet[15] = (uint8_t)((channels[8] & 0x07FF) >> 8 | (channels[9] & 0x07FF) << 3);
    packet[16] = (uint8_t)((channels[9] & 0x07FF) >> 5 | (channels[10] & 0x07FF) << 6);
    packet[17] = (uint8_t)((channels[10] & 0x07FF) >> 2);
    packet[18] = (uint8_t)((channels[10] & 0x07FF) >> 10 | (channels[11] & 0x07FF) << 1);
    packet[19] = (uint8_t)((channels[11] & 0x07FF) >> 7 | (channels[12] & 0x07FF) << 4);
    packet[20] = (uint8_t)((channels[12] & 0x07FF) >> 4 | (channels[13] & 0x07FF) << 7);
    packet[21] = (uint8_t)((channels[13] & 0x07FF) >> 1);
    packet[22] = (uint8_t)((channels[13] & 0x07FF) >> 9 | (channels[14] & 0x07FF) << 2);
    packet[23] = (uint8_t)((channels[14] & 0x07FF) >> 6 | (channels[15] & 0x07FF) << 5);
    packet[24] = (uint8_t)((channels[15] & 0x07FF) >> 3);
    packet[25] = crc; 
}
```
Пример для распаковки пакетов:

```cpp
void unpackChannels(const uint8_t *packet, int16_t *channels) {
    channels[0]  = ((int16_t)packet[4] << 8) & 0x0700;
    channels[0] |= ((int16_t)packet[5] >> 3) & 0x00FF;

    channels[1]  = ((int16_t)packet[5] << 5) & 0x0700;
    channels[1] |= ((int16_t)packet[6] >> 6) & 0x001F;
    channels[1] |= ((int16_t)packet[7] << 2) & 0x07E0;

    channels[2]  = ((int16_t)packet[7] >> 1) & 0x0003;
    channels[2] |= ((int16_t)packet[8] << 7) & 0x0780;
    channels[2] |= ((int16_t)packet[9] >> 4) & 0x007F;

    channels[3]  = ((int16_t)packet[9] << 4) & 0x0700;
    channels[3] |= ((int16_t)packet[10] >> 7) & 0x000F;
    channels[3] |= ((int16_t)packet[11] << 1) & 0x07FE;

    channels[4]  = ((int16_t)packet[11] >> 2) & 0x0001;
    channels[4] |= ((int16_t)packet[12] << 6) & 0x0740;
    channels[4] |= ((int16_t)packet[13] >> 5) & 0x003F;

    channels[5]  = ((int16_t)packet[13] << 3) & 0x0700;
    channels[5] |= (int16_t)packet[14] >> 8 &0; //channel[14] is always a single value
    channels[5] |= (int16_t)packet[14] & 0x07FF;

    channels[6]  = ((int16_t)packet[15] << 5) & 0x0700;
    channels[6] |= ((int16_t)packet[16] >> 6) & 0x001F;
    channels[6] |= ((int16_t)packet[17] << 2) & 0x07E0;

    channels[7]  = ((int16_t)packet[17] >> 1) & 0x0003;
    channels[7] |= ((int16_t)packet[18] << 7) & 0x0780;
    channels[7] |= ((int16_t)packet[19] >> 4) & 0x007F;

    channels[8]  = ((int16_t)packet[19] << 4) & 0x0700;
    channels[8] |= ((int16_t)packet[20] >> 7) & 0x000F;
    channels[8] |= ((int16_t)packet[21] << 1) & 0x07FE;

    channels[9]  = ((int16_t)packet[21] >> 2) & 0x0001;
    channels[9] |= ((int16_t)packet[22] << 6) & 0x0740;
    channels[9] |= ((int16_t)packet[23] >> 5) & 0x003F;

    channels[10] = ((int16_t)packet[23] << 3) & 0x0700;
    channels[10] |= (int16_t)packet[24] >> 8 &0; //channel[24] is always a single value
    channels[10] |= (int16_t)packet[24] & 0x07FF;

    channels[11]  = 0;
    channels[12] = 0;
    channels[13] = 0;
    channels[14] = 0;
    channels[15] = 0;
}

```
- ### **CRC**.

Для посчета контрольной суммы (CRC (Cyclic Redundancy Check)) в протоколе CRSF используется crc8.

Пример кода:

``` cpp
static uint8_t crsf_crc8tab[256] = {
    0x00, 0xD5, 0x7F, 0xAA, 0xFE, 0x2B, 0x81, 0x54, 0x29, 0xFC, 0x56, 0x83, 0xD7, 0x02, 0xA8, 0x7D,
    0x52, 0x87, 0x2D, 0xF8, 0xAC, 0x79, 0xD3, 0x06, 0x7B, 0xAE, 0x04, 0xD1, 0x85, 0x50, 0xFA, 0x2F,
    0xA4, 0x71, 0xDB, 0x0E, 0x5A, 0x8F, 0x25, 0xF0, 0x8D, 0x58, 0xF2, 0x27, 0x73, 0xA6, 0x0C, 0xD9,
    0xF6, 0x23, 0x89, 0x5C, 0x08, 0xDD, 0x77, 0xA2, 0xDF, 0x0A, 0xA0, 0x75, 0x21, 0xF4, 0x5E, 0x8B,
    0x9D, 0x48, 0xE2, 0x37, 0x63, 0xB6, 0x1C, 0xC9, 0xB4, 0x61, 0xCB, 0x1E, 0x4A, 0x9F, 0x35, 0xE0,
    0xCF, 0x1A, 0xB0, 0x65, 0x31, 0xE4, 0x4E, 0x9B, 0xE6, 0x33, 0x99, 0x4C, 0x18, 0xCD, 0x67, 0xB2,
    0x39, 0xEC, 0x46, 0x93, 0xC7, 0x12, 0xB8, 0x6D, 0x10, 0xC5, 0x6F, 0xBA, 0xEE, 0x3B, 0x91, 0x44,
    0x6B, 0xBE, 0x14, 0xC1, 0x95, 0x40, 0xEA, 0x3F, 0x42, 0x97, 0x3D, 0xE8, 0xBC, 0x69, 0xC3, 0x16,
    0xEF, 0x3A, 0x90, 0x45, 0x11, 0xC4, 0x6E, 0xBB, 0xC6, 0x13, 0xB9, 0x6C, 0x38, 0xED, 0x47, 0x92,
    0xBD, 0x68, 0xC2, 0x17, 0x43, 0x96, 0x3C, 0xE9, 0x94, 0x41, 0xEB, 0x3E, 0x6A, 0xBF, 0x15, 0xC0,
    0x4B, 0x9E, 0x34, 0xE1, 0xB5, 0x60, 0xCA, 0x1F, 0x62, 0xB7, 0x1D, 0xC8, 0x9C, 0x49, 0xE3, 0x36,
    0x19, 0xCC, 0x66, 0xB3, 0xE7, 0x32, 0x98, 0x4D, 0x30, 0xE5, 0x4F, 0x9A, 0xCE, 0x1B, 0xB1, 0x64,
    0x72, 0xA7, 0x0D, 0xD8, 0x8C, 0x59, 0xF3, 0x26, 0x5B, 0x8E, 0x24, 0xF1, 0xA5, 0x70, 0xDA, 0x0F,
    0x20, 0xF5, 0x5F, 0x8A, 0xDE, 0x0B, 0xA1, 0x74, 0x09, 0xDC, 0x76, 0xA3, 0xF7, 0x22, 0x88, 0x5D,
    0xD6, 0x03, 0xA9, 0x7C, 0x28, 0xFD, 0x57, 0x82, 0xFF, 0x2A, 0x80, 0x55, 0x01, 0xD4, 0x7E, 0xAB,
    0x84, 0x51, 0xFB, 0x2E, 0x7A, 0xAF, 0x05, 0xD0, 0xAD, 0x78, 0xD2, 0x07, 0x53, 0x86, 0x2C, 0xF9};

uint8_t crsf_crc8(const uint8_t *ptr, uint8_t len) {
uint8_t crc = 0;
for (uint8_t i = 0; i < len; i++){
    crc = crsf_crc8tab[crc ^ *ptr++];}
return crc;
}

packet[25] = crsf_crc8(&packet[2], packet[1] - 1);

```

**Примеры пакетов, полученные с помощью логического анализатора:**


![](assets/1.png)
![](assets/2.png)
![](assets/3.png)
![](assets/4.png)

Частота пакетов 50Гц, Скорость 115200Кбод/с.


## 2.Обработка пакетов с помощью ESP32.
Структурная схема:
![](assets/5.png)

Загрузка прошивки на esp32 через wifi, который она сама раздает.

Для этого понадобится библиотека ElegantOTA https://github.com/ayushsharma82/ElegantOTA.

 Чтобы билиотека заработала с esp32 в PlatformIo:

 Поскольку ElegantOTA поддерживает несколько архитектур, PlatformIO попытается автоматически включить все зависимости, что часто приводит к ошибкам компиляции. Чтобы решить эту проблему, выполните следующие действия:

- Удалите .pio/libdeps папку (если она существует) в вашем проекте, прежде чем продолжить.

- Откройте platformio.ini файл вашего проекта.

-  Добавьте следующую строку внутри вашего platformio.ini файла:
```
lib_compat_mode = strict
```

- Сохраните изменения в platformio.ini файле.

После данной процедуры можно устанавливать библиотеку(например, через менеджер библиотек).


Для загрузки прошивки:
``` c++
#include <WiFi.h>
#include <WebServer.h>
#include <ElegantOTA.h>

const char* apSSID = "ESP32-OTA";          // Имя вашей точки доступа
const char* apPassword = "password";       // Пароль для подключения к точке доступа (минимум 8 символов)

WebServer server(80);

// Колбэк, вызываемый после успешного завершения OTA-обновления
void onOTAEnd(bool success) {
  Serial.println("OTA update finished!");
  if (success) {
    Serial.println("Rebooting...");
    ESP.restart(); // Перезагружаем ESP32
  } else {
    Serial.println("OTA update failed!");
  }
}

void setup() {
  Serial.begin(115200);

  // Настраиваем ESP32 в режиме точки доступа (AP)
  WiFi.softAP(apSSID, apPassword);

  IPAddress apIP = WiFi.softAPIP();
  Serial.print("Access Point \"");
  Serial.print(apSSID);
  Serial.println("\" started");
  Serial.print("Connect to it using password: ");
  Serial.println(apPassword);
  Serial.print("AP IP address: ");
  Serial.println(apIP);

  server.on("/", []() {
    server.send(200, "text/plain", "Hello from ElegantOTA!");
  });

   // ElegantOTA callbacks
  ElegantOTA.onEnd(onOTAEnd);
  ElegantOTA.begin(&server);    // Initialise ElegantOTA

  server.begin();
  Serial.println("HTTP server started");
}

void loop() {
  server.handleClient();
  // Добавьте ваш основной код здесь.  ElegantOTA будет работать в фоне.
}
```

- Скомпилируйте и загрузите код на вашу ESP32 плату.

- Найдите в списке доступных сетей Wi-Fi сеть с именем ESP32-OTA (или то имя, которое вы указали в apSSID).

- Подключитесь к этой сети, используя пароль password (или тот пароль, который вы указали в apPassword).

- В Serial Monitor вы увидите IP-адрес точки доступа ESP32 (обычно 192.168.4.1).

- Откройте веб-браузер на вашем компьютере или телефоне и перейдите по этому IP-адресу.

- Вы должны увидеть веб-страницу, предоставленную ElegantOTA.

- Загрузите файл прошивки (.bin) и нажмите “Update”.

