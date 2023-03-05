# MAINTAINER
- Name: David Aybar
- Email: daybar4@gmail.com
- Web: https://www.davidaybar.com/

## Proceso de instalación
Se expondrán los siguientes puertos:
- 5000: Logstash TCP input
- 9200: Elasticsearch HTTP
- 9300: Elasticsearch TCP transport
- 5601: Kibana
- 9000: Cerebro HTTP

### Clonemos el proyecto git al directorio actual
```
git clone <url-to-git-project>
```

### Cambiamos directorio a la ruta del proyecto
```
cd <to-project-directory>
```

### Generamos archivos docker-compose.yml, .env
```
cp docker-compose.yml.dist docker-compose.yml
cp .env.dist .env
```

### Customizar credenciales y volumen paths si es necesario
```
vim .env
```

### Creamos el directorio data de Elastic persistido para docker
#### Para instalaciones en un entorno no local, recomiendo separar la data en una partición a parte más amplia
- Ver docker-compose.yml.dist para lista completa de volúmenes persistidos
```
mkdir -p ./elasticsearch/data

```
Aplicamos permisos para la carepta data de elastic
```
sudo chown 1000:1000 ./elasticsearch/data
```

### Creamos el stack, se bajará todo por primera vez, hay que esperar un poco
```
docker-compose up -d --build
```

### El stack está preconfigurado con el siguiente usuario privilegiado de inicio:
```
usuario: elastic
contraseña: david1234
```

### Inicializar las contraseñas de los usuarios del stack:
```
docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```
Las contraseñas para los usuarios se generarán aleatoriamente, guárdalas.
> Comenta la variable "ELASTIC_PASSWORD" del environment del servicio elasticsearch dentro del archivo Compose(docker-compose.yml). 
> Sólo se utiliza para inicializar una clave durante el arranque inicial de Elasticsearch.

### Reemplaza usuarios y contraseñas en los archivos de configuración
Reemplaza la contraseña del usuario Kibana dentro del archivo de configuración kibana/config/kibana.yml y la del usuario logstash_system dentro del archivo de configuración logstash/config/logstash.yml en vez de la del usuario actual elastic.

Reemplazar también la contraseña del usuario elastic dentro del archivo pipeline de Logstash logstash/pipeline/logstash.conf.
> Es importante no usar el usuario logstash_system dentro del archivo de pipeline Logstash ya que no tiene permisos suficientes para crear indices.

### Reiniciamos Kibana y Logstash para aplicar cambios
```
docker-compose restart kibana logstash
```

Despues de dejar un rato para que se inicie Kibana podemos iniciar sesión con el usuario elastic en http://localhost:5601/login
Este será nuestro usuario administrador de Kibana a partir de ahora, después podremos crear tantos usuarios de Kibana como necesitemos.


### Injectar data de prueba

Vamos a injectar data de un log del sistema a logstash:
```
sudo cat /var/log/syslog | nc -q0 localhost 5000
```

De esta manera se va a crear el index de la configuración de pipeline: index => "%{[@metadata][beat]}-%{[@metadata][version]}"

### Kibana crear index pattern
Accedemos a Kibana y navegamos hasta Analytics>Discover. Se nos pedirá que creemos un patrón de índice. 
Lo podemos llamar logstash-*, añadimos el index pattern: "%{[@metadata][beat]}-%{[@metadata][version]}" y el Timestamp por defecto.

### También podemos crear patterns por la línea de comandos
```
curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 8.4.1' \
    -u elastic:<generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

### Configuración de Kibana extra
Para acceder a través de un dominio personal, descomentar la línea de configuración yml de Kibana: #server.publicBaseUrl: https://hostname:5601
Reemplazar hostname por el dominio deseado y reconstruir contenedor Kibana

### Cómo activar las funciones de pago
Cambiar la opción de configuración de elasticsearch xpack.license.self_generated.type de basic a trial

### Cómo añadir plugins
Para añadir plugins a cualquier componente de ELK tenemos que

Añadir una sentencia RUN al Dockerfile correspondiente (ej. RUN logstash-plugin install logstash-filter-json)
Añadir la configuración del código del plugin asociado a la configuración del servicio (ej. Logstash input/output)
Reconstruir las imágenes utilizando el comando: docker-compose up -d build

### JVM tuning
Por defecto, tanto Elasticsearch como Logstash comienzan con 1/4 de la memoria total del host asignada al tamaño del Heap de la JVM.
Veréis que el docker-compose está seteada con 256M de memória

Por ejemplo, para aumentar el tamaño máximo del Heap de la JVM para Logstash:
```
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### Integración extra
Cerebro es una herramienta de administración web para Elasticsearch. Sustituye al antiguo plugin kopf.
Para instalarlo descomentamos las líneas que ya de dejado en el docker-compose.yml y ejecutamos un docker-compose up -d

Una vez iniciado, accedemos mediante la URL http://10.137.110.30:9000/#/connect
Añadimos la URL de acceso a elastic: http://elasticsearch:9200
Y el usuario elastic y contraseña generada colocada en la pipeline de logstash
