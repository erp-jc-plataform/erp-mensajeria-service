# Business-Mensajeria

Microservicio de envio de emails y SMS para Business ERP. Gestiona el envio de mensajes a traves de multiples proveedores (Nodemailer/SMTP, SendGrid, Twilio, AWS SNS), procesa envios en cola con BullMQ y persiste el historial en MongoDB. Desarrollado con TypeScript, Express y arquitectura limpia (Hexagonal + Event-Driven).

---

## Lenguaje y Stack Tecnologico

| Capa | Tecnologia | Version |
|------|-----------|---------|
| Lenguaje | TypeScript | 5.3.2 |
| Runtime | Node.js | >= 18 |
| Framework HTTP | Express | 4.18.2 |
| Base de datos | MongoDB | 6.3.0 |
| Cola de mensajes | BullMQ | 5.0.0 |
| Cache / Broker | Redis (ioredis) | 5.3.2 |
| Email SMTP | Nodemailer | 6.9.7 |
| Email Cloud | SendGrid | 8.1.0 |
| SMS | Twilio | 4.19.0 |
| SMS alternativo | AWS SNS (SDK v3) | 3.460.0 |
| Plantillas | Handlebars | 4.7.8 |
| Mensajeria async | KafkaJS | 2.2.4 |
| Validacion | Zod | 3.22.4 |
| Logging | Winston | 3.11.0 |
| Dev server | nodemon | 3.0.2 |
| Puerto | 3006 | - |

---

## Caracteristicas

- Envio de emails individuales y en lote via SMTP o SendGrid
- Envio de SMS individuales y en lote via Twilio o AWS SNS
- Sistema de colas con BullMQ — envios procesados asincrona y confiablemente
- Workers independientes: EmailWorker y SMSWorker (pueden correr como proceso separado)
- Reintentos automaticos de mensajes fallidos con backoff exponencial (3 intentos)
- Plantillas Handlebars — emails HTML pre-disenados (bienvenida, codigo verificacion)
- Envio de email con reporte adjunto — integracion con Business-Report
- Historial completo en MongoDB (estado, timestamps, trazabilidad, errores)
- Priorizacion de mensajes: URGENT, HIGH, NORMAL, LOW
- Soporte Kafka para consumir eventos de otros microservicios

---

## Estructura del Proyecto

```
Business-Mensajeria/
├── src/
│   ├── index.ts                              # Punto de entrada
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── Message.ts                    # Entidad base de mensaje
│   │   │   ├── EmailMessage.ts               # Entidad email
│   │   │   └── SMSMessage.ts                 # Entidad SMS
│   │   ├── repositories/
│   │   │   └── IMessageRepository.ts         # Contrato de persistencia
│   │   └── services/
│   │       ├── MessageDomainService.ts
│   │       └── TemplateService.ts            # Motor Handlebars
│   ├── application/
│   │   ├── usecases/
│   │   │   ├── SendEmail.ts
│   │   │   ├── SendSMS.ts
│   │   │   ├── QueryMessages.ts
│   │   │   ├── RetryFailedMessage.ts
│   │   │   └── SendEmailWithReport.ts
│   │   └── dto/MessageDTO.ts
│   ├── infrastructure/
│   │   ├── database/mongodb/
│   │   │   └── MongoMessageRepository.ts
│   │   ├── providers/
│   │   │   ├── email/
│   │   │   │   ├── NodemailerProvider.ts     # SMTP
│   │   │   │   └── SendGridProvider.ts       # Cloud email
│   │   │   └── sms/
│   │   │       ├── TwilioProvider.ts
│   │   │       └── AWSSNSProvider.ts
│   │   ├── queue/bullmq/
│   │   │   ├── EmailQueue.ts
│   │   │   ├── SMSQueue.ts
│   │   │   └── workers/
│   │   │       ├── EmailWorker.ts
│   │   │       └── SMSWorker.ts
│   │   ├── clients/
│   │   │   └── ReportServiceClient.ts        # HTTP hacia Business-Report
│   │   └── http/express/
│   │       ├── routes.ts
│   │       └── middleware/auth.middleware.ts
│   └── shared/
│       ├── config/config.ts
│       └── utils/logger.ts
├── templates/
│   ├── welcome.hbs                           # Email de bienvenida
│   └── verification-code.hbs                # Email con OTP
├── docker-compose.yml                        # MongoDB + Redis + Kafka
├── Dockerfile
├── nodemon.json
├── package.json
└── tsconfig.json
```

---

## Instalacion

### Requisitos previos

- Node.js >= 18
- MongoDB (local o Atlas)
- Redis >= 6
- Credenciales de algun proveedor de email (SMTP/Gmail o SendGrid)
- Opcional: credenciales Twilio o AWS SNS para SMS

### Pasos

```powershell
# 1. Entrar al directorio
cd C:\Proyectos\BusinessApp\Business-Mensajeria

# 2. Instalar dependencias
npm install

# 3. Configurar variables de entorno
copy .env.example .env
# Editar .env con tus credenciales
```

### Levantar infraestructura con Docker

```powershell
# Levanta MongoDB + Redis (minimo requerido)
docker-compose up -d
```

---

## Variables de entorno (.env)

