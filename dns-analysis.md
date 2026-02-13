# Análisis DNS - caminosdelassierras.com.ar

> **Nota**: El dominio correcto es `caminosdelassierras.com.ar` (doble "s", TLD argentino), no `caminosdelasierras.com` como figura en el documento de requerimientos.

## Metodología

La información DNS se obtuvo mediante consultas directas con `dig` contra los servidores autoritativos del dominio. Los comandos ejecutados fueron:

```bash
# Nameservers
dig NS caminosdelassierras.com.ar

# MX Records (email entrante)
dig MX caminosdelassierras.com.ar

# SPF (registro TXT del dominio raíz)
dig TXT caminosdelassierras.com.ar

# DKIM - selector de Google Workspace
dig TXT google._domainkey.caminosdelassierras.com.ar

# DKIM - selector de Brevo/Sendinblue
dig TXT mail._domainkey.caminosdelassierras.com.ar

# DMARC
dig TXT _dmarc.caminosdelassierras.com.ar

# A Records (sitio web)
dig A caminosdelassierras.com.ar
```

> Los selectores DKIM (`google`, `mail`) se verificaron por ser los más comunes para Google Workspace y Brevo respectivamente.

## Resumen Ejecutivo

El dominio tiene una **política DMARC de rechazo (`p=reject`)**, lo que significa que cualquier email enviado desde `@caminosdelassierras.com.ar` que no pase alineación SPF **ni** DKIM será **rechazado completamente por los servidores destinatarios**. DMARC requiere que pase **al menos uno** de los dos (SPF alineado O DKIM alineado). Esto es el hallazgo más crítico: no se puede enviar desde este dominio con un proveedor nuevo sin que al menos uno de los mecanismos de autenticación esté correctamente configurado en DNS.

## Registros DNS Encontrados

### Nameservers (DNS Provider)

| Registro | Valor                      |
| -------- | -------------------------- |
| NS       | `kate.ns.cloudflare.com`   |
| NS       | `austin.ns.cloudflare.com` |

**Implicación**: El DNS está gestionado en **Cloudflare**. Esto facilita agregar registros ya que Cloudflare tiene una interfaz amigable y propagación rápida.

### MX Records (Email Entrante)

| Prioridad | Servidor                  |
| --------- | ------------------------- |
| 1         | `aspmx.l.google.com`      |
| 5         | `alt1.aspmx.l.google.com` |
| 5         | `alt2.aspmx.l.google.com` |
| 10        | `alt3.aspmx.l.google.com` |
| 10        | `alt4.aspmx.l.google.com` |

**Implicación**: Usan **Google Workspace** para recibir email. Todos los emails a `@caminosdelassierras.com.ar` llegan a Gmail/Google Workspace.

### SPF Record

```
v=spf1 include:_spf.google.com ~all
```

**Análisis**:

- Solo autoriza servidores de **Google** para enviar email desde este dominio
- Usa `~all` (soft fail) - emails de otros servidores no se rechazan por SPF solo, pero se marcan como sospechosos
- **Para agregar un proveedor nuevo** (SendGrid, SES, etc.) se necesita modificar este registro para incluir sus servidores

### DKIM Records

| Selector            | Proveedor          | Estado                           |
| ------------------- | ------------------ | -------------------------------- |
| `google._domainkey` | Google Workspace   | Configurado (clave RSA 2048-bit) |
| `mail._domainkey`   | Brevo (Sendinblue) | Configurado (clave RSA 1024-bit) |

**Análisis**: Tienen DKIM configurado para Google (email normal) y para **Brevo/Sendinblue** (email transaccional o marketing). Esto indica que ya hubo un intento o uso de un servicio de email externo.

### DMARC Record

```
v=DMARC1; p=reject; rua=mailto:noresponder@caminosdelassierras.com.ar
```

**Análisis CRÍTICO**:

- **Política: `reject`** - La más estricta posible
- Cualquier email que no pase alineación SPF **Y** DKIM será **rechazado completamente**
- Los reportes de DMARC se envían a `noresponder@caminosdelassierras.com.ar`
- No hay `pct` definido, por lo que aplica al 100% de los emails

### Otros Registros

