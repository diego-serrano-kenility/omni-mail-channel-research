# Evaluacion Tecnica: SendGrid (Twilio)

> **Proveedor**: SendGrid (Twilio)
> **Sitio**: https://sendgrid.com
> **API Docs**: https://docs.sendgrid.com
> **Fecha de investigacion**: 2026-02
> **Estado**: Contrato existente con Twilio/SendGrid. Experiencia previa del equipo con la plataforma.

---

## Contexto Organizacional

**El equipo ya tiene experiencia y contrato activo con SendGrid/Twilio.** Esto representa una ventaja significativa:

1. **Contrato existente**: No se requiere proceso de adquisicion ni aprobacion de nuevo proveedor
2. **Experiencia del equipo**: Menor curva de aprendizaje y tiempo de implementacion
3. **Soporte**: Acceso a soporte bajo contrato existente
4. **Facturacion**: Ya integrado en los procesos de pago de la empresa

---

## Challenge A - Envio y Recepcion

### A1. Inbound Parse Webhook

SendGrid ofrece **Inbound Parse** nativo: intercepta emails enviados a un dominio (o subdominio) cuyos MX apunten a SendGrid, los parsea y los envia via HTTP POST a un webhook.

**Setup**:
1. Agregar MX record para el subdominio receptor apuntando a `mx.sendgrid.net` (priority 10)
2. Configurar en SendGrid dashboard (Settings > Inbound Parse): hostname + webhook URL
3. SendGrid hace POST a tu webhook cada vez que llega un email

**Payload** (modo parsed, `multipart/form-data`):

| Campo | Descripcion |
|---|---|
| `headers` | Headers raw del email entrante |
| `dkim` | Resultado de verificacion DKIM |
| `to` | Destinatario(s) |
| `from` | Remitente |
| `cc` | Con copia |
| `subject` | Asunto |
| `text` | Cuerpo texto plano |
| `html` | Cuerpo HTML |
| `sender_ip` | IP del servidor remitente |
| `envelope` | JSON con envelope SMTP (to, from) |
| `attachments` | Cantidad de adjuntos |
| `attachment1`, `attachment2`, ... | Archivos adjuntos (binary) |
| `attachment-info` | JSON con metadata de adjuntos |
| `SPF` | Resultado de verificacion SPF |

**Modo Raw**: Tambien disponible, envia el email completo en formato MIME en el campo `email`.

### A2. Estrategia de Inbound sin cambiar MX del dominio principal

El MX de `caminosdelassierras.com.ar` apunta a Google Workspace. Opciones:

| Opcion | Descripcion | Viabilidad |
|---|---|---|
| **Forwarding desde Google Workspace** | Regla de reenvio en Google Admin hacia direccion en subdominio con MX a SendGrid | **Recomendada** |
| **Subdominio dedicado** | MX de `parse.caminosdelassierras.com.ar` → `mx.sendgrid.net` | Viable (requiere 1 registro MX en Cloudflare) |
| **Forwarding + webhook directo** | Google Workspace reenvia a un endpoint propio, sin usar SendGrid para inbound | Viable |

**Recomendacion**: Usar forwarding desde Google Workspace hacia un subdominio con MX apuntando a SendGrid, o directamente al backend.

### A3. DNS Records Requeridos para Envio

Para enviar como `@caminosdelassierras.com.ar` con Domain Authentication:

| # | Tipo | Nombre | Valor | Proposito |
|---|---|---|---|---|
| 1 | CNAME | `em1234.caminosdelassierras.com.ar` | `u1234567.wl.sendgrid.net` | Autenticacion de dominio |
| 2 | CNAME | `s1._domainkey.caminosdelassierras.com.ar` | `s1.domainkey.u1234567.wl.sendgrid.net` | DKIM (selector 1) |
| 3 | CNAME | `s2._domainkey.caminosdelassierras.com.ar` | `s2.domainkey.u1234567.wl.sendgrid.net` | DKIM (selector 2) |
| 4 | TXT | `caminosdelassierras.com.ar` | Modificar SPF: agregar `include:sendgrid.net` | SPF |

**Total**: 3 CNAME nuevos + 1 modificacion SPF = **4 cambios DNS**

**Nota sobre Cloudflare**: Los CNAME deben configurarse como **DNS Only (grey cloud)**, no Proxied (orange cloud).

**SPF actualizado**:
```
v=spf1 include:_spf.google.com include:sendgrid.net ~all
```

### A4. Alineacion DMARC

Con Domain Authentication configurado:
- **DKIM alineado**: Si (`d=caminosdelassierras.com.ar` via CNAME chain)
- **SPF alineado**: No por default (Return-Path usa `bounces.sendgrid.net`). Alineacion SPF disponible con Return-Path custom (plan Pro+)
- **DMARC**: **PASS** (DKIM alignment es suficiente con `p=reject`)

