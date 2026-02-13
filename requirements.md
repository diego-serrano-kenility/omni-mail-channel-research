## **Objetivo General**

Realizar una POC técnica para validar proveedores de e-mail que permitan:

1. Recibir e-mails enviados a la casilla oficial de Caminos de la Sierras.
2. Procesarlos en nuestros servidores.
3. Responder simulando ser Caminos de la Sierras (@caminosdelasierras.com).
4. Mantener threads/conversaciones.
5. Trackear métricas de interacción (open, click, etc.).
6. Evaluar costos y viabilidad técnica.

# **Escenario Funcional**

### **Flujo esperado:**

1. Cliente envía e-mail a info@caminosdelasierras.com
2. Se configura una regla de forwarding hacia una casilla técnica nuestra (ej: inbound-caminos@omnibrein.com)
3. Nuestro backend:
   - Recibe el e-mail
   - Lo procesa
   - Lo asocia a una interacción nueva o existente
4. Se genera una respuesta y se agrega a la interacción con marca de needs interaction:
5. Un operador aprueba la respuesta, opcionalmente modificando su contenido.
6. Se envía la respuesta por email con un remitente @caminosdelasierras.com
7. Se mantiene el thread correctamente para identificar la interacción si el cliente responde.
8. Se trackean métricas del e-mail enviado.

# **Challenges Técnicos**

## **Challenge A – Envío y Recepción**

Validar que el proveedor permita:

- Inbound parsing
- Envío desde dominio externo
- Configuración de SPF / DKIM / DMARC con terceros
- Reply-To controlado
- Manejo de headers

Validar:

- ¿Es posible enviar desde @caminosdelasierras.com?
- ¿Qué configuración DNS requiere?
- ¿Se puede hacer sin delegación completa?

## **Challenge B – Thread Tracking**

Debe permitir:

- Mantener conversación en mismo thread
- Uso correcto de:
  - Message-ID
  - In-Reply-To
  - References headers
- Recuperar conversación completa

Validar:

- ¿El proveedor mantiene threading automáticamente?
- ¿Requiere implementación custom?
- ¿Cómo se vincula con omnichannel?

## **Challenge C – Email Tracking**

Necesitamos evaluar si permite:

- Open tracking
- Click tracking
- Forward detection (si aplica)
- Webhooks de eventos:
  - delivered
  - bounced
  - opened
  - clicked
  - replied

Validar:

- Nivel de detalle disponible
- Planes que lo incluyen
- Costo adicional

# **Proveedores a Evaluar (mínimo)**

- SendGrid
- Amazon SES
- Google Workspace API (si usan Gmail)
- SMTP directo

# **Evaluación Económica**

Para cada proveedor documentar:

- Precio por envío
- Precio por inbound parsing
- Precio por tracking
- Costos mínimos mensuales
- Costos por volumen estimado

# **Supuestos / Riesgos Iniciales**

- No tenemos control del DNS de caminosdelasierras.com
- Puede requerir:
  - Agregar registros SPF/DKIM
  - Domain authentication
- Dependencia del equipo de Caminos de la Sierras para configuración

# **Criterios de Aceptación**

- Al menos 3 proveedores evaluados técnicamente.
- Se valida inbound \+ outbound en entorno de prueba.
- Se documenta si es posible enviar desde dominio externo sin control DNS.
- Se valida threading funcional.
- Se validan eventos de tracking.
- Se presenta recomendación técnica \+ económica.
