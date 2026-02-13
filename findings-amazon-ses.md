# Amazon SES (Simple Email Service) - Investigación Técnica POC

> **Provider**: Amazon Web Services
> **Service**: Simple Email Service (SES)
> **Fecha de investigación**: 2025 (basado en documentación y pricing actual de SES)
> **Contexto**: Evaluar SES para recibir emails en `info@caminosdelassierras.com.ar` (Google Workspace + Cloudflare DNS, DMARC `p=reject`), procesar en backend NestJS/Node.js en AWS, y responder como `@caminosdelassierras.com.ar`.

---

## Challenge A - Envío y Recepción

### A1. Recepción de Email Inbound

#### Cómo SES recibe email inbound

SES puede actuar como servidor de correo inbound mediante **SES Email Receiving** (también llamado "SES Receipt" o "SES Inbound"). El mecanismo funciona así:

1. Configuras SES para aceptar correo para un dominio verificado o direcciones de email específicas.
2. Creas **Receipt Rule Sets** que contienen una o más **Receipt Rules** que definen qué ocurre cuando llega el correo.
3. SES procesa el email entrante a través de las reglas y ejecuta las acciones configuradas.

#### Requisito de registro MX

**Sí, el email inbound de SES REQUIERE registros MX que apunten a SES.** El registro MX debe apuntar al endpoint SMTP inbound de SES para la región que uses:

| Región | Valor del registro MX |
|---|---|
| us-east-1 | `inbound-smtp.us-east-1.amazonaws.com` |
| us-west-2 | `inbound-smtp.us-west-2.amazonaws.com` |
| eu-west-1 | `inbound-smtp.eu-west-1.amazonaws.com` |

El formato del registro MX es:
```
10 inbound-smtp.<region>.amazonaws.com
```

**LIMITACIÓN CRÍTICA**: El email inbound de SES solo está disponible en **tres regiones**:
- `us-east-1` (N. Virginia)
- `us-west-2` (Oregon)
- `eu-west-1` (Ireland)

#### ¿Puede funcionar con forwarding desde Google Workspace en lugar de cambiar MX?

**Sí, pero con consideraciones importantes.** Hay dos enfoques:

**Enfoque 1: Forwarding de Google Workspace (Recomendado para este escenario)**

En lugar de cambiar los registros MX, puedes configurar forwarding en Google Workspace:

1. Configura una **routing rule** en Google Admin Console (o un filtro de Gmail en el buzón `info@`) para reenviar los emails entrantes a una dirección que SES pueda procesar.
2. Sin embargo, SES en sí no puede "recibir" estos emails reenviados a menos que los registros MX del *dominio de destino del forwarding* apunten a SES. El patrón típico sería:
   - Mantener los registros MX apuntando a Google (`aspmx.l.google.com`)
   - Reenviar desde Google a un subdominio que controles, p. ej. `inbound@mail.yourdomain.com`
   - Apuntar el MX de `mail.yourdomain.com` a SES
   - Las Receipt Rules de SES procesan el email reenviado

**Enfoque 2: Omitir SES Inbound por completo (Más práctico)**

Dado que Google Workspace ya recibe el email, puedes omitir SES inbound por completo:

1. Configura una routing rule de Google Workspace o filtro de Gmail para reenviar a un topic de SNS vía un intermediario (p. ej. un endpoint webhook en tu backend NestJS, o a un puente Google Pub/Sub -> Lambda).
2. **Alternativa mejor**: Usa la **Gmail API con push notifications** (Google Pub/Sub) para detectar nuevos emails y obtenerlos programáticamente desde tu backend NestJS.
3. **Aún más simple**: Configura Google Workspace para reenviar a una Lambda function URL o endpoint de API Gateway que procese el email raw.

**Arquitectura recomendada para esta POC** (evitando cambios de MX):

```mermaid
flowchart TD
    GW["Google Workspace<br/>info@caminosdelassierras.com.ar"] -->|Gmail forwarding rule| APIGW["API Gateway / Lambda Function URL"]
    APIGW --> Lambda["Lambda function<br/>parses raw email"]
    Lambda --> SQS["SNS/SQS -> NestJS Backend<br/>processes and stores"]
    SQS --> SES["SES<br/>sends reply as @caminosdelassierras.com.ar"]
```

Alternativamente, el enfoque más simple:

```mermaid
flowchart TD
    GW["Google Workspace<br/>info@caminosdelassierras.com.ar"] -->|"Gmail forwarding to ses-inbound@yourcompanydomain.com"| SESIn["SES Inbound<br/>MX for yourcompanydomain.com -> SES"]
    SESIn -->|"Receipt Rule: Store to S3 + notify SNS"| S3["S3 raw email + SNS -> SQS -> NestJS Backend"]
    S3 --> SESOut["SES Outbound<br/>reply as @caminosdelassierras.com.ar"]
```

#### Impacto del forwarding en DMARC/SPF

Cuando Google reenvía un email, el SPF del remitente original puede fallar (porque la IP de Google no está en el SPF del remitente original). Sin embargo, **las firmas DKIM típicamente sobreviven al forwarding** si el cuerpo/headers del mensaje no se modifican. Esto es relevante si quieres validar la autenticidad del remitente original, pero para este caso de uso (solo quieres recibir y procesar el contenido), no es un bloqueo.

---

### A2. SES Receipt Rules

#### Cómo funcionan

Las Receipt Rules se organizan en **Receipt Rule Sets**. Solo un rule set puede estar **activo** a la vez. Dentro de un rule set, las reglas se evalúan en orden.