### A5. Reply-To y Custom Headers

**Reply-To**: Control total, independiente del From.

**Custom Headers**: Soportados (In-Reply-To, References, X-Custom, etc.)

---

## Challenge B - Thread Tracking

### B1. Threading

**SendGrid NO mantiene threading automaticamente.** Implementacion manual requerida (igual que todos los proveedores evaluados):

1. Al recibir email inbound: extraer `Message-ID`, `In-Reply-To`, `References`, `Subject`
2. Almacenar en base de datos vinculado al caso/contacto
3. Al enviar respuesta: setear `In-Reply-To`, `References`, `Subject` con "Re: "

**Message-ID**: SendGrid genera automaticamente. Se puede setear custom via headers (verificar en testing).

**Compatibilidad de threading**:
- Gmail: Funciona via `References`/`In-Reply-To` + mismo Subject
- Outlook: Funciona via `In-Reply-To`/`References` + `Thread-Topic` (opcional)
- Apple Mail: Funciona via `In-Reply-To`/`References`

---

## Challenge C - Email Tracking

### C1. Funcionalidades de Tracking

SendGrid ofrece el **tracking mas completo y maduro** de los proveedores evaluados:

| Metrica | Disponible | Detalle |
|---|---|---|
| **Delivered** | Si | Email aceptado por el servidor destino |
| **Opened** | Si | Via pixel de tracking (1x1 transparent GIF) |
| **Clicked** | Si | Via reescritura de URLs |
| **Bounced** | Si | Hard bounce + soft bounce diferenciados |
| **Dropped** | Si | Email no enviado (suppression list, invalido) |
| **Deferred** | Si | Fallo temporal, SendGrid reintenta |
| **Spam Report** | Si | Destinatario marco como spam |
| **Unsubscribe** | Si | Click en link de unsub |
| **Processed** | Si | Aceptado por SendGrid para envio |

### C2. Event Webhook

Eventos entregados via **Event Webhook** (HTTP POST, JSON array, batched).

**Seguridad**: Firma ECDSA para verificar autenticidad de webhooks (`@sendgrid/eventwebhook`).

**Retry**: SendGrid reintenta Event Webhooks con backoff exponencial si el endpoint falla.

### C3. Tracking por Plan

El tracking esta **incluido en TODOS los planes** (incluyendo trial):
- Open tracking: Si
- Click tracking: Si
- Event Webhook: Si
- Deliverability Insights: Solo Pro+

---

## Pricing (2025-2026)

> **Actualizacion importante (mayo 2025)**: Twilio retiro los planes gratuitos permanentes de SendGrid. Solo existe un **free trial de 60 dias**, tras el cual se requiere upgrade a un plan pago.

| Plan | Precio/mes | Emails/mes | IP Dedicada | Notas |
|---|---|---|---|---|
| **Free Trial** | $0 (60 dias) | 100/dia (~3,000) | No | Temporal, no permanente |
| **Essentials** | $19.95 | 50,000 | No | Minimo sostenido |
| **Pro** | $89.95 | 100,000 | Incluida | Return-Path custom, analytics avanzados |
| **Premier** | Custom | Custom | Incluida | CSM dedicado, SSO |

**Costo por 1,000 emails**:

| Plan | Volumen | Costo/1K |
|---|---|---|
| Essentials 50K | 50,000/mes | ~$0.40 |
| Essentials 100K | 100,000/mes | ~$0.35 |
| Pro 100K | 100,000/mes | ~$0.90 |
| Pro 300K | 300,000/mes | ~$0.50 |

**Nota**: Bajo contrato existente, los precios pueden ser mas favorables que los listados publicamente.

---

## Integracion Tecnica

### SDK y API

| Componente | Paquete |
|---|---|
| Envio de email | `@sendgrid/mail` |
| Cliente REST general | `@sendgrid/client` |
| Parser inbound | `@sendgrid/inbound-mail-parser` |
| Verificacion webhooks | `@sendgrid/eventwebhook` |

---

## Limitaciones y Consideraciones

| Limitacion | Detalle | Impacto |
|---|---|---|
| **Sin free tier permanente** | Solo trial 60 dias, luego minimo $19.95/mes | Bajo (contrato existente) |
| **Inbound Parse sin retry** | Si el webhook esta caido, emails se pierden | Alto (mitigar manteniendo copia en Google Workspace) |
| **4 DNS changes** | 3 CNAME + SPF modification | Moderado (gestion coordinada con IT del cliente) |
| **IP compartida en Essentials** | Deliverability depende de otros senders en la IP | Bajo-Medio |
| **Event Webhook batching** | Eventos pueden llegar con delay (segundos a minutos) | Bajo |
| **Open tracking poco confiable** | Apple Mail Privacy, image blocking | Bajo (limitacion universal) |
| **Timeout de webhook** | 3s Inbound Parse, 5s Event Webhook | Bajo (procesar async) |
| **Tamanio max Inbound Parse** | 30 MB | Bajo |

