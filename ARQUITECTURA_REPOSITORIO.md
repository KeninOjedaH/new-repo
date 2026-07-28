# Arquitectura de Software del Repositorio

## 1. Descripción general

El repositorio implementa una plataforma de procesamiento y consulta de datos organizada como un **monorepositorio modular**. En una misma solución se integran:

- Una interfaz web desarrollada con **React, Vite y TypeScript**.
- Una API backend desarrollada con **FastAPI**.
- Un servicio independiente para la ejecución de procesos **ETL**.
- Una base de datos **PostgreSQL**.
- Contenedores administrados mediante **Docker Compose**.

La arquitectura combina separación por responsabilidades, comunicación mediante APIs REST, persistencia centralizada y procesamiento de datos por capas.

---

## 2. Vista general de la arquitectura

```text
Usuario
   │
   ▼
Frontend React
   │  HTTP / REST + JWT
   ▼
Backend FastAPI
   │
   ├── Autenticación y autorización
   ├── Gestión de usuarios
   ├── Carga de archivos Excel
   ├── Historial de ejecuciones ETL
   └── Comunicación con el servicio ETL
          │
          │ HTTP interno POST /run
          ▼
     Servicio ETL
          │
          ├── Ingesta
          ├── Detección de duplicados
          ├── Normalización
          ├── Procesamiento Silver
          ├── Construcción de conceptos
          ├── Generación de perfiles
          └── Publicación Gold
                 │
                 ▼
            PostgreSQL
                 │
                 ├── Usuarios
                 ├── Historial de cargas
                 ├── Datos Bronze
                 ├── Datos Silver
                 └── Vistas y tablas Gold
```

---

## 3. Estilo arquitectónico

La solución utiliza principalmente los siguientes estilos arquitectónicos:

### 3.1. Arquitectura en capas

La aplicación se divide en capas con responsabilidades claramente diferenciadas:

1. **Capa de presentación:** frontend web.
2. **Capa de aplicación:** API backend.
3. **Capa de procesamiento:** motor ETL.
4. **Capa de persistencia:** PostgreSQL.
5. **Capa de infraestructura:** Docker y volúmenes compartidos.

### 3.2. Arquitectura de servicios

El backend y el motor ETL funcionan como servicios separados:

- El backend expone la API consumida por el frontend.
- El servicio ETL expone una API interna en el puerto `9000`.
- El backend invoca al ETL mediante `POST /run`.

### 3.3. Arquitectura ETL por capas de datos

El procesamiento sigue una organización similar a una arquitectura Medallion:

- **Bronze:** datos originales o crudos.
- **Silver:** datos limpiados, normalizados y tipados.
- **Gold:** datos preparados para análisis y consumo.

### 3.4. Arquitectura monorepo

El frontend, backend, motor ETL, scripts e infraestructura se encuentran dentro del mismo repositorio. Esto facilita:

- La administración centralizada del código.
- El versionamiento coordinado.
- La integración de cambios entre servicios.
- El despliegue conjunto mediante Docker Compose.

---

## 4. Componentes principales

## 4.1. Frontend web

**Tecnologías:** React, Vite y TypeScript.

**Ubicación:**

```text
frontend/
```

### Responsabilidades

- Mostrar la pantalla de inicio de sesión.
- Almacenar y utilizar el token JWT.
- Proteger las rutas privadas.
- Mostrar el panel principal.
- Permitir la carga de archivos Excel.
- Consultar el historial de cargas ETL.
- Consumir los endpoints del backend.

### Archivos principales

```text
frontend/src/
├── main.tsx
├── App.tsx
├── routes.tsx
├── auth/
│   ├── auth.ts
│   └── ProtectedRoute.tsx
├── pages/
│   ├── Login/
│   │   ├── LoginPage.tsx
│   │   └── login.css
│   └── Panel/
│       ├── PanelPage.tsx
│       └── panel.css
└── assets/
```

### Rutas principales

```text
/          Página de inicio de sesión
/login     Página de inicio de sesión
/panel     Panel protegido
```

