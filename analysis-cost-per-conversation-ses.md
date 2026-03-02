# Análisis de Costo por Conversación - Estrategia Amazon SES

> **Fecha**: 2026-03-02
>
> **Objetivo**: Estimar el costo total de cada conversación de email utilizando Amazon SES como estrategia de recepción/envío, S3 Intelligent Tiering para almacenamiento, y un LLM para interpretación y respuesta automática. Se analizan tres escenarios de duración de conversación y cuatro rangos de tamaño del primer mail.

---

## 1. Supuestos Generales

### 1.1 Escenarios de Duración de Conversación

| Escenario | Mails totales | Inbound | Outbound | Descripción |
|---|---|---|---|---|
| **Corta** | 2 | 1 | 1 | Pregunta inicial + respuesta definitiva |
| **Estándar** | 6 | 3 | 3 | Tres idas y vueltas (caso promedio) |
| **Larga** | 10 | 5 | 5 | Discusión extendida, cinco idas y vueltas |

**Reglas comunes a todos los escenarios:**
- Solo el **primer mail entrante** tiene tamaño variable (ver distribución)
- Todos los demás mails (entrantes posteriores y salientes) pesan **≤ 256 KB**
- Los mails **salientes no se almacenan** en S3 (solo pagan fee SES outbound)

### 1.2 Distribución de Tamaño del Primer Mail Entrante

| Percentil | Tamaño máximo | Chunks SES (256 KB c/u) |
|---|---|---|
| **P60** (60% de mails) | ≤ 256 KB | 1 |
| **P90** (61%-90%) | ≤ 4 MB | 16 |
| **P95** (91%-95%) | ≤ 20 MB | 80 |
| **P99** (96%-99%) | ≤ 40 MB | 160 |

### 1.3 Estrategia de Almacenamiento S3

- Cada mail inbound se almacena como **raw email** (depositado por SES)
- El raw email se **mueve a otro prefix** (COPY + DELETE)
- Cada **part del mail** se almacena como objeto independiente
- Partes por email: **2 a 5** (promedio estimado: **3 partes**)
- Total de objetos por email inbound: **1 (raw) + 3 (partes) = 4 objetos**
- **Solo el 1%** de los mails se consultan pasados los 30 días → 99% migra a capas más baratas

| Escenario | Emails almacenados | Objetos S3 |
|---|---|---|
| Corta (2 mails) | 1 inbound | 4 |
| Estándar (6 mails) | 3 inbound | 12 |
| Larga (10 mails) | 5 inbound | 20 |

### 1.4 Storage Class: S3 Intelligent Tiering

| Capa | Condición | Precio/GB/mes |
|---|---|---|
| **Frequent Access** | Acceso reciente (< 30 días) | $0.023 |
| **Infrequent Access** | Sin acceso por 30+ días | $0.0125 |
| **Archive Instant Access** | Sin acceso por 90+ días | $0.004 |
| **Archive Access** | Sin acceso por 90-270+ días | $0.0036 |
| **Deep Archive** | Sin acceso por 180-730+ días | $0.00099 |

> **Nota**: Objetos menores a 128 KB no se mueven de capa y permanecen siempre en Frequent Access. No hay cargos por retrieval ni por transición entre capas.

### 1.5 Costos LLM por Escenario

Las tasas por token se derivan del caso estándar: **$0.30/1M input tokens** y **$2.50/1M output tokens**.

| Concepto | Corta (2 mails) | Estándar (6 mails) | Larga (10 mails) |
|---|---|---|---|
| **Input**: prompt inicial | 14,000 | 14,000 | 14,000 |
| **Input**: consultas entrantes | 1 × 1,000 = 1,000 | 3 × 1,000 = 3,000 | 5 × 1,000 = 5,000 |
| **Total input tokens** | 15,000 | 17,000 | 19,000 |
| **Costo input** | $0.0045 | $0.0051 | $0.0057 |
| **Output tokens** (respuestas) | ~1,333 | 4,000 | ~6,667 |
| **Costo output** | $0.0033 | $0.0100 | $0.0167 |
| **Total LLM** | **$0.0078** | **$0.0151** | **$0.0224** |

---

## 2. Desglose de Costos por Componente

### 2.1 Amazon SES - Recepción (Inbound)

