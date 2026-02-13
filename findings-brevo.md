# Evaluacion Tecnica: Brevo

> **Proveedor**: Brevo (renombrado desde Sendinblue en mayo 2023)
>
> **Sitio**: https://www.brevo.com
>
> **API Docs**: https://developers.brevo.com
>
> **Fecha de investigacion**: 2026-02-11
>
> **Estado del dominio**: Parcialmente integrado (DKIM + verificacion ya configurados)

---

## Hallazgo Clave: Integracion Parcial Pre-existente

El dominio `caminosdelassierras.com.ar` **ya tiene configuracion parcial con Brevo**:

| Componente              | Estado                         | Registro DNS                                                            |
| ----------------------- | ------------------------------ | ----------------------------------------------------------------------- |
| Verificacion de dominio | Completado                     | `brevo-code:2e5dc272262e89f4a55726410d925b25` (TXT)                     |
| DKIM                    | Completado                     | `mail._domainkey.caminosdelassierras.com.ar` (TXT, RSA 1024-bit)        |
| SPF                     | **NO configurado**             | Falta `include:sendinblue.com` en el registro SPF                       |
| DMARC                   | Existente (no requiere cambio) | `v=DMARC1; p=reject; rua=mailto:noresponder@caminosdelassierras.com.ar` |

**Esto representa una ventaja significativa**: Brevo es el unico proveedor evaluado que ya tiene 2 de 3 pasos de autenticacion completados. SendGrid, SES y Mailgun requieren configurar todo desde cero.

---

## Challenge A - Envio y Recepcion

### A1. Inbound Email (Recepcion)

**Brevo SI soporta Inbound Parsing** a traves de su funcionalidad llamada **"Inbound Parsing"** (anteriormente conocida como "Sendinblue Inbound").

**Como funciona:**

1. **Metodo principal - Webhook de inbound parsing**: Brevo puede recibir emails y enviar el contenido parseado a un webhook URL via HTTP POST.

2. **Configuracion**:
   - Se configura desde la consola de Brevo en la seccion Transactional > Settings > Inbound Parsing
   - Se define un webhook URL (ej: `https://api.tudominio.com/webhooks/brevo/inbound`)
   - Brevo parsea el email y envia un JSON payload al webhook

3. **Datos que recibe el webhook**:
   - `Uuid`: Identificador unico del email entrante
   - `MessageId`: Message-ID del email original
   - `InReplyTo`: Header In-Reply-To (crucial para threading)
   - `From`: Remitente (objeto con email y nombre)
   - `To`: Destinatarios
   - `Cc`: Con copia
   - `ReplyTo`: Reply-To del email original
   - `SentAtDate`: Fecha de envio
   - `Subject`: Asunto
   - `RawHtmlBody`: Cuerpo HTML completo
   - `RawTextBody`: Cuerpo texto plano
   - `Attachments`: Array de adjuntos (Base64 encoded o URL para descarga)
   - `Headers`: Headers completos del email original

4. **Para activar inbound en un dominio personalizado**, hay dos opciones:
   - **Opcion A (Cambio de MX)**: Apuntar los MX records a Brevo - **NO viable** en este caso porque se perderia Google Workspace
   - **Opcion B (Forwarding desde Gmail)**: Configurar regla de reenvio en Google Workspace para enviar copias de emails de `info@caminosdelassierras.com.ar` a una direccion de inbound de Brevo. Esta es la **opcion recomendada**.

### A2. Inbound Parse Webhook - Equivalente a SendGrid

**Si, Brevo tiene un equivalente directo al Inbound Parse de SendGrid.**

