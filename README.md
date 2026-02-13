# Mail Channel Research - POC Caminos de las Sierras

> Investigacion tecnica para validar proveedores de email que permitan recibir, procesar, responder y trackear emails desde la plataforma Omni, enviando como `@caminosdelassierras.com.ar`.

## Contexto

- **Dominio**: `caminosdelassierras.com.ar` (Google Workspace, Cloudflare DNS)
- **DMARC**: `p=reject` (requiere SPF o DKIM alineado para enviar)
- **Restriccion principal**: No se tiene control directo del DNS; cambios requieren coordinacion con el equipo IT del cliente
- **Stack**: NestJS sobre AWS ECS, MongoDB, Redis
- **Contratos existentes**: AWS (infraestructura completa), Twilio/SendGrid (email)

## Indice de Documentos

### Requerimientos y Analisis Base

| Documento | Descripcion |
|---|---|
| [requirements.md](requirements.md) | Objetivo, challenges tecnicos, criterios de aceptacion y entregables esperados de la POC |
| [dns-analysis.md](dns-analysis.md) | Analisis de registros DNS del dominio (MX, SPF, DKIM, DMARC). Hallazgo: DMARC `p=reject` y configuracion parcial pre-existente con Brevo |

### Evaluacion por Proveedor

| Documento | Proveedor | Hallazgo clave |
|---|---|---|
| [findings-amazon-ses.md](findings-amazon-ses.md) | Amazon SES | **Contrato existente + experiencia del equipo**. Costo mas bajo a escala, integracion AWS nativa, 62K free/mes desde ECS. Requiere 5 DNS changes y pipeline DIY para inbound |
| [findings-sendgrid.md](findings-sendgrid.md) | SendGrid (Twilio) | **Contrato existente + experiencia del equipo**. Tracking mas completo y maduro, SDK robusto, inbound parse nativo. Sin free tier permanente ($19.95/mes min) |
| [findings-google-workspace-gmail-api.md](findings-google-workspace-gmail-api.md) | Gmail API | Zero DNS changes y costo $0, pero sin tracking nativo y limite de 2,000 emails/dia |
| [findings-direct-smtp.md](findings-direct-smtp.md) | SMTP Directo + Provider Inbound | Zero DNS changes usando SMTP del cliente para outbound + forwarding a provider para inbound. Enfoque hibrido para multi-tenant |
| [findings-brevo.md](findings-brevo.md) | Brevo (Sendinblue) | DKIM ya configurado en DNS, solo 1 cambio DNS pendiente (SPF). Sin experiencia ni contrato existente |

### Documentos Adicionales

| Documento | Descripcion |
|---|---|
| [sendgrid-poc-evaluation.md](sendgrid-poc-evaluation.md) | Evaluacion tecnica detallada original de SendGrid (referencia complementaria a findings-sendgrid.md) |

### Comparativas y Recomendacion

| Documento | Descripcion |
|---|---|
| [comparison-features.md](comparison-features.md) | Tabla comparativa de features por challenge (envio/recepcion, threading, tracking, integracion, deliverability, limites) |
| [comparison-costs.md](comparison-costs.md) | Comparativa de costos por proveedor, escenarios de volumen, TCO a 12 meses, ranking por costo |
| [recommendation.md](recommendation.md) | Recomendacion final, matriz de decision ponderada, diagramas de flujo, riesgos, supuestos, dependencias y plan de implementacion |

## Recomendacion Final (resumen)

**Opcion A (Recomendada): Amazon SES** - Score 7.8/10

- Experiencia del equipo y contratos vigentes con AWS
- Costo mas bajo a escala ($0.10/1K, 62K free/mes desde ECS)
- Integracion nativa con stack existente (IAM, SNS, SQS, S3)

**Opcion B: SendGrid** - Score 7.4/10

- Contrato existente con Twilio/SendGrid, experiencia del equipo
- Tracking mas maduro y setup mas rapido (inbound parse nativo)
- Desde $19.95/mes

**Opcion C (Fallback sin DNS): Gmail API** - Score 7.2/10

- Zero cambios DNS, costo $0
- Ideal si la coordinacion DNS resulta muy lenta
- Sin tracking nativo, limite 2,000/dia

**Opcion D: SMTP Directo** - Score 6.7/10

- Zero DNS, zero costo, DMARC nativo
- Opcion valida para vision multi-tenant futura
- Limitado por tracking DIY y gestion de credenciales

**Opcion E: Brevo** - Score 6.6/10

- DKIM pre-existente en DNS, solo 1 cambio DNS
- Sin experiencia ni contrato existente

## Estado del Analisis

- [x] Investigacion de DNS y autenticacion del dominio
- [x] Evaluacion tecnica de 5 proveedores/enfoques (SES, SendGrid, Gmail API, SMTP Directo, Brevo)
- [x] Comparativa de features y costos
- [x] Recomendacion final con matriz de decision (incluye factores organizacionales)
- [x] Identificacion de riesgos y dependencias
- [ ] Validacion practica en entorno de prueba (POC)