Cada Receipt Rule tiene:
- **Recipients**: Direcciones de email o dominios a los que aplica la regla (p. ej. `info@yourdomain.com` o `yourdomain.com`)
- **Actions**: Lista ordenada de acciones a ejecutar (hasta varias por regla)
- **TLS requirement**: Opción de requerir TLS para conexiones inbound
- **Spam/virus scanning**: Opción de habilitar escaneo

#### Acciones disponibles

| Acción | Descripción |
|---|---|
| **S3 Action** | Almacena el email raw (formato MIME) en un bucket S3. Opcionalmente puede usar un prefijo de key y cifrado KMS. |
| **SNS Action** | Publica una notificación en un topic de SNS. Puede incluir el contenido completo del email (hasta 150 KB) o solo metadata. |
| **Lambda Action** | Invoca una función Lambda de forma síncrona o asíncrona. La Lambda recibe el evento del email y puede controlar el flujo de correo (STOP_RULE, STOP_RULE_SET, CONTINUE). |
| **Bounce Action** | Envía una respuesta bounce al remitente. |
| **Stop Action** | Detiene el procesamiento de más reglas en el rule set. |
| **WorkMail Action** | Reenvía a Amazon WorkMail. |
| **Add Header Action** | Añade un header personalizado al email. |

#### Restricciones importantes

- **Límite de tamaño de email**: SES puede recibir emails de hasta **40 MB** (el mensaje MIME). Para la acción S3, el mensaje completo se almacena. Para la acción SNS, si el mensaje supera **150 KB**, la notificación SNS contendrá solo la metadata (headers, etc.) y debes obtener el contenido completo desde S3.
- **El dominio o dirección de email debe estar verificado** en SES para que las receipt rules funcionen.
- **Las receipt rules solo funcionan en las 3 regiones soportadas para inbound** (us-east-1, us-west-2, eu-west-1).

---

### A3. Envío desde dominio externo - Requisitos DNS

Para enviar emails como `@caminosdelassierras.com.ar` vía SES, necesitas:

#### Paso 1: Verificar el dominio en SES

Añade `caminosdelassierras.com.ar` como identidad verificada en SES. Esto requiere añadir un **registro TXT** al DNS. Alternativamente, con SES v2, la verificación de dominio se hace vía DKIM (ver abajo) — añadir los 3 registros CNAME de DKIM sirve tanto para verificación de dominio como para setup de DKIM simultáneamente.

#### Paso 2: Configuración DKIM (Easy DKIM)

SES genera **tres registros CNAME** que deben añadirse al DNS. Los selectores reales son tokens únicos generados por SES (p. ej. `abcdef1234ghijkl._domainkey...`). Los valores exactos se proporcionan en la consola de SES o en la respuesta de la API cuando inicias la verificación del dominio.

**Detalle clave**: Los registros CNAME apuntan a `*.dkim.amazonses.com`, lo que permite a AWS **rotar automáticamente las claves DKIM** sin requerir cambios en DNS. Esta es una ventaja significativa frente a proveedores que requieren que gestiones las claves manualmente.

#### Paso 3: SPF vía Custom MAIL FROM Domain

SES envía emails usando su propio dominio MAIL FROM por defecto (p. ej. `amazonses.com`). La verificación SPF pasa contra `amazonses.com`, pero la alineación SPF con `caminosdelassierras.com.ar` falla.

**Con DMARC `p=reject`, el fallo de alineación SPF es aceptable SIEMPRE QUE la alineación DKIM pase.** DMARC requiere alineación SPF O alineación DKIM, no ambas.

Sin embargo, para máxima deliverability, deberías configurar un **Custom MAIL FROM domain**:

1. Elige un subdominio, p. ej. `mail.caminosdelassierras.com.ar`
2. Añade un registro MX para el subdominio apuntando a `feedback-smtp.<region>.amazonses.com`
3. Añade un registro SPF para el subdominio: `v=spf1 include:amazonses.com ~all`

**Importante para este dominio**: El registro SPF actual es `v=spf1 include:_spf.google.com ~all`. Si usas un Custom MAIL FROM subdomain (p. ej. `mail.caminosdelassierras.com.ar`), NO necesitas modificar el registro SPF del dominio raíz. La verificación SPF ocurre contra el dominio MAIL FROM (el subdominio), no contra el dominio `From` del header.

**Sin embargo**, algunos servidores receptores pueden verificar también el SPF del dominio raíz como señal secundaria. Por precaución, podrías añadir `include:amazonses.com` al SPF raíz: `v=spf1 include:_spf.google.com include:amazonses.com ~all`

#### Resumen de registros DNS necesarios

| Tipo de registro | Nombre | Valor | Propósito |
|---|---|---|---|
| CNAME | `token1._domainkey.caminosdelassierras.com.ar` | `token1.dkim.amazonses.com` | DKIM (1 de 3) |
| CNAME | `token2._domainkey.caminosdelassierras.com.ar` | `token2.dkim.amazonses.com` | DKIM (2 de 3) |
| CNAME | `token3._domainkey.caminosdelassierras.com.ar` | `token3.dkim.amazonses.com` | DKIM (3 de 3) |
| MX | `mail.caminosdelassierras.com.ar` | `10 feedback-smtp.us-east-1.amazonses.com` | Custom MAIL FROM |
| TXT | `mail.caminosdelassierras.com.ar` | `v=spf1 include:amazonses.com ~all` | SPF para MAIL FROM |
| TXT (opcional) | `caminosdelassierras.com.ar` | Modificar SPF existente para añadir `include:amazonses.com` | SPF raíz (opcional) |

**Total de cambios DNS**: 5 registros obligatorios + 1 modificación opcional. Todos pueden añadirse en Cloudflare.

---

### A4. ¿Puede SES enviar desde un dominio que no posees?

