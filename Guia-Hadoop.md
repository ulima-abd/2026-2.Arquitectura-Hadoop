# Demostración: Configuración de Hadoop con Docker

## Requisitos previos

- Docker Desktop ≥ 20.x con WSL2 habilitado
- Al menos 4 GB de RAM disponible para Docker

---

## Fase 1 — Prerrequisitos y entorno

### Paso 1: Verificar Docker instalado

```bash
# Verificar versión de Docker
docker --version

# Verificar que el daemon está activo
docker info
```

### Paso 2: Crear directorio del proyecto

```bash
mkdir hadoop-demo && cd hadoop-demo
```

---

## Fase 2 — Levantar el clúster Hadoop

Utilizar el archivo `docker-compose.yml` que se le entrega.

> **Puertos expuestos:**
> - `9870` → UI del HDFS NameNode
> - `9000` → RPC del NameNode (HDFS)
> - `8088` → UI de YARN ResourceManager
> - `8032` → RPC del ResourceManager
> - `8188` → UI del Job History Server


### Paso 4: Levantar el clúster

```bash
# Levantar en background
docker compose up -d

# Verificar contenedores activos
docker compose ps

# Ver logs del NameNode
docker compose logs namenode
```

> Espera ~30 segundos para que el NameNode salga del modo seguro (*safe mode*) y para que el `nodemanager` termine de registrarse en el ResourceManager. Verifica el estado del HDFS en [http://localhost:9870](http://localhost:9870) y el de YARN (debe mostrar 1 nodo activo, "Active Nodes") en [http://localhost:8088](http://localhost:8088).

---

## Fase 3 — Demo: WordCount con MapReduce

### Paso 6: Subir datos a HDFS y ejecutar WordCount

Entra al contenedor del NameNode, crea directorios en HDFS y ejecuta el JAR de ejemplo incluido en la imagen.

```bash
# Entrar al contenedor NameNode
docker exec -it namenode bash
```

Una vez dentro del contenedor:

```bash
# Crear directorio de entrada en HDFS
hdfs dfs -mkdir -p /user/root/input

# Crear archivo de prueba y subirlo a HDFS
echo "hello hadoop hello world hadoop is great" > texto.txt
hdfs dfs -put texto.txt /user/root/input/

# Ejecutar el job WordCount (se enviará a YARN, no en modo local)
hadoop jar \
  $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar \
  wordcount /user/root/input /user/root/output

# Ver el resultado
hdfs dfs -cat /user/root/output/part-r-00000
```

> Mientras el job corre, puedes verlo en la UI de YARN ([http://localhost:8088](http://localhost:8088)) con estado `RUNNING` y luego `SUCCEEDED`, y su historial en el Job History Server ([http://localhost:8188](http://localhost:8188)).

> **Nota — configuración YARN necesaria:** el `docker-compose.yml` entregado ya incluye la configuración necesaria para que el job de `wordcount` encuentre el `resourcemanager` y pueda ejecutar los mappers/reducers. Si adaptas el compose y el job falla, revisa que:
> - El servicio `namenode` (desde donde se corre `hadoop jar`) tenga las variables `YARN_CONF_yarn_resourcemanager_hostname`, `_address`, `_scheduler_address` y `_resource___tracker_address` — sin ellas, el cliente no sabe dónde está el resourcemanager y el job nunca arranca (se queda reintentando la conexión).
> - El servicio `nodemanager` tenga `YARN_CONF_yarn_nodemanager_aux___services=mapreduce_shuffle` y `YARN_CONF_yarn_nodemanager_aux___services_mapreduce_shuffle_class=org.apache.hadoop.mapred.ShuffleHandler` — sin esto, los mappers fallan con `InvalidAuxServiceException: The auxService:mapreduce_shuffle does not exist`.

**Salida esperada:**

```
great   1
hadoop  2
hello   2
is      1
world   1
```

> **Nota:** El nombre del contenedor puede variar. Usa `docker ps` para confirmar el nombre exacto antes de ejecutar `docker exec`.

### Paso 7: Explorar las UIs web

Muestra a los estudiantes la interfaz gráfica de HDFS y YARN para visualizar el estado del clúster.

| Interfaz | URL | Qué mostrar |
|---|---|---|
| HDFS NameNode UI | [http://localhost:9870](http://localhost:9870) | Utilities → Browse filesystem para ver los archivos y bloques |
| YARN ResourceManager UI | [http://localhost:8088](http://localhost:8088) | Nodos activos (Nodes) y la lista de aplicaciones (Applications) con el job WordCount |
| Job History Server UI | [http://localhost:8188](http://localhost:8188) | Detalle del job ya finalizado: mappers, reducers y contadores |