El componente `ProtectedRoute` verifica la autenticación del usuario antes de permitir el acceso al panel.

### Puerto

```text
5173
```

---

## 4.2. Backend API

**Tecnología:** FastAPI.

**Ubicación:**

```text
backend/
```

### Responsabilidades

- Autenticar usuarios.
- Generar y validar tokens JWT.
- Consultar el usuario autenticado.
- Recibir archivos Excel.
- Guardar temporalmente los archivos cargados.
- Invocar al servicio ETL.
- Registrar el resultado de cada ejecución.
- Exponer el historial de cargas.
- Gestionar la conexión con PostgreSQL mediante SQLAlchemy.

### Estructura principal

```text
backend/app/
├── main.py
├── config.py
├── db.py
├── deps.py
├── security.py
├── models.py
├── models_etl.py
├── schemas.py
└── etl_routes.py
```

### Descripción de archivos

| Archivo | Responsabilidad |
|---|---|
| `main.py` | Inicialización de FastAPI, CORS, rutas y eventos de inicio. |
| `config.py` | Lectura de variables de configuración. |
| `db.py` | Configuración de SQLAlchemy y conexión a PostgreSQL. |
| `deps.py` | Dependencias para base de datos y usuario autenticado. |
| `security.py` | Hash de contraseñas, validación y JWT. |
| `models.py` | Modelo de usuarios. |
| `models_etl.py` | Modelo del historial de ejecuciones ETL. |
| `schemas.py` | Esquemas de entrada y salida de la API. |
| `etl_routes.py` | Carga de archivos, ejecución ETL e historial. |

### Endpoints principales

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/health` | Verifica el estado del backend. |
| `POST` | `/auth/login` | Valida credenciales y genera un token JWT. |
| `GET` | `/auth/me` | Devuelve los datos del usuario autenticado. |
| `POST` | `/etl/upload` | Recibe un archivo Excel y ejecuta el ETL. |
| `GET` | `/etl/history` | Consulta el historial de cargas. |

### Seguridad

El backend utiliza:

- Contraseñas almacenadas mediante hash `bcrypt`.
- Tokens JWT firmados con el algoritmo `HS256`.
- Dependencias de FastAPI para validar al usuario autenticado.
- Rutas protegidas mediante encabezado de autorización.

### Puerto

```text
8000
```

---

## 4.3. Servicio ETL

**Tecnología:** Python y FastAPI.

**Ubicación:**

```text
etl_engine/
```

### Responsabilidades

- Recibir la ruta de un archivo ubicado en el volumen compartido.
- Ejecutar el comando `etl-engine run`.
- Ingerir archivos Excel o datos procedentes de fuentes externas.
- Calcular hashes para detectar archivos duplicados.
- Normalizar nombres y valores.
- Convertir datos a estructuras tipadas.
- Generar datos Silver.
- Construir conceptos y perfiles.
- Crear o actualizar resultados Gold.
- Persistir los resultados en PostgreSQL.

### API interna

El archivo `api_server.py` expone:

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/health` | Verifica el estado del servicio ETL. |
| `POST` | `/run` | Ejecuta el motor ETL sobre un archivo compartido. |

El endpoint `/run` solamente admite rutas ubicadas dentro de:

```text
/shared/
```

### Puerto interno

```text
9000
```

Este puerto se expone únicamente dentro de la red de Docker y no se publica directamente hacia el host.

---

## 4.4. Pipeline ETL

La lógica principal del motor ETL se encuentra en:

```text
etl_engine/src/etl_engine/
```

### Estructura

```text
etl_engine/src/etl_engine/
├── api_server.py
├── cli.py
├── config.py
├── core/
│   ├── ingest.py
│   ├── dedupe.py
│   ├── silver.py
│   ├── concept.py
│   ├── silver_concept.py
│   └── profile.py
├── sources/
│   ├── airtable_sync.py
│   ├── freshdesk_sync.py
│   ├── infobip_sync.py
│   └── wordpress_watermark.py
├── storage/
│   ├── filesystem.py
│   ├── postgres.py
│   ├── postgres_gold.py
│   ├── schema.sql
│   └── gold_schema.sql
└── utils/
    ├── hashing.py
    └── text.py
```

