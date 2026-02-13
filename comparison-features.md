# Comparativa de Features - Proveedores de Email

> Evaluación técnica para la POC de mail channel de Caminos de las Sierras

## Tabla Comparativa General

| Feature                | Amazon SES                | SendGrid            | Gmail API                  | SMTP Directo     | Brevo               |
| ---------------------- | ------------------------- | ------------------- | -------------------------- | ---------------- | ------------------- |
| **Tipo de servicio**   | Email transaccional (AWS) | Email transaccional | API de usuario (Workspace) | SMTP del cliente | Email transaccional |
| **SDK Node.js**        | `@aws-sdk/client-sesv2`   | `@sendgrid/mail`    | `googleapis`               | `nodemailer`     | `@getbrevo/brevo`   |
| **API REST**           | SES v2 API                | v3 REST             | Gmail API v1               | N/A (SMTP)       | v3 REST             |
| **SMTP Relay**         | Si                        | Si                  | No (solo API)              | Si (nativo)      | Si                  |
| **Contrato existente** | Si                        | Si                  | No (Workspace del cliente) | N/A              | No                  |
| **Experiencia equipo** | Si                        | Si                  | No                         | N/A              | No                  |

---

## Challenge A - Envio y Recepcion

| Criterio                                    | Amazon SES                       | SendGrid              | Gmail API                  | SMTP Directo                  | Brevo                         |
| ------------------------------------------- | -------------------------------- | --------------------- | -------------------------- | ----------------------------- | ----------------------------- |
| **Inbound parsing**                         | DIY (S3 + SNS)                   | Nativo (webhook POST) | Nativo (Pub/Sub + history) | Forwarding + provider         | Nativo (webhook POST)         |
| **Formato inbound**                         | Raw MIME (necesita mailparser)   | multipart/form-data   | JSON (Gmail API format)    | Depende del provider          | JSON                          |
| **Requiere cambio MX**                      | No (usar forwarding)             | No (usar forwarding)  | No                         | No                            | No (usar forwarding)          |
| **Envio desde @caminosdelassierras.com.ar** | Si (con DNS auth)                | Si (con DNS auth)     | Si (nativo)                | Si (nativo, SMTP del cliente) | Si (parcialmente configurado) |
| **DNS changes requeridos**                  | 5 (3 CNAME + MX + SPF subdomain) | 4 (3 CNAME + SPF)     | 0                          | 0                             | 1 (solo SPF)                  |
| **DMARC p=reject compatible**               | Si (via Easy DKIM)               | Si (via DKIM)         | Si (nativo)                | Si (nativo)                   | Si (DKIM ya configurado)      |
| **Reply-To controlable**                    | Si                               | Si                    | Si                         | Si                            | Si                            |
| **Custom headers**                          | Si (v2 API o raw)                | Si                    | Si (raw RFC 2822)          | Si (Nodemailer)               | Si                            |
| **Sin caambios DNS**                        | No                               | No                    | Si                         | Si                            | Parcial (DKIM ya esta)        |

### Veredicto Challenge A

- **Gmail API y SMTP Directo**: No requieren cambios DNS (email sale directamente de Google Workspace / SMTP del cliente)
- **Brevo**: Menor esfuerzo DNS (1 solo cambio) gracias a DKIM pre-existente, pero sin contrato ni experiencia
- **SendGrid**: 4 cambios DNS, contrato y experiencia existentes
- **Amazon SES**: 5 registros DNS (en subdominios, bajo riesgo), contrato y experiencia existentes, mejor costo a escala

---

## Challenge B - Thread Tracking

| Criterio                   | Amazon SES                | SendGrid                     | Gmail API                     | SMTP Directo           | Brevo                     |
| -------------------------- | ------------------------- | ---------------------------- | ----------------------------- | ---------------------- | ------------------------- |
| **Message-ID automatico**  | Si                        | Si                           | Si                            | Si (generado por SMTP) | Si                        |
| **Custom Message-ID**      | No (SES genera el suyo)   | Via headers (no garantizado) | No (Gmail genera el suyo)     | Si (via Nodemailer)    | No (Brevo genera el suyo) |
| **In-Reply-To soportado**  | Si (v2 API Headers / raw) | Si (via headers)             | Si (raw RFC 2822)             | Si (via Nodemailer)    | Si (via headers API)      |
| **References soportado**   | Si (v2 API Headers / raw) | Si (via headers)             | Si (raw RFC 2822)             | Si (via Nodemailer)    | Si (via headers API)      |
| **Threading automatico**   | No (manual)               | No (manual)                  | **Parcial** (threadId nativo) | No (manual)            | No (manual)               |
| **Thread ID nativo**       | No                        | No                           | **Si** (threadId de Gmail)    | No                     | No                        |
| **Recuperar conversacion** | Manual (via DB propia)    | Manual (via DB propia)       | **Nativo** (threads.get)      | Manual (via DB propia) | Manual (via DB propia)    |

