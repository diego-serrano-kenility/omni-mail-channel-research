# Comparativa de Costos - Proveedores de Email

> Evaluación económica para la POC de mail channel de Caminos de las Sierras

## Resumen de Pricing por Proveedor

### SendGrid

> **Actualización importante (mayo 2025)**: Twilio retiró los planes gratuitos permanentes de SendGrid. Actualmente solo existe un **free trial de 60 días**, tras el cual se requiere upgrade a un plan pago. [Fuente](https://www.twilio.com/en-us/changelog/changes-coming-to-sendgrid-s-free-plans)

| Plan | Precio/mes | Emails/mes | Inbound Parse | Tracking | IP Dedicada |
|---|---|---|---|---|---|
| **Free Trial (60 días)** | $0 (temporal) | 100/día (~3,000) | Incluido | Incluido | No |
| **Essentials 50K** | $19.95 | 50,000 | Incluido | Incluido | No |
| **Essentials 100K** | $34.95 | 100,000 | Incluido | Incluido | No |
| **Pro 100K** | $89.95 | 100,000 | Incluido | Incluido | Incluida |
| **Pro 300K** | $249.00 | 300,000 | Incluido | Incluido | Incluida |
| **Pro 700K** | $499.00 | 700,000 | Incluido | Incluido | Incluida |
| **Premier** | Custom | Custom | Incluido | Incluido | Incluida |

### Amazon SES

| Concepto | Precio |
|---|---|
| **Envío (desde ECS)** | Primeros 62,000/mes: **$0** / Luego: $0.10 por 1,000 |
| **Recepción** | Primeros 1,000/mes: **$0** / Luego: $0.10 por 1,000 |
| **Attachments** | $0.12 por GB |
| **IP Dedicada** | $24.95/mes por IP |
| **Virtual Deliverability Manager** | $0.07 por 100 emails |
| **SNS notifications** | Free (primer 1M requests/mes) |
| **S3 storage** | ~$0.023/GB/mes |

### Gmail API (Google Workspace)

| Concepto | Precio |
|---|---|
| **Google Workspace** | Ya pagado por el cliente (desde $7.20/user/mes) |
| **Gmail API** | $0 (incluido en Workspace) |
| **Pub/Sub** | $0 (free tier cubre el uso) |
| **Costo adicional total** | **$0/mes** |

### Brevo

| Plan | Precio/mes | Emails/mes | Inbound | Tracking | Webhooks |
|---|---|---|---|---|---|
| **Free** | $0 | 300/día (~9,000) | Incluido | Incluido | Incluido |
| **Starter** | $9 | 5,000 | Incluido | Incluido | Incluido |
| **Business** | $18 | 5,000 | Incluido | Incluido | Incluido |
| **Transaccional 20K** | ~$15/mes | 20,000 | Incluido | Incluido | Incluido |
| **Transaccional 40K** | ~$25/mes | 40,000 | Incluido | Incluido | Incluido |

---

## Precio por Cada 1,000 Emails

### Costo por 1,000 Emails Enviados

| Proveedor | Free Tier | Precio por 1,000 enviados (post free tier) | Notas |
|---|---|---|---|
| **SendGrid** | 100/día por 60 días (trial, no permanente) | **$0.35 - $0.90** (varía según plan) | Essentials 50K: $0.40/1K · Essentials 100K: $0.35/1K · Pro 100K: $0.90/1K · Pro 300K: $0.83/1K · Pro 700K: $0.71/1K. Sin free tier permanente desde mayo 2025 |
| **Amazon SES** | 62,000/mes gratis (desde ECS) | **$0.10** | Precio fijo independiente del volumen. El más económico a escala |
| **Gmail API** | Incluido en Workspace | **$0.00** | Sin costo por email. Hard cap de 2,000 emails/día por usuario |
| **Brevo** | 300/día (~9,000/mes) gratis | **$0.63 - $0.75** | 20K: $0.75/1K · 40K: $0.63/1K · 100K: $0.65/1K · 150K: $0.63/1K |

### Costo por 1,000 Emails Recibidos

| Proveedor | Free Tier | Precio por 1,000 recibidos (post free tier) | Notas |
|---|---|---|---|
| **SendGrid** | Ilimitado (en planes pagos) | **$0.00** | Inbound Parse incluido en todos los planes sin costo adicional |
| **Amazon SES** | 1,000/mes gratis | **$0.10** + $0.09/1K chunks (256KB c/u) | Email de 1KB = 1 chunk · Email de 300KB = 2 chunks |
| **Gmail API** | Incluido en Workspace | **$0.00** | Lectura de inbox via API, sin costo por email recibido |
| **Brevo** | Ilimitado | **$0.00** | Inbound Parsing incluido en todos los planes sin costo adicional |

### Resumen Comparativo: Costo por 1,000 Emails (Envío + Recepción)

| Proveedor | Envío /1K | Recepción /1K | **Total /1K (envío + recepción)** |
|---|---|---|---|
| **Gmail API** | $0.00 | $0.00 | **$0.00** |
| **Amazon SES** | $0.10 | $0.10 | **$0.20** |
| **SendGrid** | $0.35 - $0.90 | $0.00 | **$0.35 - $0.90** |
| **Brevo** | $0.63 - $0.75 | $0.00 | **$0.63 - $0.75** |

> **Nota**: Los precios de envío son post-free tier. Gmail API tiene un hard cap de 2,000/día que no puede superarse. Amazon SES es el más económico a escala para envíos que superen el free tier. SendGrid ya no tiene free tier permanente (solo trial de 60 días desde mayo 2025). Brevo incluye la recepción sin costo en todos sus planes.

---

## Escenarios de Costo por Volumen

### Escenario POC (< 100 emails/día)

| Proveedor | Plan recomendado | Costo mensual | Notas |
|---|---|---|---|
| **SendGrid** | Free Trial (60 días) → Essentials | **$0 → $19.95** | Trial temporal; post-trial requiere plan Essentials |
| **Amazon SES** | Pay-as-you-go | **~$0** | Free tier desde ECS |
| **Gmail API** | Workspace existente | **$0** | Sin costo adicional |
| **Brevo** | Free | **$0** | Límite 300/día suficiente para POC (free tier permanente) |

### Escenario Producción Baja (500-1,000 emails/día)

| Proveedor | Plan recomendado | Costo mensual | Notas |
|---|---|---|---|
| **SendGrid** | Essentials | **$19.95** | 50K emails/mes |
| **Amazon SES** | Pay-as-you-go | **~$0-1** | Cubierto por free tier (62K/mes) |
| **Gmail API** | Workspace existente | **$0** | Dentro del límite de 2,000/día |
| **Brevo** | Starter/Transaccional | **$9-15** | 5K-20K emails/mes |

### Escenario Producción Media (2,000-5,000 emails/día)

| Proveedor | Plan recomendado | Costo mensual | Notas |
|---|---|---|---|
| **SendGrid** | Pro 100K | **$89.95** | 100K emails/mes + IP dedicada |
| **Amazon SES** | Pay-as-you-go | **~$5-9** | ~60K-150K emails/mes |
| **Gmail API** | **No viable** | N/A | Supera límite de 2,000/día |
| **Brevo** | Business/Transaccional | **$25-65** | 40K-100K emails/mes |

### Escenario Producción Alta (10,000+ emails/día)

| Proveedor | Plan recomendado | Costo mensual | Notas |
|---|---|---|---|
| **SendGrid** | Pro/Premier | **$249-499+** | Pro 300K-700K o Premier custom |
| **Amazon SES** | Pay-as-you-go | **~$24** | ~300K emails @ $0.10/1K |
| **Gmail API** | **No viable** | N/A | Hard cap de 2,000/día |
| **Brevo** | Business/Enterprise | **$95+** | 150K+ emails/mes |

---

## Costo Total de Ownership (TCO) a 12 meses

### Escenario más probable: Producción Baja (500 emails/día, ~15,000/mes)

| Concepto | SendGrid | Amazon SES | Gmail API | Brevo |
|---|---|---|---|---|
| **Suscripción mensual** | $19.95 (mínimo, sin free tier permanente) | $0 | $0 | $9-15 |
| **Costo por email** | Incluido | ~$0 (free tier) | $0 | Incluido |
| **IP Dedicada** | No incluida | $24.95 (opcional) | N/A | No incluida |
| **Infra adicional** | $0 | ~$1 (S3, SNS) | ~$0 (Pub/Sub) | $0 |
| **Costo mensual total** | **$19.95** | **~$1** | **~$0** | **$9-15** |
| **Costo anual** | **$239.40** | **~$12** | **~$0** | **$108-180** |

### Costos Ocultos / No Monetarios

| Factor | SendGrid | Amazon SES | Gmail API | Brevo |
|---|---|---|---|---|
| **Tiempo de desarrollo POC** | 1-2 días | 2-4 días | 2-4 días | 1-2 días |
| **Tiempo de config DNS** | ~30 min (3 records) | ~1 hora (5 records) | $0 | ~10 min (1 record) |
| **Coordinación con cliente** | Media | Alta | Baja (solo Workspace admin) | Baja |
| **Mantenimiento ongoing** | Bajo | Medio | Bajo (solo watch renewal) | Bajo |
| **Riesgo de bloqueo DNS** | Medio | Medio | Ninguno | Bajo |

---

## Ranking por Costo

### Para la POC
1. **Gmail API** - $0 (sin costo adicional)
2. **Amazon SES** - ~$0 (free tier desde ECS, **contrato existente**)
3. **Brevo** - $0 (free tier permanente, sin contrato)
4. **SendGrid** - $0 por 60 dias (trial) o bajo contrato existente

### Para Produccion (volumen bajo-medio)
1. **Gmail API** - $0/mes (si el volumen es < 2,000/dia)
2. **Amazon SES** - $0-5/mes (free tier desde ECS, **contrato existente**)
3. **SendGrid** - $19.95-499/mes segun plan (**contrato existente**, posibles condiciones preferenciales)
4. **Brevo** - $9-25/mes (sin contrato)

### Para Produccion (volumen alto)
1. **Amazon SES** - $24+/mes (el mas economico a escala, **significativamente menor que alternativas**)
2. **SendGrid** - $249-499+/mes (contrato existente)
3. **Brevo** - $65+/mes (sin contrato)
4. **Gmail API** - No viable (limite 2,000/dia)

---

## Notas Importantes

1. **Amazon SES** es el proveedor mas economico a cualquier escala gracias al free tier desde EC2/ECS y el precio base de $0.10/1,000 emails. **A partir de volumen moderado, la diferencia de costo es significativa**: $0.10/1K (SES) vs $0.35-0.90/1K (SendGrid segun plan) vs $0.63/1K (Brevo)
2. **Amazon SES y SendGrid** cuentan con **contratos existentes** y **experiencia del equipo**, lo que reduce costos ocultos de onboarding y curva de aprendizaje
3. **Gmail API** es completamente gratis pero tiene un hard cap de 2,000 emails/dia que no se puede aumentar
4. **SendGrid** **ya no tiene free tier permanente** (solo trial de 60 dias desde mayo 2025). El costo minimo sostenido es $19.95/mes (Essentials), pero el contrato existente puede ofrecer condiciones distintas
5. **Brevo** tiene free tier permanente (300 emails/dia) pero **no hay contrato ni experiencia previa**, lo que agrega costos de onboarding y evaluacion de proveedor
6. Todos los proveedores incluyen tracking y webhooks sin costo adicional (excepto Gmail API que no tiene tracking nativo)