### Etapas del procesamiento

```text
Archivo o fuente externa
        │
        ▼
1. Ingesta
        │
        ▼
2. Cálculo de hash
        │
        ▼
3. Detección de duplicados
        │
        ▼
4. Almacenamiento Bronze
        │
        ▼
5. Limpieza y normalización
        │
        ▼
6. Conversión y tipado Silver
        │
        ▼
7. Construcción de conceptos y perfiles
        │
        ▼
8. Publicación de resultados Gold
```

### Módulos del núcleo

| Módulo | Función principal |
|---|---|
| `ingest.py` | Ingesta y lectura de archivos. |
| `dedupe.py` | Identificación de registros o cargas duplicadas. |
| `silver.py` | Transformación y tipado de datos. |
| `concept.py` | Detección y construcción de conceptos. |
| `silver_concept.py` | Asociación entre datos Silver y conceptos. |
| `profile.py` | Generación de perfiles y estadísticas. |

---

## 4.5. Fuentes externas de datos

El motor ETL posee conectores para sincronizar información procedente de diferentes servicios.

```text
etl_engine/src/etl_engine/sources/
```

### Fuentes identificadas

- Airtable.
- Freshdesk.
- Infobip.
- WordPress.
- Archivos Excel cargados desde el frontend.

### Scripts de sincronización

```text
scripts/
├── sync_airtable_daily.sh
├── sync_airtable_reportes_daily.sh
├── sync_freshdesk_hourly.sh
├── sync_infobip_daily.sh
└── sync_wordpress_daily.sh
```

Estos scripts permiten programar procesos batch con diferentes frecuencias.

---

## 4.6. Base de datos PostgreSQL

**Tecnología:** PostgreSQL 16.

**Servicio Docker:**

```text
db
```

### Responsabilidades

- Almacenar usuarios del sistema.
- Registrar el historial de ejecuciones ETL.
- Persistir datos crudos y transformados.
- Mantener las tablas de soporte del motor ETL.
- Proporcionar tablas y vistas Gold para análisis.

### Tablas de aplicación

#### `app_users`

Almacena los usuarios que pueden ingresar al sistema.

Campos principales:

- `id`
- `username`
- `hashed_password`
- `role`
- `is_active`

#### `etl_upload_log`

Registra cada archivo enviado al motor ETL.

Campos principales:

- `id`
- `department`
- `filename`
- `upload_id`
- `content_hash`
- `is_duplicate`
- `n_rows`
- `gold_refreshed`
- `status`
- `message`
- `stdout`
- `stderr`
- `created_at`

### Estados de ejecución

```text
RUNNING
SUCCESS
DUPLICATE
FAILED
```

### Puerto

Dentro del contenedor:

```text
5432
```

Expuesto en el host:

```text
15432
```

### Persistencia

La base de datos utiliza el volumen:

```text
pgdata
```

---

## 4.7. Volumen compartido de archivos

El backend y el servicio ETL comparten el volumen:

```text
shared_uploads
```

El volumen se monta en ambos servicios bajo:

```text
/shared
```

La carpeta utilizada para las cargas es:

```text
/shared/uploads
```

### Funcionamiento

1. El backend recibe el archivo.
2. El backend genera un nombre único mediante UUID.
3. El archivo se guarda temporalmente en `/shared/uploads`.
4. El backend envía la ruta al servicio ETL.
5. El ETL procesa el archivo desde la misma ubicación.
6. Al finalizar, el backend elimina el archivo temporal.

Este mecanismo evita enviar archivos grandes directamente entre los contenedores mediante HTTP.

---

## 4.8. Infraestructura Docker

La infraestructura se define en:

```text
docker-compose.yml
```

### Servicios