| Caracteristica           | SendGrid Inbound Parse | Brevo Inbound Parsing |
| ------------------------ | ---------------------- | --------------------- |
| Webhook HTTP POST        | Si                     | Si                    |
| JSON payload             | Si                     | Si                    |
| Headers completos        | Si                     | Si                    |
| Adjuntos                 | Si (multipart)         | Si (Base64 o URL)     |
| Texto plano + HTML       | Si                     | Si                    |
| Message-ID / In-Reply-To | Si                     | Si                    |
| Configuracion            | Via UI o API           | Via UI (Settings)     |
| Requiere MX propio       | Si (o forwarding)      | Si (o forwarding)     |

**Diferencia clave**: El inbound parsing de Brevo es mas sencillo de configurar pero tiene menos opciones de filtrado que SendGrid. Para el caso de uso con forwarding desde Gmail, ambos funcionan de manera equivalente.

### A3. Envio desde Dominio Autenticado

Dado que el dominio **ya tiene verificacion y DKIM configurados**, lo que falta para enviar es:

1. **Agregar SPF** (unico paso DNS restante):

   ```
   v=spf1 include:_spf.google.com include:sendinblue.com ~all
   ```

2. **Verificar que la cuenta Brevo este activa** y tenga acceso a la API transaccional

3. **Agregar el sender email** en Brevo:
   - Ir a Senders & IPs > Senders
   - Agregar `info@caminosdelassierras.com.ar` como sender verificado
   - La verificacion del sender es adicional a la del dominio (requiere confirmar via email o puede validarse automaticamente si el dominio ya esta autenticado)

4. **No se requiere delegacion completa de DNS** - Solo el agregado del include SPF

**Resumen de gaps para envio completo**:

| Paso                             | Estado        | Accion requerida                                 |
| -------------------------------- | ------------- | ------------------------------------------------ |
| Verificacion de dominio en Brevo | Completado    | Ninguna                                          |
| DKIM configurado                 | Completado    | Ninguna                                          |
| SPF configurado                  | **Pendiente** | Agregar `include:sendinblue.com` al registro SPF |
| Sender verificado                | **Pendiente** | Agregar sender en consola Brevo                  |
| API key generada                 | **Pendiente** | Generar en Settings > API Keys                   |

### A4. Configuracion SPF Requerida

**Registro SPF actual**:

```
v=spf1 include:_spf.google.com ~all
```

**Registro SPF necesario**:

```
v=spf1 include:_spf.google.com include:sendinblue.com ~all
```

**Notas**:

- Brevo usa `sendinblue.com` para SPF (no `brevo.com`) - esto es legacy del rebranding y sigue vigente
- Solo se agrega un `include` adicional, no afecta el email existente de Google Workspace
- El cambio se hace en Cloudflare, en el registro TXT del dominio raiz
- Propagacion DNS: tipicamente 5-15 minutos en Cloudflare (TTL bajo)

### A5. Reply-To Handling

**Si, Brevo permite control total del Reply-To.**

Opciones de Reply-To:

- **Mismo sender**: `replyTo` = `sender` (el cliente responde a info@caminosdelassierras.com.ar)
- **Direccion de inbound**: `replyTo` apunta a una casilla que Brevo monitorea para inbound parsing
- **Direccion del backend**: `replyTo` puede apuntar a un alias que el backend procesa

**Para el flujo planteado**, lo ideal es que el Reply-To sea `info@caminosdelassierras.com.ar` asi el cliente responde ahi, Google Workspace lo recibe, y el forwarding lo envia a Brevo inbound.

### A6. Alineacion DMARC

**Analisis critico dado DMARC p=reject**:

DMARC requiere que pase **al menos uno** de:

- SPF alineado (el dominio del Return-Path coincide con el From)
- DKIM alineado (el dominio de la firma DKIM coincide con el From)

**Situacion actual (sin agregar SPF)**:

| Check              | Resultado             | Detalle                                                          |
| ------------------ | --------------------- | ---------------------------------------------------------------- |
| SPF                | FAIL                  | Brevo no esta en el SPF actual                                   |
| DKIM               | **PASS**              | `mail._domainkey.caminosdelassierras.com.ar` ya esta configurado |
| DMARC (alineacion) | **PASS**              | Porque DKIM pasa y esta alineado con el dominio                  |
| Entrega            | **Deberia funcionar** | DMARC pasa por DKIM aunque SPF falle                             |