**SES requiere verificación de dominio vía registros DNS.** No puedes enviar desde un dominio a menos que puedas añadir los registros de verificación (CNAMEs de DKIM o token de verificación TXT) al DNS de ese dominio.

En este caso, la empresa no posee `caminosdelassierras.com.ar`, pero necesitan la cooperación de quien gestione el DNS de Cloudflare para añadir los registros. **Sin acceso a DNS, SES no puede enviar desde este dominio.**

El proceso de verificación:
1. Ir a SES Console -> Verified Identities -> Create Identity
2. Elegir "Domain" e introducir `caminosdelassierras.com.ar`
3. Habilitar Easy DKIM (recomendado: longitud de clave DKIM 2048-bit)
4. Opcionalmente configurar Custom MAIL FROM domain
5. SES proporciona los registros DNS a añadir
6. Añadir registros en Cloudflare
7. SES verifica periódicamente el DNS y marca el dominio como "Verified" una vez que los registros se propagan (normalmente minutos a horas)

**El dominio permanece verificado mientras existan los registros DNS.** Si se eliminan los registros, SES revoca la verificación tras un período.

---

### A5. Manejo de Reply-To

SES te da **control total** sobre el header `Reply-To`:

- Al enviar vía API de SES (`SendEmail`, `SendRawEmail`, o v2 `SendEmail`), puedes establecer el header `Reply-To` a cualquier dirección de email.
- El header `Reply-To` NO necesita ser una identidad verificada en SES.
- El header `From` DEBE ser una identidad verificada.

**Caso de uso para esta POC**: Establecer `From` como `info@caminosdelassierras.com.ar` y `Reply-To` a la misma dirección o una dirección de routing. También podrías establecer Reply-To a una dirección de tracking única (p. ej. `case-12345@inbound.yourdomain.com`) para enrutar las respuestas de vuelta a tu sistema manteniendo la apariencia de enviar desde el dominio del cliente.

---

### A6. DKIM - Easy DKIM

#### Cómo funciona Easy DKIM

1. Cuando verificas un dominio con Easy DKIM habilitado, SES genera un **par de claves DKIM** (pública/privada).
2. SES almacena la clave privada internamente y la usa para firmar cada email outbound de ese dominio.
3. La clave pública se publica vía los 3 registros CNAME que añades al DNS.
4. Los registros CNAME apuntan a `*.dkim.amazonses.com`, donde AWS aloja los registros TXT DKIM reales (las claves públicas).
5. Cuando un servidor receptor recibe el email, busca la firma DKIM, sigue el CNAME para obtener la clave pública, y verifica la firma.

#### Características principales

- **Rotación automática de claves**: Como el CNAME apunta a registros alojados en AWS, AWS puede rotar claves sin que cambies el DNS.
- **Opciones de longitud de clave**: Claves RSA de 1024-bit o 2048-bit. **Se recomienda 2048-bit.**
- **El signing DKIM es automático**: Cada email enviado desde el dominio verificado a través de SES se firma DKIM automáticamente. No se necesitan cambios de código.
- **Alineación DKIM**: El dominio `d=` en la firma DKIM coincide con `caminosdelassierras.com.ar`, por lo que la alineación DMARC DKIM pasa.

#### BYODKIM (Bring Your Own DKIM)

Como alternativa, SES también soporta BYODKIM donde proporcionas tu propio par de claves. Es útil si necesitas selectores específicos o quieres gestionar tus propias claves. Para esta POC, Easy DKIM es más simple y recomendado.

---

## Challenge B - Thread Tracking

### B1. Message-ID

**Sí, SES genera automáticamente un header `Message-ID`** para cada email enviado. El formato es `<unique-id@email.amazonses.com>`, o si usas un Custom MAIL FROM domain, puede aparecer como `<unique-id@mail.caminosdelassierras.com.ar>`.

**Importante**: SES también devuelve un **SES Message ID** en la respuesta de la API (diferente del header RFC Message-ID). Puedes correlacionar el SES Message ID con el RFC Message-ID en los headers del email. SES usa el formato `Message-ID: <SES-Message-ID@email.amazonses.com>`. Esto significa que la respuesta de la API de SES te da suficiente información para construir o predecir el valor del header `Message-ID`.

### B2. Headers personalizados (In-Reply-To, References)

**Sí, SES soporta completamente headers personalizados.** Hay dos formas:

#### Opción 1: Usando `SendRawEmail` / v2 `SendEmail` con contenido Raw

Construyes todo el mensaje MIME tú mismo, incluyendo todos los headers (From, To, Subject, Message-ID, In-Reply-To, References, MIME-Version, Content-Type). Esto te da **control completo** sobre todos los headers.

#### Opción 2: Usando SES v2 `SendEmail` con parámetro Headers

La acción `SendEmail` de la API SES v2 con `Content.Simple` soporta un parámetro `Headers` que te permite añadir headers personalizados (In-Reply-To, References) sin construir MIME raw. **Nota**: El parámetro `Headers` en la ruta de contenido simple se añadió en SES v2. Si no está disponible en la versión del SDK que usas, usa `SendRawEmail`.

### B3. Soporte de threading incorporado

**SES NO tiene soporte de threading incorporado.** La gestión de threads es enteramente tu responsabilidad:

- Debes almacenar el `Message-ID` de cada email enviado.
- Cuando llega una respuesta (vía procesamiento inbound), debes extraer los headers `In-Reply-To` y `References` del email entrante.
- Al enviar una respuesta, debes establecer:
  - `In-Reply-To`: El Message-ID del email al que respondes
  - `References`: La cadena de Message-IDs del thread (separados por espacios)
  - `Subject`: Prefijo con "Re: " coincidiendo con el asunto original