**Tarifa base**: $0.10 / 1,000 emails = **$0.0001 por email**

**Tarifa por chunks**: $0.09 / 1,000 chunks = **$0.00009 por chunk** (cada 256 KB)

#### Conversación Corta (2 mails: 1 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (1 email) | $0.000100 | $0.000100 | $0.000100 | $0.000100 |
| Chunks email 1° | 1 | 16 | 80 | 160 |
| Costo chunks | $0.000090 | $0.001440 | $0.007200 | $0.014400 |
| **Total SES inbound** | **$0.000190** | **$0.001540** | **$0.007300** | **$0.014500** |

#### Conversación Estándar (6 mails: 3 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (3 emails) | $0.000300 | $0.000300 | $0.000300 | $0.000300 |
| Chunks (email 1° + 2 × 1) | 3 | 18 | 82 | 162 |
| Costo chunks | $0.000270 | $0.001620 | $0.007380 | $0.014580 |
| **Total SES inbound** | **$0.000570** | **$0.001920** | **$0.007680** | **$0.014880** |

#### Conversación Larga (10 mails: 5 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (5 emails) | $0.000500 | $0.000500 | $0.000500 | $0.000500 |
| Chunks (email 1° + 4 × 1) | 5 | 20 | 84 | 164 |
| Costo chunks | $0.000450 | $0.001800 | $0.007560 | $0.014760 |
| **Total SES inbound** | **$0.000950** | **$0.002300** | **$0.008060** | **$0.015260** |

### 2.2 Amazon SES - Envío (Outbound)

**Tarifa base**: $0.10 / 1,000 emails = **$0.0001 por email**

**Tarifa data**: $0.12 / GB

| Escenario | Emails | Tarifa base | Data (256KB c/u) | **Total outbound** |
|---|---|---|---|---|
| Corta (2 mails) | 1 | $0.000100 | $0.000030 | **$0.000130** |
| Estándar (6 mails) | 3 | $0.000300 | $0.000090 | **$0.000390** |
| Larga (10 mails) | 5 | $0.000500 | $0.000146 | **$0.000646** |

### 2.3 Amazon S3 - Almacenamiento

#### Volumen almacenado por conversación

El almacenamiento total incluye el raw email original más las partes extraídas. El volumen total es aproximadamente **2× el tamaño del raw email** (se conserva tanto el raw como las partes descompuestas).

**Conversación Corta (1 email almacenado):**

| Componente | P60 | P90 | P95 | P99 |
|---|---|---|---|---|
| Email 1° (raw + partes) | 512 KB | 8 MB | 40 MB | 80 MB |
| **Total en GB** | 0.00050 | 0.00781 | 0.03906 | 0.07813 |

**Conversación Estándar (3 emails almacenados):**

| Componente | P60 | P90 | P95 | P99 |
|---|---|---|---|---|
| Email 1° (raw + partes) | 512 KB | 8 MB | 40 MB | 80 MB |
| Emails 2° y 3° (raw + partes) | 1 MB | 1 MB | 1 MB | 1 MB |
| **Total** | 1.5 MB | 9 MB | 41 MB | 81 MB |
| **Total en GB** | 0.00146 | 0.00879 | 0.04004 | 0.07910 |

**Conversación Larga (5 emails almacenados):**

| Componente | P60 | P90 | P95 | P99 |
|---|---|---|---|---|
| Email 1° (raw + partes) | 512 KB | 8 MB | 40 MB | 80 MB |
| Emails 2°-5° (raw + partes) | 2 MB | 2 MB | 2 MB | 2 MB |
| **Total** | 2.5 MB | 10 MB | 42 MB | 82 MB |
| **Total en GB** | 0.00244 | 0.00977 | 0.04102 | 0.08008 |

#### Costo de almacenamiento (primer mes - Frequent Access $0.023/GB)

| Percentil | Corta (1 email) | Estándar (3 emails) | Larga (5 emails) |
|---|---|---|---|
| P60 | $0.000012 | $0.000034 | $0.000056 |
| P90 | $0.000180 | $0.000202 | $0.000225 |
| P95 | $0.000898 | $0.000921 | $0.000943 |
| P99 | $0.001797 | $0.001819 | $0.001842 |

