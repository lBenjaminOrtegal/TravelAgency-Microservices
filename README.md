# TravelAgency — Plataforma Web de Microservicios Orquestados en Kubernetes

## Descripción del Proyecto

**TravelAgency** es una plataforma web distribuida para la comercialización, gestión de reservas y procesamiento de pagos de paquetes turísticos nacionales e internacionales. El sistema moderniza la operación de la agencia mediante una **arquitectura orientada a microservicios desacoplados** desarrollada con **Spring Boot** y **React + TypeScript**, orquestada en un clúster de **Kubernetes (Minikube)**.

La plataforma implementa patrones de diseño de sistemas distribuidos como **Database-per-Service**, **Service Discovery (Eureka)**, **Centralized Configuration (Config Server)**, **API Gateway reactivo** y autenticación/autorización federada basada en el estándar **OAuth2 / OpenID Connect (OIDC)** con **Keycloak**.

---

## Arquitectura del Sistema

```text
                                 ┌────────────────────────┐
                                 │   Keycloak (IAM)       │
                                 │   (NodePort: 30708)    │
                                 └───────────▲────────────┘
                                             │ JWT Validation
┌─────────────────────────┐      HTTP        │
│   Frontend (React)      ├──────────────────┼───────────────────────────────┐
│   (NodePort: 30807)     │                  │                               │
└─────────────────────────┘                  ▼                               │
                                 ┌────────────────────────┐                  │
                                 │   API Gateway          │                  │
                                 │   (NodePort: 30808)    │                  │
                                 └───────────┬────────────┘                  │
                                             │ Load Balanced (lb://)         │
                                             ▼                               │
                            ┌──────────────────────────────────┐             │
                            │   Eureka Service Discovery       │             │
                            │   (ClusterIP: 8761)              │             │
                            └────────────────┬─────────────────┘             │
                                             │ Service Registration          │
         ┌───────────────────────────────────┼───────────────────────────────┼───────────────────────────────────┐
         │                                   │                               │                                   │
         ▼                                   ▼                               ▼                                   ▼
┌──────────────────┐               ┌──────────────────┐            ┌──────────────────┐                ┌──────────────────┐
│   m2-service     │               │   m3-service     │            │   m4-service     │                │   m5-service     │
│ Paquetes (CRUD)  │               │ Búsqueda/Filtros │            │ Reservas/Desc.   │                │ Pagos/Transacc.  │
└────────┬─────────┘               └────────┬─────────┘            └────────┬─────────┘                └────────┬─────────┘
         │ DB-per-Service                   │ DB-per-Service                │ DB-per-Service                    │ DB-per-Service
         ▼                                  ▼                               ▼                                   ▼
┌──────────────────┐               ┌──────────────────┐            ┌──────────────────┐                ┌──────────────────┐
│  MySQL m2_db     │               │  MySQL m3_db     │            │  MySQL m4_db     │                │  MySQL m5_db     │
└──────────────────┘               └──────────────────┘            └──────────────────┘                └──────────────────┘

                       [ + m6-service (Seguimiento) y m7-service (Reportes) con sus BDs MySQL ]
                       [ + config-server (Configuración Centralizada desde Git) ]
```

### Principales Patrones y Componentes de Infraestructura:

1. **API Gateway (`api-gateway`)**: Punto de entrada único para el frontend. Encaminamiento dinámico no bloqueante (Spring Cloud Gateway WebFlux) con validación de tokens JWT.
2. **Service Discovery (`eureka-server`)**: Registro dinámico de microservicios. Los servicios de negocio se inician en puertos asignados dinámicamente por el sistema (`server.port=0`) eliminando conflictos de puertos.
3. **Centralized Config Server (`config-server`)**: Servidor de configuración centralizada que expone las propiedades YAML almacenadas en un repositorio remoto.
4. **Database-per-Service**: Cada microservicio cuenta con su propia instancia independiente de **MySQL 8.0**, garantizando el aislamiento de datos y previniendo acoplamientos a nivel de persistencia.
5. **IAM Decentralized Security (`Keycloak`)**: Keycloak actúa como el servidor de identidad. Los microservicios operan como *OAuth2 Resource Servers*, validando las firmas criptográficas de los tokens JWT de forma independiente mediante las claves públicas expuestas por Keycloak (`jwk-set-uri`).

