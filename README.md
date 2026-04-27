# MPO — Fundamentos de Computación en la Nube
## Agencia Espanito — Arquitectura Cloud en AWS

**Proyecto Intermodular · 1º ASIR · Yassin Mansouri · Prometeo by ThePower**

---

## ¿Qué es este módulo?

Diseño y análisis de la arquitectura cloud para la agencia Espanito en **Amazon Web Services (AWS)**, región **eu-central-1 (Frankfurt)**. El objetivo no es desplegar la infraestructura completa, sino demostrar cómo se desplegaría en un entorno profesional real, con una estimación de costes justificada.

---

## Proveedor elegido: Amazon Web Services (AWS)

AWS se elige frente a Google Cloud, Azure y DigitalOcean por tres razones principales:

- **Región eu-central-1 (Frankfurt):** baja latencia para estudiantes en Europa e Irán
- **Free Tier 12 meses:** permite arrancar con coste prácticamente cero el primer año (~8-12 €/mes)
- **Liderazgo de mercado:** mayor ecosistema de servicios gestionados y documentación

---

## Arquitectura propuesta

La arquitectura sigue el patrón de **tres capas** dentro de una VPC privada con subredes pública y privada:

```
INTERNET / USUARIOS
        │ HTTPS (443)
        ▼
    Route 53          → DNS: espanito.com
        │
    CloudFront        → CDN: archivos estáticos, imágenes, assets
        │
Application Load Balancer (ALB)   → HTTP → HTTPS · Health checks
        │
┌── SUBRED PÚBLICA (172.31.0.0/24) ─────────────────────┐
│   EC2 t3.small — 172.31.10.10                          │
│   Ubuntu 22.04 LTS · Nginx + PHP/Python                │
│   Servidor de aplicación                               │
└────────────────────────┬───────────────────────────────┘
                         │ TCP 3306 (privado)
┌── SUBRED PRIVADA (172.31.20.0/24) ─────────────────────┐
│   RDS MySQL 8.0 — 172.31.20.10                         │
│   db.t3.micro · espanito_db (11 tablas)                │
│                                                        │
│   S3 Bucket: espanito-documentos                       │
│   Pasaportes, diplomas, certificados · AES-256         │
└────────────────────────────────────────────────────────┘
        │
Servicios de gestión: CloudWatch · IAM · SNS · AWS Shield
```

El diagrama completo está disponible en `Diagram.png`.

---

## Servicios cloud utilizados

| Servicio | Especificación | Función |
|---|---|---|
| **EC2 t3.small** | 2 vCPU, 2 GB RAM, Ubuntu 22.04 | Servidor web + backend de la aplicación |
| **RDS MySQL 8.0** | db.t3.micro, 20 GB SSD | Base de datos espanito_db (11 tablas) |
| **S3** | Standard, cifrado AES-256, versionado | Documentos de estudiantes (PDF, imágenes) |
| **Route 53** | 1 zona alojada | DNS gestionado para espanito.com |
| **CloudFront** | CDN global | Distribución de contenido estático |
| **ALB** | Application Load Balancer público | Balanceo + redirección HTTP→HTTPS |
| **CloudWatch** | Métricas + alarmas | Monitorización EC2, RDS y S3 |
| **IAM** | 3 roles definidos | Gestión de identidades y permisos |
| **SNS** | Alertas por email | Notificaciones de eventos y alarmas |
| **AWS Shield** | Estándar (incluido) | Protección DDoS automática |
| **AWS Backup** | Retención 7 días | Backups automáticos de RDS y S3 |

---

## Estimación de costes mensuales

### Costes en producción (región eu-central-1)

| Servicio | Especificación | Coste/mes (€) |
|---|---|---|
| EC2 t3.small | 1 instancia, uso 24/7 | 17,20 |
| RDS db.t3.micro MySQL | 1 instancia, 20 GB SSD | 14,80 |
| S3 Standard | 50 GB + 10K peticiones | 1,20 |
| Route 53 | 1 zona + consultas DNS | 0,60 |
| CloudFront | 10 GB transferencia/mes | 0,85 |
| CloudWatch | Métricas básicas + 2 alarmas | 1,00 |
| ALB | 1 balanceador, tráfico bajo | 2,50 |
| Transferencia de datos | ~20 GB salida/mes | 1,80 |
| **TOTAL ESTIMADO** | | **~40 €/mes** |

*Basado en AWS Pricing Calculator — abril 2025.*

### Con Free Tier (primeros 12 meses)

Coste efectivo reducido a **~8-12 €/mes** — solo se pagan los servicios fuera del free tier (ALB y transferencia adicional).

### Cloud vs infraestructura local — comparativa a 3 años

| Concepto | Cloud (AWS) | Local (servidor físico) |
|---|---|---|
| Inversión inicial | 0 € | 800–1.500 € |
| Coste mensual | ~40 €/mes | ~15 €/mes (electricidad) |
| Mantenimiento | AWS lo gestiona | Administrador necesario |
| Escalabilidad | Inmediata (clic) | Requiere nuevo hardware |
| Backups | Automáticos | Configuración manual |
| Disponibilidad | 99,99% garantizada | Depende del hardware |
| **Coste total 3 años** | **~1.440 €** | **~2.340 € (hw + mant.)** |

---

## Security Groups (firewall cloud)

| Security Group | Puerto | Origen permitido |
|---|---|---|
| SG-Web (ALB) | 80, 443 | 0.0.0.0/0 (todo Internet) |
| SG-App (EC2) | 8080 | Solo desde ALB |
| SG-App (EC2) | 22 (SSH) | Solo IP admin |
| SG-DB (RDS) | 3306 | Solo desde EC2 (SG-App) |

---

## Integración con el proyecto global

La arquitectura cloud es coherente con todos los módulos del Proyecto Intermodular:

| Módulo | Integración |
|---|---|
| **0372 — Bases de Datos** | `espanito_db` migra a RDS MySQL — misma estructura, mismas 11 tablas, mismas consultas SQL |
| **0373 — Lenguajes de Marcas** | Los XML de exportación se almacenan en S3 en la ruta `/exports/solicitudes/` |
| **0370 — Redes** | La red local de Espanito se conecta a AWS mediante VPN Site-to-Site |
| **0371 — Hardware** | El servidor físico de la sala de redes se sustituye por la instancia EC2 |

> La migración de local a cloud **no requiere cambiar el código ni las consultas SQL** — solo la cadena de conexión de la base de datos: de `172.20.10.10:3306` al endpoint de RDS.

---

## Archivos incluidos

| Archivo | Descripción |
|---|---|
| `Diagram.png` | Diagrama completo de la arquitectura cloud en AWS |
| `cloud_espanito.md` | Análisis completo: proveedor, arquitectura, servicios y costes |
| `README_cloud.md` | Este archivo |