### B4. Envío de email raw

SES proporciona capacidad completa de email raw vía **API SES v1** (`SendRawEmail`) y **API SES v2** (`SendEmail` con `Content.Raw`). Ambos enfoques te dan control completo sobre cada header, parte MIME y adjunto.

---

## Challenge C - Email Tracking

### C1. Características de tracking/notificación

SES proporciona tracking extensivo mediante **event notifications**:

| Tipo de evento | Descripción |
|---|---|
| **Send** | El email fue aceptado exitosamente por SES para entrega |
| **Delivery** | SES entregó exitosamente el email al servidor de correo del destinatario |
| **Bounce** | El email rebotó (hard bounce = permanente, soft bounce = temporal) |
| **Complaint** | El destinatario marcó el email como spam |
| **Reject** | SES rechazó el email (p. ej. virus detectado) |
| **Open** | El destinatario abrió el email (requiere open tracking habilitado) |
| **Click** | El destinatario hizo click en un enlace (requiere click tracking habilitado) |
| **Rendering Failure** | Falló el rendering del template |
| **Delivery Delay** | Fallo temporal de entrega (aún reintentando) |
| **Subscription** | Relacionado con gestión de suscripciones de SES |

### C2. Configuración de notificaciones SNS

#### Notificaciones a nivel de dominio/identidad (estilo SES v1)

Puedes configurar topics de SNS por identidad para tres tipos de eventos:
- **Bounce notifications** -> SNS Topic A
- **Complaint notifications** -> SNS Topic B
- **Delivery notifications** -> SNS Topic C

#### Configuration Sets + Event Destinations (estilo SES v2 - Recomendado)

Este es el enfoque más moderno y flexible:

1. **Crear un Configuration Set** con opciones de tracking (p. ej. CustomRedirectDomain para click tracking).

2. **Crear Event Destinations** en el Configuration Set:

Los event destinations pueden enviar a:
- **SNS Topic** (todos los tipos de eventos)
- **Kinesis Data Firehose** (todos los tipos de eventos, para streaming a S3/Redshift/etc.)
- **CloudWatch** (métricas basadas en dimensiones)
- **EventBridge** (todos los tipos de eventos, permite routing a cualquier servicio AWS)
- **Pinpoint** (para analytics de Pinpoint)

3. **Especificar el Configuration Set al enviar** vía el parámetro `ConfigurationSetName` en el comando SendEmail. Esto habilita el tracking para ese email.

### C3. Open Tracking y Click Tracking

#### Open Tracking

- **Soporte nativo**: Sí, SES soporta open tracking.
- **Mecanismo**: SES inserta un pixel de tracking transparente de 1x1 (`<img>`) en el cuerpo HTML del email.
- **Activación**: Habilitar en el Configuration Set vía `PutConfigurationSetTrackingOptionsCommand` (o como parte de TrackingOptions al crear el configuration set). El open tracking se **habilita por configuration set** teniendo un event destination que escucha eventos `OPEN`.
- **Limitaciones**: Solo funciona para emails HTML. No funciona si el cliente de correo del destinatario bloquea imágenes (común en Outlook, funciones de privacidad de Apple Mail, etc.).

#### Click Tracking

- **Soporte nativo**: Sí, SES soporta click tracking.
- **Mecanismo**: SES reescribe todos los enlaces en el cuerpo HTML para que pasen por un dominio de tracking de SES. Cuando el destinatario hace click, SES registra el evento y redirige a la URL original.
- **Dominio de tracking por defecto**: Un subdominio de `amazonses.com`.
- **Dominio de tracking personalizado**: Puedes establecer un dominio personalizado (p. ej. `track.yourdomain.com`) vía `TrackingOptions.CustomRedirectDomain` del Configuration Set. Esto requiere:
  - Un registro CNAME: `track.yourdomain.com CNAME <varía por región, p. ej. r.us-east-1.awstrack.me>`
- **Activación**: Similar al open tracking, habilitar teniendo event destinations para eventos `CLICK`.

**Nota importante sobre tracking y DKIM**: El open/click tracking modifica el cuerpo del email, lo cual ocurre ANTES del signing DKIM. Por tanto las firmas DKIM permanecen válidas.

### C4. SES + CloudWatch Metrics

SES publica estas métricas a CloudWatch automáticamente:

| Métrica | Descripción |
|---|---|
| `Send` | Número de llamadas a la API de envío |
| `Delivery` | Número de entregas exitosas |
| `Bounce` | Número de bounces |
| `Complaint` | Número de complaints |
| `Reject` | Número de envíos rechazados |
| `Open` | Número de opens (si tracking habilitado) |
| `Click` | Número de clicks (si tracking habilitado) |
| `RenderingFailure` | Número de fallos de rendering de templates |

Con **Configuration Sets** que tienen CloudWatch como event destination, puedes añadir dimensiones personalizadas (p. ej. campaign, dimension value source desde MESSAGE_TAG, EMAIL_HEADER, o LINK_TAG). Luego puedes crear dashboards y alarmas de CloudWatch basados en estas métricas.

**Métricas a nivel de cuenta SES** (siempre disponibles sin configuration sets):
- Utilización de cuota de envío
- Tasa de bounce (SES monitorea esto; si supera ~5%, tu cuenta puede ponerse en probation)
- Tasa de complaint (SES monitorea esto; el umbral es ~0.1%)

### C5. Notificaciones de eventos estilo webhook

SES no tiene una función "webhook" nativa, pero puedes lograr comportamiento estilo webhook mediante varios patrones:

