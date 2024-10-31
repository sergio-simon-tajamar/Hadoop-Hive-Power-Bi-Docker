# Levantar un clúster de Hadoop con YARN y Hive

Este repositorio proporciona una configuración para levantar un clúster de Hadoop que incluye YARN y Hive, utilizando Docker. Se presentan los pasos para configurar, iniciar el clúster y realizar pruebas básicas de consulta SQL.

## Ejecutar Docker Compose

En este archivo `docker-compose.yml`, se definen los servicios que forman el clúster de Hadoop junto con Hive:

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
      - ./hadoop-hive.env
    ports:
      - "50070:50070"
  
  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    volumes:
      - datanode:/hadoop/dfs/data
    env_file:
      - ./hadoop-hive.env
    environment:
      SERVICE_PRECONDITION: "namenode:50070"
    ports:
      - "50075:50075"
  
  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      SERVICE_PRECONDITION: "hive-metastore:9083"
    ports:
      - "10000:10000"
  
  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
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

volumes:
  namenode:
  datanode:
```

Cada servicio en este archivo representa una parte crítica del clúster de Hadoop:

- **NameNode**: Administra el sistema de archivos HDFS y dirige las operaciones en los datos almacenados.
- **DataNode**: Almacena físicamente los bloques de datos y se comunica con el NameNode.
- **ResourceManager**: Gestiona los recursos en el clúster para las aplicaciones YARN.
- **NodeManager**: Administra los recursos de cada nodo y ejecuta las aplicaciones.
- **Hive Metastore**: Almacena metadatos de las tablas y estructuras de datos de Hive.
- **Hive Server**: Permite realizar consultas SQL sobre los datos en Hive.

### Configuración de variables de entorno para Hadoop

El archivo de entorno `hadoop.env` configura las opciones de HDFS y YARN. Estas variables personalizan la conexión y funcionamiento del clúster. Crea un archivo llamado `hadoop.env` y agrega la siguiente configuración:

```bash
# Configuración de la conexión a la base de datos del metastore de Hive
HIVE_SITE_CONF_javax_jdo_option_ConnectionURL=jdbc:postgresql://hive-metastore-postgresql/metastore # URL de conexión JDBC para PostgreSQL
HIVE_SITE_CONF_javax_jdo_option_ConnectionDriverName=org.postgresql.Driver # Controlador JDBC para PostgreSQL
HIVE_SITE_CONF_javax_jdo_option_ConnectionUserName=hive # Nombre de usuario para la conexión a la base de datos
HIVE_SITE_CONF_javax_jdo_option_ConnectionPassword=hive # Contraseña para la conexión a la base de datos
HIVE_SITE_CONF_datanucleus_autoCreateSchema=false # No crear automáticamente el esquema en la base de datos
HIVE_SITE_CONF_hive_metastore_uris=thrift://hive-metastore:9083 # URI del servicio Thrift para el metastore de Hive

# Configuración del sistema de archivos HDFS
HDFS_CONF_dfs_namenode_datanode_registration_ip___hostname___check=false # Desactivar la verificación de registro de IP/hostname para nodos

# Configuración del sistema de archivos por defecto
CORE_CONF_fs_defaultFS=hdfs://namenode:8020 # Sistema de archivos HDFS por defecto
CORE_CONF_hadoop_http_staticuser_user=root # Usuario estático para acceso HTTP en Hadoop
CORE_CONF_hadoop_proxyuser_hue_hosts=* # Hosts permitidos para el proxy user de Hue
CORE_CONF_hadoop_proxyuser_hue_groups=* # Grupos permitidos para el proxy user de Hue

# Configuración del HDFS Web
HDFS_CONF_dfs_webhdfs_enabled=true # Habilitar acceso a WebHDFS
HDFS_CONF_dfs_permissions_enabled=false # Desactivar permisos en HDFS

# Configuración de YARN (Yet Another Resource Negotiator)
YARN_CONF_yarn_log___aggregation___enable=true # Habilitar la agregación de logs de aplicaciones
YARN_CONF_yarn_resourcemanager_recovery_enabled=true # Habilitar recuperación del Resource Manager
YARN_CONF_yarn_resourcemanager_store_class=org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore # Clase para almacenamiento del estado del Resource Manager
YARN_CONF_yarn_resourcemanager_fs_state___store_uri=/rmstate # URI del sistema de archivos para almacenar el estado del Resource Manager
YARN_CONF_yarn_nodemanager_remote___app___log___dir=/app-logs # Directorio remoto para logs de aplicaciones en Node Manager
YARN_CONF_yarn_log_server_url=http://historyserver:8188/applicationhistory/logs/ # URL del servidor de logs de la historia de las aplicaciones
YARN_CONF_yarn_timeline___service_enabled=true # Habilitar el servicio de línea de tiempo
YARN_CONF_yarn_timeline___service_generic___application___history_enabled=true # Habilitar el historial de aplicaciones genéricas en el servicio de línea de tiempo
YARN_CONF_yarn_resourcemanager_system___metrics___publisher_enabled=true # Habilitar la publicación de métricas del Resource Manager
YARN_CONF_yarn_resourcemanager_hostname=resourcemanager # Nombre del host del Resource Manager
YARN_CONF_yarn_timeline___service_hostname=historyserver # Nombre del host del servicio de línea de tiempo
YARN_CONF_yarn_resourcemanager_address=resourcemanager:8032 # Dirección del Resource Manager
YARN_CONF_yarn_resourcemanager_scheduler_address=resourcemanager:8030 # Dirección del scheduler del Resource Manager
YARN_CONF_yarn_resourcemanager_resource__tracker_address=resourcemanager:8031 # Dirección del tracker de recursos del Resource Manager
```

## Iniciar el clúster

Arranca todos los servicios del clúster en segundo plano:

```bash
docker-compose up -d
```

## Verificar que todos los contenedores están funcionando

Confirma que los contenedores están en ejecución:

```bash
docker-compose ps
```

Para verificar que el clúster está configurado correctamente, realiza una prueba simple en HDFS.

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

4. Verifica que el archivo está en HDFS:
   ```bash
   hdfs dfs -cat /user/root/test/test.txt
   ```

## Ejemplo de consulta SQL en Hive

Para realizar consultas SQL en Hive, sigue estos pasos:

1. Accede al contenedor de Hive:
   ```bash
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

4. Realiza una consulta:
   ```sql
   SELECT * FROM pokes;
   ```

## Detener el clúster

Para detener y eliminar los servicios:

```bash
docker-compose down
```

Para eliminar también los volúmenes de datos:

```bash
docker-compose down -v
```

Este README proporciona una guía completa sobre cómo levantar un clúster de Hadoop con YARN y Hive, realizar