**IMPORTANTE**: Aunque tecnicamente los emails podrian entregarse solo con DKIM (ya que DMARC requiere solo 1 de 2), **se recomienda fuertemente agregar SPF tambien** porque:

1. Algunos servidores de email son mas estrictos que el estandar DMARC
2. Mejor reputacion del dominio con ambos alineados
3. Redundancia: si DKIM falla por alguna razon, SPF rescata la entrega
4. Best practice de la industria

**Conclusion**: Los emails enviados via Brevo **probablemente se entregan hoy** gracias al DKIM existente, pero agregar SPF es altamente recomendado y es un cambio minimo.

---

## Challenge B - Thread Tracking

### B1. Soporte de Threading (Message-ID, In-Reply-To, References)

**Brevo SI soporta email threading a traves de headers personalizados.**

**Headers relevantes para threading**:

| Header        | Comportamiento en Brevo                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `Message-ID`  | Generado automaticamente por Brevo al enviar. Se incluye en el response de la API y en el webhook `delivered` |
| `In-Reply-To` | Se puede setear manualmente via `headers` en la API                                                           |
| `References`  | Se puede setear manualmente via `headers` en la API                                                           |
| `Subject`     | Debe incluir "Re: " para que Gmail/Outlook lo agrupen en el thread                                            |

### B2. Custom Headers

**Si, Brevo permite custom headers** en la API transaccional.

**Limitaciones**:

- No se pueden sobreescribir headers protegidos (From, To, Subject, Date)
- El `Message-ID` es generado por Brevo y no puede ser sobreescrito
- Los custom headers con prefijo `X-` se mantienen tal cual
- Los headers estandar como `In-Reply-To` y `References` se respetan

### B3. Threading: Automatico vs Manual

**El threading requiere implementacion manual, pero es straightforward.**

Brevo **no mantiene threading automaticamente**. El flujo que se debe implementar en el backend NestJS es:

1. **Al recibir email inbound** (via webhook de inbound parsing):
   - Extraer `MessageId` del email original
   - Extraer `Subject` del email original
   - Almacenar en base de datos vinculado al caso/contacto

2. **Al enviar respuesta** (via API transaccional):
   - Setear `In-Reply-To` con el `MessageId` del ultimo email del thread
   - Setear `References` con la cadena completa de Message-IDs del thread
   - Setear `Subject` como `"Re: " + subject_original`

3. **Brevo retorna el `messageId`** del email enviado en el response de la API. Este ID debe almacenarse para encadenar futuros replies.

---

## Challenge C - Email Tracking

### C1. Funcionalidades de Tracking

Brevo ofrece tracking **nativo y automatico** en emails transaccionales:

| Metrica          | Disponible | Detalle                                                               |
| ---------------- | ---------- | --------------------------------------------------------------------- |
| **Delivered**    | Si         | Confirmacion de entrega al servidor destino                           |
| **Opened**       | Si         | Via pixel de tracking (1x1 transparent image)                         |
| **Clicked**      | Si         | Via reescritura de URLs (link tracking)                               |
| **Bounced**      | Si         | Hard bounce y soft bounce diferenciados                               |
| **Soft Bounce**  | Si         | Reintentos automaticos configurables                                  |
| **Hard Bounce**  | Si         | Se agrega a lista de supresion automaticamente                        |
| **Spam Report**  | Si         | Cuando el destinatario marca como spam                                |
| **Unsubscribed** | Si         | Para emails con link de unsub (mas relevante en marketing)            |
| **Blocked**      | Si         | Emails bloqueados por Brevo pre-envio (ej: destinatario en blacklist) |
| **Deferred**     | Si         | Email aceptado pero pendiente de entrega final                        |
| **Invalid**      | Si         | Direccion de email invalida                                           |
| **Forward**      | No nativo  | No hay deteccion de forward como SendGrid                             |