```mermaid
flowchart LR
    subgraph p1 ["Pattern 1: SNS -> HTTPS (Most Common)"]
        SES1["SES Event"] --> SNS1["SNS Topic"] --> HTTPS1["HTTPS Subscription<br/>your webhook endpoint"]
    end
    subgraph p2 ["Pattern 2: SNS -> SQS (Recommended)"]
        SES2["SES Event"] --> SNS2["SNS Topic"] --> SQS2["SQS Queue"] --> NestJS2["NestJS Backend<br/>polls SQS"]
    end
    subgraph p3 ["Pattern 3: EventBridge (Newest)"]
        SES3["SES Event"] --> EB3["EventBridge"] --> API3["API Destination<br/>any HTTP endpoint"]
    end
    subgraph p4 ["Pattern 4: Firehose (Analytics)"]
        SES4["SES Event"] --> KF4["Kinesis Firehose"] --> S3R4["S3 / Redshift"]
    end
```

- **Pattern 1** (SNS -> HTTPS): SNS puede entregar a un endpoint HTTPS, que es efectivamente un webhook. Tu backend NestJS expone un endpoint que SNS llama con el payload del evento.
- **Pattern 2** (SNS -> SQS, recomendado): Más resiliente porque SQS proporciona entrega garantizada, retry y soporte de dead-letter queue.
- **Pattern 3** (EventBridge): Proporciona filtrado, transformación y entrega confiable a endpoints HTTP. Lo más cercano a un webhook nativo, permite routing complejo de eventos.
- **Pattern 4** (Kinesis Firehose): Bueno para analytics históricos y reporting.

---

## Pricing

### Modelo de pricing actual de SES (a 2025)

El pricing de SES es directo y está entre los más baratos del mercado.

#### Email Outbound (Envío)

| Escenario | Precio |
|---|---|
| Envío desde aplicación alojada en EC2 | **$0.00** para los primeros **62,000 emails/mes** (Free Tier) |
| Más allá del free tier / no-EC2 | **$0.10 por 1,000 emails** ($0.0001 por email) |
| Adjuntos | **$0.12 por GB** de adjuntos enviados |

**Detalle del Free Tier**: Los 62,000 emails/mes del free tier aplican cuando los emails se envían desde una aplicación alojada en **Amazon EC2** (o servicios que corren en EC2 como ECS, EKS, Elastic Beanstalk, Lambda). Como la empresa corre en ECS, califican.

#### Email Inbound (Recepción)

| Escenario | Precio |
|---|---|
| Primeros 1,000 emails/mes | **Gratis** |
| Más de 1,000 | **$0.10 por 1,000 emails** |
| Chunks de email entrante (por 256 KB) | **$0.09 por 1,000 chunks** |

**Nota**: "Chunks" significa que cada 256 KB de un email entrante cuenta como un chunk. Un email de 1 KB = 1 chunk. Un email de 300 KB = 2 chunks.

#### Costes adicionales

| Característica | Coste |
|---|---|
| **Dedicated IP** | **$24.95/mes por IP** (mínimo 1 IP por región) |
| **Dedicated IPs (managed)** | **$24.95/mes por IP** con warm-up y escalado automático |
| **Virtual Deliverability Manager** | **$0.07 por 100 emails** (dashboard consultivo para insights de deliverability) |
| **SES Mailbox Simulator** | Gratis (para testing de bounces, complaints, etc.) |

#### Estimación de coste para esta POC

Asumiendo:
- Alojado en ECS (califica para free tier)
- ~500 emails inbound/mes
- ~500 respuestas outbound/mes
- Adjuntos pequeños (< 1 GB total)

| Ítem | Coste mensual |
|---|---|
| Outbound (500 emails, cubierto por free tier) | $0.00 |
| Inbound (500 emails, cubierto por 1,000 gratis) | $0.00 |
| Notificaciones SNS | ~$0.00 (SNS primeros 1M requests gratis) |
| Almacenamiento S3 para emails raw | ~$0.01 |
| Invocaciones Lambda | ~$0.00 (cubierto por free tier) |
| **Total** | **~$0.01/mes** |

Incluso a volúmenes mayores (10,000 emails/mes):

| Ítem | Coste mensual |
|---|---|
| Outbound (10,000 emails, 62,000 gratis en EC2) | $0.00 |
| Inbound (10,000 emails) | $0.90 |
| Adjuntos (~2 GB) | $0.24 |
| **Total** | **~$1.14/mes** |

---

## Integración

### I2. API SES v2 vs API SES v1

| Característica | SES v1 (`@aws-sdk/client-ses`) | SES v2 (`@aws-sdk/client-sesv2`) |
|---|---|---|
| Estilo de API | Operaciones más granulares, más antiguas | Operaciones modernas, consolidadas |
| Enviar email | `SendEmail`, `SendRawEmail`, `SendTemplatedEmail`, `SendBulkTemplatedEmail` | Único `SendEmail` con opciones de tipo de contenido (Simple, Raw, Template) |
| Configuration sets | Soportado | First-class, más características |
| Headers personalizados en envíos simples | No soportado (debe usarse `SendRawEmail`) | Soportado vía parámetro `Headers` |
| Event destinations | SNS, CloudWatch, Kinesis Firehose | SNS, CloudWatch, Kinesis Firehose, **EventBridge**, Pinpoint |
| Contact lists / gestión de suscripciones | No disponible | Gestión de listas incorporada |
| Virtual Deliverability Manager | No disponible | Soportado |
| Verificación de dominio | Registro TXT o DKIM | Basado en DKIM (preferido) |
| **Recomendación** | Legacy; evitar para proyectos nuevos | **Usar esto para todos los proyectos nuevos** |

### I3. Procesamiento de emails inbound

#### Arquitectura: S3 + SNS + Lambda (o SQS -> NestJS)