---

## Módulos del Sistema (Épicas)

| Microservicio | Épica / Dominio | Descripción | Base de Datos |
| :--- | :--- | :--- | :--- |
| **Keycloak (M1)** | Épica 1: Usuarios e Identidad | Gestión de credenciales, login/registro, perfiles y asignación de roles (`ADMIN`, `USER`). | MySQL / H2 |
| **`m2-service`** | Épica 2: Gestión de Paquetes | Creación, actualización, control de cupos y publicación de paquetes turísticos (Rol `ADMIN`). | `db_m2` (MySQL) |
| **`m3-service`** | Épica 3: Exploración y Búsqueda | Catálogo interactivo con filtros combinados (destino, rango de precio, fechas, temporada, tipo de viaje, categoría). | `db_m3` (MySQL) |
| **`m4-service`** | Épica 4: Reservas y Descuentos | Creación de reservas, validación de stock y motor dinámico de reglas de negocio de descuentos. | `db_m4` (MySQL) |
| **`m5-service`** | Épica 5: Gestión de Pagos | Procesamiento y simulación de transacciones financieras con tarjeta de crédito/débito. | `db_m5` (MySQL) |
| **`m6-service`** | Épica 6: Seguimiento de Reservas | Cancelaciones, consulta de historial y transiciones de estado (`PENDING`, `CONFIRMED`, `IN_PROGRESS`, `COMPLETED`, `CANCELED`). | `db_m6` (MySQL) |
| **`m7-service`** | Épica 7: Reportes e Inteligencia | Generación de reportes de transacciones por rango de fechas y ranking de ventas (por reservas o por número de pasajeros). | `db_m7` (MySQL) |

---

## Stack Tecnológico

### Backend & Microservicios
* **Lenguaje:** Java 21 LTS
* **Framework Principal:** Spring Boot 3.x / 4.x
* **Ecosistema Spring Cloud:**
  * Spring Cloud Gateway (WebFlux / Reactive Routing)
  * Spring Cloud Netflix Eureka (Service Registry & Discovery)
  * Spring Cloud Config Server (Centralized Configuration)
* **Seguridad:** Spring Security, OAuth2 Resource Server, OpenID Connect, JWT
* **Persistencia:** Spring Data JPA / Hibernate
* **Herramientas de Construcción:** Apache Maven / Gradle

### Frontend
* **Librería UI:** React 19 + TypeScript
* **Bundler & Tooling:** Vite, ESBuild
* **Estilos:** React Bootstrap, Bootstrap 5, CSS3 Custom Styling
* **Cliente HTTP & Autenticación:** Axios (con interceptores JWT), `@react-keycloak/web`, `keycloak-js`

### Infraestructura, Base de Datos & DevOps
* **Orquestación:** Kubernetes (Minikube con driver VirtualBox/Docker)
* **Contenedores:** Docker & Docker Hub
* **Base de Datos:** MySQL 8.0 (6 instancias persistentes aisladas con `PersistentVolumeClaim`)
* **Gestor de Identidades (IAM):** Keycloak 26.2
* **Testing & Calidad:** JUnit 5, Playwright (Pruebas E2E / Gherkin BDD), K6 (Pruebas de Carga y Estrés), SonarQube (Análisis Estático)

---

## Estructura del Repositorio

