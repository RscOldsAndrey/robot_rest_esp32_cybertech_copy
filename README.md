# ROBOT REST ESP32 CYBERTECH

Для прототипа используется [nanoESP32-S3](https://github.com/wuxx/nanoESP32-S3/tree/master).

## Описание API

1) Управление движением робота - `PUT` - `/move`

```json
{"id": "3036393632076C54", "direction": "forward", "len": 100, "speed": 30}
{"id": "3036393632076C54", "direction": "right", "len": 100, "speed": 30}
```

Робот переместится в указанном направлениии на заданный угол или на расстояние в mm. `"direction"` может името следущие значения:

- `forward` - движение вперед, `len` будет длинной в `mm`.
- `backward` - движение назад, `len` будет длинной в `mm`.
- `left` - движение влево, `len` будет углом в градусах.
- `right` - движение вправо, `len` будет углом в градусах.

Для линейного движения `len` может быть **больше** `0 mm` и **меньше или ровно** `10000 mm`. Значения передаются с точность 1 mm.
Для поворота `len` может быть **больше** `0 градусов` и **меньше или ровно** `360 градусам`. Значения передаются с точность 1 градуса.

`speed` задает скорость в `mm/s`. Для поворототов скорость задается тоже с помощью `mm/s`.

Данные комманды не являются блокирующими. После запроса робот сразу вернет `200` если комманда валидна. Для того чтобы узнать что происходит на моторах в данный момент нужно отправить запрос в сенсор который вернет текущий pwm.

2) Запрос значений сенсоров - `POST` - `/sensor`

```json
{"id": "3036393632076C54", "type": "all"} // Запрос всех сенсоров
{"id": "3036393632076C54", "type": "laser"} // Запрос показаний лазерных сенсоров
{"id": "3036393632076C54", "type": "imu"} // Запрос данных с гироскопа
{"id": "3036393632076C54", "type": "motor"} // Запрос текущего PWM с каждого из моторов
```

Ответ будет иметь следующий формат:

```json
{ 
	"laser": {"1": 12, "2": 12, "3": 12}, 
	"imu": {"roll": 20, "pitch": 30, "yaw": 40},
    "motor": {"left_pwm": 20, "right_pwm": -30},
} // Запрос all.
{"laser": {"left": 12, "left45": 17, "forward": 20, "right45": 100, "right": 100, "backward": 100}} // Дальномер mm.
{"imu": {"roll": 20, "pitch": 30, "yaw": 40}} // imu - поворот в пространстве относительно конкретной оси в градусах.
{"motor": {"left_pwm": 20, "right_pwm": -30}} // PWM который физически выставлен на моторе в данный момент
```

Все числа в запросах и ответах это целые значения (int).

`pitch` и `yaw` имею значения от -180 .. 180 то угол смещения от положения во время включения сенсора. `roll` имеет значения от 0 ... 360 это поворот по оси Z. Все измеряется в градусах.

В случае если значение за пределом видимости дальномера он вернет `8190` или больше.

2) Запрос настройки сенсоров - `POST` - `/sensor_config`

```json
{
    "id": "F535AF9628574A53",
    "interval": 33,
    "enabled_sensors": ["left", "right", "forward"]
}
```
interval - интервал в мс от 20 до 200

## Сборка проекта

1) Установить `IDF`:

```shell
mkdir esp32 && cd esp32
sudo apt install git wget flex bison gperf python3 python3-pip python3-venv cmake ninja-build ccache libffi-dev libssl-dev dfu-util libusb-1.0-0
git clone -b v5.3 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf && ./install.sh
cd ..
```

2) Распаковать (скачать) данный проект собрать его:

```shell
. esp-idf/export.sh
idf.py build
idf.py flash
idf.py monitor 
```

Последняя команда чтобы посмотреть логи работы.

## Тестирование

Для работы с API необходимо указать в `main/inc/config.hpp` название сети и пароль для WiFi. Потом в логах необходимо будет узнать IP адрес и уникальный ID. Они будут в следующих строках:

```
...
I (425) server.cpp:  <--- Chip ID: 3036393632076C54 ---> 
...
I (1495) server.cpp:  <--- IP Address: 192.168.1.179 ---> 
...
```

REST API можно протестировать с помощью curl, примеры комманд:

```shell
curl -X PUT -H "Content-Type: application/json" -d '{"id": "3036393632076C54", "l": 100, "r": -100, "l_time": 2000, "r_time": 4000}' http://192.168.1.179/motor
curl -X POST -H "Content-Type: application/json" -d '{"id": "3036393632076C54", "type": "all"}' http://192.168.1.179/sensor
curl -X POST -H "Content-Type: application/json" -d '{"id": "3036393632076C54", "type": "laser"}' http://192.168.1.179/sensor
curl -X POST -H "Content-Type: application/json" -d '{"id": "3036393632076C54", "type": "imu"}' http://192.168.1.179/sensor
curl -X POST -H "Content-Type: application/json" -d '{"id": "F535AF9628574A53", "interval": 20, "enabled_sensors": ["left", "right", "forward"]}' http://192.168.69.144/sensor_config
```

----

## Проблемы с IDF

Данный код тестировался на ветке `v5.3` и в данном релизе есть проблемы с I2C. Более подробно можно посмотреть [тут](https://github.com/espressif/esp-idf/issues/14401). Без данных исправлений IMU не заработает.

Для того чтобы быстро все исправить нужно открыть файл `esp-idf/components/esp_driver_i2c/i2c_master.c` внутри IDF и закомментировать строку:

```c

    // Эту строку закомментировать (примерно 555)
    // i2c_hal_master_set_scl_timeout_val(hal, i2c_dev->scl_wait_us i2c_master->base->clk_src_freq_hz);

    // .........

    I2C_CLOCK_SRC_ATOMIC() {
        i2c_hal_set_bus_timing(hal, i2c_dev->scl_speed_hz, i2c_master->base->clk_src, i2c_master->base->clk_src_freq_hz);
    }
    // Сюда строку добавляем (примерно 573)
    i2c_hal_master_set_scl_timeout_val(hal, i2c_dev->scl_wait_us, i2c_master->base->clk_src_freq_hz); // эту строку добавить
    i2c_ll_master_set_fractional_divider(hal->dev, 0, 0);
    i2c_ll_update(hal->dev);
```

Данную строку необходимо переместить чтобы Драйвер не затирал значение таймаута которое он сам-же и выставил.

----

Если у вашего пользователя нет доступа к сериал порту, можно сделать это:

```shell
sudo usermod -aG dialout $USER
sudo newgrp dialout 
```

Чтобы выйти из режима просмотра логов - `Ctrl + ]`.

----

Изменения в `config` файде делаются через `idf.py menuconfig` (их не нужно делать, это уже выставленно в проекте):

- `(Top) → Serial flasher config → Flash size` - 4Mb
- `(Top) → Component config → ESP System Settings → CPU frequency` - 240Mhz