### Veredicto Challenge B

- **Gmail API**: Ventaja significativa con `threadId` nativo y capacidad de recuperar conversaciones completas via API. Aun requiere setear headers manualmente para que el threading funcione en otros clientes.
- **Todos los demas**: Requieren implementacion manual equivalente (almacenar Message-IDs, setear In-Reply-To/References). El esfuerzo es el mismo independientemente del proveedor.

---

## Challenge C - Email Tracking

| Criterio                         | Amazon SES                  | SendGrid              | Gmail API             | SMTP Directo          | Brevo                 |
| -------------------------------- | --------------------------- | --------------------- | --------------------- | --------------------- | --------------------- |
| **Open tracking**                | Nativo (config set)         | Nativo (pixel)        | DIY (pixel manual)    | DIY (pixel manual)    | Nativo (pixel)        |
| **Click tracking**               | Nativo (config set)         | Nativo (link rewrite) | DIY (link wrapping)   | DIY (link wrapping)   | Nativo (link rewrite) |
| **Delivered**                    | SNS/SQS/EventBridge         | Webhook               | No                    | No                    | Webhook               |
| **Bounced**                      | SNS (hard + soft)           | Webhook               | Manual (parsear NDRs) | Manual (parsear NDRs) | Webhook (hard + soft) |
| **Spam report**                  | SNS (complaint)             | Webhook               | No                    | No                    | Webhook               |
| **Deferred**                     | No nativo                   | Webhook               | No                    | No                    | Webhook               |
| **Forward detection**            | No                          | No                    | No                    | No                    | No                    |
| **Replied detection**            | No (solo via inbound)       | No (solo via inbound) | Si (via Pub/Sub)      | No                    | No (solo via inbound) |
| **Webhooks real-time**           | Si (via SNS)                | Si (batched JSON)     | Parcial (Pub/Sub)     | No                    | Si (individual)       |
| **Dashboard analytics**          | CloudWatch                  | Si (30 dias)          | No                    | No                    | Si                    |
| **Supression lists**             | Automaticas (account-level) | Automaticas           | No                    | No                    | Automaticas           |
| **Incluido en todos los planes** | Si                          | Si                    | N/A                   | N/A                   | Si                    |

### Veredicto Challenge C

- **SendGrid**: Tracking mas completo y maduro out-of-the-box. **Contrato y experiencia existentes**.
- **Amazon SES**: Tracking completo via Configuration Sets + SNS/SQS pipeline. Requiere mas setup pero se integra con infra AWS existente. **Contrato y experiencia existentes**.
- **Brevo**: Tracking completo y simple de configurar via webhooks. Sin contrato ni experiencia.
- **SMTP Directo**: **Sin tracking nativo** en outbound. Requiere implementacion DIY (pixel + link wrapping).
- **Gmail API**: **Sin tracking nativo**. Requiere implementacion DIY con menor confiabilidad.

---

## Integracion con Stack Existente

| Criterio                   | Amazon SES | SendGrid | Gmail API | SMTP Directo    | Brevo |
| -------------------------- | ---------- | -------- | --------- | --------------- | ----- |
| **NestJS compatible**      | Si         | Si       | Si        | Si (Nodemailer) | Si    |
| **AWS nativo**             | Si         | No       | No        | No              | No    |
| **SNS/SQS integracion**    | Si         | No       | No        | No              | No    |
| **Contrato existente**     | Si         | Si       | No        | N/A             | No    |
| **Experiencia del equipo** | Si         | Si       | No        | N/A             | No    |
| **Complejidad de setup**   | Media-Alta | Baja     | Media     | Media           | Baja  |

### Veredicto Integracion

- **Amazon SES**: Mejor integracion con el stack AWS existente (ECS, SNS, SQS, S3, IAM). **Contrato existente y experiencia del equipo.**
- **SendGrid**: Setup rapido, webhook-based. **Contrato existente y experiencia del equipo.**
- **SMTP Directo**: Usa Nodemailer (estandar en NestJS), sin dependencia de terceros para outbound.
- **Gmail API**: Requiere setup de GCP (service account, Pub/Sub, domain-wide delegation)
- **Brevo**: Setup rapido pero **sin experiencia ni contrato**. SDK con limitaciones de tipado.

