# 8. Despliegue en AWS y demostración del funcionamiento

Para esta parte te recomiendo una estrategia concreta: crear una instancia EC2 con Ubuntu y ejecutar dentro de ella todo el clúster mediante Docker Compose. Así tendrás en AWS los tres servidores de metadatos, los cinco nodos de datos y el cliente, cada uno como un contenedor independiente.

Esta opción encaja con el `docker-compose.yml` que ya tiene el repositorio y evita tener que modificar las direcciones de los servicios para distribuirlos entre ocho máquinas virtuales. También permite controlar los fallos deteniendo contenedores concretos y capturar las pruebas desde la consola de AWS.

Hay una diferencia importante que debes dejar clara en el informe: será un clúster de ocho servicios distribuidos en contenedores, alojados en una misma instancia EC2. Sirve para demostrar la comunicación entre procesos, Raft, replicación, transferencia y recuperación de nodos; no demuestra tolerancia a la caída de toda una máquina física.

La secuencia que vamos a seguir será:

1. Crear la infraestructura de red y la instancia EC2.
2. Instalar y configurar Docker.
3. Clonar el repositorio.
4. Generar certificados y secretos.
5. Construir e iniciar DFSha.
6. Verificar que los tres servidores de metadatos y los cinco nodos de datos estén activos.
7. Ejecutar las pruebas funcionales.
8. Demostrar replicación, concurrencia y recuperación ante fallos.
9. Ejecutar las pruebas de rendimiento.
10. Capturar las evidencias para el informe.

## 1. Crear la infraestructura en AWS Academy

### 1.1. Iniciar el laboratorio

1. Ingresa a AWS Academy.
2. Abre Learner Lab.
3. Presiona Start Lab y espera a que el indicador aparezca en verde.
4. Abre la consola de AWS desde el enlace del laboratorio.
5. Selecciona la región que te proporciona el laboratorio y mantenla durante todo el despliegue.