```env
# Servidor
PORT=3006
NODE_ENV=development

# MongoDB
MONGODB_URI=mongodb://localhost:27017/business_mensajeria
MONGODB_DB_NAME=business_mensajeria

# Redis (BullMQ)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Email — opcion A: SMTP (Gmail, Outlook, etc.)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=tu-email@gmail.com
SMTP_PASSWORD=tu-app-password
EMAIL_FROM=noreply@businessapp.com

# Email — opcion B: SendGrid
SENDGRID_API_KEY=tu-api-key-sendgrid

# SMS — opcion A: Twilio
TWILIO_ACCOUNT_SID=tu-account-sid
TWILIO_AUTH_TOKEN=tu-auth-token
TWILIO_PHONE_NUMBER=+1234567890

# SMS — opcion B: AWS SNS
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=tu-access-key
AWS_SECRET_ACCESS_KEY=tu-secret-key

# Integracion Business-Report
REPORT_SERVICE_URL=http://localhost:3008
REPORT_SERVICE_API_KEY=tu-api-key-report

# Colas
QUEUE_MAX_RETRIES=3
QUEUE_RETRY_DELAY_MS=60000

# Kafka (opcional)
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=business-mensajeria
```

---

## Levantar el Microservicio

### Desarrollo

```powershell
cd C:\Proyectos\BusinessApp\Business-Mensajeria
npm run dev
```

Arranca en http://localhost:3006 con nodemon (hot-reload). Los workers se inician integrados.

### Workers independientes (produccion recomendada)

```powershell
# Solo workers (separado del API server)
npm run worker

# Workers con hot-reload
npm run worker:dev
```

### Produccion

```powershell
npm run build
npm start
```

### Verificar que esta corriendo

```powershell
Invoke-RestMethod -Uri http://localhost:3006/health
```

---

## URLs Disponibles

| URL | Descripcion |
|-----|-------------|
| http://localhost:3006/health | Health check del servicio |
| http://localhost:3006/api/messages | API de mensajes |

---

## Endpoints de la API

Todos los endpoints requieren autenticacion: `Authorization: Bearer <token>` o `x-api-key: <key>`.

### Emails

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| POST | /api/messages/email | Enviar email individual |
| POST | /api/messages/email/batch | Enviar emails en lote |
| POST | /api/messages/email/with-report | Enviar email con reporte adjunto |

### SMS

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| POST | /api/messages/sms | Enviar SMS individual |
| POST | /api/messages/sms/batch | Enviar SMS en lote |

### Historial y reintentos

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| GET | /api/messages | Consultar historial (paginado) |
| GET | /api/messages/:id | Obtener mensaje por ID |
| GET | /api/messages/:id/status | Estado de un mensaje |
| POST | /api/messages/:id/retry | Reintentar mensaje fallido |
| POST | /api/messages/retry/all | Reintentar todos los fallidos |

### Sistema

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| GET | /health | Health check |

---

## Ejemplos de Request

### Enviar email con plantilla

```json
POST /api/messages/email
{
  "to": "usuario@empresa.com",
  "subject": "Bienvenido al sistema",
  "template": "welcome",
  "templateData": { "nombre": "Juan", "empresa": "Acme SA" }
}
```

### Enviar SMS

```json
POST /api/messages/sms
{
  "to": "+593999999999",
  "body": "Tu codigo de verificacion es: 123456"
}
```

### Email con reporte adjunto

```json
POST /api/messages/email/with-report
{
  "to": "gerente@empresa.com",
  "subject": "Reporte mensual de ventas",
  "reportId": "uuid-del-reporte-generado",
  "reportFormat": "pdf",
  "template": "report-delivery"
}
```

---

## Arquitectura de Colas

```
HTTP Request
    |
    v
API (Express) -- enqueue --> EmailQueue / SMSQueue (BullMQ + Redis)
                                    |
                          EmailWorker / SMSWorker
                                    |
                         +----------+----------+
                         |                     |
                  Nodemailer / SendGrid    Twilio / AWS SNS
                         |                     |
                         v                     v
                  MongoDB (historial: status, timestamps, errores)
```

Estados de mensaje: `pending` → `queued` → `sending` → `sent` / `failed`

---

## Plantillas Handlebars

Los archivos `.hbs` en la carpeta `templates/` son plantillas HTML reutilizables:

| Archivo | Uso |
|---------|-----|
| welcome.hbs | Email de bienvenida al registrar usuario |
| verification-code.hbs | Email con codigo OTP o verificacion |

Para agregar una nueva plantilla, crear `templates/mi-plantilla.hbs` y referenciarla con `"template": "mi-plantilla"`.

---

## Scripts npm

| Comando | Descripcion |
|---------|-------------|
| npm run dev | Desarrollo con nodemon (hot-reload) |
| npm run build | Compilar TypeScript a dist/ |
| npm start | Ejecutar compilado (produccion) |
| npm run worker | Levantar solo los workers de BullMQ |
| npm run worker:dev | Workers con hot-reload |
| npm test | Tests con Jest |
| npm run lint | Linting ESLint |
| npm run format | Formatear con Prettier |

---

## Docker

```powershell
# Levantar MongoDB + Redis + Kafka
docker-compose up -d

# Ver logs
docker-compose logs -f

# Detener
docker-compose down
```

---

## Licencia

Proyecto interno — Business ERP.