**Open tracking** funciona automaticamente insertando un pixel invisible. Se puede desactivar por email individual si se desea.

**Click tracking** funciona reescribiendo los URLs del email para pasar por los servidores de Brevo. Se puede desactivar globalmente o por email.

### C2. Webhook Events

Brevo proporciona **webhooks en tiempo real** para todos los eventos de email transaccional.

**Eventos disponibles via webhook**:

| Evento         | Trigger                                 | Payload incluye                   |
| -------------- | --------------------------------------- | --------------------------------- |
| `delivered`    | Email entregado al servidor destino     | messageId, email, date, ts, event |
| `request`      | Email enviado a los servidores de Brevo | messageId, email, date            |
| `hardBounce`   | Bounce permanente                       | messageId, email, reason, date    |
| `softBounce`   | Bounce temporal                         | messageId, email, reason, date    |
| `opened`       | Email abierto (pixel cargado)           | messageId, email, date, ip        |
| `click`        | Link clickeado                          | messageId, email, link, date, ip  |
| `spam`         | Reportado como spam                     | messageId, email, date            |
| `blocked`      | Bloqueado pre-envio                     | messageId, email, reason, date    |
| `unsubscribed` | Click en unsub link                     | messageId, email, date            |
| `invalid`      | Email invalido                          | messageId, email, reason, date    |
| `deferred`     | Entrega diferida                        | messageId, email, reason, date    |
| `uniqueOpened` | Primera apertura unica                  | messageId, email, date, ip        |

**Configuracion de webhooks**:

- Via UI: Transactional > Settings > Webhook
- Via API: `POST /v3/webhooks` endpoint
- Se puede configurar un webhook URL por evento o uno global para todos
- Autenticacion opcional via secret token en headers

### C3. Tracking por Plan

| Plan       | Open Tracking | Click Tracking | Webhooks | Real-time |
| ---------- | ------------- | -------------- | -------- | --------- |
| Free       | Si            | Si             | Si       | Si        |
| Starter    | Si            | Si             | Si       | Si        |
| Business   | Si            | Si             | Si       | Si        |
| Enterprise | Si            | Si             | Si       | Si        |

**El tracking esta incluido en TODOS los planes, incluyendo el free tier.** No hay costo adicional por tracking.

### C4. Webhooks en Tiempo Real

**Si, los webhooks son en tiempo real** (near real-time):

- Los eventos se disparan segundos despues de ocurrir
- No hay batching (cada evento genera un POST individual)
- Reintentos automaticos: si el webhook falla, Brevo reintenta hasta 5 veces con backoff exponencial
- Se puede verificar la autenticidad del webhook via un header de firma

---

## Pricing (2025-2026)

### Estructura de Precios de Brevo

Brevo tiene **dos lineas de producto relevantes**: Marketing Platform y Transactional Email (API/SMTP). Para este caso, nos interesan ambas pero principalmente la transaccional.

### Pricing - Marketing Platform (referencial)

| Plan           | Precio        | Emails/mes                  | Contactos  |
| -------------- | ------------- | --------------------------- | ---------- |
| **Free**       | $0/mes        | 300 emails/dia (~9,000/mes) | Ilimitados |
| **Starter**    | Desde $9/mes  | 5,000/mes                   | Ilimitados |
| **Business**   | Desde $18/mes | 5,000/mes                   | Ilimitados |
| **Enterprise** | Custom        | Custom                      | Ilimitados |

_Nota: Los precios pueden variar; consultar brevo.com/pricing para valores exactos actualizados._

### Pricing - Transactional Email (API/SMTP)

**La API transaccional tiene un modelo de pricing separado** basado en volumen de emails:

