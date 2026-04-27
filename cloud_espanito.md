# MPO — Fundamentos de Computación en la Nube
## Agencia Espanito — Arquitectura Cloud en AWS
**Proyecto Intermodular · 1º ASIR · Yassin Mansouri**

---

## 1. Elección del proveedor cloud

### Proveedor elegido: Amazon Web Services (AWS)

Se ha elegido AWS como proveedor cloud para el despliegue de la infraestructura de Espanito por las siguientes razones:

**Madurez y fiabilidad**
AWS es el proveedor cloud líder mundial con más de 200 servicios disponibles y una disponibilidad garantizada del 99.99% en sus servicios principales. Cuenta con centros de datos en Europa (Frankfurt, Irlanda, París, Estocolmo) lo que garantiza baja latencia para los usuarios de la agencia.

**Escalabilidad automática**
La agencia Espanito puede empezar con una infraestructura mínima y escalar automáticamente según el volumen de estudiantes, solicitudes y documentos gestionados, sin necesidad de invertir en hardware físico adicional.

**Free Tier disponible**
AWS ofrece un nivel gratuito durante los primeros 12 meses que incluye los servicios básicos necesarios para la agencia, lo que permite comenzar sin coste durante el período de prueba.

**Ecosistema de servicios educativos**
AWS dispone de AWS Educate, que proporciona créditos y recursos gratuitos para instituciones educativas y proyectos académicos, alineado con la naturaleza de la agencia Espanito.

**Comparativa con otras opciones:**

| Criterio | AWS | Google Cloud | Azure | DigitalOcean |
|---|---|---|---|---|
| Madurez | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★☆☆ |
| Free Tier | 12 meses | 90 días | 12 meses | 60 días |
| Presencia en Europa | Frankfurt, Irlanda... | Bélgica, Países Bajos... | Amsterdam, Frankfurt... | Amsterdam, Frankfurt |
| Facilidad de uso | Media | Media | Media | Alta |
| Documentación | Excelente | Muy buena | Muy buena | Buena |
| Coste estimado | ~35€/mes | ~32€/mes | ~38€/mes | ~25€/mes |

Se descarta DigitalOcean por menor ecosistema de servicios gestionados. Se elige AWS por su posición de liderazgo, la presencia de la región eu-central-1 (Frankfurt) como punto óptimo para una agencia con estudiantes europeos, y su amplio free tier.

---

## 2. Arquitectura cloud propuesta

### Descripción general

La arquitectura cloud de Espanito sigue el patrón de **tres capas** (presentación, aplicación y datos), desplegado en la región **eu-central-1 (Frankfurt)** dentro de una VPC privada con subredes públicas y privadas.

### Diagrama de arquitectura

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INTERNET / USUARIOS                          │
└────────────────────────────┬────────────────────────────────────────┘
                             │ HTTPS (443)
                    ┌────────▼────────┐
                    │   Route 53      │  DNS: espanito.com
                    │  (DNS + SSL)    │      *.espanito.com
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   CloudFront    │  CDN — documentos estáticos,
                    │   (CDN global)  │  imágenes, assets web
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │    Application Load Balancer │  HTTP → HTTPS redirect
              │       (ALB público)          │  Health checks
              └──────────┬──────────────────┘
                         │
         ┌───────────────▼───────────────┐
         │         Subred PÚBLICA         │
         │  ┌─────────────────────────┐  │
         │  │  EC2 t3.small           │  │
         │  │  Servidor de aplicación │  │
         │  │  Ubuntu 22.04 LTS       │  │
         │  │  Nginx + PHP/Python     │  │
         │  │  172.31.10.10           │  │
         │  └────────────┬────────────┘  │
         └───────────────┼───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │         Subred PRIVADA         │
         │                               │
         │  ┌─────────────────────────┐  │
         │  │  RDS MySQL 8.0          │  │  ← espanito_db
         │  │  db.t3.micro            │  │     (11 tablas)
         │  │  Multi-AZ: No (básico)  │  │
         │  │  172.31.20.10           │  │
         │  └─────────────────────────┘  │
         │                               │
         │  ┌─────────────────────────┐  │
         │  │  S3 Bucket              │  │  ← Documentos estudiantes
         │  │  espanito-documentos    │  │     (pasaportes, títulos...)
         │  │  Cifrado AES-256        │  │
         │  └─────────────────────────┘  │
         └───────────────────────────────┘
                         │
         ┌───────────────▼───────────────┐
         │    Servicios de gestión        │
         │  CloudWatch (monitorización)   │
         │  IAM (usuarios y permisos)     │
         │  SNS (alertas por email)       │
         └───────────────────────────────┘