| Servicio | Responsabilidad | Puerto |
|---|---|---|
| `frontend` | Interfaz React | `5173` |
| `backend` | API principal | `8000` |
| `etl` | Procesamiento ETL interno | `9000` interno |
| `db` | PostgreSQL | `15432:5432` |

### Dependencias

```text
frontend
   └── depende de backend

backend
   └── depende de db saludable

etl
   └── depende de db saludable
```

### Volúmenes

| Volumen | Uso |
|---|---|
| `pgdata` | Persistencia de PostgreSQL. |
| `shared_uploads` | Intercambio temporal de archivos entre backend y ETL. |

---

## 5. Flujo principal de ejecución

### 5.1. Inicio de sesión

```text
1. El usuario ingresa sus credenciales en React.
2. El frontend envía POST /auth/login.
3. FastAPI consulta al usuario en PostgreSQL.
4. El backend verifica la contraseña con bcrypt.
5. El backend genera un JWT.
6. El frontend guarda el token.
7. ProtectedRoute habilita el acceso al panel.
```

### 5.2. Carga y procesamiento de un archivo

```text
1. El usuario selecciona un archivo Excel.
2. El frontend envía el archivo a POST /etl/upload.
3. El backend valida la extensión del archivo.
4. El backend crea un registro RUNNING en etl_upload_log.
5. El archivo se guarda en /shared/uploads.
6. El backend invoca http://etl:9000/run.
7. El ETL ejecuta el comando etl-engine run.
8. El pipeline procesa y persiste la información.
9. El ETL devuelve stdout, stderr y código de salida.
10. El backend interpreta el resultado.
11. El registro se actualiza como SUCCESS, DUPLICATE o FAILED.
12. El frontend muestra el resultado y actualiza el historial.
13. El archivo temporal se elimina.
```

---

## 6. Comunicación entre componentes

| Origen | Destino | Mecanismo | Propósito |
|---|---|---|---|
| Usuario | Frontend | Navegador web | Interacción con el sistema. |
| Frontend | Backend | HTTP REST | Login, carga e historial. |
| Frontend | Backend | JWT | Autenticación de solicitudes. |
| Backend | PostgreSQL | SQLAlchemy | Usuarios e historial ETL. |
| Backend | ETL | HTTP interno | Solicitar ejecución del pipeline. |
| Backend | Volumen compartido | Sistema de archivos | Guardar archivos temporales. |
| ETL | Volumen compartido | Sistema de archivos | Leer archivos cargados. |
| ETL | PostgreSQL | SQL | Persistir Bronze, Silver y Gold. |
| Scripts | Fuentes externas | APIs | Sincronizar información externa. |

---

## 7. Estructura general del repositorio

```text
Fronted_React-main/
├── frontend/
│   ├── src/
│   │   ├── auth/
│   │   ├── pages/
│   │   ├── assets/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── routes.tsx
│   ├── Dockerfile
│   ├── package.json
│   └── vite.config.ts
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── config.py
│   │   ├── db.py
│   │   ├── deps.py
│   │   ├── security.py
│   │   ├── models.py
│   │   ├── models_etl.py
│   │   ├── schemas.py
│   │   └── etl_routes.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── etl_engine/
│   ├── src/etl_engine/
│   │   ├── core/
│   │   ├── sources/
│   │   ├── storage/
│   │   ├── utils/
│   │   ├── api_server.py
│   │   ├── cli.py
│   │   └── config.py
│   ├── gold_sql/
│   ├── sql/
│   ├── Dockerfile
│   └── pyproject.toml
│
├── scripts/
│   ├── sync_airtable_daily.sh
│   ├── sync_airtable_reportes_daily.sh
│   ├── sync_freshdesk_hourly.sh
│   ├── sync_infobip_daily.sh
│   └── sync_wordpress_daily.sh
│
├── docker-compose.yml
├── package-lock.json
└── .gitignore
```

---

## 8. Características arquitectónicas

### Modularidad

Cada componente se encuentra en una carpeta independiente y posee responsabilidades específicas.

### Separación de responsabilidades

