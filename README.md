|    Metrics    |                                                                                     Master                                                                                     |                                                                                  Develop                                                                                 |
|:-------------:|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| CI status     | [![pipeline status](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/badges/master/pipeline.svg)](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/commits/master) | [![pipeline status](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/badges/dev/pipeline.svg)](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/commits/dev) |
| Test coverage | [![coverage report](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/badges/master/coverage.svg)](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/commits/master) | [![coverage report](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/badges/dev/coverage.svg)](https://gitlab.com/redmic-project/device/oag-buoy/acmplus/commits/dev) |


# Falmouth Scientific, Inc. ACM-Plus

![FSI ACM-Plus](images/acmplus.jpg)

* Lee datos desde el correntímetro ACM-Plus de la empresa Falmouth Scientific, Inc. 
* Guarda los datos en una base de datos PostgreSQL
* Publica los datos utlizando el protocolo MQTT


## Detección del dispositivo
Para conectar el correntímetro al equipo se han utilizado los conversores FTDI. De esta forma se facilita la tarea
de identificación del dispositivo utilizando reglas UDEV. En este caso hay 2 reglas ya que hay 2 conectores, uno
para pruebas/ reemplazo y el que está en funcionamiento.

```
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idProduct}=="6001", ATTRS{idVendor}=="0403", ATTRS{serial}=="FT0I1IP5", SYMLINK+="current_meter_acmplus"
ACTION=="add", SUBSYSTEM=="tty", ATTRS{idProduct}=="6001", ATTRS{idVendor}=="0403", ATTRS{serial}=="FT0I104U", SYMLINK+="current_meter_acmplus"
```

## Configuración
La configuración del servicio es necesario pasarla a través de un fichero YAML, por defecto debe estar en /etc/buoy/acmplus y
se debe llamar device.yaml.

```buildoutcfg
service:
    path_pidfile: /var/run/buoy/
    start_timeout: 1

database:
    database: database
    user: username
    password: password
    host: localhost

serial:
    port: /dev/current_meter_acmplus
    baudrate: 115200
    stopbits: 1
    parity: N
    bytesize: 8
    timeout: 0

mqtt:
    broker_url: iot.eclipse.org
    client_id: client_id
    topic_data: topic_name
    username: username
    password: changeme
```

También debe de haber otro fichero logging.yaml donde se define el nivel de log y donde se vuelcan.

```buildoutcfg
version: 1
disable_existing_loggers: False
formatters:
    simple:
        format: '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    detail:
        format: '%(asctime)s - %(levelname)s - File: %(filename)s - %(funcName)s() - Line: %(lineno)d -  %(message)s'

handlers:
    console:
        class: logging.StreamHandler
        level: INFO
        formatter: simple
        stream: ext://sys.stdout
root:
    level: INFO
    handlers: [console]
```

## Arranque del servicio
Utilizando las reglas de UDEV, se puede detectar cuando el dispositivo está conectado y entonces iniciar
el servicio de recogida, guardado y envío de datos de forma automática. Para ello, se hay de crear un servicio
en systemd que se arranque con la presencia del dispositivo.

```buildoutcfg
[Unit]
Description=Current Meter - ACMPlus
After=multi-user.target dev-current_meter_acmplus.device
BindsTo=dev-current_meter_acmplus.device

[Service]
Type=idle
ExecStart=/usr/local/bin/current-meter-acmplus
PIDFile=/var/run/buoy/acmplus.pid
ExecStop=/usr/bin/pkill -15 /var/run/buoy/acmplus.pid
TimeoutStopSec=5
Restart=on-failure

[Install]
WantedBy=dev-current_meter_acmplus.device
```

El fichero es necesario copiarlo en /etc/systemd/system/, recargar y activar el servicio.
```
sudo cp current-meter-acmplus.service /etc/systemd/system/
sudo systemctl daemon-reload 
sudo systemctl enable current-meter-acmplus.service
```

Y por último iniciar el servicio.

```
sudo systemctl start current-meter-acmplus.service
```