```mermaid
flowchart TD
    Email["Email arrives at SES"] --> RR["Receipt Rule"]
    RR --> S3["S3: Store raw email<br/>s3://bucket/emails/{messageId}"]
    RR --> SNS["SNS: Publish notification to topic"]
    SNS --> SQS["SQS Queue<br/>reliable processing"]
    SQS --> NestJS["NestJS Backend<br/>polls SQS, processes email"]
    SNS --> Lambda["Lambda<br/>immediate processing/parsing"]
```

### I4. Integración con infraestructura AWS existente

Dado que la empresa ya usa AWS extensivamente, SES se integra naturalmente:

| Servicio AWS | Integración con SES |
|---|---|
| **ECS** | El backend en ECS usa AWS SDK para enviar vía SES. El IAM task role proporciona credenciales. No se necesitan API keys. |
| **S3** | Almacenar emails raw inbound. Almacenar templates de email. Archivar emails enviados. |
| **SNS** | Recibir notificaciones de eventos SES (bounces, deliveries, opens, clicks). Fan out a múltiples consumidores. |
| **SQS** | Buffer de procesamiento de email inbound. Desacoplar manejo de eventos. Dead-letter queue para procesamiento fallido. |
| **Lambda** | Procesar emails inbound. Manejar eventos SES. |
| **CloudWatch** | Métricas SES, alarmas en tasas de bounce/complaint, dashboards. |
| **EventBridge** | Enrutar eventos SES a cualquier servicio AWS con reglas de filtrado. |
| **IAM** | Permisos granulares para envío (puede restringir a direcciones From específicas). |
| **KMS** | Cifrar emails almacenados en reposo (cifrado server-side S3). |
| **CloudFormation/CDK** | Soporte completo de IaC para todos los recursos SES. |

---

## Limitaciones

### L1. Restricciones del modo Sandbox

**Toda cuenta SES nueva comienza en modo Sandbox.** Restricciones:

| Restricción | Sandbox | Producción |
|---|---|---|
| Enviar a | Solo direcciones de email/dominios verificados | Cualquier destinatario |
| Enviar desde | Solo identidades verificadas | Solo identidades verificadas |
| Límite de envío diario | **200 emails/día** | Comienza en 50,000/día (ajustable) |
| Tasa de envío | **1 email/segundo** | Comienza en 14 emails/segundo (ajustable) |

**Para salir del Sandbox**: Enviar una solicitud vía AWS Console o Support. Necesitas proporcionar:
- Tu caso de uso
- Cómo manejas bounces y complaints
- Volumen de envío esperado
- Tipo de contenido (transactional vs. marketing)

**Tiempo de aprobación típico**: 24-48 horas. A veces requiere ida y vuelta con soporte de AWS.

**Importante para POC**: Puedes hacer toda la POC en modo sandbox verificando las direcciones de email de destinatario específicas con las que probarás.

### L2. Límites de envío

| Métrica | Por defecto (Producción) | Máximo |
|---|---|---|
| Cuota de envío diaria | 50,000 emails/día | Puede aumentarse a millones vía solicitud |
| Tasa máxima de envío | 14 emails/segundo | Puede aumentarse a cientos/segundo |
| Tamaño máximo de mensaje | **10 MB** (incluyendo adjuntos, tras encoding) | No puede aumentarse |
| Máximo de destinatarios por mensaje | 50 | No puede aumentarse |
| Máximo de llamadas API `SendEmail` | 1 por mensaje | N/A |

**Aumentos de tasa de envío**: SES aumenta automáticamente tus límites con el tiempo si mantienes buena reputación de envío. También puedes solicitar aumentos manualmente.

### L3. Disponibilidad por región

#### SES Sending (Outbound) - Disponible en muchas regiones:

- us-east-1 (N. Virginia)
- us-east-2 (Ohio)
- us-west-1 (N. California)
- us-west-2 (Oregon)
- af-south-1 (Cape Town)
- ap-south-1 (Mumbai)
- ap-northeast-1 (Tokyo)
- ap-northeast-2 (Seoul)
- ap-northeast-3 (Osaka)
- ap-southeast-1 (Singapore)
- ap-southeast-2 (Sydney)
- ca-central-1 (Canada)
- eu-central-1 (Frankfurt)
- eu-west-1 (Ireland)
- eu-west-2 (London)
- eu-west-3 (Paris)
- eu-north-1 (Stockholm)
- eu-south-1 (Milan)
- me-south-1 (Bahrain)
- sa-east-1 (Sao Paulo)
- il-central-1 (Tel Aviv)

#### SES Receiving (Inbound) - SOLO 3 regiones:

| Región | Endpoint |
|---|---|
| us-east-1 (N. Virginia) | `inbound-smtp.us-east-1.amazonaws.com` |
| us-west-2 (Oregon) | `inbound-smtp.us-west-2.amazonaws.com` |
| eu-west-1 (Ireland) | `inbound-smtp.eu-west-1.amazonaws.com` |

**Esta es una limitación significativa.** Si tu infraestructura AWS principal está en otra región, el procesamiento de email inbound debe estar en una de estas tres regiones, y necesitarás integración cross-region.

### L4. Gotchas con procesamiento de email inbound

1. **Límite de tamaño de mensaje SNS**: Si el email entrante supera **150 KB**, la notificación SNS NO contendrá el contenido del email. DEBES almacenar en S3 primero y obtener desde ahí. Siempre diseña tu arquitectura para manejar ambos caminos.

2. **Sin filtrado de spam incorporado para acciones personalizadas**: SES proporciona escaneo básico de spam/virus (vía SpamAssassin) en receipt rules, pero solo puedes elegir hacer bounce al spam o dejarlo pasar. No hay filtrado sofisticado.

