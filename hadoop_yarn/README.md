# Levantar un clúster de Hadoop con los servicios de YARN y realizar una prueba de WordCount

## Ejecutar Docker Compose

En este archivo `docker-compose.yml`, se definen los servicios que forman el clúster de Hadoop.

```yaml
services:
  namenode:
    container_name: namenode
    image: bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8
    restart: always
    ports:
      - 9870:9870    # Puerto web de NameNode
      - 9000:9000    # Puerto de comunicación de HDFS
    volumes:
      - hadoop_namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop.env   # Archivo con configuración adicional para Hadoop

  datanode:
    container_name: datanode
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    restart: always
    ports:
      - 9864:9864    # Puerto web de DataNode
    volumes:
      - hadoop_datanode:/hadoop/dfs/data
    environment:
      SERVICE_PRECONDITION: "namenode:9870"  # Nombre del servicio NameNode para la comunicación
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode

  resourcemanager:
    container_name: resourcemanager
    image: bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8
    restart: always
    ports:
      - 8088:8088    # Puerto web de ResourceManager
    environment:
      SERVICE_PRECONDITION: "namenode:9000 namenode:9870 datanode:9864"
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode
      - datanode

  nodemanager:
    container_name: nodemanager
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8
    restart: always
    ports:
      - 8042:8042    # Puerto web de NodeManager
    environment:
      SERVICE_PRECONDITION: "namenode:9000 namenode:9870 datanode:9864 resourcemanager:8088"
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode
      - datanode
      - resourcemanager

volumes:
  hadoop_namenode:
  hadoop_datanode:
```

Cada servicio en este archivo representa una parte crítica del clúster de Hadoop:

- **NameNode**: Administra el sistema de archivos HDFS y dirige las operaciones en los datos almacenados.
- **DataNode**: Almacena físicamente los bloques de datos y se comunica con el NameNode.
- **ResourceManager**: Gestiona los recursos en el clúster para las aplicaciones YARN.
- **NodeManager**: Administra los recursos de cada nodo y ejecuta las aplicaciones.

### Configuración de variables de entorno para Hadoop

El archivo de entorno configura las opciones de HDFS y YARN. Estas variables personalizan la conexión y funcionamiento del clúster.

```bash
CORE_CONF_fs_defaultFS=hdfs://namenode:9000
CORE_CONF_hadoop_http_staticuser_user=root
CORE_CONF_hadoop_proxyuser_hue_hosts=*
CORE_CONF_hadoop_proxyuser_hue_groups=*
CORE_CONF_io_compression_codecs=org.apache.hadoop.io.compress.SnappyCodec

HDFS_CONF_dfs_webhdfs_enabled=true
HDFS_CONF_dfs_permissions_enabled=false
HDFS_CONF_dfs_namenode_datanode_registration_ip___hostname___check=false

YARN_CONF_yarn_log___aggregation___enable=true
YARN_CONF_yarn_log_server_url=http://historyserver:8188/applicationhistory/logs/
YARN_CONF_yarn_resourcemanager_recovery_enabled=true
YARN_CONF_yarn_resourcemanager_store_class=org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore
YARN_CONF_yarn_resourcemanager_scheduler_class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler
YARN_CONF_yarn_scheduler_capacity_root_default_maximum___allocation___mb=8192
YARN_CONF_yarn_scheduler_capacity_root_default_maximum___allocation___vcores=4
YARN_CONF_yarn_resourcemanager_fs_state___store_uri=/rmstate
YARN_CONF_yarn_resourcemanager_system___metrics___publisher_enabled=true
YARN_CONF_yarn_resourcemanager_hostname=resourcemanager
YARN_CONF_yarn_resourcemanager_address=resourcemanager:8032
YARN_CONF_yarn_resourcemanager_scheduler_address=resourcemanager:8030
YARN_CONF_yarn_resourcemanager_resource__tracker_address=resourcemanager:8031
YARN_CONF_yarn_timeline___service_enabled=true
YARN_CONF_yarn_timeline___service_generic___application___history_enabled=true
YARN_CONF_yarn_nodemanager_remote___app___log___dir=/app-logs
```

Estas configuraciones controlan aspectos importantes, como la ruta base del sistema de archivos, el uso de la compresión y la configuración de los servicios YARN para la recuperación de aplicaciones y el registro de logs.

## Iniciar el clúster

Arranca todos los servicios del clúster en segundo plano:

```bash
docker compose up -d
```

## Verificar que todos los contenedores están funcionando

Confirma que los contenedores están en ejecución:

```bash
docker compose ps
```

## Prueba del clúster

Para verificar que el clúster está configurado correctamente, realiza una prueba simple en HDFS.

1. Accede al contenedor `namenode`:
   ```bash
   docker compose exec namenode bash
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

## Ejemplo de WordCount

Ejecuta un trabajo de `WordCount` para contar las palabras en un archivo de entrada y verificar la operación de MapReduce en el clúster.

1. Dentro del contenedor `namenode`, configura la entrada y ejecuta `WordCount`:
   ```bash
   # Crear directorio para input
   hdfs dfs -mkdir -p /user/root/wordcount/input

   # Crear archivo de prueba
   echo "Hola mundo Hadoop YARN mundo" > input.txt
   hdfs dfs -put input.txt /user/root/wordcount/input

   # Ejecutar WordCount
   hadoop jar /opt/hadoop-3.2.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcount /user/root/wordcount/input /user/root/wordcount/output

   # Ver resultados
   hdfs dfs -cat /user/root/wordcount/output/part-r-00000
   ```

## Acceso a las interfaces web

Hadoop y YARN ofrecen interfaces web para gestionar el sistema:

- **HDFS NameNode**: [http://localhost:9870](http://localhost:9870) – Monitorea el estado de HDFS.
- **YARN ResourceManager**: [http://localhost:8088](http://localhost:8088) – Supervisa los recursos y trabajos YARN.
- **NodeManager**: [http://localhost:8042](http://localhost:8042) – Consulta el estado del NodeManager.

## Detener el clúster

Para detener y eliminar los servicios:

```bash
docker compose down
```

Para eliminar también los volúmenes de datos:

```bash
docker compose down -v
```