---

## Confiabilidad y Deliverability

| Criterio                            | Amazon SES                      | SendGrid                   | Gmail API         | SMTP Directo                      | Brevo                |
| ----------------------------------- | ------------------------------- | -------------------------- | ----------------- | --------------------------------- | -------------------- |
| **IP dedicada**                     | $24.95/IP/mes                   | Desde plan Pro ($89.95/mo) | N/A (Google IPs)  | N/A (IPs del cliente)             | Desde plan Business  |
| **IP compartida**                   | Si (default)                    | Si (planes bajos)          | N/A (Google IPs)  | N/A                               | Si (planes bajos)    |
| **Reputacion de IPs**               | Alta                            | Alta                       | Muy alta (Google) | Depende del proveedor del cliente | Media-Alta           |
| **Warmup necesario**                | Si (IP nueva)                   | Si (IP nueva)              | No                | No                                | Si (IP nueva)        |
| **Bounce rate monitoring**          | Si (automatico, threshold 5%)   | Si                         | No                | No                                | Si                   |
| **Complaint rate monitoring**       | Si (automatico, threshold 0.1%) | Si                         | No                | No                                | Si                   |
| **Sandbox/Restricciones iniciales** | Si (sandbox, 200/dia)           | No (envio inmediato)       | No                | No                                | No (envio inmediato) |

---

## Limites de Envio

| Criterio                 | Amazon SES                 | SendGrid                      | Gmail API                                                                             | SMTP Directo                            | Brevo                |
| ------------------------ | -------------------------- | ----------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------- | -------------------- |
| **Free tier envio**      | 62,000/mes (desde EC2/ECS) | Trial 60 dias (no permanente) | 2,000/dia/usuario (API/SMTP), hasta 10,000/dia (SMTP Relay)                           | N/A (usa cuota del cliente)             | 300/dia (permanente) |
| **Limite maximo diario** | 50,000/dia (ajustable)     | Segun plan                    | 2,000/dia via API (hard cap); hasta 10,000/dia via SMTP Relay (limite fijo de Google) | 2,000-10,000/dia (segun proveedor SMTP) | Segun plan           |
| **Escalabilidad**        | Muy alta (millones)        | Alta (millones)               | Baja-Media (2,000/dia API; hasta 10,000/dia SMTP Relay)                               | Limitada por SMTP del cliente           | Alta (segun plan)    |
| **Tamanio max mensaje**  | 10 MB (v1) / 40 MB (v2)   | 30 MB                         | 25 MB (35 MB upload via API)                                                          | 25 MB (Gmail) / 25 MB (M365)            | 4 MB (5 MB total)    |
| **Max recipients/msg**   | 50                         | 1,000                         | N/A                                                                                   | Depende del SMTP                        | 99                   |

---

## Resumen de Fortalezas y Debilidades

### Amazon SES

- **Fortaleza**: Costo mas bajo a escala ($0.10/1K), integracion AWS nativa, 62K free/mes, **contrato existente y experiencia del equipo**
- **Debilidad**: Setup inbound requiere pipeline DIY (S3/SNS/SQS), sandbox inicial (24-48h), 5 DNS changes (en subdominios, bajo riesgo)

### SendGrid

- **Fortaleza**: Tracking mas completo y maduro, Inbound Parse nativo (setup rapido), SDK robusto, **contrato existente y experiencia del equipo**
- **Debilidad**: Costo mayor que SES ($0.40/1K), requiere 4 DNS changes, Inbound Parse no tiene retry, **sin free tier permanente** (solo trial 60 dias, minimo $19.95/mes)

### Gmail API

- **Fortaleza**: 0 DNS changes, 0 costo adicional, threading nativo, entregabilidad de Google
- **Debilidad**: Sin tracking nativo, limite 2,000/dia via API (hasta 10,000/dia via SMTP Relay, limite fijo de Google), requiere Workspace admin

### SMTP Directo

- **Fortaleza**: Zero DNS changes, zero costo por email, DMARC nativo, email genuino del cliente
- **Debilidad**: Sin tracking nativo en outbound (DIY), limites del SMTP del cliente, gestion de credenciales multi-tenant

### Brevo

- **Fortaleza**: Solo 1 DNS change (DKIM ya configurado en dominio), setup rapido, tracking completo
- **Debilidad**: **Sin experiencia ni contrato existente**, SDK menos maduro, inbound parsing menos documentado, cuenta pre-existente inaccesible