3. **Las receipt rules son regionales**: Si configuras inbound en us-east-1, no puedes acceder a esas receipt rules desde eu-west-1.

4. **Verificación de dominio para recepción**: El dominio debe estar verificado en SES para que las receipt rules acepten correo para él. Pero esta es la misma verificación necesaria para envío.

5. **Sin parsing de email incorporado**: SES entrega contenido MIME raw. Necesitas una librería como `mailparser` para parsearlo a datos estructurados (headers, body, adjuntos).

6. **Los emails reenviados rompen SPF/DKIM del remitente original**: Si usas forwarding de Google Workspace a SES, el SPF del remitente original puede fallar (ya que Google está relayando, no el servidor original). DKIM puede sobrevivir si Google no modifica el mensaje. Esto afecta tu capacidad de verificar la autenticidad del remitente original pero no bloquea la recepción.

7. **Sin webhook/API de inbound parse como SendGrid**: A diferencia del Inbound Parse de SendGrid (que hace POST de datos de email parseados a tu webhook), el inbound de SES requiere que configures tú mismo el pipeline S3/SNS/Lambda. Es más flexible pero requiere más setup.

8. **Preocupaciones de cifrado**: Los emails almacenados en S3 vía receipt rules están sin cifrar por defecto. Deberías habilitar cifrado server-side de S3 (SSE-S3 o SSE-KMS) en el bucket.

9. **SES no maneja bien catch-all**: Las receipt rules coinciden con direcciones exactas o dominios completos. No hay regex o pattern matching para direcciones de destinatario.

10. **Monitoreo de tasa de bounce**: SES monitorea activamente tus tasas de bounce y complaint. Si tu tasa de bounce supera ~5% o la tasa de complaint supera ~0.1%, SES puede poner tu cuenta en probation o suspender el envío. Es importante monitorear esto, especialmente al enviar a direcciones de clientes que no has validado.

---

## Recomendación de arquitectura para esta POC específica

Dadas las restricciones (Google Workspace, Cloudflare DNS, DMARC `p=reject`, NestJS en ECS):

### Arquitectura recomendada

#### Flujo Inbound

```mermaid
flowchart TD
    Customer["Customer -> info@caminosdelassierras.com.ar"] --> GW["Google Workspace"]
    GW -->|"Routing rule (forward copy)"| Inbound["inbound@your-company-domain.com"]
    Inbound -->|"MX -> SES Inbound (us-east-1)"| RR["SES Receipt Rule"]
    RR --> S3["S3 Bucket<br/>raw email"]
    RR --> SNS["SNS Topic"]
    SNS --> SQS["SQS Queue"]
    SQS --> NestJS["NestJS on ECS<br/>parse, process, store in DB"]
```

#### Flujo Outbound

```mermaid
flowchart TD
    NestJS["NestJS on ECS"] -->|"AWS SDK v3 (SESv2Client)"| SES["SES Outbound"]
    SES -->|"DKIM-signed as caminosdelassierras.com.ar"| Mailbox["Customer's mailbox"]
    SES -->|"SES Event"| SNS["SNS -> SQS -> NestJS<br/>tracking events"]
```

### Registros DNS requeridos (en Cloudflare)

Para **caminosdelassierras.com.ar**:

| # | Tipo | Nombre | Valor | Propósito |
|---|---|---|---|---|
| 1 | CNAME | `token1._domainkey` | `token1.dkim.amazonses.com` | SES DKIM (1/3) |
| 2 | CNAME | `token2._domainkey` | `token2.dkim.amazonses.com` | SES DKIM (2/3) |
| 3 | CNAME | `token3._domainkey` | `token3.dkim.amazonses.com` | SES DKIM (3/3) |
| 4 | MX | `mail` | `10 feedback-smtp.us-east-1.amazonses.com` | Custom MAIL FROM |
| 5 | TXT | `mail` | `v=spf1 include:amazonses.com ~all` | SPF para subdominio MAIL FROM |

**Los registros MX existentes del dominio raíz permanecen sin cambios** (Google Workspace).
**El SPF existente del dominio raíz se actualiza opcionalmente** para añadir `include:amazonses.com`.

### Ventajas clave para este caso de uso

1. **Ya en AWS**: Integración nativa, auth basada en IAM, no hay API keys que gestionar, misma facturación.
2. **Extremadamente barato**: Efectivamente gratis a escala POC (y bajo coste a escala de producción).
3. **Control total de headers**: Puede establecer In-Reply-To, References, Message-ID personalizado para threading.
4. **Event notifications**: Tracking comprehensivo vía SNS/SQS (ya en su stack).
5. **Alineación DKIM**: Easy DKIM con auto-rotación satisface DMARC `p=reject`.
6. **Escalabilidad**: Sin límites prácticos para su volumen.

### Desventajas clave / Riesgos

1. **Sin webhook de inbound parse incorporado**: Debes construir tú mismo el pipeline S3/SNS/Lambda (a diferencia del Inbound Parse más simple de SendGrid).
2. **Inbound solo en 3 regiones**: Puede requerir setup cross-region si la infra principal está en otro lugar.
3. **Salida de Sandbox requerida**: Necesitas solicitar acceso a producción (24-48 horas).
4. **Sin threading incorporado**: Toda la gestión de threads es manual (pero esto es cierto para todos los proveedores).
5. **Acceso a DNS requerido**: Los 5+ registros DNS deben añadirse en Cloudflare por quien controle el dominio.
6. **Sin UI de email templating incorporada**: A diferencia de los dynamic templates o características de marketing de SendGrid, SES es más básico. Los templates existen (SES Templates API) pero son Handlebars-style básicos.
7. **Detección de forward**: SES no proporciona tracking de evento "forwarded". Solo open, click, bounce, complaint, delivery.