#### Costo de almacenamiento a largo plazo (99% sin acceso después de 30 días)

| Período | Capa | Precio/GB/mes | Ahorro vs Frequent |
|---|---|---|---|
| 0-30 días | Frequent Access | $0.023 | — |
| 30-90 días | Infrequent Access | $0.0125 | -46% |
| 90+ días | Archive Instant Access | $0.004 | -83% |
| 180+ días | Deep Archive | $0.00099 | -96% |

#### Costo de operaciones S3

| Escenario | PUTs | GETs | DELETEs | **Costo ops** |
|---|---|---|---|---|
| Corta (1 email) | ~5 | ~2 | ~1 | **$0.000031** |
| Estándar (3 emails) | ~15 | ~6 | ~3 | **$0.000092** |
| Larga (5 emails) | ~25 | ~10 | ~5 | **$0.000154** |

> Precios: PUT/COPY/DELETE = $0.005/1K ($0.000005 c/u), GET = $0.0004/1K ($0.0000004 c/u)

#### Monitoring de Intelligent Tiering ($0.0025/1K objetos/mes)

| Escenario | Objetos | Costo monitoring/mes |
|---|---|---|
| Corta | 4 | $0.000010 |
| Estándar | 12 | $0.000030 |
| Larga | 20 | $0.000050 |

#### Total S3 por conversación (primer mes)

| Escenario | Componente | P60 | P90 | P95 | P99 |
|---|---|---|---|---|---|
| **Corta** | Storage + Ops + Monitor | **$0.000053** | **$0.000221** | **$0.000939** | **$0.001838** |
| **Estándar** | Storage + Ops + Monitor | **$0.000156** | **$0.000324** | **$0.001043** | **$0.001941** |
| **Larga** | Storage + Ops + Monitor | **$0.000260** | **$0.000429** | **$0.001147** | **$0.002046** |

### 2.4 Amazon SNS - Notificaciones

| Concepto | Costo |
|---|---|
| Notificaciones SNS | **$0.00** (free tier: primer 1M requests/mes) |

### 2.5 LLM - Procesamiento de Lenguaje Natural

| Escenario | Input tokens | Output tokens | Costo input | Costo output | **Total LLM** |
|---|---|---|---|---|---|
| **Corta** (1 ida y vuelta) | 15,000 | ~1,333 | $0.0045 | $0.0033 | **$0.0078** |
| **Estándar** (3 idas y vueltas) | 17,000 | 4,000 | $0.0051 | $0.0100 | **$0.0151** |
| **Larga** (5 idas y vueltas) | 19,000 | ~6,667 | $0.0057 | $0.0167 | **$0.0224** |

---

## 3. Costo Total por Conversación

### 3.1 Conversación Corta (2 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000190 | $0.001540 | $0.007300 | $0.014500 |
| SES Outbound | $0.000130 | $0.000130 | $0.000130 | $0.000130 |
| S3 | $0.000053 | $0.000221 | $0.000939 | $0.001838 |
| LLM | $0.007800 | $0.007800 | $0.007800 | $0.007800 |
| **TOTAL** | **$0.008173** | **$0.009691** | **$0.016169** | **$0.024268** |

### 3.2 Conversación Estándar (6 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000570 | $0.001920 | $0.007680 | $0.014880 |
| SES Outbound | $0.000390 | $0.000390 | $0.000390 | $0.000390 |
| S3 | $0.000156 | $0.000324 | $0.001043 | $0.001941 |
| LLM | $0.015100 | $0.015100 | $0.015100 | $0.015100 |
| **TOTAL** | **$0.016216** | **$0.017734** | **$0.024213** | **$0.032311** |

### 3.3 Conversación Larga (10 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000950 | $0.002300 | $0.008060 | $0.015260 |
| SES Outbound | $0.000646 | $0.000646 | $0.000646 | $0.000646 |
| S3 | $0.000260 | $0.000429 | $0.001147 | $0.002046 |
| LLM | $0.022400 | $0.022400 | $0.022400 | $0.022400 |
| **TOTAL** | **$0.024256** | **$0.025775** | **$0.032253** | **$0.040352** |

### 3.4 Comparativa Cruzada: Duración × Tamaño

| | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| **Corta (2 mails)** | $0.0082 | $0.0097 | $0.0162 | $0.0243 |
| **Estándar (6 mails)** | $0.0162 | $0.0177 | $0.0242 | $0.0323 |
| **Larga (10 mails)** | $0.0243 | $0.0258 | $0.0323 | $0.0404 |

> El incremento de costo entre la conversación corta y la larga en P60 es de **$0.016** (+197%), dominado casi en su totalidad por el LLM. El incremento entre P60 y P99 dentro de una misma duración es de **$0.016** (+197% para corta, +99% para estándar, +66% para larga), dominado por los chunks de SES.

### 3.5 Distribución porcentual del costo (caso más común: Estándar P60)

| Componente | Costo | % del total |
|---|---|---|
| LLM | $0.015100 | 93.1% |
| SES Inbound | $0.000570 | 3.5% |
| SES Outbound | $0.000390 | 2.4% |
| S3 | $0.000156 | 1.0% |
| **Total** | **$0.016216** | **100%** |

### 3.6 Distribución porcentual del costo (caso extremo: Larga P99)

| Componente | Costo | % del total |
|---|---|---|
| LLM | $0.022400 | 55.5% |
| SES Inbound | $0.015260 | 37.8% |
| S3 | $0.002046 | 5.1% |
| SES Outbound | $0.000646 | 1.6% |
| **Total** | **$0.040352** | **100%** |

---

## 4. Estimación Mensual: 1,000 Conversaciones (P60, 6 mails)

### 4.1 Supuestos del escenario mensual

| Parámetro | Valor |
|---|---|
| Conversaciones por mes | 1,000 |
| Tamaño del primer mail | P60 (≤ 256 KB) |
| Duración de conversación | Estándar (6 mails = 3 in + 3 out) |
| Total mails inbound/mes | 3,000 |
| Total mails outbound/mes | 3,000 |
| Total mails/mes | 6,000 |
| Total objetos S3 nuevos/mes | 12,000 (12 por conversación) |
| Storage nuevo/mes | ~1.5 GB (1.5 MB × 1,000 conversaciones) |

### 4.2 Costo mensual desglosado

| Componente | Cálculo | Costo/mes |
|---|---|---|
| **SES Inbound base** | 3,000 emails × $0.0001 | $0.30 |
| **SES Inbound chunks** | 3,000 chunks × $0.00009 | $0.27 |
| **SES Outbound base** | 3,000 emails × $0.0001 | $0.30 |
| **SES Outbound data** | 3,000 × 256KB = 0.73GB × $0.12 | $0.09 |
| **S3 storage (mes corriente)** | 1.5 GB × $0.023 | $0.03 |
| **S3 operaciones** | 1,000 × $0.000092 | $0.09 |
| **S3 monitoring** | 12,000 obj × $0.0000025 | $0.03 |
| **SNS** | Free tier | $0.00 |
| **LLM** | 1,000 × $0.0151 | $15.10 |
| | | |
| **TOTAL MES** | | **$16.21** |

### 4.3 Distribución del costo mensual

| Componente | Costo/mes | % |
|---|---|---|
| **LLM** | $15.10 | 93.2% |
| **SES (inbound + outbound)** | $0.96 | 5.9% |
| **S3 (storage + ops + monitoring)** | $0.15 | 0.9% |
| **Total** | **$16.21** | **100%** |

### 4.4 Evolución del costo S3 acumulado a lo largo del tiempo

A medida que se acumulan conversaciones sin borrar datos, el almacenamiento S3 crece pero el costo unitario baja gracias a Intelligent Tiering. Asumiendo 1,000 conversaciones/mes constantes y que el 99% no se accede pasados los 30 días:

| Mes | Conv. acumuladas | Storage acum. | Capa predominante datos antiguos | Costo S3 storage/mes |
|---|---|---|---|---|
| 1 | 1,000 | 1.5 GB | 100% Frequent | $0.03 |
| 2 | 2,000 | 3.0 GB | Mes anterior → Infrequent | $0.05 |
| 3 | 3,000 | 4.5 GB | 2 meses → Infrequent | $0.07 |
| 6 | 6,000 | 9.0 GB | 3+ meses → Archive Instant | $0.08 |
| 12 | 12,000 | 18.0 GB | 6+ meses → Deep Archive | $0.08 |

