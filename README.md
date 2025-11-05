# 🧩 Sistema de Archivos Distribuido con Docker y GlusterFS

## 📘 Guía Completa de Implementación

---

## 📋 Índice

1. [Descripción General](#descripción-general)
2. [Requisitos Previos](#requisitos-previos)
3. [Fundamentos Teóricos](#fundamentos-teóricos)
4. [Arquitectura del Sistema](#arquitectura-del-sistema)
5. [Implementación Paso a Paso](#implementación-paso-a-paso)
6. [Pruebas y Verificación](#pruebas-y-verificación)
7. [Resolución de Problemas](#resolución-de-problemas)
8. [Conclusiones](#conclusiones)

---

## 📘 Descripción General

Este proyecto implementa un **Sistema de Archivos Distribuido (DFS)** utilizando contenedores Docker para simular múltiples nodos de red y GlusterFS como sistema de archivos distribuido.

### Objetivos

- ✅ Comprender la arquitectura de un sistema de archivos distribuido
- ✅ Simular un entorno de red distribuida con Docker
- ✅ Configurar un volumen compartido y replicado con GlusterFS
- ✅ Montar el volumen distribuido en un cliente
- ✅ Evaluar la consistencia, replicación y tolerancia a fallos

---

## 🔧 Requisitos Previos

### Software necesario

- **Docker Desktop** instalado y en ejecución
- **Docker Compose** (incluido en Docker Desktop)
- Terminal o línea de comandos
- Editor de texto (VS Code, Notepad++, etc.)

### Conocimientos básicos

- Comandos básicos de terminal/bash
- Conceptos básicos de Docker
- Nociones de sistemas de archivos

---

## 🧠 Fundamentos Teóricos

### ¿Qué es un Sistema de Archivos Distribuido?

Un **DFS (Distributed File System)** permite que varios equipos (nodos) compartan y accedan de forma concurrente a archivos almacenados en distintos puntos de una red, manteniendo la transparencia de acceso.

### Características clave

- **Transparencia de localización**: El usuario accede a los archivos sin saber en qué nodo están almacenados
- **Replicación y tolerancia a fallos**: El sistema mantiene copias sincronizadas de los datos
- **Escalabilidad**: Se pueden agregar nodos para aumentar capacidad o disponibilidad
- **Seguridad y consistencia**: Garantiza que los datos sean coherentes en toda la red

### ¿Qué es GlusterFS?

GlusterFS es un sistema de archivos distribuido moderno, basado en el concepto de "volúmenes distribuidos y replicados", que agrupa múltiples servidores en un mismo sistema lógico de archivos.

### ¿Qué es Docker?

Docker es una plataforma que permite crear "contenedores" - entornos virtuales aislados que actúan como servidores independientes, pero comparten el kernel del sistema operativo host.

---

## 🏗️ Arquitectura del Sistema

### Escenario: DataLink Corp

La empresa DataLink Corp necesita garantizar el acceso compartido a documentos y recursos por parte de distintos servidores regionales.

### Componentes del Sistema

```
┌─────────────────┐         ┌─────────────────┐
│     NODE1       │◄───────►│     NODE2       │
│  (Servidor A)   │         │  (Servidor B)   │
│   - Almacena    │         │   - Réplica     │
│   - Sincroniza  │         │   - Sincroniza  │
└────────▲────────┘         └────────▲────────┘
         │                           │
         │    Red GlusterNet         │
         │                           │
         │     ┌─────────────┐      │
         └─────►   CLIENT    ◄──────┘
               │  (Usuario)  │
               │  Accede a   │
               │  archivos   │
               └─────────────┘
```

### Descripción de nodos

- **Node1**: Servidor de almacenamiento A (nodo primario)
- **Node2**: Servidor de almacenamiento B (réplica sincronizada)
- **Client**: Nodo cliente que monta y accede al sistema de archivos distribuido

---

## 🚀 Implementación Paso a Paso

### PASO 1: Crear la estructura del proyecto

```bash
# Crear directorio del proyecto
mkdir dfs-lab
cd dfs-lab

# Crear subdirectorio para pruebas
mkdir pruebas
```

Tu estructura debe quedar así:

```
dfs-lab/
├── docker-compose.yml
├── README.md
└── pruebas/
```

---

### PASO 2: Crear el archivo docker-compose.yml

Crea un archivo llamado `docker-compose.yml` con el siguiente contenido:

```yaml
services:
  node1:
    image: gluster/gluster-centos:latest
    container_name: node1
    privileged: true
    networks:
      - glusternet
    volumes:
      - data1:/data/brick1
    hostname: node1
    command: /bin/bash -c "glusterd && tail -f /dev/null"

  node2:
    image: gluster/gluster-centos:latest
    container_name: node2
    privileged: true
    networks:
      - glusternet
    volumes:
      - data2:/data/brick1
    hostname: node2
    command: /bin/bash -c "glusterd && tail -f /dev/null"

  client:
    image: almalinux:9
    container_name: client
    privileged: true
    networks:
      - glusternet
    depends_on:
      - node1
      - node2
    tty: true
    stdin_open: true

networks:
  glusternet:
    driver: bridge

volumes:
  data1:
  data2:
```

#### Explicación de la configuración

- **image**: Imágenes Docker oficiales (gluster-centos para nodos, almalinux para cliente)
- **privileged**: Necesario para que GlusterFS funcione correctamente
- **networks**: Red virtual compartida entre contenedores
- **volumes**: Almacenamiento persistente para cada nodo
- **command**: Inicia automáticamente el demonio de GlusterFS

---

### PASO 3: Inicializar el entorno

```bash
# Levantar todos los servicios
docker-compose up -d

# Esperar a que los contenedores arranquen
sleep 10

# Verificar que estén corriendo
docker ps
```

**Resultado esperado:**

```
CONTAINER ID   NAME      IMAGE                          STATUS
xxxxx          node1     gluster/gluster-centos:latest  Up
xxxxx          node2     gluster/gluster-centos:latest  Up
xxxxx          client    almalinux:9                    Up
```

---

### PASO 4: Verificar que glusterd esté corriendo

```bash
# Verificar node1
docker exec node1 ps aux | grep glusterd

# Verificar node2
docker exec node2 ps aux | grep glusterd
```

Deberías ver procesos `glusterd` activos en ambos nodos.

---

### PASO 5: Configurar el clúster GlusterFS

```bash
# Conectar node1 con node2 (formar el clúster)
docker exec -it node1 gluster peer probe node2

# Verificar el estado del clúster
docker exec -it node1 gluster peer status
```

**Resultado esperado:**

```
Number of Peers: 1

Hostname: node2
Uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
State: Peer in Cluster (Connected)
```

✅ Esto confirma que ambos nodos están conectados y forman un clúster.

---

### PASO 6: Crear el volumen distribuido y replicado

```bash
# Crear el volumen con réplica en 2 nodos
docker exec -it node1 gluster volume create datavolume replica 2 transport tcp node1:/data/brick1 node2:/data/brick1 force

# Iniciar el volumen
docker exec -it node1 gluster volume start datavolume

# Ver información del volumen
docker exec -it node1 gluster volume info
```

**Resultado esperado:**

```
Volume Name: datavolume
Type: Replicate
Volume ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node1:/data/brick1
Brick2: node2:/data/brick1
```

#### Explicación

- **Type: Replicate**: Los datos se replican en ambos nodos
- **Number of Bricks: 1 x 2 = 2**: Un volumen con 2 réplicas
- **Status: Started**: El volumen está activo y funcionando

---

### PASO 7: Instalar GlusterFS en el cliente

```bash
# Entrar al contenedor client
docker exec -it client bash
```

Ahora estás **dentro del contenedor**. El prompt cambiará a:

```
[root@client /]#
```

Ejecuta:

```bash
# Actualizar el sistema (opcional pero recomendado)
dnf update -y

# Instalar el cliente GlusterFS
dnf install -y glusterfs glusterfs-fuse

# Verificar la instalación
glusterfs --version
```

**NO salgas del contenedor aún**, continúa en el siguiente paso.

---

### PASO 8: Montar el volumen en el cliente

**Dentro del contenedor client** (donde aún estás), ejecuta:

```bash
# Crear el punto de montaje
mkdir -p /mnt/dfs

# Montar el volumen GlusterFS
mount -t glusterfs node1:/datavolume /mnt/dfs

# Verificar el montaje
df -hT | grep gluster
```

**Resultado esperado:**

```
node1:/datavolume  fuse.glusterfs  1007G  14G  953G  2% /mnt/dfs
```

✅ El volumen está correctamente montado en `/mnt/dfs`.

#### Explicación del comando mount

- `-t glusterfs`: Tipo de sistema de archivos
- `node1:/datavolume`: Servidor y nombre del volumen
- `/mnt/dfs`: Punto de montaje local

---

### PASO 9: Realizar pruebas de escritura/lectura

**Dentro del contenedor client**, ejecuta:

```bash
# Crear archivo de prueba
echo "Hola DFS Distribuido!" > /mnt/dfs/prueba.txt

# Leer el archivo
cat /mnt/dfs/prueba.txt

# Crear más archivos
echo "Este es el archivo 2" > /mnt/dfs/archivo2.txt
echo "Este es el archivo 3" > /mnt/dfs/archivo3.txt

# Crear una carpeta con contenido
mkdir /mnt/dfs/documentos
echo "Documento interno" > /mnt/dfs/documentos/doc1.txt

# Listar todos los archivos
ls -la /mnt/dfs/
ls -la /mnt/dfs/documentos/
```

Ahora **sal del contenedor**:

```bash
exit
```

---

## 🧪 Pruebas y Verificación

### PRUEBA 1: Verificar replicación entre nodos

**Desde tu terminal (fuera de los contenedores):**

```bash
# Verificar archivos en node1
docker exec node1 ls -la /data/brick1/

# Leer contenido en node1
docker exec node1 cat /data/brick1/prueba.txt

# Verificar archivos en node2
docker exec node2 ls -la /data/brick1/

# Leer contenido en node2
docker exec node2 cat /data/brick1/prueba.txt
```

✅ **Los archivos deben estar idénticos en ambos nodos**. Esto confirma que la replicación funciona correctamente.

---

### PRUEBA 2: Tolerancia a fallos (Detener node1)

```bash
# Apagar el nodo1
docker stop node1

# Entrar al cliente
docker exec -it client bash

# Intentar leer archivos (desde dentro del client)
cat /mnt/dfs/prueba.txt
ls /mnt/dfs/

# Crear un nuevo archivo con node1 apagado
echo "Archivo creado sin node1" > /mnt/dfs/nuevo.txt

# Salir
exit
```

✅ **El sistema debe seguir funcionando** porque node2 tiene todas las réplicas.

---

### PRUEBA 3: Recuperación del nodo

```bash
# Volver a encender node1
docker start node1

# Esperar a que se reconecte
sleep 10

# Verificar reconexión
docker exec -it node1 gluster peer status

# Verificar que el archivo nuevo se replicó
docker exec node1 cat /data/brick1/nuevo.txt
```

✅ GlusterFS **sincroniza automáticamente** los cambios cuando el nodo vuelve.

---

### PRUEBA 4: Escritura concurrente

```bash
# Abrir dos terminales

# Terminal 1 - Entrar al client
docker exec -it client bash
echo "Escritura desde terminal 1" >> /mnt/dfs/concurrente.txt

# Terminal 2 - Escribir directamente en node1
docker exec -it node1 bash
echo "Escritura desde node1" >> /data/brick1/concurrente.txt

# Leer el archivo desde client
docker exec client cat /mnt/dfs/concurrente.txt
```

Observar cómo GlusterFS maneja la sincronización.

---

### PRUEBA 5: Medición de rendimiento

```bash
# Entrar al client
docker exec -it client bash

# Probar velocidad de escritura (crear archivo de 100MB)
dd if=/dev/zero of=/mnt/dfs/test.img bs=1M count=100 conv=fdatasync

# Probar velocidad de lectura
dd if=/mnt/dfs/test.img of=/dev/null bs=1M

# Salir
exit
```

Documentar los tiempos obtenidos.

---

## 🔍 Guardar Logs y Resultados

```bash
# Crear directorio de pruebas si no existe
mkdir -p pruebas

# Guardar logs de node1
docker logs node1 > pruebas/logs-node1.txt

# Guardar logs de node2
docker logs node2 > pruebas/logs-node2.txt

# Guardar información del volumen
docker exec node1 gluster volume info > pruebas/volumen-info.txt

# Guardar estado del clúster
docker exec node1 gluster peer status > pruebas/peer-status.txt

# Guardar lista de archivos replicados
docker exec node1 ls -laR /data/brick1/ > pruebas/archivos-node1.txt
docker exec node2 ls -laR /data/brick1/ > pruebas/archivos-node2.txt
```

---

## 🛠️ Resolución de Problemas

### Problema 1: glusterd no está corriendo

**Síntoma:**
```
Connection failed. Please check if gluster daemon is operational.
```

**Solución:**
```bash
docker exec -d node1 glusterd
docker exec -d node2 glusterd
sleep 5
```

---

### Problema 2: Error al montar el volumen

**Síntoma:**
```
Mount failed. Please check the log file for more details.
```

**Solución:**
```bash
# Verificar que el volumen esté activo
docker exec node1 gluster volume status datavolume

# Si está detenido, iniciarlo
docker exec node1 gluster volume start datavolume

# Reintentar montaje
docker exec -it client mount -t glusterfs node1:/datavolume /mnt/dfs
```

---

### Problema 3: Contenedores no se comunican

**Síntoma:** Los nodos no pueden conectarse entre sí.

**Solución:**
```bash
# Verificar que estén en la misma red
docker network inspect dfs-lab_glusternet

# Verificar conectividad
docker exec node1 ping -c 3 node2
docker exec node2 ping -c 3 node1
```

---

### Problema 4: Archivos no se replican

**Solución:**
```bash
# Ver estado detallado del volumen
docker exec node1 gluster volume status datavolume detail

# Verificar que ambos bricks estén online
docker exec node1 gluster volume heal datavolume info

# Forzar sincronización
docker exec node1 gluster volume heal datavolume full
```

---

## 🎯 Comandos Útiles de Referencia

### Gestión de contenedores

```bash
# Ver contenedores activos
docker ps

# Ver todos los contenedores
docker ps -a

# Ver logs de un contenedor
docker logs node1

# Reiniciar un contenedor
docker restart node1

# Detener todos los servicios
docker-compose down

# Iniciar todos los servicios
docker-compose up -d

# Reconstruir contenedores
docker-compose up -d --build
```

### Gestión de volúmenes Docker

```bash
# Listar volúmenes
docker volume ls

# Inspeccionar un volumen
docker volume inspect dfs-lab_data1

# Eliminar volúmenes no usados
docker volume prune
```

### Comandos GlusterFS

```bash
# Ver estado del clúster
docker exec node1 gluster peer status

# Ver información del volumen
docker exec node1 gluster volume info

# Ver estado detallado del volumen
docker exec node1 gluster volume status datavolume detail

# Detener un volumen
docker exec node1 gluster volume stop datavolume

# Eliminar un volumen
docker exec node1 gluster volume delete datavolume

# Ver logs de GlusterFS
docker exec node1 tail -f /var/log/glusterfs/glusterd.log
```

---

## 💭 Análisis Conceptual

### ¿Qué ocurre si ambos nodos fallan?

- **Pérdida total de acceso**: Sin nodos disponibles, el cliente no puede acceder a los datos
- **Datos intactos**: Los datos permanecen en los volúmenes Docker persistentes
- **Recuperación**: Al reiniciar los nodos, el sistema se recupera automáticamente

### ¿Cómo se asegura la consistencia entre copias?

GlusterFS utiliza:
- **Self-heal daemon**: Proceso que detecta y repara inconsistencias
- **Replicación síncrona**: Las escrituras se confirman solo cuando se replican en todos los nodos
- **Metadata tracking**: Seguimiento de versiones y cambios

### ¿Qué mecanismos detectan fallos automáticamente?

- **Heartbeat**: Comunicación periódica entre nodos
- **Health checks**: Verificación del estado de los servicios
- **Auto-heal**: Reparación automática de archivos divididos (split-brain)

---

## 📊 Conclusiones

### Ventajas de GlusterFS

✅ **Alta disponibilidad**: Sin puntos únicos de fallo  
✅ **Escalabilidad horizontal**: Fácil agregar más nodos  
✅ **Transparencia**: Los usuarios no perciben la distribución  
✅ **Integración**: Compatible con contenedores, VMs y cloud  
✅ **Open Source**: Gratuito y con comunidad activa  

### Desafíos

❌ **Consistencia**: Requiere mecanismos de sincronización complejos  
❌ **Latencia**: En redes geográficamente distribuidas  
❌ **Complejidad**: Configuración y mantenimiento en gran escala  
❌ **Split-brain**: Posibles conflictos en escrituras concurrentes  

### Casos de uso reales

- **Almacenamiento de medios**: Servidores de streaming
- **Big Data**: Hadoop, Analytics
- **Virtualización**: Almacenamiento compartido para VMs
- **Backups**: Replicación geográfica de datos críticos
- **Cloud Storage**: Infraestructura de almacenamiento distribuido

---

## 🚀 Extensiones Avanzadas (Opcional)

### 1. Agregar un tercer nodo

Modificar `docker-compose.yml`:

```yaml
  node3:
    image: gluster/gluster-centos:latest
    container_name: node3
    privileged: true
    networks:
      - glusternet
    volumes:
      - data3:/data/brick1
    hostname: node3
    command: /bin/bash -c "glusterd && tail -f /dev/null"

volumes:
  data1:
  data2:
  data3:  # Agregar
```

Agregar al clúster:

```bash
docker exec node1 gluster peer probe node3
docker exec node1 gluster volume add-brick datavolume replica 3 node3:/data/brick1 force
```

### 2. Servidor web compartido

```bash
# Crear un contenedor nginx
docker run -d --name webserver \
  --network dfs-lab_glusternet \
  -v dfs-lab_data1:/usr/share/nginx/html:ro \
  -p 8080:80 \
  nginx

# Acceder en el navegador
# http://localhost:8080
```

### 3. Monitoreo con Prometheus

Integrar métricas de GlusterFS con Prometheus y visualizar en Grafana.

### 4. Cifrado y autenticación

```bash
# Habilitar autenticación
docker exec node1 gluster volume set datavolume auth.allow 192.168.*

# Configurar SSL/TLS
docker exec node1 gluster volume set datavolume client.ssl on
docker exec node1 gluster volume set datavolume server.ssl on
```

---

## 📚 Referencias y Recursos

### Documentación oficial

- [GlusterFS Documentation](https://docs.gluster.org/)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/)

### Repositorios

- [GlusterFS GitHub](https://github.com/gluster/glusterfs)
- [Docker Gluster Images](https://hub.docker.com/r/gluster/gluster-centos)

### Tutoriales adicionales

- [GlusterFS Architecture](https://docs.gluster.org/en/latest/Quick-Start-Guide/Architecture/)
- [Docker Networking Guide](https://docs.docker.com/network/)
- [FUSE Filesystem](https://www.kernel.org/doc/html/latest/filesystems/fuse.html)

---

## ✅ Checklist Final

Antes de finalizar el proyecto, verifica:

- [ ] Los 3 contenedores están corriendo
- [ ] El clúster GlusterFS está formado (peer status OK)
- [ ] El volumen está creado y activo
- [ ] El volumen está montado en el cliente
- [ ] Los archivos se replican en ambos nodos
- [ ] La tolerancia a fallos funciona (probado)
- [ ] Logs guardados en carpeta `pruebas/`
- [ ] Capturas de pantalla tomadas
- [ ] Informe documentado

---

## 🧹 Limpieza del entorno

Cuando termines el proyecto:

```bash
# Detener todos los contenedores
docker-compose down

# Eliminar volúmenes (CUIDADO: borra todos los datos)
docker volume rm dfs-lab_data1 dfs-lab_data2

# Eliminar imágenes descargadas (opcional)
docker rmi gluster/gluster-centos:latest almalinux:9

# Limpiar sistema completo
docker system prune -a
```
