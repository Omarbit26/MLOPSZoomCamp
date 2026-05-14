# MLflow Local Stack — Escenario 3 (sin AWS)

Replica del **Escenario 3 de MLflow** (múltiples data scientists, múltiples modelos) usando servicios locales con Docker en lugar de AWS.

## Arquitectura

```
┌─────────────────────────────────────────────────────┐
│                   Docker Network                     │
│                  (mlflow-net)                        │
│                                                      │
│   ┌─────────────┐        ┌─────────────────────┐    │
│   │  PostgreSQL  │◄──────│   MLflow Server      │    │
│   │  :5432       │       │   :5000              │    │
│   │  (metadata)  │       │   (tracking server)  │    │
│   └─────────────┘        └─────────────────────┘    │
│                                    ▲                 │
│   ┌─────────────┐                  │                 │
│   │    MinIO    │◄─────────────────┘                 │
│   │  :9000 API  │                                    │
│   │  :9001 UI   │                                    │
│   │ (artifacts) │                                    │
│   └─────────────┘                                    │
└─────────────────────────────────────────────────────┘
```

| Componente | Equivalente AWS | Descripción |
|---|---|---|
| MLflow Server | EC2 | Tracking server accesible por URL |
| PostgreSQL | RDS | Backend store: guarda runs, métricas, parámetros |
| MinIO | S3 | Artifact store: guarda modelos, plots, archivos |

## Requisitos

- Docker Engine 20+
- Docker Compose v2+
- Python 3.8+ (para los scripts de experimentos)

Verificar instalación:
```bash
docker --version
docker compose version
```

## Estructura del proyecto

```
mlflow-local-stack/
├── compose.yaml        # Definición de servicios Docker
├── minio-data/         # Datos persistentes de MinIO (artifact store)
├── postgres-data/      # Datos persistentes de PostgreSQL (backend store)
└── README.md
```

## Instalación y uso

### 1. Clonar / crear el proyecto

```bash
mkdir mlflow-local-stack
cd mlflow-local-stack
mkdir -p minio-data postgres-data
```

### 2. Crear el archivo `compose.yaml`

```yaml
services:

  postgres:
    image: postgres:15
    container_name: mlflow_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: mlflow123
      POSTGRES_DB: mlflow_db
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    networks:
      - mlflow-net

  minio:
    image: minio/minio:latest
    container_name: mlflow_minio
    restart: unless-stopped
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - ./minio-data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - mlflow-net

  minio-setup:
    image: minio/mc:latest
    container_name: mlflow_minio_setup
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      sleep 5;
      mc alias set local http://minio:9000 minioadmin minioadmin123;
      mc mb --ignore-existing local/mlflow-artifacts;
      exit 0;
      "
    networks:
      - mlflow-net

  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    container_name: mlflow_server
    restart: unless-stopped
    depends_on:
      - postgres
      - minio
    environment:
      MLFLOW_S3_ENDPOINT_URL: http://minio:9000
      AWS_ACCESS_KEY_ID: minioadmin
      AWS_SECRET_ACCESS_KEY: minioadmin123
    ports:
      - "5000:5000"
    command: >
      mlflow server
      --backend-store-uri postgresql://mlflow:mlflow123@postgres:5432/mlflow_db
      --artifacts-destination s3://mlflow-artifacts
      --host 0.0.0.0
      --port 5000
    networks:
      - mlflow-net

networks:
  mlflow-net:
    driver: bridge
```

> **Nota:** Si ya tienes PostgreSQL instalado localmente en el puerto 5432, no expongas ese puerto en el compose (ya está configurado así arriba). MLflow se comunica con PostgreSQL por la red interna Docker.

### 3. Levantar los servicios

```bash
docker compose up -d
```

La primera vez descargará las imágenes (~2-3 minutos). Verificar que todo esté corriendo:

```bash
docker compose ps
```

Salida esperada:
```
NAME              IMAGE                          STATUS
mlflow_minio      minio/minio:latest             Up
mlflow_postgres   postgres:15                    Up
mlflow_server     ghcr.io/mlflow/mlflow:latest   Up
```

### 4. Acceder a las interfaces

| Servicio | URL | Usuario | Contraseña |
|---|---|---|---|
| MLflow UI | http://localhost:5000 | — | — |
| MinIO Console | http://localhost:9001 | `minioadmin` | `minioadmin123` |

## Conectar Python al servidor

### Instalar dependencias

```bash
pip install mlflow boto3 psycopg2-binary scikit-learn
```

### Configurar el cliente MLflow