```

### Flujo de una solicitud de admisión

```
Estudiante (Teherán)
      │  HTTPS
      ▼
Route 53 → espanito.com → IP del ALB
      │
      ▼
Application Load Balancer
      │  HTTP interno
      ▼
EC2 t3.small (servidor aplicación)
      │  TCP 3306 (privado)
      ▼
RDS MySQL (espanito_db)
      │  Lee/escribe solicitudes, estudiantes, visados...
      │
      │  S3 API
      ▼
S3 Bucket (documentos)
      │  Almacena/recupera PDFs de pasaportes, diplomas...
```

### Componentes de la arquitectura

**VPC (Virtual Private Cloud)**
Red virtual privada en eu-central-1 con CIDR 172.31.0.0/16. Contiene dos subredes:
- Subred pública (172.31.10.0/24): EC2 con acceso a Internet vía Internet Gateway
- Subred privada (172.31.20.0/24): RDS sin acceso directo desde Internet

**Security Groups (equivalente a firewall)**
- SG-Web: permite entrada en puerto 80/443 desde cualquier IP, salida libre
- SG-App: permite entrada en puerto 8080 solo desde el ALB
- SG-DB: permite entrada en puerto 3306 solo desde el EC2

---

## 3. Servicios cloud utilizados

### EC2 — Amazon Elastic Compute Cloud

**Instancia:** t3.small (2 vCPU, 2 GB RAM)
**Sistema operativo:** Ubuntu Server 22.04 LTS
**Función:** Servidor principal de la aplicación web de Espanito. Ejecuta el servidor web (Nginx), el backend de la aplicación y gestiona las conexiones con la base de datos y el almacenamiento S3.
**Justificación del tamaño:** Para una agencia con 10-50 usuarios concurrentes, t3.small es suficiente. Si el volumen crece, se puede escalar verticalmente a t3.medium sin cambiar la arquitectura.

### RDS — Amazon Relational Database Service

**Motor:** MySQL 8.0 (mismo motor que espanito_db local)
**Instancia:** db.t3.micro (1 vCPU, 1 GB RAM)
**Almacenamiento:** 20 GB SSD gp2 con autoescalado hasta 100 GB
**Función:** Alojar la base de datos espanito_db con sus 11 tablas. RDS gestiona automáticamente las copias de seguridad diarias (retención 7 días), actualizaciones de seguridad y monitorización.
**Ventaja sobre MySQL local:** Backups automáticos, actualizaciones gestionadas, alta disponibilidad configurable.

### S3 — Amazon Simple Storage Service

**Bucket:** espanito-documentos-prod
**Función:** Almacenar todos los documentos de los estudiantes (pasaportes, diplomas, certificados de idioma, cartas de motivación) que actualmente se guardan como rutas de archivo en la tabla documentos de espanito_db.
**Configuración:** Cifrado en reposo (AES-256), control de acceso por carpeta por estudiante, versionado habilitado.
**Integración con XML:** Los archivos XML de exportación de solicitudes (módulo 0373) también se almacenarían en S3 en la ruta `/exports/solicitudes/`.

### Route 53 — DNS gestionado

**Función:** Gestión del dominio espanito.com con resolución DNS de alta disponibilidad. Permite configurar registros A, CNAME y MX para el correo de la agencia.

### CloudFront — CDN

**Función:** Distribuir los archivos estáticos del portal web (imágenes, CSS, JavaScript) desde puntos de presencia en Europa y Oriente Medio, reduciendo la latencia para los estudiantes iraníes que acceden desde Teherán.

### CloudWatch — Monitorización

**Función:** Monitorizar el estado del EC2 (CPU, memoria, red), el rendimiento de RDS (conexiones, latencia de consultas) y el espacio en S3. Configura alertas por email cuando algún indicador supera el umbral definido.

### IAM — Identity and Access Management

**Función:** Gestionar los permisos de acceso a los servicios AWS. Se crean tres roles:
- Rol para la aplicación EC2 (acceso a S3 y RDS)
- Rol para el administrador (acceso completo)
- Rol de solo lectura para informes y auditoría

---

## 4. Estimación de costes

### Costes mensuales estimados (región eu-central-1, Frankfurt)

| Servicio | Especificación | Coste/mes (€) |
|---|---|---|
| EC2 t3.small | 1 instancia, uso 24/7 | 17.20 |
| RDS db.t3.micro MySQL | 1 instancia, 20 GB SSD | 14.80 |
| S3 Standard | 50 GB almacenamiento + 10K peticiones | 1.20 |
| Route 53 | 1 zona alojada + consultas DNS | 0.60 |
| CloudFront | 10 GB transferencia/mes | 0.85 |
| CloudWatch | Métricas básicas + 2 alarmas | 1.00 |
| ALB | 1 balanceador, tráfico bajo | 2.50 |
| Transferencia de datos | ~20 GB salida/mes | 1.80 |
| **TOTAL ESTIMADO** | | **~40 €/mes** |

*Precios aproximados basados en la calculadora de AWS Pricing (abril 2025). Los precios reales pueden variar según el uso.*

### Free Tier disponible (primeros 12 meses)

| Servicio | Incluido gratis |
|---|---|
| EC2 | 750 horas/mes de t2.micro o t3.micro |
| RDS | 750 horas/mes de db.t3.micro |
| S3 | 5 GB de almacenamiento estándar |
| Route 53 | 1 zona alojada el primer mes |
| CloudFront | 1 TB de transferencia/mes |
| CloudWatch | 10 métricas + 3 dashboards |

Durante el primer año, con el free tier activo, el coste efectivo se reduce a aproximadamente **8-12 €/mes** (solo los servicios fuera del free tier como el ALB y la transferencia de datos adicional).

### Comparativa: cloud vs infraestructura local

| Concepto | Cloud (AWS) | Local (servidor físico) |
|---|---|---|
| Inversión inicial | 0 € | 800-1500 € (servidor) |
| Coste mensual | ~40 €/mes | ~15 €/mes (electricidad) |
| Mantenimiento | AWS lo gestiona | Administrador necesario |
| Escalabilidad | Inmediata (clic) | Requiere nuevo hardware |
| Backups | Automáticos | Configuración manual |
| Disponibilidad | 99.99% garantizada | Depende del hardware |
| Coste a 3 años | ~1.440 € | ~2.340 € (hw + mant.) |

**Conclusión:** Para una agencia del tamaño de Espanito, la nube es más económica a largo plazo y elimina la carga de mantenimiento de hardware.

---

## 5. Coherencia con el proyecto global

La arquitectura cloud propuesta integra directamente los entregables de los otros módulos:

| Módulo | Integración cloud |
|---|---|
| 0372 — BBDD | `espanito_db` migra a RDS MySQL — misma estructura, mismas consultas |
| 0373 — Marcas | Los XML de exportación se almacenan en S3 (`/exports/solicitudes/`) |
| 0370 — Redes | La red local de Espanito se conectaría a AWS mediante VPN Site-to-Site |
| 0371 — Hardware | El servidor físico de la sala de redes se sustituye por EC2 |

La migración de local a cloud no requiere cambiar el código ni las consultas SQL — solo cambiar la cadena de conexión de la base de datos de `172.20.10.10:3306` a el endpoint de RDS.
