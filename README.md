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

| Documento                          | Descripcion                                                                                                                              |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| [requirements.md](requirements.md) | Objetivo, challenges tecnicos, criterios de aceptacion y entregables esperados de la POC                                                 |
| [dns-analysis.md](dns-analysis.md) | Analisis de registros DNS del dominio (MX, SPF, DKIM, DMARC). Hallazgo: DMARC `p=reject` y configuracion parcial pre-existente con Brevo |

### Evaluacion por Proveedor

| Documento                                                                        | Proveedor                       | Hallazgo clave                                                                                                                                                           |
| -------------------------------------------------------------------------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [findings-amazon-ses.md](findings-amazon-ses.md)                                 | Amazon SES                      | Contrato existente + experiencia del equipo. Costo mas bajo a escala ($0.10/1K), integracion AWS nativa. Requiere 5 DNS changes y pipeline DIY para inbound |
| [findings-sendgrid.md](findings-sendgrid.md)                                     | SendGrid (Twilio)               | Contrato existente + experiencia del equipo. Tracking mas completo y maduro, SDK robusto, inbound parse nativo. Sin free tier permanente ($19.95/mes min)                |
| [findings-google-workspace-gmail-api.md](findings-google-workspace-gmail-api.md) | Gmail API                       | Sin cambios de DNS y costo $0, pero sin tracking nativo y limite de emails/dia                                                                                           |
| [findings-direct-smtp.md](findings-direct-smtp.md)                               | SMTP Directo + Provider Inbound | Sin cambios de DNS usando SMTP del cliente para outbound + forwarding a provider para inbound.                                                                           |
| [findings-brevo.md](findings-brevo.md)                                           | Brevo (Sendinblue)              | DKIM ya configurado en DNS, sólo un cambio DNS necesario (SPF). Sin experiencia ni contrato existente                                                                    |

### Comparativas y Recomendacion

| Documento                                        | Descripcion                                                                                                                      |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| [comparison-features.md](comparison-features.md) | Tabla comparativa de features por challenge (envio/recepcion, threading, tracking, integracion, deliverability, limites)         |
| [comparison-costs.md](comparison-costs.md)       | Comparativa de costos por proveedor, escenarios de volumen, TCO a 12 meses, ranking por costo                                    |
| [recommendation.md](recommendation.md)           | Recomendacion final, matriz de decision ponderada, diagramas de flujo, riesgos, supuestos, dependencias y plan de implementacion |

## Recomendacion Final (resumen)

**Opcion A (Recomendada): Amazon SES** - Score 7.8/10

- Experiencia del equipo y contratos vigentes con AWS
- Costo mas bajo a escala ($0.10/1K, sin suscripcion mensual)
- Integracion nativa con stack existente (SNS, SQS, S3)

**Opcion B: SendGrid** - Score 7.4/10

- Contrato existente con SendGrid, experiencia del equipo
- Tracking mas maduro y setup mas rapido (inbound parse nativo)
- Desde $19.95/mes (Essentials 50K) hasta $499/mes (Pro 700K)

**Opcion C (Fallback sin DNS): Gmail API** - Score 7.2/10

- Sin cambios en DNS, costo $0
- Ideal si la coordinacion para los cambios del DNS resulta muy lenta
- Sin tracking nativo, limite 2000/dia via Gmail API y 10000/dia via SMTP Relay

**Opcion D: SMTP Directo** - Score 6.7/10

- Sin cambios en DNS, sin costo, DMARC nativo
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