> El costo de S3 storage converge a ~$0.08/mes para 1,000 conv/mes gracias a que los datos antiguos migran a Deep Archive ($0.00099/GB). **El storage nunca supera el 0.5% del costo total mensual.**

### 4.5 Costo total mensual proyectado (incluyendo storage acumulado)

| Mes | SES | S3 (todo) | LLM | **Total/mes** |
|---|---|---|---|---|
| 1 | $0.96 | $0.15 | $15.10 | **$16.21** |
| 3 | $0.96 | $0.19 | $15.10 | **$16.25** |
| 6 | $0.96 | $0.20 | $15.10 | **$16.26** |
| 12 | $0.96 | $0.20 | $15.10 | **$16.26** |

> El costo mensual se estabiliza rápidamente en **~$16.26/mes** para 1,000 conversaciones. El crecimiento del storage acumulado es absorbido por las capas baratas de Intelligent Tiering, agregando menos de **$0.05/mes** adicionales.

### 4.6 Costo anual total (12 meses a 1,000 conv/mes)

| Concepto | Costo anual |
|---|---|
| SES (inbound + outbound) | $11.52 |
| S3 (storage + ops + monitoring) | ~$2.25 |
| LLM | $181.20 |
| **Total anual** | **~$194.97** |
| **Promedio mensual** | **~$16.25** |

---

## 5. Conclusiones

### Costo total estimado por conversación según duración y tamaño

| Duración \ Tamaño | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| **Corta (2 mails)** | **$0.0082** | $0.0097 | $0.0162 | $0.0243 |
| **Estándar (6 mails)** | **$0.0162** | $0.0177 | $0.0242 | $0.0323 |
| **Larga (10 mails)** | **$0.0243** | $0.0258 | $0.0323 | $0.0404 |

### Costo mensual para 1,000 conversaciones estándar (P60)

| Métrica | Valor |
|---|---|
| **Costo por conversación** | $0.0162 |
| **Costo mensual total** | **$16.21** |
| **Costo anual total** | **~$195** |
| **Costo por mail individual** | $0.0027 |

### Hallazgos Clave

1. **El LLM es el componente dominante en todos los escenarios**: Representa entre el **55%** (larga P99) y el **93%** (estándar P60) del costo total. Es el único componente que escala significativamente con la duración de la conversación.

2. **El almacenamiento S3 es despreciable**: Incluso acumulando storage a lo largo del año, S3 nunca supera el **1%** del costo total mensual. Intelligent Tiering reduce el costo de datos antiguos hasta en un **96%**.

3. **Agregar idas y vueltas es barato en SES**: El costo incremental de SES por cada ida y vuelta adicional (2 mails ≤ 256KB) es solo **$0.00038**, trivial comparado con el incremento del LLM (~**$0.0073** por ida y vuelta).

4. **El tamaño del primer mail solo importa en el top 5%**: Entre P60 y P90 la diferencia es solo **+9%** ($0.0015). Recién a partir de P95 el costo de chunks SES se vuelve significativo (**+49%** vs P60).

5. **A 1,000 conversaciones/mes el costo es extremadamente bajo**: **$16.21/mes** (~$195/año) para atender 1,000 consultas por email con IA, incluyendo almacenamiento persistente de todos los mails recibidos.

6. **El costo por conversación es 100-1000x menor que un agente humano**: A ~$0.016 por conversación estándar, el costo operativo de resolver una consulta por email con IA equivale a menos de 1 segundo de trabajo de un agente humano.

---

## Fuentes de Precios

- [Amazon SES Pricing](https://aws.amazon.com/ses/pricing/) — Inbound: $0.10/1K emails + $0.09/1K chunks; Outbound: $0.10/1K emails + $0.12/GB data
- [Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/) — Intelligent Tiering: $0.023/GB (Frequent), $0.0125/GB (Infrequent), $0.004/GB (Archive Instant); PUT: $0.005/1K; GET: $0.0004/1K
- [S3 Intelligent Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) — Monitoring: $0.0025/1K objetos/mes; sin cargos de retrieval ni transición
- [CloudFix - S3 Intelligent Tiering Guide](https://cloudfix.com/blog/aws-s3-intelligent-tiering/) — Referencia de precios por capa