| Registro    | Valor                                         | Significado                        |
| ----------- | --------------------------------------------- | ---------------------------------- |
| TXT (Brevo) | `brevo-code:2e5dc272262e89f4a55726410d925b25` | Dominio verificado en Brevo        |
| A           | `104.21.21.182`, `172.67.199.201`             | Sitio web detrás de Cloudflare CDN |

## Hallazgos Clave

### 1. DMARC `reject` bloquea envíos no autorizados

Con la política `p=reject`, cualquier proveedor nuevo que quiera enviar como `@caminosdelassierras.com.ar` **debe** cumplir al menos una de estas condiciones:

- Estar incluido en el registro SPF (alineación SPF), **O**
- Tener una firma DKIM alineada con el dominio (alineación DKIM)

**Nota importante**: DMARC `p=reject` **no exige** que pasen SPF y DKIM simultáneamente. Basta con que **uno de los dos** esté alineado con el dominio del header `From`. Esto es relevante porque Brevo ya tiene DKIM alineado (`mail._domainkey`), lo que significa que sus emails probablemente pasen DMARC incluso sin estar incluido en el registro SPF. Sin embargo, se recomienda configurar ambos como best practice.

Sin al menos uno de estos mecanismos, los emails serán rechazados por los servidores destinatarios (Gmail, Outlook, Yahoo, etc.).

### 2. Brevo ya está parcialmente configurado

- El dominio ya está verificado en Brevo (registro TXT de verificación presente)
- Tiene DKIM configurado para Brevo (`mail._domainkey`)
- **Sin embargo**, el SPF **no incluye** los servidores de Brevo (`include:sendinblue.com`), lo que significa que los emails enviados vía Brevo actualmente fallan SPF (pero pueden pasar DMARC solo por DKIM)

### 3. Se requieren cambios DNS obligatorios

Para cualquier proveedor elegido, se necesitará como mínimo:

| Proveedor           | Cambio SPF requerido             | Cambio DKIM requerido   |
| ------------------- | -------------------------------- | ----------------------- |
| **SendGrid**        | Agregar `include:sendgrid.net`   | Agregar 2 CNAME records |
| **Amazon SES**      | Agregar `include:amazonses.com`  | Agregar 3 CNAME records |
| **Brevo**           | Agregar `include:sendinblue.com` | Ya configurado          |
| **Mailgun**         | Agregar `include:mailgun.org`    | Agregar 2 TXT records   |
| **Google (actual)** | Ya incluido                      | Ya configurado          |

### 4. Forwarding de inbound es viable

Como el email entrante llega a Google Workspace, se puede configurar un forwarding rule en Gmail/Google Admin para reenviar emails de `info@caminosdelassierras.com.ar` a un endpoint de inbound parsing del proveedor elegido.

## Impacto en la POC

| Aspecto                                   | Impacto                                                | Mitigacion                                                                                                                                           |
| ----------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Envio desde `@caminosdelassierras.com.ar` | Requiere cambios DNS (CNAME, TXT en Cloudflare)        | **Bajo** - Tramite operativo estandar. Los registros son en subdominios y no afectan la configuracion existente.                                     |
| DMARC `reject`                            | Sin DNS correcto, emails rechazados                    | **Medio** - Validar SPF+DKIM antes de enviar en produccion. Se puede testear en sandbox (SES) o con dominio de testing.                              |
| Brevo pre-existente                       | Existe DKIM configurado desde cuenta Brevo desconocida | **Informativo** - DKIM de Brevo ya esta en DNS. Sin embargo, no hay experiencia ni contrato con Brevo, y la cuenta original no ha sido identificada. |
| Google Workspace activo                   | El inbound parsing puede usar forwarding de Gmail      | **Positivo** - No requiere cambiar MX records del dominio principal.                                                                                 |

### Valoracion de los cambios DNS requeridos

Los cambios DNS necesarios para cualquier proveedor (SES, SendGrid, etc.) son **operaciones de bajo riesgo**:

- Se trata de **agregar** registros nuevos (CNAME, TXT), no de modificar ni eliminar registros existentes
- Los registros DKIM se agregan en subdominios (`_domainkey.*`) que no interfieren con el funcionamiento actual del email
- La unica modificacion a un registro existente es agregar un `include` al SPF, que no afecta los includes previos
- El proceso completo se puede comunicar en un unico pedido al equipo IT con instrucciones precisas