[AWS: Introducción a AWS Academy y sus herramientas | mcherrera](https://images.openai.com/static-rsc-4/YosJPv6TW_p3zRclPcA3AAB8ZIkDv6_mPzUhI7RjFS-JQga2Dz-3pVMUjl3fPMPMpDGHnyhOQYPppeuG0ImTIK8DyQnwVgYujUugjJRgNlLcN5t6L8mzJTtzhDYsENDW4rd16w8uPkp-qyIxYNjIFP3SwskGuoTbGQhkE2a0O_k?purpose=inline)

[Launching Your First EC2 Web Server on AWS | by Bentsiwa | Jan, 2026 | Medium](https://images.openai.com/static-rsc-4/YRW75yq3GflhJ6j4LinKor2kCtFKCH6L14U3iHUwyRzN9zpgjUCCrp0I7Awo_Q_476xXOmnDPzmOwfic1U1QROJAPafDx5vbGUQcsGi8iITXgBQ3AUF_kptyyHiV3M6EYLyrapYanBGrwwkTZ7Tc23_YQq1Z_jLN70vRMPID4kE?purpose=inline)

No cambies de región después de crear los recursos: las instancias y redes creadas en una región no aparecen automáticamente en otra.

### 1.2. Crear un grupo de seguridad

En la consola:

1. Busca EC2.
2. En el menú lateral, entra en Network & Security → Security Groups.
3. Presiona Create security group.
4. Configúralo así:

| Campo               | Valor                              |
| ------------------- | ---------------------------------- |
| Security group name | `SG-DFSha`                         |
| Description         | `Acceso SSH para el clúster DFSha` |
| VPC                 | Default VPC                        |

En Inbound rules, agrega únicamente esta regla:

| Type | Protocol | Port | Source |
| ---- | -------- | ---- | ------ |
| SSH  | TCP      | 22   | My IP  |

No abras los puertos 7000, 7001 ni 7100 a Internet. Los servicios se comunicarán internamente por la red de Docker y tú administrarás el clúster mediante SSH.

AWS recomienda restringir el acceso SSH a las direcciones IP necesarias y evitar exponerlo a cualquier origen.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Elastic Compute Cloud+1



[การใช้ Mountpoint for Amazon S3 เพื่อเข้าถึง S3 จาก EC2 เสมือนเป็นไดเร็กทอรีในเครื่อง | DevelopersIO](https://images.openai.com/static-rsc-4/aHaOqarDemTg283tqJMboY0evFf3suwCTOW6D4lxl6AIJhRkWwIsxCtMpJD__jknrj3ORcAhxe-qIhAJj4RzqQLUTnOqv1aMoqcn4c1hJIzdAKOXaXA5QHRbjddiiAOCMCeYYi6kNpINrKKrFt8iQNubs1so-lD4qAIcnfN0Zk8?purpose=inline)

[dev.classmethod.jp](https://dev.classmethod.jp/articles/Mountpoint-for-Amazon-S3-th/?utm_source=chatgpt.com)

Presiona Create security group.

### 1.3. Crear un par de claves

1. En el menú de EC2, selecciona Key Pairs.
2. Presiona Create key pair.
3. Nombre: `dfsha-key`.
4. Tipo: RSA.
5. Formato: `.pem`.
6. Presiona Create key pair.

Se descargará un archivo llamado `dfsha-key.pem`.



Importante

&#x20;Guarda ese archivo en un lugar seguro. No lo subas al repositorio ni lo compartas.



### 1.4. Crear la instancia EC2

En EC2, selecciona Instances → Launch instances.

Configura la instancia de la siguiente manera:

Configuración recomendada

| Campo                 | Selección                      |
| --------------------- | ------------------------------ |
| Name                  | `DFSha-Server`                 |
| AMI                   | Ubuntu Server 24.04 LTS        |
| Architecture          | 64-bit x86                     |
| Instance type         | `t3.xlarge` si está disponible |
| Key pair              | `dfsha-key`                    |
| VPC                   | Default VPC                    |
| Subnet                | Subnet pública predeterminada  |
| Auto-assign public IP | Enable                         |
| Security group        | `SG-DFSha`                     |
| Storage               | 30 GiB, gp3                    |

Recursos recomendados

# 4

vCPU

# 16 GB

RAM

Si AWS Academy no permite t3.xlarge, utiliza la instancia de 2 vCPU y 8 GB de RAM más cercana que permita el laboratorio. En ese caso, evita las pruebas de rendimiento con archivos de 1 GB.

¿Por qué estos recursos? Tendremos nueve servicios ejecutándose simultáneamente: tres servidores de metadatos, cinco nodos de datos y un cliente. Además, necesitaremos recursos para construir las imágenes y ejecutar pruebas.

Si el laboratorio tiene restricciones de tipos de instancia, elige una alternativa disponible con al menos 2 vCPU y 8 GB de RAM. No todos los laboratorios de AWS Academy permiten los mismos tipos de instancia.

En almacenamiento, los 30 GiB permiten alojar el sistema operativo, las imágenes Docker, los volúmenes de los nodos y los archivos de prueba.

Finalmente, presiona Launch instance.

[How to Create an EC2 instance in AWS free tier: A Step-by-Step Guide - DEV Community](https://images.openai.com/static-rsc-4/8perLnsr1rGVgjs8qQHXKKaNTHn-bRFgNIVpj6MYRIk2WPTUKD644b9iIo6NdCKE5SAyUMfNtc1urKgDayeRA9J9U5TBlFX6MwwFqtEs9Ty5j0jJVG69HNwobMdIBLb1oS5KHWInYvFts0XsPm_6kessRfwPr5dhLTt6qz9p9EU?purpose=inline)

[Beginners guide to AWS EC2 - DEV Community](https://images.openai.com/static-rsc-4/JkAH60Iz4sDXUDjI-iFiM72tm4lKNbt6fGO1i4rjduogJKlqbWdxwIYg_ew2M5gy10vyshqCF5Q3cCGVwZYjvyfZ9EUVa6U1dICTW-6CDUcaTzk3oOfKGxPQUyxgL2CVDhJDqIg9-WsTAtejReBsjb4cTnsckQbFABBC5gCku9Q?purpose=inline)

### 1.5. Conectarse a la instancia

Cuando la instancia esté en estado `Running` y las comprobaciones de estado estén aprobadas:

1. Selecciónala.
2. Copia su Public IPv4 address.
3. Abre una terminal en tu computador.

Como estás utilizando Linux, ejecuta:

```
chmod 400 ~/Descargas/dfsha-key.pem
```

Conéctate:

```
ssh -i ~/Descargas/dfsha-key.pem ubuntu@IP_PUBLICA
```

Reemplaza `IP_PUBLICA` por la dirección que aparece en AWS.

Si la conexión funciona, tendrás una terminal de Ubuntu dentro de EC2.



Evidencia 1

&#x20;Captura la instancia en estado `Running`, mostrando su tipo, IP pública y zona de disponibilidad. No muestres información sensible de acceso.



## 2. Instalar y configurar Docker en EC2

Ahora vamos a preparar el servidor para ejecutar el proyecto.

### 2.1. Actualizar Ubuntu

Dentro de la instancia EC2:

```
sudo apt update
sudo apt upgrade -y
```

Instala las herramientas necesarias:

```
sudo apt install -y \
    git \
    curl \
    wget \
    unzip \
    openssl \
    ca-certificates \
    gnupg \
    lsb-release \
    python3 \
    python3-venv \
    python3-pip
```

### 2.2. Instalar Docker Engine

Utilizaremos el repositorio oficial de Docker para Ubuntu.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.docker.com\&sz=32)

Docker Docs



Ejecuta:

```
sudo install -m 0755 -d /etc/apt/keyrings
```

Descarga la clave:

```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
-o /etc/apt/keyrings/docker.asc
```

Configura los permisos:

```
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Agrega el repositorio:

```
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Actualiza los paquetes:

```
sudo apt update
```

Instala Docker y Docker Compose:

```
sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

Inicia Docker:

```
sudo systemctl enable --now docker
```

### 2.3. Permitir utilizar Docker sin sudo

Ejecuta:

```
sudo usermod -aG docker $USER
```

Después:

```
newgrp docker
```

Esto permite ejecutar los scripts del repositorio, que invocan `docker compose` directamente. Ten presente que pertenecer al grupo Docker otorga privilegios equivalentes a root en el servidor; úsalo únicamente en esta instancia de laboratorio.

Verifica la instalación:

```
docker --version
docker compose version
```

Ejecuta la imagen de prueba:

```
docker run hello-world
```

Si aparece el mensaje `Hello from Docker!`, Docker está funcionando.



Evidencia 2

&#x20;Captura la terminal mostrando las versiones de Docker y Docker Compose, además de la ejecución exitosa de `hello-world`.



## 3. Clonar el repositorio DFSha

Ahora vamos a descargar el proyecto que contiene los componentes que analizamos.

### 3.1. Clonar desde GitHub

En la instancia EC2:

```
git clone URL_DE_TU_REPOSITORIO
```

Por ejemplo:

```
git clone https://github.com/USUARIO/DFSha.git
```

Entra en el directorio:

```
cd DFSha
```

Si tu repositorio tiene otro nombre, utiliza el directorio que se haya creado.

Verifica su contenido:

```
ls -la
```

Deberías encontrar:

```
client/
data-node/
metadata-server/
proto/
shared/
scripts/
tests/
docs/
docker-compose.yml
requirements.txt
README.md
```

Si el repositorio es privado, tendrás que autenticarte con el mecanismo de acceso que utilicen en GitHub. No introduzcas tokens personales directamente en los comandos que puedan quedar guardados en el historial.

### 3.2. Verificar los archivos de configuración

Ejecuta:

```
docker compose config
```

Esto permite verificar que Docker Compose pueda interpretar el archivo de configuración.

Si no muestra errores, podemos continuar.

### 3.3. Preparar los scripts

Ejecuta:

```
chmod +x scripts/*.sh
```

Verifica que tienes espacio suficiente:

```
df -h
free -h
nproc
```

Evidencia 3: captura el árbol del repositorio y la salida de `docker compose config` sin errores.

## 4. Configurar DFSha

Esta parte es especialmente importante porque el repositorio genera automáticamente los certificados y secretos que necesitan los servicios.

### 4.1. Generar certificados y secretos

El proyecto tiene un script que realiza esta preparación:

```
scripts/start_cluster.sh
```

Este script realiza tres tareas principales:

1. Genera una autoridad certificadora privada (CA).
2. Genera un certificado para cada uno de los componentes que utilizan mTLS.
3. Genera un archivo `.env` con secretos aleatorios y claves maestras para los cinco nodos de datos.

En concreto, genera certificados para:

```
ms1
ms2
ms3
dn1
dn2
dn3
dn4
dn5
client
```

No necesitas generar manualmente cada certificado.

### 4.2. Iniciar el clúster

Ejecuta:

```
scripts/start_cluster.sh
```

Este comando construye las imágenes Docker e inicia los servicios mediante:

```
docker compose up -d --build
```

El primer arranque puede tardar algunos minutos porque necesita descargar la imagen de Python, instalar dependencias y generar los stubs de gRPC.

[Disable the new fancy TUI in favor of old simple build/run output · Issue #8753 · docker/compose · GitHub](https://images.openai.com/static-rsc-4/w9dFkvV_lSCyuy39tc7uYZ5ofGZEsbhgC6VXEM_JU4xfmUSuma52GvNvf3WSDfe0D62-w_gCnqrtvVGiNQzMUk1l-y7oKZY3EZwRTFPllaFv8XrvpLCmYVfWGscQDn5dHeGvNdNpuf3W407qoK4SASXm2i_t-x1oHtbrwKxQywo?purpose=inline)

[github.com](https://github.com/docker/compose/issues/8753?utm_source=chatgpt.com)

### 4.3. Comprobar que todos los contenedores están activos

Ejecuta:

```
docker compose ps
```

Debes encontrar estos servicios:

| Contenedor | Función   | Estado esperado |
| ---------- | --------- | --------------- |
| ms1        | Metadatos | Up              |
| ms2        | Metadatos | Up              |
| ms3        | Metadatos | Up              |
| dn1        | Datos     | Up              |
| dn2        | Datos     | Up              |
| dn3        | Datos     | Up              |
| dn4        | Datos     | Up              |
| dn5        | Datos     | Up              |
| client     | Cliente   | Up              |



Evidencia 4

&#x20;Esta es una de las capturas más importantes: debe mostrar los nueve contenedores ejecutándose.



### 4.4. Revisar los registros de los servicios

Ejecuta:

```
docker compose logs --tail=50 ms1 ms2 ms3
```

Y después:

```
docker compose logs --tail=50 dn1 dn2 dn3 dn4 dn5
```

Busca mensajes que indiquen que los servicios están listos, como:

```
MS ... listo
DN ... listo
```

También puedes observar continuamente los registros:

```
docker compose logs -f
```

Presiona `Ctrl + C` para salir de la visualización sin detener los contenedores.

Si algún servicio aparece como `Restarting` o `Exited`, no continúes con las pruebas: primero hay que revisar sus registros.

## 5. Registrar un usuario y ejecutar las primeras operaciones

Ya tenemos el clúster iniciado. Ahora vamos a utilizar el cliente.

### 5.1. Registrar un usuario

Ejecuta:

```
docker compose exec client python client/main.py register demo
```

El sistema solicitará una contraseña.

Utiliza una contraseña de al menos ocho caracteres.

Después, inicia sesión:

```
docker compose exec client python client/main.py login demo
```

El cliente conservará la sesión en su volumen persistente.

### 5.2. Crear directorios

Ejecuta:

```
docker compose exec client python client/main.py mkdir documentos
```

Después:

```
docker compose exec client python client/main.py mkdir documentos/pruebas
```

Comprueba el contenido:

```
docker compose exec client python client/main.py ls
```

Deberías observar el directorio `documentos`.

Entra en él:

```
docker compose exec client python client/main.py cd documentos
```

Y consulta el directorio actual:

```
docker compose exec client python client/main.py pwd
```

El resultado debería corresponder a:

```
/users/demo/documentos
```

Evidencia 5: captura donde se observe el usuario autenticado, la creación de directorios y el listado del contenido.

### 5.3. Crear un archivo de prueba

Vamos a generar un archivo de 20 MB dentro del contenedor cliente:

```
docker compose exec client sh -c \
'head -c 20971520 /dev/urandom > /tmp/prueba.bin'
```

Esto genera datos binarios aleatorios.

Ahora súbelo a DFSha:

```
docker compose exec client \
python client/main.py put /tmp/prueba.bin pruebas.bin
```

Deberías obtener un mensaje indicando el tamaño transferido y la velocidad de escritura.

### 5.4. Descargar el archivo

Ejecuta:

```
docker compose exec client \
python client/main.py get pruebas.bin /tmp/recuperado.bin
```

Compara los hashes:

```
docker compose exec client sh -c \
'sha256sum /tmp/prueba.bin /tmp/recuperado.bin'
```



Evidencia 6

&#x20;Captura los dos hashes idénticos y las salidas de `put` y `get`.



Esto demuestra que el archivo fue transferido al sistema distribuido y posteriormente recuperado sin alterar su contenido.

## 6. Demostrar que los bloques están distribuidos y replicados

Esta prueba es fundamental porque demuestra que DFSha no está funcionando como un almacenamiento centralizado.

Vamos a consultar los metadatos directamente desde el cliente.

### 6.1. Consultar las ubicaciones de los bloques

Ejecuta:

```
docker compose exec client python -c '
from dfs_client import DFSClient

c = DFSClient()
path = c.abspath("pruebas.bin")
info = c.meta.get_block_locations(path)

print("Archivo:", path)
print("Versión:", info.file_version)
print("Tamaño:", info.total_size)
print("Número de bloques:", len(info.blocks))

for b in info.blocks:
    print("Bloque:", b.block_id)
    print("Réplicas:", list(b.replicas))
    print("Cantidad:", len(b.replicas))
    print("---")
'
```

El archivo de 20 MB debe dividirse en tres bloques:

- Bloque 1: 8 MiB.
- Bloque 2: 8 MiB.
- Bloque 3: el contenido restante.

Cada bloque debe contar con tres réplicas en nodos diferentes.



Evidencia 7

&#x20;Captura la salida que muestre los tres bloques y sus tres ubicaciones de almacenamiento.



### 6.2. Demostrar que se utilizan varios nodos

En la misma salida, observa las direcciones de las réplicas.

Deberías encontrar direcciones similares a:

```
dn1:7001
dn2:7001
dn3:7001
dn4:7001
dn5:7001
```

No todos los bloques tienen que utilizar exactamente la misma combinación de nodos. Lo importante es que las tres réplicas de cada bloque sean distintas y que los bloques se distribuyan entre varios nodos.

## 7. Demostrar recuperación ante fallos de nodos de datos

Ahora vamos a provocar un fallo controlado.

La idea es detener uno de los nodos que almacena una réplica y demostrar que el archivo continúa disponible.

### 7.1. Identificar una réplica

Utiliza el comando del punto anterior y toma nota de uno de los nodos que almacena el primer bloque.

Por ejemplo, supongamos que aparece:

```
Réplicas:
dn1:7001
dn3:7001
dn5:7001
```

En ese caso, vamos a detener `dn1`.

### 7.2. Detener el nodo

Ejecuta:

```
docker compose stop dn1
```

Comprueba su estado:

```
docker compose ps
```

El nodo debe aparecer detenido.



Evidencia 8

&#x20;Captura el clúster con un nodo de datos detenido.



### 7.3. Leer el archivo con el nodo detenido

Ejecuta:

```
docker compose exec client \
python client/main.py get pruebas.bin /tmp/recuperado_fallo.bin
```

Comprueba su integridad:

```
docker compose exec client sh -c \
'sha256sum /tmp/prueba.bin /tmp/recuperado_fallo.bin'
```

Los hashes deben coincidir.

Esto demuestra que el cliente pudo utilizar otra réplica cuando el nodo original no estaba disponible.

### 7.4. Esperar la recuperación de R=3

El sistema considera sospechoso un nodo después de 10 segundos y caído después de 30 segundos sin heartbeat.

Por tanto, espera aproximadamente 45–60 segundos después de detener el nodo.

Vuelve a consultar las ubicaciones:

```
docker compose exec client python -c '
from dfs_client import DFSClient

c = DFSClient()
info = c.meta.get_block_locations(c.abspath("pruebas.bin"))

for b in info.blocks:
    print(b.block_id, list(b.replicas), len(b.replicas))
'
```

El objetivo es comprobar que los bloques afectados recuperaron tres réplicas y que el nodo detenido dejó de figurar como una réplica disponible.

El tiempo puede variar según el momento en que se detecte el fallo y se complete la copia.

### 7.5. Volver a iniciar el nodo

Ejecuta:

```
docker compose start dn1
```

Después:

```
docker compose ps
```

Evidencia 9: captura la lectura correcta durante el fallo y la recuperación posterior de las tres réplicas.

## 8. Demostrar tolerancia a fallos de los servidores de metadatos

Esta prueba es distinta a la anterior: ahora vamos a detener el líder de Raft.

### 8.1. Identificar el líder

Ejecuta:

```
docker compose exec client python -c '
from dfs_client import DFSClient
c = DFSClient()
c.meta.list_dir(c.abspath("."))
print("Líder detectado:", c.meta.router.leader)
'
```

El resultado debería mostrar una dirección similar a:

```
ms2:7100
```

En ese caso, `ms2` es el servidor que debes detener.

### 8.2. Detener el líder

Si el líder es `ms2`, ejecuta:

```
docker compose stop ms2
```

No detengas dos servidores de metadatos: Raft necesita una mayoría de dos de los tres servidores para continuar.

### 8.3. Ejecutar una operación durante la elección

Intenta crear un directorio:

```
docker compose exec client \
python client/main.py mkdir recuperacion_raft
```

El cliente debe volver a contactar los servidores, detectar al nuevo líder y completar la operación si se recupera el quorum.

Verifica:

```
docker compose exec client \
python client/main.py ls
```

Deberías encontrar `recuperacion_raft`.

### 8.4. Reiniciar el servidor detenido

```
docker compose start ms2
```

Comprueba nuevamente:

```
docker compose ps
```

Evidencia 10: captura del líder identificado, su detención y la operación exitosa después de la elección de un nuevo líder.

## 9. Demostrar RF3: open, read, write, close y locks

Para validar las operaciones de archivo abierto vamos a utilizar la biblioteca de DFSha.

### 9.1. Probar escritura y lectura

Ejecuta:

```
docker compose exec -T client sh -c \
'printf "Contenido de prueba RF3" | python client/main.py write documentos/rf3.txt'
```

Después:

```
docker compose exec client \
python client/main.py read documentos/rf3.txt 0 23
```

Debe devolver el contenido que escribimos.

### 9.2. Probar que close publica la versión

El comportamiento esperado es que los datos escritos durante una sesión no sean visibles hasta confirmar el archivo mediante `close`.

Esta propiedad se verifica específicamente con la prueba:

```
test_escritura_no_visible_hasta_close
```

La prueba abre dos sesiones, verifica que el contenido anterior sigue siendo visible antes de cerrar la escritura y después comprueba que la nueva versión aparece tras el `close`.

### 9.3. Probar locks y concurrencia

Los locks se verifican con dos clientes diferentes:

- Un cliente solicita un bloqueo exclusivo.
- El segundo cliente intenta obtener el mismo bloqueo.
- El sistema debe rechazar el segundo intento con `BLOCK_LOCKED`.
- Cuando el primero libera el bloqueo, el segundo puede adquirirlo.

También se comprueba que dos escrituras que parten de la misma versión no puedan sobrescribirse silenciosamente. Una de ellas debe fallar con `VERSION_CONFLICT`.

Estas pruebas se ejecutarán mediante `pytest` en el siguiente apartado.

## 10. Ejecutar la batería completa de pruebas automatizadas

Hay una particularidad importante del repositorio que debemos respetar.

Las pruebas de integración están preparadas para ejecutarse en modo local, porque utilizan direcciones `127.0.0.1`, controlan procesos locales y algunas acceden directamente a los archivos del almacenamiento. Por ello, no vamos a ejecutar `pytest` directamente contra los contenedores Docker: el cliente de Docker Compose utiliza direcciones internas como `dn1:7001`, que no son accesibles directamente desde el entorno Python del host.

La forma más sencilla y reproducible es:

- Utilizar Docker Compose para demostrar el despliegue y las operaciones del clúster.
- Ejecutar la batería automatizada en modo local sobre la misma instancia EC2, con los mismos tres servidores de metadatos y cinco nodos de datos.

Esto permite validar las pruebas sin modificar la lógica del proyecto ni falsear resultados.

### 10.1. Detener el clúster Docker Compose

Primero termina la demostración con Docker:

```
docker compose down
```

No utilices `-v`: esa opción eliminaría los volúmenes persistentes de Docker.

### 10.2. Preparar el entorno Python

Desde la raíz del repositorio:

```
python3 -m venv .venv
```

Activa el entorno:

```
source .venv/bin/activate
```

Instala las dependencias:

```
pip install --upgrade pip
pip install -r requirements-dev.txt
```

### 10.3. Iniciar el clúster en modo local

Ejecuta:

```
scripts/start_cluster.sh --local
```

Este comando inicia:

| Componente | Dirección      |
| ---------- | -------------- |
| MS1        | 127.0.0.1:7100 |
| MS2        | 127.0.0.1:7101 |
| MS3        | 127.0.0.1:7102 |
| DN1        | 127.0.0.1:7201 |
| DN2        | 127.0.0.1:7202 |
| DN3        | 127.0.0.1:7203 |
| DN4        | 127.0.0.1:7204 |
| DN5        | 127.0.0.1:7205 |

En este modo se ejecutan procesos Python independientes, sin Docker y sin TLS.

Configura las variables de entorno:

```
export DFSHA_LOCAL=1
export DFSHA_RUN=/tmp/dfsha_run

export PYTHONPATH=$PWD:$PWD/proto/generated:$PWD/client

export DFSHA_MS=127.0.0.1:7100,127.0.0.1:7101,127.0.0.1:7102
```

Comprueba que los procesos se iniciaron:

```
ls -l /tmp/dfsha_run
```

Deberías encontrar archivos de registros y PID para los servidores y nodos.

### 10.4. Ejecutar todas las pruebas

Ahora sí:

```
python -m pytest tests -v --tb=short
```

Esto ejecuta las categorías del repositorio:

| Categoría      | Qué demuestra                                                             |
| -------------- | ------------------------------------------------------------------------- |
| `functional/`  | Gestión de directorios, archivos, lectura y escritura.                    |
| `concurrency/` | Locks, leases y conflictos de versiones.                                  |
| `replication/` | Tres réplicas y distribución de bloques.                                  |
| `failures/`    | Lectura y escritura con nodos caídos, re-replicación y elección de líder. |
| `performance/` | Transferencia y consulta de metadatos.                                    |

La prueba de integridad que modifica directamente un archivo almacenado está habilitada en este modo mediante `DFSHA_LOCAL=1`.

El resultado esperado es que las pruebas terminen en estado `PASSED`, salvo algún `SKIPPED` justificado por las condiciones de ejecución. Si alguna falla, conserva la salida completa para identificar el problema antes de repetirla.



Evidencia 11

&#x20;Captura el resumen final de `pytest`, incluyendo el número de pruebas aprobadas, fallidas y omitidas.



### 10.5. Ejecutar una prueba de rendimiento de 100 MB

El proyecto permite cambiar el tamaño del archivo de la prueba mediante `DFSHA_PERF_MB`.

Ejecuta:

```
DFSHA_PERF_MB=100 python -m pytest tests/performance -s -v
```

El resultado debe mostrar algo parecido a:

```
RESULTADO {
    "MB": 100,
    "put_MBps": ...,
    "get_MBps": ...,
    "metadatos_ms": ...
}
```

Los valores reales dependerán de la instancia EC2 y de las condiciones de ejecución. No copies los resultados locales anteriores del informe como si fueran mediciones de AWS.

Si la instancia tiene suficiente memoria y espacio, posteriormente puedes probar con 500 MB o 1000 MB. Para la primera validación recomiendo 100 MB.

Evidencia 12: captura de la salida `RESULTADO`, con los valores obtenidos realmente en EC2.

### 10.6. Detener el clúster de pruebas

Cuando termines:

```
scripts/stop_cluster.sh --local
```

Y desactiva el entorno virtual:

```
deactivate
```

## 11. Capturas que debes incluir en el punto 8 del informe

No necesitas incluir todas las terminales ni cada comando. Te recomiendo organizar las evidencias en el orden siguiente.

Lista de evidencias para el informe

0 de 13 evidencias listas

0%

8.1 — Instancia EC2 creada

Estado Running, tipo de instancia y región.

8.2 — Configuración de seguridad

Grupo SG-DFSha con SSH limitado a My IP.

8.3 — Entorno instalado

Ubuntu, Docker y Docker Compose funcionando.

8.4 — Repositorio preparado

Estructura del proyecto y configuración validada.

8.5 — Clúster desplegado

Los nueve contenedores en estado Up.

8.6 — Autenticación y RF1

Registro, inicio de sesión y operaciones de directorios.

8.7 — Transferencia RF2

Put, get y comparación de hashes.

8.8 — Distribución y replicación

Bloques con tres réplicas en nodos distintos.

8.9 — Recuperación de nodos

Lectura durante una caída y restauración de R=3.

8.10 — Tolerancia a fallos de metadatos

Elección de nuevo líder y operación exitosa.

8.11 — Pruebas RF3 y concurrencia

Lectura, escritura, cierre, locks y versiones.

8.12 — Pruebas automatizadas

Resumen final de pytest.

8.13 — Rendimiento

Resultados de transferencia y metadatos en EC2.

## 12. Limpieza y control de costos

Cuando termines las pruebas:

1\. Detén los contenedores:

```
docker compose down
```

2. Detén la instancia EC2 si todavía vas a necesitarla y quieres conservar su configuración y almacenamiento.
3. Cuando hayas terminado definitivamente el proyecto, termina la instancia y elimina los recursos que ya no necesites.

Recuerda que detener una instancia no equivale a eliminar todos los recursos asociados. Revisa especialmente los volúmenes EBS y las direcciones IP públicas reservadas. AWS recomienda considerar por separado la persistencia del almacenamiento y el ciclo de vida de las instancias.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Elastic Compute Cloud+1



## 13. Referencias oficiales para la configuración

- [AWS — Primeros pasos con EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html?utm_source=chatgpt.com).
- [AWS — Crear un grupo de seguridad](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-security-group.html?utm_source=chatgpt.com).
- [Docker — Instalar Docker Engine en Ubuntu](https://docs.docker.com/engine/install/ubuntu/?utm_source=chatgpt.com).
- [AWS — Buenas prácticas para EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-best-practices.html?utm_source=chatgpt.com).

## Cómo vamos a trabajar para evitar errores

Mi recomendación es no ejecutar todos los pasos de una sola vez. Primero crea la instancia y deja Docker funcionando. Después desplegamos DFSha y revisamos sus logs antes de empezar a probar.

Comienza por el punto 1 y el punto 2. Cuando tengas la instancia EC2 conectada y Docker funcionando, compárteme la salida de:

```
docker --version
docker compose version
docker run hello-world
```

Con eso podremos comprobar que el servidor está listo antes de desplegar los nueve componentes.