| Volumen mensual      | Precio estimado               |
| -------------------- | ----------------------------- |
| Hasta 300 emails/dia | **Gratis** (incluido en Free) |
| 20,000 emails/mes    | ~$15/mes                      |
| 40,000 emails/mes    | ~$25/mes                      |
| 60,000 emails/mes    | ~$39/mes                      |
| 100,000 emails/mes   | ~$65/mes                      |
| 150,000 emails/mes   | ~$95/mes                      |

**Detalles importantes de pricing**:

1. **Free tier**: 300 emails/dia tanto transaccionales como marketing. Para una POC es mas que suficiente.

2. **Sin costo por contactos**: Brevo no cobra por numero de contactos almacenados (a diferencia de otros proveedores).

3. **Transaccional vs Marketing**: Comparten la cuota de emails pero el acceso API transaccional esta disponible en todos los planes.

4. **Inbound parsing**: **Incluido sin costo adicional** en todos los planes.

5. **Webhooks**: **Incluidos sin costo adicional** en todos los planes.

6. **Dedicated IP**: Disponible desde el plan Business (~$12/mes adicional). No requerido para la POC ni para volumenes bajos.

### Costo estimado para el caso de uso

Asumiendo un volumen estimado de comunicaciones con clientes de Caminos de las Sierras:

| Escenario        | Volumen estimado    | Plan recomendado    | Costo mensual |
| ---------------- | ------------------- | ------------------- | ------------- |
| POC / Prueba     | < 300/dia           | Free                | $0            |
| Produccion baja  | 5,000 - 10,000/mes  | Starter             | $9 - $16/mes  |
| Produccion media | 20,000 - 40,000/mes | Starter/Business    | $15 - $25/mes |
| Produccion alta  | 100,000+/mes        | Business/Enterprise | $65+/mes      |

---

## Integracion Tecnica

### I1. Node.js SDK

**Brevo tiene un SDK oficial para Node.js**: `@getbrevo/brevo`

**Anteriormente**: El paquete se llamaba `sib-api-v3-sdk` (Sendinblue). El nuevo paquete `@getbrevo/brevo` es el oficial luego del rebranding.

**Nota sobre el SDK**: El SDK de Node.js de Brevo ha tenido historial de actualizaciones irregulares. Como alternativa robusta, se puede usar la **API REST directamente** con `axios` o `node-fetch`, lo cual es frecuentemente preferido en produccion.

### I2. REST API

La API REST v3 de Brevo es completa y bien documentada.

**Endpoint base**: `https://api.brevo.com/v3/`

**Autenticacion**: Header `api-key: xkeysib-YOUR-API-KEY`

**Endpoints principales para este caso de uso**:

| Endpoint                               | Metodo     | Uso                         |
| -------------------------------------- | ---------- | --------------------------- |
| `/v3/smtp/email`                       | POST       | Enviar email transaccional  |
| `/v3/smtp/statistics/events`           | GET        | Obtener eventos de email    |
| `/v3/smtp/statistics/aggregatedReport` | GET        | Reporte agregado            |
| `/v3/webhooks`                         | POST       | Crear webhook               |
| `/v3/webhooks`                         | GET        | Listar webhooks             |
| `/v3/webhooks/{webhookId}`             | PUT/DELETE | Actualizar/eliminar webhook |
| `/v3/inbound/events`                   | GET        | Eventos de inbound parsing  |
| `/v3/senders`                          | POST/GET   | Gestionar senders           |

### I3. SMTP Relay

**Brevo ofrece SMTP relay** como alternativa a la API REST:

| Parametro     | Valor                                                                               |
| ------------- | ----------------------------------------------------------------------------------- |
| Host          | `smtp-relay.brevo.com` (anteriormente `smtp-relay.sendinblue.com`, ambos funcionan) |
| Puerto        | 587 (TLS) o 465 (SSL)                                                               |
| Autenticacion | Login: email de cuenta Brevo, Password: SMTP key (distinta de API key)              |