---

## Notas de comparación (vs. otros proveedores en evaluación)

| Característica | Amazon SES | SendGrid | Notas |
|---|---|---|---|
| **Inbound parsing** | DIY (S3+SNS+Lambda) | Built-in Inbound Parse webhook | SendGrid más simple para inbound |
| **Outbound sending** | API/SMTP | API/SMTP | Comparable |
| **Setup DKIM** | 3 registros CNAME | 2 registros CNAME | SES ligeramente más registros DNS |
| **Gestión de threads** | Manual (control total de headers) | Manual (control total de headers) | Mismo esfuerzo |
| **Open/Click tracking** | Nativo (config set) | Nativo (incorporado) | Comparable |
| **Event webhooks** | SNS->SQS/HTTP | Native Event Webhook | SES requiere más plumbing |
| **Pricing (envío)** | $0.10/1K ($0 en free tier EC2) | $0.35-0.80/1K (por niveles) | SES 3-8x más barato |
| **Pricing (inbound)** | $0.10/1K | Incluido en plan | SES más barato a volumen |
| **Integración AWS** | Nativa | Third-party | Gran ventaja para SES |
| **Facilidad de setup** | Media (más DIY) | Fácil (más batteries-included) | SendGrid más rápido para prototipar |
| **Herramientas de deliverability** | Virtual Deliverability Manager ($) | Deliverability insights (depende del plan) | Ambos adecuados |

---

## Resumen de evaluación

| Criterio | Valoración | Notas |
|---|---|---|
| **¿Puede recibir email inbound?** | Sí (con cambio de MX o patrón de forwarding) | Requiere MX para SES inbound, O usar forwarding de Google + MX del propio dominio |
| **¿Puede enviar como @caminosdelassierras.com.ar?** | Sí (con registros DNS) | 3 CNAMEs DKIM + MX/TXT Custom MAIL FROM = 5 registros DNS en Cloudflare |
| **¿Pasa DMARC p=reject?** | Sí | La alineación DKIM vía Easy DKIM satisface DMARC |
| **¿Continuidad de thread?** | Sí (manual) | Control total vía headers In-Reply-To/References |
| **¿Email tracking?** | Sí | Open, click, delivery, bounce, complaint vía Configuration Sets |
| **¿Coste para POC?** | ~$0/mes | El free tier cubre fácilmente el volumen POC |
| **¿Coste a escala (10K/mes)?** | ~$1-2/mes | Extremadamente competitivo |
| **¿Integración con AWS existente?** | Excelente | IAM nativo, SNS, SQS, S3, CloudWatch, EventBridge |
| **¿Complejidad de setup?** | Media-Alta | Más DIY que SendGrid, pero más flexible |
| **¿Tiempo a producción?** | 2-4 días | Salida de Sandbox (24-48h) + propagación DNS + desarrollo |

---

## Referencias

### Pricing
- SES Pricing: <https://aws.amazon.com/ses/pricing/>
- SES FAQs: <https://aws.amazon.com/ses/faqs/>

### Recepción de email inbound (Receipt Rules)
- Email receiving with Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/receiving-email.html>
- SES email receiving concepts and use cases: <https://docs.aws.amazon.com/ses/latest/dg/receiving-email-concepts.html>
- Creating receipt rules (console walkthrough): <https://docs.aws.amazon.com/ses/latest/dg/receiving-email-receipt-rules-console-walkthrough.html>
- ReceiptRule API Reference: <https://docs.aws.amazon.com/ses/latest/APIReference/API_ReceiptRule.html>

### Verificación de dominio y DKIM
- Verified identities in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/verify-addresses-and-domains.html>
- Configuring identities in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/configure-identities.html>
- Easy DKIM in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html>
- Authenticating email with DKIM in SES: <https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim.html>

### SPF y Custom MAIL FROM Domain
- Authenticating email with SPF in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-spf.html>
- Complying with DMARC in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dmarc.html>

### Configuration Sets, Event Destinations y Tracking
- Setting up event notifications for SES: <https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html>
- Creating Amazon SES event destinations: <https://docs.aws.amazon.com/ses/latest/dg/event-destinations-manage.html>
- Set up an Amazon SNS event destination: <https://docs.aws.amazon.com/ses/latest/dg/event-publishing-add-event-destination-sns.html>
- Create a configuration set: <https://docs.aws.amazon.com/ses/latest/dg/event-publishing-create-configuration-set.html>
- PutConfigurationSetTrackingOptions (open/click tracking): <https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_PutConfigurationSetTrackingOptions.html>

### Límites de envío y Sandbox
- Service quotas in Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/quotas.html>
- Managing your Amazon SES sending limits: <https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas.html>
- Increasing your SES sending quotas: <https://docs.aws.amazon.com/ses/latest/dg/manage-sending-quotas-request-increase.html>

### Disponibilidad por región
- Regions and Amazon SES: <https://docs.aws.amazon.com/ses/latest/dg/regions.html>
- SES email receiving expands to new regions (anuncio 2023): <https://aws.amazon.com/about-aws/whats-new/2023/09/amazon-ses-email-service-7-regions/>

### SDK e integración
- `@aws-sdk/client-sesv2` (npm): <https://www.npmjs.com/package/@aws-sdk/client-sesv2>
- AWS SDK for JavaScript v3 - SESv2 Client: <https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/sesv2>
- `@aws-sdk/client-ses` (npm, API v1): <https://www.npmjs.com/package/@aws-sdk/client-ses>
- Nodemailer with SES transport: <https://nodemailer.com/transports/ses/>