```python
import os
import mlflow

# Apuntar al servidor local
os.environ["MLFLOW_S3_ENDPOINT_URL"] = "http://localhost:9000"
os.environ["AWS_ACCESS_KEY_ID"] = "minioadmin"
os.environ["AWS_SECRET_ACCESS_KEY"] = "minioadmin123"

mlflow.set_tracking_uri("http://localhost:5000")
```

### Script de ejemplo completo

```python
import mlflow
import mlflow.sklearn
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
import os

os.environ["MLFLOW_S3_ENDPOINT_URL"] = "http://localhost:9000"
os.environ["AWS_ACCESS_KEY_ID"] = "minioadmin"
os.environ["AWS_SECRET_ACCESS_KEY"] = "minioadmin123"

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("iris-experiment")

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

configs = [
    {"C": 0.1, "max_iter": 100},
    {"C": 1.0, "max_iter": 200},
    {"C": 10.0, "max_iter": 300},
]

for cfg in configs:
    with mlflow.start_run():
        model = LogisticRegression(C=cfg["C"], max_iter=cfg["max_iter"])
        model.fit(X_train, y_train)
        acc = accuracy_score(y_test, model.predict(X_test))

        mlflow.log_param("C", cfg["C"])
        mlflow.log_param("max_iter", cfg["max_iter"])
        mlflow.log_metric("accuracy", acc)
        mlflow.sklearn.log_model(model, "model")

        print(f"C={cfg['C']} | accuracy={acc:.4f}")
```

## Gestión de los servicios

```bash
# Levantar
docker compose up -d

# Ver estado
docker compose ps

# Ver logs en tiempo real
docker compose logs mlflow --follow

# Detener (sin borrar datos)
docker compose stop

# Detener y eliminar contenedores (datos persisten en carpetas)
docker compose down

# Borrar TODO incluyendo datos (irreversible)
docker compose down
rm -rf postgres-data minio-data
```

## Persistencia de datos

Los datos **sobreviven** a reinicios y a `docker compose down` porque están montados en carpetas locales:

- `./postgres-data/` → todos los experimentos, runs, métricas y parámetros
- `./minio-data/` → todos los artifacts (modelos entrenados, plots, archivos)

Para retomar el trabajo en cualquier momento:

```bash
docker compose up -d
```

Y todo estará exactamente como se dejó.

## Inspeccionar la base de datos

### Opción A — Consola SQL (sin instalar nada)

Entrar directo al contenedor:

```bash
docker exec -it mlflow_postgres psql -U mlflow -d mlflow_db
```

Comandos útiles dentro de la consola:

```sql
-- ver todas las tablas que creó MLflow
\dt

-- ver experimentos
SELECT * FROM experiments;

-- ver todos los runs
SELECT run_uuid, experiment_id, status, start_time FROM runs;

-- ver métricas
SELECT * FROM metrics;

-- ver parámetros
SELECT * FROM params;

-- salir
\q
```

### Opción B — pgAdmin (interfaz gráfica vía Docker)

```bash
docker run -d \
  --name pgadmin \
  --network mlflow-local-stack_mlflow-net \
  -e PGADMIN_DEFAULT_EMAIL=admin@admin.com \
  -e PGADMIN_DEFAULT_PASSWORD=admin \
  -p 8080:80 \
  dpage/pgadmin4
```

Acceder a `http://localhost:8080` y conectar con:

| Campo              | Valor |
|--------------------|---|
| Usuario pgadming   | `admin@admin.com` |
| Contraseña pgadmin | `admin` |
| Server | `mlflow_postgres` |
| Host | `mlflow_postgres` |
| Puerto | `5432` |
| Usuario | `mlflow` |
| Contraseña | `mlflow123` |
| Base de datos | `mlflow_db` |


### Opción C — DBeaver (app de escritorio gratuita)

```bash
sudo snap install dbeaver-ce
```

Conectar con los mismos datos de arriba pero usando `localhost` como host.

## Solución de problemas

**Puerto 5432 ya en uso:**
El sistema tiene PostgreSQL instalado localmente. La solución es no exponer el puerto en el compose (ya aplicado en la configuración de arriba).

**MLflow no conecta con PostgreSQL al inicio:**
Es normal en la primera ejecución. MLflow reintenta automáticamente mientras PostgreSQL termina de inicializarse. Esperar ~30 segundos.

**`docker compose logs mlflow --follow` dice "no such service":**
El nombre del servicio en el compose es `mlflow`, no `mlflow_server`. Usar siempre el nombre del servicio, no el del contenedor.
