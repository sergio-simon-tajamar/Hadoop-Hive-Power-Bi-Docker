# Levantar un Clúster de Hadoop con YARN, Hive y Power BI

Este repositorio proporciona una configuración para desplegar un clúster de Hadoop que incluye YARN, Hive y Power BI utilizando Docker. A continuación, se detallan los pasos necesarios para configurar e iniciar el clúster de manera efectiva.

## Requisitos Previos

Antes de comenzar, asegúrate de tener instalados los siguientes programas:

- [Docker](https://www.docker.com/get-started)
- [Docker Compose](https://docs.docker.com/compose/)

## Estructura del Proyecto

El archivo `docker-compose.yml` define los servicios necesarios para el clúster de Hadoop. A continuación, se presenta la configuración básica:

```yaml
version: "3"

services:
  namenode:
    container_name: namenode
    image: bde2020/hadoop-namenode:2.0.0-hadoop2.7.4-java8
    volumes:
      - namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop-powerbi.env
    ports:
      - "50070:50070"

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    volumes:
      - datanode:/hadoop/dfs/data
    env_file:
      - ./hadoop-powerbi.env
    environment:
      SERVICE_PRECONDITION: "namenode:50070"
    ports:
      - "50075:50075"

  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-powerbi.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      SERVICE_PRECONDITION: "hive-metastore:9083"
    ports:
      - "10000:10000"

  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-powerbi.env
    command: /opt/hive/bin/hive --service metastore
    environment:
      SERVICE_PRECONDITION: "namenode:50070 datanode:50075 hive-metastore-postgresql:5432"
    ports:
      - "9083:9083"

  hive-metastore-postgresql:
    image: bde2020/hive-metastore-postgresql:2.3.0

  presto-coordinator:
    image: shawnzhu/prestodb:0.181
    ports:
      - "8080:8080"

  superset:
    image: apache/superset:latest
    environment:
      SUPERSET_ENV: production
      SUPERSET_LOAD_EXAMPLES: "yes"
      SUPERSET_SECRET_KEY: "thisisaverysecurekey"
    ports:
      - "8088:8088"
    depends_on:
      - hive-server
    entrypoint: >
      /bin/sh -c "
      superset db upgrade;
      superset init;
      superset run -h 0.0.0.0 -p 8088
      "

volumes:
  namenode:
  datanode:
```

## Descripción de los Servicios

Cada servicio en el archivo `docker-compose.yml` tiene un rol específico en el clúster:

- **NameNode**: Administra el sistema de archivos HDFS y controla las operaciones sobre los datos almacenados.
- **DataNode**: Almacena los bloques de datos físicamente y se comunica con el NameNode.
- **ResourceManager**: Gestiona los recursos del clúster para las aplicaciones YARN.
- **NodeManager**: Administra los recursos de cada nodo y ejecuta las aplicaciones.
- **Hive Metastore**: Almacena metadatos sobre las tablas y estructuras de datos de Hive.
- **Hive Server**: Permite realizar consultas SQL sobre los datos en Hive.
- **Power BI**: Herramienta de visualización y análisis de datos. Crea informes interactivos y dashboards a partir de datos procesados en el clúster.

## Configuración de Variables de Entorno

Para personalizar la conexión y funcionamiento del clúster, crea un archivo llamado `hadoop-powerbi.env` con el siguiente contenido:

```bash
HIVE_SITE_CONF_javax_jdo_option_ConnectionURL=jdbc:postgresql://hive-metastore-postgresql/metastore
HIVE_SITE_CONF_javax_jdo_option_ConnectionDriverName=org.postgresql.Driver
HIVE_SITE_CONF_javax_jdo_option_ConnectionUserName=hive
HIVE_SITE_CONF_javax_jdo_option_ConnectionPassword=hive
HIVE_SITE_CONF_datanucleus_autoCreateSchema=false
HIVE_SITE_CONF_hive_metastore_uris=thrift://hive-metastore:9083
HDFS_CONF_dfs_namenode_datanode_registration_ip___hostname___check=false

CORE_CONF_fs_defaultFS=hdfs://namenode:8020
CORE_CONF_hadoop_http_staticuser_user=root
CORE_CONF_hadoop_proxyuser_hue_hosts=*
CORE_CONF_hadoop_proxyuser_hue_groups=*

HDFS_CONF_dfs_webhdfs_enabled=true
HDFS_CONF_dfs_permissions_enabled=false

YARN_CONF_yarn_log___aggregation___enable=true
YARN_CONF_yarn_resourcemanager_recovery_enabled=true
YARN_CONF_yarn_resourcemanager_store_class=org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore
YARN_CONF_yarn_resourcemanager_fs_state___store_uri=/rmstate
YARN_CONF_yarn_nodemanager_remote___app___log___dir=/app-logs
YARN_CONF_yarn_log_server_url=http://historyserver:8188/applicationhistory/logs/
YARN_CONF_yarn_timeline___service_enabled=true
YARN_CONF_yarn_timeline___service_generic___application___history_enabled=true
YARN_CONF_yarn_resourcemanager_system___metrics___publisher_enabled=true
YARN_CONF_yarn_resourcemanager_hostname=resourcemanager
YARN_CONF_yarn_timeline___service_hostname=historyserver
YARN_CONF_yarn_resourcemanager_address=resourcemanager:8032
YARN_CONF_yarn_resourcemanager_scheduler_address=resourcemanager:8030
YARN_CONF_yarn_resourcemanager_resource__tracker_address=resourcemanager:8031
```

## Iniciar el Clúster

Para arrancar todos los servicios del clúster en segundo plano, ejecuta:

```bash
docker-compose up -d
```

## Verificar el Estado de los Contenedores

Para confirmar que todos los contenedores están en ejecución, utiliza:

```bash
docker-compose ps
```

## Prueba de Configuración en HDFS

Para verificar que el clúster está correctamente configurado, realiza una prueba simple en HDFS:

1. Accede al contenedor `namenode`:
   ```bash
   docker-compose exec namenode bash
   ```

2. Crea un directorio de prueba en HDFS:
   ```bash
   hdfs dfs -mkdir -p /user/root/test
   ```

3. Crea un archivo de prueba y súbelo a HDFS:
   ```bash
   echo "Hola Mundo desde Hadoop YARN en Docker" > test.txt
   hdfs dfs -put test.txt /user/root/test
   ```

4. Verifica que el archivo se haya subido correctamente:
   ```bash
   hdfs dfs -cat /user/root/test/test.txt
   ```

## Ejemplo de Consulta SQL en Hive

Para realizar consultas SQL en Hive, sigue estos pasos:

1. Accede al contenedor de Hive:
   ```bash
   docker-compose exec hive-server bash

   /opt/hive/bin/beeline -u jdbc:hive2://localhost:10000
   ```

2. Crea una tabla en Hive:
   ```sql
   CREATE TABLE pokes (foo INT, bar STRING);
   ```

3. Carga datos en la tabla:
   ```sql
   LOAD DATA LOCAL INPATH '/opt/hive/examples/files/kv1.txt' OVERWRITE INTO TABLE pokes;
   ```

4. Realiza una consulta para verificar los datos:
   ```sql
   SELECT * FROM pokes;
   ```

## Comprobación de Power BI

Accede a Power BI en la siguiente URL:

- **URL**: [http://localhost:8088/](http://localhost:8088/)

Por defecto, puedes acceder con las siguientes credenciales:

- **Usuario**: admin
- **Contraseña**: admin123

### Conectarse a la Base de Datos en Power BI

1. Dirígete a `Data > Connect to dataset`.
   
   ![Conexión a dataset](./images/image.png)

2. Selecciona el tipo de base de datos **Hive** y proporciona la URI de la base de datos en el siguiente formato:
   - `hive://@hive-server:10000/default`
   - `hive://username:password@hive-server:10000/default`

   ![Selección de base de datos](./images/image2.png)

   ![Selección de base de datos](./images/image3.png)

3. Conéctate a la base de datos. Si la conexión falla, dirígete a `SQL > SQL Lab`.

   ![SQL Lab](./images/image4.png)

### Ejemplo de Datos de Prueba

Puedes utilizar los siguientes comandos para insertar datos de prueba en la tabla `ventas`:

```sql
CREATE TABLE ventas (
    fecha DATE,
    producto STRING,
    cantidad INT,
    precio DECIMAL(10, 2)
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

INSERT INTO ventas (fecha, producto, cantidad, precio) VALUES
('2024-01-05', 'Producto A', 10, 15.50),
('2024-01-12', 'Producto B', 5, 20.00),
('2024-01-20', 'Producto A', 8, 15.50),
('2024-01-25', 'Producto C', 12, 7.50),
('2024-02-02', 'Producto B', 20, 20.00),
('2024-02-10', 'Producto A', 15, 15.50),
('2024-02-15', 'Producto C', 10, 7.50),
('2024-02-22', 'Producto B', 7, 20.00),
('2024-03-01', 'Producto A', 5, 15.50),
('2024-03-10', 'Producto C', 6, 7.50),
('2024-03-15', 'Producto A', 11, 15.50),
('2024-03-20', 'Producto B', 4, 20.00),
('2024-03-30', 'Producto C', 8, 7.50),
('2024-04-02', 'Producto A', 9, 15.50),
('2024-04-10', 'Producto B', 3, 20.00),
('2024-04-15', 'Producto C', 15, 7.50);
```

Para comprobar la conexión, ejecuta la siguiente consulta en la tabla `ventas`:

```sql
SELECT 
    DATE_FORMAT(fecha, 'yyyy-MM') AS mes,
    producto,
    SUM(cantidad) AS total_cantidad,
    SUM(cantidad * precio) AS total_ventas
FROM 
    ventas
GROUP BY 
    DATE_FORMAT(fecha, 'yyyy-MM'), producto
ORDER BY 
    mes, producto;
```

![Consulta SQL en Power BI](./images/image-1.png)

## Detener el Clúster

Para detener y eliminar los servicios del clúster, ejecuta:

```bash
docker-compose down
```

Para eliminar también los volúmenes de datos, usa:

```bash
docker-compose down -v
```