**Ventaja del SMTP relay**: Mas facil de integrar con librerias existentes como Nodemailer (muy comun en NestJS). Las metricas y webhooks siguen funcionando con SMTP relay.

**Desventaja**: Menor control que la API REST (no retorna messageId inmediatamente en la respuesta SMTP de la misma forma).

### I4. Webhook Configuration para Eventos

**Configuracion via UI**:

- Ir a Transactional > Settings > Webhooks
- Agregar URL del webhook
- Seleccionar eventos deseados
- Testear con boton "Test webhook"

---

## Ventajas de la Integracion Pre-existente

### Gap Analysis: De estado actual a funcionalidad completa

```
Estado actual:                    Lo que falta:
==============                    ==============

[x] Dominio verificado            [ ] Agregar SPF include
[x] DKIM configurado              [ ] Verificar sender email
[x] DMARC existente               [ ] Generar API key
[ ] SPF no incluye Brevo          [ ] Configurar inbound parsing
                                   [ ] Configurar webhooks
                                   [ ] Implementar backend
```

### Ventajas especificas

1. **Menor riesgo de DMARC failure**: El DKIM ya esta alineado, por lo que incluso si se comete un error con SPF, los emails se entregaran.

2. **Tiempo de setup reducido**: Solo 1 cambio DNS vs 3-5 para otros proveedores.

3. **Validacion inmediata**: Se puede probar el envio casi inmediatamente despues de agregar SPF (con DKIM ya funcional).

4. **Evidencia de uso previo**: Alguien en la organizacion ya configuro Brevo, lo que sugiere familiaridad o al menos aprobacion previa del proveedor.

5. **Menor coordinacion con el equipo DNS**: Solo se necesita un cambio en Cloudflare, no multiples records.

---

## Limitaciones y Consideraciones

### Limitaciones de Brevo

1. **Inbound parsing menos maduro**: Comparado con SendGrid, el inbound parsing de Brevo tiene menos documentacion y menos opciones de filtrado. Funciona, pero es menos flexible.

2. **SDK de Node.js**: El SDK oficial (`@getbrevo/brevo`) ha recibido criticas por falta de tipado completo en TypeScript y actualizaciones lentas. Se recomienda usar la API REST directamente con un wrapper custom.

3. **Rate limits**: El plan free tiene limites de 300 emails/dia. Los planes pagos tienen rate limits mas generosos pero aun presentes.

4. **IP compartida en planes bajos**: En Free y Starter, los emails se envian desde IPs compartidas, lo que puede afectar deliverability. El plan Business ofrece IP dedicada.

5. **Soporte tecnico**: El soporte en planes Free y Starter es basicamente email/ticketing. Soporte prioritario disponible desde Business.

6. **Retencion de logs**: Los logs de eventos transaccionales se retienen por un periodo limitado (tipicamente 60 dias en planes basicos). Para historico largo, se deben consumir via webhooks y almacenar en base propia.

### Riesgos identificados

| Riesgo                        | Probabilidad | Impacto | Mitigacion                                                |
| ----------------------------- | ------------ | ------- | --------------------------------------------------------- |
| DKIM key 1024-bit (no 2048)   | Baja         | Bajo    | Funcional pero se podria actualizar a futuro              |
| Inbound parsing pierde emails | Baja         | Alto    | Mantener copia en Google Workspace (no borrar originales) |
| Rate limit en free tier       | Media        | Medio   | Migrar a plan pago si se excede                           |
| SDK con bugs                  | Media        | Bajo    | Usar API REST directamente                                |
| Cambio de pricing             | Baja         | Medio   | Contratar plan anual si se encuentra buen precio          |

---

## Resumen Ejecutivo

### Brevo como candidato para la POC