```text
Evaluacion-2-Tingeso/
├── api-gateway/          # Spring Cloud Gateway (Port 8080 / NodePort 30808)
├── config-server/        # Spring Cloud Config Server (Port 8081)
├── eureka-server/        # Netflix Eureka Discovery Server (Port 8761)
├── m2-service/           # Microservicio de Paquetes Turísticos (CRUD)
├── m3-service/           # Microservicio de Búsqueda y Filtros
├── m4-service/           # Microservicio de Reservas y Motor de Descuentos
├── m5-service/           # Microservicio de Pagos y Transacciones
├── m6-service/           # Microservicio de Seguimiento y Estado de Reservas
├── m7-service/           # Microservicio de Reportes Administrativos
├── frontend/             # Single Page Application en React + TypeScript
├── config-data/          # Repositorio de configuraciones YAML para Config Server
└── k8s/                  # Manifiestos declarativos de Kubernetes
    ├── 01-configmap.yaml     # Variables de entorno unificadas del clúster
    ├── 02-credentials.yaml   # Configuraciones del entorno
    ├── 03-databases.yaml     # PVCs, Deployments y ClusterIP Services para las BDs MySQL
    ├── 04-keycloak.yaml      # PVC, Deployment y NodePort Service para Keycloak
    ├── 05-infrastructure.yaml# Deployments y Services para Config Server y Eureka
    ├── 06-microservices.yaml  # Deployments y ClusterIP Services para m2 a m7
    ├── 07-gateway.yaml       # Deployment y NodePort Service para API Gateway
    └── 08-frontend.yaml      # Deployment y NodePort Service para Frontend
```

---

## Despliegue en Kubernetes (Minikube)

### Requisitos Previos

* **Minikube** v1.30+
* **kubectl** configurado
* **Docker Desktop** o Docker Engine
* **VirtualBox** (si se utiliza el driver de máquina virtual)

### Pasos de Instalación y Despliegue

#### 1. Iniciar Minikube

```bash
minikube start --driver=virtualbox --memory=8192 --cpus=4
```

Obtén la IP interna de Minikube:

```bash
minikube ip
# Ejemplo de salida: 192.168.59.105
```

> **Nota:** Ajusta la dirección IP en `k8s/01-configmap.yaml` y `k8s/08-frontend.yaml` si la IP de tu Minikube difiere.

#### 2. Aplicar los manifiestos de Kubernetes en orden

```bash
kubectl apply -f k8s/01-configmap.yaml
kubectl apply -f k8s/02-credentials.yaml
kubectl apply -f k8s/03-databases.yaml
kubectl apply -f k8s/04-keycloak.yaml
kubectl apply -f k8s/05-infrastructure.yaml
kubectl apply -f k8s/06-microservices.yaml
kubectl apply -f k8s/07-gateway.yaml
kubectl apply -f k8s/08-frontend.yaml
```

#### 3. Verificar el estado del clúster

Asegúrate de que todos los pods y servicios estén en estado `Running`:

```bash
kubectl get pods -w
kubectl get svc
```

#### 4. Acceder a las aplicaciones

| Servicio | Tipo de Servicio | URL de Acceso |
| :--- | :--- | :--- |
| **Frontend SPA** | `NodePort` | `http://<MINIKUBE_IP>:30807` |
| **API Gateway** | `NodePort` | `http://<MINIKUBE_IP>:30808` |
| **Keycloak IAM** | `NodePort` | `http://<MINIKUBE_IP>:30708` |
| **Eureka Server** | `ClusterIP` | Accesible internamente en `http://eureka-service:8761` |
| **Config Server** | `ClusterIP` | Accesible internamente en `http://config-service:8081` |

---

## Reglas de Negocio Destacadas

### Motor de Descuentos Dinámico (`m4-service`)
El sistema calcula el costo total de la reserva aplicando de manera automatizada las siguientes reglas:
1. **Descuento por Cantidad de Pasajeros:** Aplica un porcentaje cuando la reserva supera el mínimo configurado de acompañantes.
2. **Descuento por Cliente Frecuente:** Valida el historial histórico del usuario según sus reservas previas completadas.
3. **Descuento por Compras Múltiples:** Identifica si el cliente ha comprado más de un paquete dentro de una ventana de tiempo predefinida.
4. **Acumulación y Tope Máximo:** Permite o restringe la combinación de promociones y respeta un límite global de descuento.
