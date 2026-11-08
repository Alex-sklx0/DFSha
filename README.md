# DFSha: sistema de archivos distribuido

Cliente/servidor con servicio distribuido entre servidores (S2S). Python, gRPC y Docker.
3 servidores de metadatos (Raft), 5 nodos de datos, bloques de 8 MiB, R=3 y W=2, SHA-256,
AES-256-GCM, JWT y mTLS.

## Estructura
    client/           cli.py, dfs_client.py (open/read/write/close, put/get, lock), metadata_client.py, data_node_client.py
    data-node/        data_service.py, block_storage.py (cifrado), integrity.py, replication.py, heartbeat.py
    metadata-server/  metadata_service.py, namespace_manager.py, file_manager.py, block_manager.py,
                      replication_manager.py, user_manager.py, storage/metadata_repository.py (estado Raft)
    proto/            auth.proto, metadata.proto, data_node.proto (generated/ se crea con scripts/generate_proto.sh)
    shared/           config, constants, exceptions (códigos 6.6), security, utils
    scripts/          start_cluster.sh, stop_cluster.sh, clean_storage.sh, generate_proto.sh, generate_certs.sh
    tests/            functional, concurrency, failures, replication, performance (pytest)

## Arranque con Docker (despliegue de AWS Academy)
    scripts/start_cluster.sh        # genera certs/ (CA + un cert por nodo, mTLS) y .env (secretos), docker compose up
    docker compose exec client python client/main.py register david      # contraseña >= 8 caracteres
    docker compose exec client python client/main.py login david
    docker compose exec client python client/main.py mkdir datos
    docker compose exec client python client/main.py put /ruta/archivo.bin datos/archivo.bin
Para copiar archivos al contenedor: `docker compose cp archivo.bin client:/tmp/`.
En AWS: una EC2 por componente (o varias por VM), subred privada, grupo de seguridad que solo abra 7000, 7001 y 7100 entre nodos;
copia `certs/<nodo>` a cada VM y ajusta `docker-compose.yml` por VM (o usa los mismos servicios con `DFSHA_MS` apuntando a las IP privadas).

## Arranque local sin Docker (desarrollo)
    pip install -r requirements-dev.txt
    scripts/start_cluster.sh --local          # 3 MS (127.0.0.1:7100-7102) + 5 DN (7201-7205), sin TLS salvo que exportes TLS_DIR
    export PYTHONPATH=$PWD:$PWD/proto/generated:$PWD/client DFSHA_MS=127.0.0.1:7100,127.0.0.1:7101,127.0.0.1:7102
    python3 client/main.py register david && python3 client/main.py login david
    scripts/stop_cluster.sh --local ; scripts/clean_storage.sh --local

## CLI
`register login ls cd pwd mkdir rmdir rm put append get read write lock unlock`
(`read <ruta> <offset> <longitud>`; `write <ruta>` lee de stdin y hace open, write y close;
`lock <ruta> <bloque|-1> <X|S>`).

## API de la biblioteca (RF3)
    from dfs_client import DFSClient
    c = DFSClient(["ms1:7100"]); c.login("david", "secreto123")
    with c.open("datos/nota.txt", "w") as h: h.write(b"hola")     # visible solo al hacer close()
    with c.open("datos/nota.txt", "a") as h: h.write(b" mundo")    # anexar: lock exclusivo con lease renovable
    with c.open("datos/nota.txt") as h: h.seek(0); print(h.read(4))

## Pruebas
    export DFSHA_LOCAL=1                      # (solo si usas el clúster --local; con Docker no)
    python3 -m pytest tests -v                # tests/failures tarda ~2 min (espera la re-replicación)
    DFSHA_PERF_MB=1000 python3 -m pytest tests/performance -s
`failures` detiene nodos con `docker compose stop dnN` (o mata el proceso en modo local).

## Correspondencia con el informe
RF1: namespace_manager y user_manager. RF2: dfs_client.put/get y el pipeline en data_service/replication.
RF3: FileHandle (open/read/write/close) y file_manager (locks, CAS por versión).
Replicación y recuperación: replication_manager y heartbeat.py. Integridad: integrity.py y data_node_client.
Seguridad: shared/security.py (argon2, JWT, capability tokens, HKDF/AES-GCM, mTLS).

## Limitaciones declaradas (sección 7.9)
- Raft entre MS (PySyncObj) se protege con contraseña AES de PySyncObj y la red privada, no con mTLS.
- El registro de DN (heartbeats) vive en la memoria del líder; se reconstruye en segundos tras una elección
  (durante ~3 s un líder nuevo responde `INSUFFICIENT_REPLICAS` a una asignación; el cliente reintenta solo).
- Los DN reciben el bloque completo, lo guardan y lo reenvían (store-and-forward), no byte a byte.
- Tokens JWT/capacidad con HMAC compartido; JWT de 15 min sin refresh token (`JWT_TTL`).
- Las lecturas de metadatos se sirven en el líder sin ReadIndex.
- Sin cifrado extremo a extremo ni KMS (la clave maestra se entrega por `MASTER_KEY_HEX`); sin rename atómico entre
  directorios; sin cuotas; sin modificación parcial dentro de un bloque (solo crear, reemplazar o anexar).