| Criterio             | Evaluacion               | Nota                                                          |
| -------------------- | ------------------------ | ------------------------------------------------------------- |
| Inbound email        | **Viable**               | Via forwarding desde Google Workspace + inbound parsing       |
| Outbound autenticado | **Casi listo**           | Solo falta agregar SPF include                                |
| Threading            | **Manual pero factible** | Via custom headers en API transaccional                       |
| Tracking             | **Completo**             | Open, click, bounce, delivered - incluido en todos los planes |
| Webhooks             | **Completos**            | Tiempo real, todos los eventos, incluido en free              |
| Precio POC           | **$0**                   | Free tier suficiente para pruebas                             |
| Precio produccion    | **$9-25/mes**            | Para volumenes tipicos de soporte                             |
| Integracion NestJS   | **Buena**                | SDK disponible + API REST + SMTP relay                        |
| Setup DNS restante   | **1 cambio**             | Solo agregar `include:sendinblue.com` al SPF                  |
| DMARC compliance     | **Alta**                 | DKIM ya alineado, SPF sera bonus                              |

### Recomendacion

**Brevo debe ser evaluado como candidato prioritario** en la POC por las siguientes razones:

1. **Menor friccion de setup**: La integracion parcial existente reduce el esfuerzo de configuracion a minimos.
2. **Costo competitivo**: Free para POC, $9-25/mes para produccion.
3. **Todas las features requeridas cubiertas**: Inbound parsing, outbound autenticado, threading manual, tracking completo.
4. **Menor riesgo DMARC**: Con DKIM ya funcional, hay menor probabilidad de rechazos durante pruebas.
5. **Unico paso DNS pendiente**: Solo necesita agregar un include al SPF, frente a 3-5 cambios DNS de los otros proveedores.

### Proximo paso recomendado

1. Verificar acceso a la cuenta Brevo donde se configuro el dominio (alguien en la organizacion lo hizo)
2. Agregar `include:sendinblue.com` al registro SPF en Cloudflare
3. Generar API key en Brevo
4. Probar envio transaccional basico
5. Configurar inbound parsing + webhook
6. Implementar POC en NestJS

---

## Referencias

### Pricing

- Brevo Pricing Plans: <https://www.brevo.com/pricing/>

### Inbound Parsing

- Parse inbound email (documentacion): <https://developers.brevo.com/docs/inbound-parse-webhooks>
- Brevo Inbound Parsing (overview): <https://www.brevo.com/inbound-parsing/>
- Get inbound email events API: <https://developers.brevo.com/reference/get-inbound-email-events>

### API Transaccional (Envio)

- Send a transactional email (guia): <https://developers.brevo.com/docs/send-a-transactional-email>
- Send a transactional email (API reference): <https://developers.brevo.com/reference/send-transac-email>
- SMTP relay integration: <https://developers.brevo.com/docs/smtp-integration>

### Limites de tamaño de mensaje y attachments

- Add an attachment to an email campaign: <https://help.brevo.com/hc/en-us/articles/11098615382802-Add-an-attachment-to-an-email-campaign>
- Platform quotas (limites de objetos y recursos): <https://developers.brevo.com/docs/platform-quotas>

### Webhooks (Tracking/Eventos)

- Transactional webhooks: <https://developers.brevo.com/docs/transactional-webhooks>
- Getting started with webhooks: <https://developers.brevo.com/docs/how-to-use-webhooks>
- Create webhook (API reference): <https://developers.brevo.com/reference/createwebhook>

### SDK y Librerias

- `@getbrevo/brevo` (npm): <https://www.npmjs.com/package/@getbrevo/brevo>
- Brevo Node.js SDK (GitHub): <https://github.com/getbrevo/brevo-node>
- SDKs and libraries: <https://developers.brevo.com/docs/api-clients>

### Documentacion General

- Brevo API Documentation: <https://developers.brevo.com>
- Brevo sitio principal: <https://www.brevo.com>