---

## Ventajas Especificas para Este Caso

1. **Contrato existente**: Sin proceso de adquisicion, facturacion ya integrada
2. **Experiencia del equipo**: Menor tiempo de desarrollo y debugging
3. **Tracking completo**: El mas maduro de los evaluados, con webhook firmados (ECDSA)
4. **SDK robusto**: `@sendgrid/mail` es estable, bien mantenido y con tipado TypeScript
5. **Inbound Parse nativo**: Webhook POST con email parseado (vs DIY de SES)
6. **Seguridad de webhooks**: Firma ECDSA para verificar autenticidad

## Desventajas

1. **Costo**: Mas caro que Amazon SES a cualquier volumen ($0.40/1K vs $0.10/1K)
2. **4 DNS changes**: Mas que Brevo (1) pero menos que SES (5)
3. **Sin free tier permanente**: Trial de 60 dias solamente
4. **Inbound Parse sin retry**: Riesgo de perdida de emails si el webhook falla

---

## Resumen Ejecutivo

| Criterio | Evaluacion |
|---|---|
| **Inbound email** | Nativo via Inbound Parse (webhook POST) |
| **Outbound autenticado** | Si, con Domain Authentication (4 DNS changes) |
| **Threading** | Manual pero straightforward |
| **Tracking** | El mas completo y maduro de todos los evaluados |
| **Webhooks** | Si, con firma ECDSA y retry |
| **Precio POC** | $0 (trial 60 dias) o bajo contrato existente |
| **Precio produccion** | $19.95-89.95/mes |
| **Integracion NestJS** | Excelente (SDK oficial, tipado TS) |
| **Setup DNS** | 4 cambios (3 CNAME + SPF) |
| **DMARC compliance** | Si (DKIM alineado via Domain Auth) |
| **Experiencia del equipo** | **Si** (contrato y experiencia previa) |

---

## Referencias

### Pricing
- SendGrid Pricing & Plans: <https://sendgrid.com/en-us/pricing>
- Twilio SendGrid Trial Account Plan: <https://support.sendgrid.com/hc/en-us/articles/35270136965403-Twilio-SendGrid-Trial-Account-Plan>
- Cambios en free tier (mayo-julio 2025): <https://support.sendgrid.com/hc/en-us/articles/35270136965403-Twilio-SendGrid-Trial-Account-Plan>

### Inbound Parse
- Inbound Email Parse Webhook: <https://www.twilio.com/docs/sendgrid/for-developers/parsing-email/inbound-email>
- Configure the Inbound Parse Webhook: <https://www.twilio.com/docs/sendgrid/for-developers/parsing-email/setting-up-the-inbound-parse-webhook>
- Inbound Parse (UI settings): <https://www.twilio.com/docs/sendgrid/ui/account-and-settings/inbound-parse>
- Securing your Inbound Parse Webhooks: <https://www.twilio.com/docs/sendgrid/for-developers/parsing-email/securing-your-parse-webhooks>

### Domain Authentication (DKIM/SPF)
- DKIM Records Explained: <https://www.twilio.com/docs/sendgrid/ui/account-and-settings/dkim-records>
- SendGrid Automated Security - Domain Authentication: <https://support.sendgrid.com/hc/en-us/articles/21415314709147-SendGrid-Automated-Security>
- Custom DKIM Selector for Authentication: <https://support.sendgrid.com/hc/en-us/articles/4408024201627-Use-a-Custom-DKIM-Selector-for-Authentication>

### Event Webhook (Tracking)
- Event Webhook Reference: <https://docs.sendgrid.com/for-developers/tracking-events/event>
- Getting started with the Event Webhook: <https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/getting-started-event-webhook>
- How to Track Email Opens: <https://support.sendgrid.com/hc/en-us/articles/360041790933-How-to-Track-Email-Opens>

### SDK y API
- `@sendgrid/mail` (npm): <https://www.npmjs.com/package/@sendgrid/mail>
- `@sendgrid/client` (npm): <https://www.npmjs.com/package/@sendgrid/client>
- `@sendgrid/eventwebhook` (npm): <https://www.npmjs.com/package/@sendgrid/eventwebhook>
- SendGrid API Documentation: <https://docs.sendgrid.com>
- SendGrid v3 API Reference: <https://www.twilio.com/docs/sendgrid/api-reference>
