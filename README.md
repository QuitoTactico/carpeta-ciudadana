# Carpeta Ciudadana - Monorepo

Sistema nacional de gestión de documentos digitales para Colombia.

## ⚠️ Propósito Educativo

Este proyecto es desarrollado con **fines exclusivamente educativos** como parte del curso **Arquitecturas Avanzadas de Software** de la **Universidad EAFIT**.

**Institución:** Universidad EAFIT
**Curso:** Arquitecturas Avanzadas de Software
**Programa:** Ingeniería de Sistemas / Posgrado
**Objetivo:** Análisis y diseño arquitectónico de un sistema distribuido a escala nacional


Lee nuestra wiki: [Wiki del Proyecto](https://github.com/QuitoTactico/carpeta-ciudadana/wiki)

## Diagrama de Despliegue

```mermaid
graph TB
    subgraph "Servicios Externos"
        GovCarpeta["🌐 GovCarpeta API<br/>Heroku<br/>https://govcarpeta-apis-4905ff3c005b.herokuapp.com<br/>Protocol: HTTPS"]
        SendGrid["📧 SendGrid API<br/>Email Service<br/>Protocol: HTTPS"]
    end

    subgraph "Local - User PC"
        subgraph "OS (Any)"
            subgraph "Docker Server"
                subgraph "Kubernetes Cluster (Minikube, Docker Driver)"
                    subgraph "Frontend Layer"
            CitizenWeb["🖥️ citizen-web<br/>React + TypeScript + Vite<br/>Nginx<br/>Port: 80 (internal: 8080)<br/>NodePort: -<br/>Service: LoadBalancer"]
        end

        subgraph "Application Services"
            AuthService["🔐 auth-service<br/>Go 1.23 + Echo<br/>Port: 8080<br/>NodePort: 30080<br/>JWT Authentication<br/>Service: ClusterIP + NodePort"]
            
            CarpetaService["📂 carpeta-ciudadana-service<br/>Spring Boot 3.2 + Java 21<br/>Port: 8080<br/>NodePort: 30081<br/>Document Management<br/>Service: LoadBalancer + NodePort"]
            
            CiudadanoRegistry["👤 ciudadano-registry-service<br/>Spring Boot 3.2 + Java 17<br/>Port: 8081<br/>NodePort: -<br/>Citizen Registration<br/>Service: ClusterIP"]
            
            DocAuthService["✅ document-authentication-service<br/>Python 3.13 + FastAPI<br/>Port: 8083<br/>NodePort: 30093<br/>Document Verification<br/>Service: ClusterIP + NodePort"]
            
            NotifService["📨 notifications-service<br/>Go 1.23 + Echo<br/>Port: 8080<br/>NodePort: 30090<br/>Email Notifications<br/>Service: ClusterIP + NodePort"]
        end

        subgraph "Message Broker"
            RabbitMQ["🐰 RabbitMQ Cluster<br/>RabbitMQ 3.13-management<br/>3 Nodes (StatefulSet)<br/>───────────────<br/>carpeta-rabbitmq-server-0 (seed)<br/>carpeta-rabbitmq-server-1<br/>carpeta-rabbitmq-server-2<br/>───────────────<br/>AMQP Port: 5672<br/>Management UI: 15672<br/>Prometheus: 15692<br/>───────────────<br/>Quorum Queues:<br/>- document_verification_request<br/>- document_verified_response<br/>- document_authenticated_response<br/>- notifications.email.queue<br/>───────────────<br/>Exchanges:<br/>- microservices.topic (topic)<br/>- carpeta.events (topic)<br/>───────────────<br/>Storage: 10Gi x 3 (PVC)<br/>Service: LoadBalancer"]
        end

        subgraph "Data Layer"
            AuthPostgres["🗄️ auth-postgres<br/>PostgreSQL 15-alpine<br/>Port: 5432<br/>Database: auth_service_db<br/>Tables:<br/>- users<br/>- audit_logs<br/>Service: ClusterIP"]
            
            DynamoDB["📊 DynamoDB Local<br/>amazon/dynamodb-local<br/>Port: 8000<br/>Tables:<br/>- CarpetaCiudadano<br/>- Documento<br/>- HistorialAcceso<br/>Service: ClusterIP"]
            
                        MinIO["📦 MinIO<br/>minio/minio:latest<br/>API Port: 9000<br/>Console Port: 9001<br/>NodePort Console: 30901<br/>Bucket: carpeta-ciudadana-docs<br/>Storage:<br/>- Documentos PDF/JPEG/PNG<br/>- Presigned URLs (15min)<br/>Service: ClusterIP + NodePort (console)"]
                    end
                end
            end
        end
    end

    %% Frontend to Services
    CitizenWeb -->|"HTTP/REST<br/>Port 8080"| AuthService
    CitizenWeb -->|"HTTP/REST<br/>Port 8080"| CarpetaService
    CitizenWeb -->|"HTTP/REST<br/>Port 8083<br/>authenticateDocument"| DocAuthService
    
    %% Auth Service connections
    AuthService -->|"HTTP/REST<br/>Port 8081<br/>validate/register citizen"| CiudadanoRegistry
    AuthService -->|"AMQP<br/>Port 5672<br/>user.registration.email<br/>user.registration.complete"| RabbitMQ
    AuthService -->|"SQL<br/>Port 5432<br/>Store users & audit"| AuthPostgres
    
    %% Carpeta Service connections
    CarpetaService -->|"HTTP/REST<br/>Port 9000<br/>Upload/Download docs"| MinIO
    CarpetaService -->|"HTTP<br/>Port 8000<br/>Store metadata"| DynamoDB
    CarpetaService -->|"AMQP<br/>Port 5672<br/>document events"| RabbitMQ
    
    %% Ciudadano Registry connections
    CiudadanoRegistry -->|"HTTPS<br/>validateCitizen<br/>registerCitizen<br/>unregisterCitizen"| GovCarpeta
    CiudadanoRegistry -->|"HTTP/REST<br/>Port 8080<br/>create folder"| CarpetaService
    CiudadanoRegistry -->|"HTTP<br/>Port 8000<br/>Store registry"| DynamoDB
    
    %% Document Auth Service connections
    DocAuthService -->|"HTTP/REST<br/>Port 8080<br/>get presigned URL"| CarpetaService
    DocAuthService -->|"HTTPS<br/>authenticateDocument"| GovCarpeta
    DocAuthService -->|"AMQP<br/>Port 5672<br/>publish auth result"| RabbitMQ
    
    %% Notifications Service connections
    RabbitMQ -->|"AMQP<br/>Port 5672<br/>consume events"| NotifService
    NotifService -->|"HTTPS<br/>Send emails"| SendGrid
    
    %% Styling
    classDef frontend fill:#e1f5ff,stroke:#01579b,stroke-width:3px,color:#000
    classDef service fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000
    classDef messaging fill:#f3e5f5,stroke:#4a148c,stroke-width:3px,color:#000
    classDef database fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px,color:#000
    classDef external fill:#ffebee,stroke:#b71c1c,stroke-width:2px,color:#000
    
    class CitizenWeb frontend
    class AuthService,CarpetaService,CiudadanoRegistry,DocAuthService,NotifService service
    class RabbitMQ messaging
    class AuthPostgres,DynamoDB,MinIO database
    class GovCarpeta,SendGrid external
```

## Estructura del Proyecto

```
carpeta_ciudadana/
├── docs/                    # Documentación y análisis arquitectónico
│   ├── ADR/                # Architecture Decision Records
│   └── informacion_cruda/  # Análisis de requerimientos y DDD
├── services/               # Microservicios y aplicaciones
├── libs/                   # Librerías compartidas
├── infrastructure/         # Configuración de infraestructura (Docker, K8s, Terraform)
└── tools/                  # Scripts y herramientas de desarrollo
```

## Directorios Principales

### `docs/`
Toda la documentación del proyecto, incluyendo análisis de dominio, requerimientos funcionales/no funcionales, y decisiones arquitectónicas.

### `services/`
Microservicios y aplicaciones del sistema. Cada servicio será independiente con su propia tecnología.

Ejemplos de servicios futuros:
- API del Operador
- API del Centralizador (MinTIC)
- Aplicación Web Ciudadano
- Aplicación Web Entidad
- Servicio de Autenticación
- Servicio de Notificaciones
- Servicio de Analytics

### `libs/`
Código compartido entre servicios (tipos, modelos, utilidades, etc.).

### `infrastructure/`
Configuración de infraestructura como código (IaC) y contenedores.

### `tools/`
Scripts de automatización, herramientas de desarrollo, y utilidades del proyecto.

## Comenzar

Este es un proyecto académico de diseño arquitectónico. Consulta `CLAUDE.md` para guías de desarrollo y `docs/` para el análisis completo del sistema.