- React administra la interfaz.
- FastAPI administra la lógica de aplicación y seguridad.
- El ETL administra el procesamiento de datos.
- PostgreSQL administra la persistencia.
- Docker Compose administra la ejecución de servicios.

### Escalabilidad

El backend y el ETL pueden desplegarse o escalarse de manera independiente, siempre que se conserve la comunicación interna y el acceso a PostgreSQL.

### Seguridad

- Autenticación con JWT.
- Contraseñas protegidas mediante bcrypt.
- Rutas protegidas en frontend y backend.
- Servicio ETL no expuesto directamente al usuario.
- Validación de archivos y rutas compartidas.

### Trazabilidad

La tabla `etl_upload_log` permite conocer:

- Quién o qué departamento realizó una carga.
- El archivo procesado.
- El identificador de carga.
- El hash del contenido.
- Si el archivo era duplicado.
- La cantidad de filas procesadas.
- Si las vistas Gold fueron actualizadas.
- El estado final.
- Los mensajes de salida y error.

### Persistencia

Los datos de PostgreSQL permanecen almacenados en el volumen `pgdata`, incluso cuando los contenedores se reinician.

### Procesamiento batch e interactivo

La plataforma admite dos formas de ejecución:

- **Interactiva:** archivos cargados desde el panel web.
- **Batch:** scripts programados para Airtable, Freshdesk, Infobip y WordPress.

---

## 9. Patrón de despliegue

```text
Docker Compose
│
├── Contenedor frontend
│   └── React + Vite
│
├── Contenedor backend
│   └── FastAPI + SQLAlchemy + JWT
│
├── Contenedor ETL
│   └── FastAPI interna + CLI ETL
│
└── Contenedor PostgreSQL
    └── Persistencia de aplicación y datos ETL
```

---

## 10. Clasificación final de la arquitectura

La arquitectura del repositorio puede definirse como:

> **Arquitectura monorepo modular, distribuida en servicios contenedorizados, con frontend SPA, backend REST, motor ETL independiente, persistencia PostgreSQL y procesamiento de datos por capas Bronze, Silver y Gold.**

No corresponde a una arquitectura de microservicios completa, porque los componentes forman una solución pequeña y fuertemente coordinada. Sin embargo, presenta características de una arquitectura orientada a servicios, debido a que el backend y el motor ETL se ejecutan y comunican como servicios independientes.

---

## 11. Observaciones técnicas

1. Existen dos archivos con nombres similares dentro del motor ETL:

```text
etl_engine/DockerFile
etl_engine/Dockerfile
```

En sistemas Windows esto puede producir conflictos por diferencias únicamente en mayúsculas y minúsculas. Se recomienda conservar solo:

```text
etl_engine/Dockerfile
```

2. El archivo `docker-compose.yml` referencia certificados dentro de:

```text
postgres_ssl/
```

Debe verificarse que esta carpeta y sus certificados se encuentren disponibles en el entorno de despliegue.

3. El backend elimina el archivo temporal después de completar o fallar la ejecución, lo cual evita la acumulación de archivos en el volumen compartido.

4. El servicio ETL utiliza `subprocess.run` para ejecutar el comando del motor. En cargas pesadas convendría evaluar una cola de trabajos asíncrona, aunque la implementación actual es adecuada para una carga moderada y controlada.

---

## 12. Conclusión

El repositorio presenta una arquitectura clara y modular para una plataforma de integración y procesamiento de datos. La separación entre frontend, backend, servicio ETL y base de datos permite mantener responsabilidades bien definidas y facilita el mantenimiento.

La decisión de utilizar un servicio ETL independiente es especialmente importante, ya que evita que el backend concentre toda la lógica de procesamiento. PostgreSQL funciona como punto central de persistencia, mientras que Docker Compose permite levantar todos los componentes de forma coordinada.

La arquitectura es apropiada para un sistema interno de carga, transformación, seguimiento y análisis de datos, con capacidad de evolución hacia mecanismos de procesamiento asíncrono, mayor observabilidad y despliegues independientes.
