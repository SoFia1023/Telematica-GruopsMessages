# GroupChat — Entrega 1: Monolito funcional

Proyecto 1 del curso **ST0263 Tópicos Especiales en Telemática / SI3007 Sistemas Distribuidos** — EAFIT 2026-1.

Aplicación de mensajería instantánea inspirada en **Microsoft Teams**: grupos con canales de texto, mensajes directos, archivos compartidos y presencia online. Construida como monolito modular en Spring Boot, con la estructura pensada para migrar a microservicios en la Entrega 2.

> **Entrega 1 — Semana 7 (8 de marzo):** monolito funcional desplegado en AWS.  
> **Entrega 2 — Semana 13 (26 de abril):** arquitectura distribuida con microservicios, Kafka/RabbitMQ, EKS.

---

## Stack

| | |
|---|---|
| **Lenguaje / Framework** | Java 17 + Spring Boot 3.2.5 |
| **Autenticación** | JWT stateless (jjwt 0.12.6) |
| **Mensajería en tiempo real** | WebSocket + STOMP (broker en memoria) |
| **Persistencia** | Spring Data JPA + PostgreSQL 15 |
| **Seguridad** | Spring Security + BCrypt |
| **Frontend** | HTML + CSS + JS vanilla (servido por Spring Boot) |

---

## Estructura del proyecto

El código está organizado por módulo funcional, no por capa técnica. Cada módulo es un candidato directo a microservicio en la Entrega 2.

```
src/main/java/eafit/gruopChat/
├── user/           # Registro, login, gestión de perfil y roles
├── group/          # Grupos, canales, membresías e invitaciones
├── messaging/      # Mensajes de grupo/canal + receipts SENT/DELIVERED/READ
├── dm/             # Conversaciones directas 1-a-1
├── presence/       # Estado online/offline y last seen
├── file/           # Subida y descarga de archivos
├── infrastructure/ # JWT, WebSocket, CORS — configuración transversal
└── shared/         # Enums globales y manejo centralizado de errores
```

---

## Correr el proyecto

**Requisitos:** Java 17+, PostgreSQL corriendo.

```bash
./mvnw spring-boot:run
```

La app queda en `http://localhost:8080`. El frontend está disponible en esa misma URL.

**Con Docker:**

```bash
docker build -t groupchat .
docker run -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://host:5432/groupchatdb \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=groupchat2024 \
  groupchat
```

---

## Configuración (`application.properties`)

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/groupchatdb
spring.datasource.username=postgres
spring.datasource.password=groupchat2024

jwt.secret=clave-secreta-de-al-menos-32-caracteres
jwt.expiration-ms=86400000

spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
```

---

## Autenticación

Todas las rutas requieren el header `Authorization: Bearer <token>` salvo:

| Ruta | Motivo |
|---|---|
| `POST /api/users/register` | Registro público |
| `POST /api/users/login` | Devuelve el JWT |
| `GET /api/groups/invite/*` | Preview de grupo sin cuenta |
| `GET /api/files/**` | Descarga de archivos desde el chat |
| `/`, `/index.html`, `/css/**`, `/js/**` | Frontend estático |

Para WebSocket, el token va en el header `Authorization` del frame STOMP `CONNECT`.

---

## API REST

### Usuarios — `/api/users`

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/register` | Registrar usuario |
| `POST` | `/login` | Login → devuelve JWT |
| `GET` | `/{id}` | Obtener usuario por ID |
| `GET` | `/email/{email}` | Obtener usuario por email |
| `GET` | `` | Listar usuarios activos |
| `PUT` | `/{id}` | Actualizar perfil (solo el dueño) |
| `PATCH` | `/{id}/password` | Cambiar contraseña (solo el dueño) |
| `PATCH` | `/{id}/disable` | Deshabilitar cuenta |
| `PATCH` | `/{id}/enable` | Habilitar cuenta |
| `PATCH` | `/{id}/role?role=` | Cambiar rol (`ROLE_USER` / `ROLE_ADMIN`) |
| `DELETE` | `/{id}` | Eliminar cuenta (soft delete) |

### Grupos — `/api/groups`

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `` | Crear grupo |
| `GET` | `` | Mis grupos |
| `GET` | `/{groupId}` | Detalle de grupo |
| `PUT` | `/{groupId}` | Editar grupo (ADMIN) |
| `DELETE` | `/{groupId}` | Eliminar grupo (solo el creador) |
| `GET` | `/invite/{code}` | Preview público por código de invitación |
| `POST` | `/invite/{code}/join` | Unirse por código |
| `GET` | `/{groupId}/members` | Listar miembros |
| `DELETE` | `/{groupId}/members/{userId}` | Expulsar miembro (ADMIN) |
| `PATCH` | `/{groupId}/members/{userId}/role?role=` | Cambiar rol en el grupo |
| `POST` | `/{groupId}/leave` | Abandonar grupo |
| `POST` | `/{groupId}/invitations?invitedUserId=` | Invitar usuario (ADMIN) |
| `PATCH` | `/invitations/{id}?accept=true\|false` | Aceptar o rechazar invitación |
| `GET` | `/invitations/pending` | Mis invitaciones pendientes |
| `POST` | `/{groupId}/channels` | Crear canal (ADMIN) |
| `GET` | `/{groupId}/channels` | Listar canales |
| `PUT` | `/channels/{channelId}` | Editar canal (ADMIN) |
| `DELETE` | `/channels/{channelId}` | Eliminar canal (ADMIN) |

### Mensajes — `/api/messages`

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/channel/{channelId}?page=0&size=50` | Historial de canal (paginado) |
| `GET` | `/group/{groupId}?page=0&size=50` | Historial del grupo general (paginado) |
| `PATCH` | `/{messageId}?content=` | Editar mensaje propio |
| `DELETE` | `/{messageId}` | Eliminar mensaje propio (soft delete) |
| `GET` | `/channel/{channelId}/files` | Archivos compartidos en el canal |
| `GET` | `/group/{groupId}/files` | Archivos compartidos en el grupo |

### Mensajes directos — `/api/dm`

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/start?targetEmail=` | Iniciar conversación |
| `GET` | `` | Mis conversaciones |
| `GET` | `/requests` | Solicitudes entrantes pendientes |
| `POST` | `/{id}/accept` | Aceptar solicitud |
| `DELETE` | `/{id}/decline` | Rechazar solicitud |
| `GET` | `/{id}/messages?page=0&size=50` | Historial (paginado) |
| `DELETE` | `/messages/{messageId}` | Eliminar mensaje |

### Archivos — `/api/files`

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/upload` | Subir archivo (`multipart/form-data`, campo `file`, máx 10 MB) |
| `GET` | `/{fileId}` | Descargar o previsualizar archivo |

### Presencia — `/api/presence`

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `/group/{groupId}` | Estado online/offline de los miembros del grupo |

---

## WebSocket (STOMP)

**Endpoint:** `ws://localhost:8080/ws` (SockJS fallback incluido)

### El cliente publica en:

| Destino | Descripción |
|---|---|
| `/app/chat.send` | Enviar mensaje a grupo o canal |
| `/app/chat.read` | Marcar mensaje como leído |
| `/app/dm.send` | Enviar mensaje directo |

### El cliente se suscribe a:

| Topic | Descripción |
|---|---|
| `/topic/channel.{channelId}` | Mensajes nuevos de un canal |
| `/topic/group.{groupId}` | Mensajes del área general del grupo |
| `/topic/receipts.channel.{channelId}` | Actualizaciones de lectura (canal) |
| `/topic/receipts.group.{groupId}` | Actualizaciones de lectura (grupo) |
| `/topic/presence.{groupId}` | Cambios de presencia (online/offline) |
| `/topic/dm.{conversationId}` | Mensajes directos entrantes |
| `/topic/dm.requests.{userId}` | Solicitudes DM nuevas |

---

## Base de datos

Esquema generado automáticamente con `ddl-auto=update`.

| Tabla | Descripción |
|---|---|
| `users` | Usuarios |
| `groups` | Grupos |
| `group_members` | Membresías con rol (`ADMIN` / `MEMBER`) |
| `channels` | Canales dentro de un grupo |
| `group_invitations` | Invitaciones directas |
| `messages` | Mensajes de grupo y canal |
| `message_receipts` | Confirmaciones de lectura (`SENT` → `DELIVERED` → `READ`) |
| `direct_conversations` | Conversaciones 1-a-1 |
| `dm_messages` | Mensajes directos |
| `files` | Archivos (Base64 en DB — se migra a S3 en Entrega 2) |
| `user_presence` | Último `last_seen` por usuario |

---

## Documentación

- `decisiones_diseno.docx` — Por qué lo construimos así y cómo escala para la Entrega 2.
- `documentacion_tecnica.docx` — Referencia completa de modelos, DTOs y endpoints.

---

## Uso de IA

Todo el diseño de arquitectura, la planeación, el levantamiento de requisitos, la investigación de herramientas y tecnologías, y las decisiones técnicas fueron realizados por el equipo. El código fue desarrollado por nosotras con apoyo de Claude (Anthropic) para agilizar partes operativas, y depurado y validado por el equipo. La documentación fue escrita principalmente por nosotras y apoyada en redacción por Claude y ChatGPT.

---

**ST0263 / SI3007 — EAFIT 2026-1**
