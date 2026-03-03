# Análisis de Costo por Conversación - Estrategia Amazon SES

> **Fecha**: 2026-03-03
>
> **Objetivo**: Estimar el costo total de cada conversación de email utilizando Amazon SES como estrategia de recepción/envío, S3 Standard para almacenamiento, y un LLM para interpretación y respuesta automática. Se analizan tres escenarios de duración de conversación, cuatro rangos de tamaño del primer mail, y dos modelos de LLM.

---

## Índice

1. [Resumen Ejecutivo](#1-resumen-ejecutivo)
2. [Supuestos y Parámetros](#2-supuestos-y-parámetros)
3. [Detalle: Amazon SES](#3-detalle-amazon-ses)
4. [Detalle: Amazon S3](#4-detalle-amazon-s3)
5. [Detalle: LLM](#5-detalle-llm)
6. [Detalle: MongoDB Atlas](#6-detalle-mongodb-atlas)
7. [Estimación Mensual y Anual (4,000 conv/mes)](#7-estimación-mensual-y-anual-4000-convmes)
8. [Nota: S3 Intelligent Tiering vs Standard](#8-nota-s3-intelligent-tiering-vs-standard)
9. [Fuentes de Precios](#9-fuentes-de-precios)

---

## 1. Resumen Ejecutivo

### Costo por conversación según duración y tamaño

**Con Gemini** ($0.30/1M input, $2.50/1M output):

| Duración \ Tamaño | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| **Corta (2 mails)** | **$0.0082** | $0.0097 | $0.0162 | $0.0243 |
| **Estándar (6 mails)** | **$0.0162** | $0.0177 | $0.0242 | $0.0323 |
| **Larga (10 mails)** | **$0.0242** | $0.0257 | $0.0322 | $0.0403 |

**Con GPT-OSS** ($0.15/1M input, $0.60/1M output):

| Duración \ Tamaño | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| **Corta (2 mails)** | **$0.0034** | $0.0049 | $0.0114 | $0.0195 |
| **Estándar (6 mails)** | **$0.0060** | $0.0076 | $0.0140 | $0.0221 |
| **Larga (10 mails)** | **$0.0087** | $0.0102 | $0.0167 | $0.0248 |

### Proyección mensual y anual (4,000 conversaciones estándar P60)

| Métrica | Gemini | GPT-OSS |
|---|---|---|
| **Costo por conversación** | $0.0162 | $0.0060 |
| **Costo mensual (mes 1)** | **$64.82** | **$24.22** |
| **Costo mensual (mes 12, con storage acumulado)** | **$67.19** | **$26.59** |
| **Costo anual total** | **~$792** | **~$305** |
| **Costo por mail individual** | $0.0027 | $0.0010 |

### Distribución del costo (caso más común: Estándar P60)

| Componente | Gemini | % | GPT-OSS | % |
|---|---|---|---|---|
| LLM | $0.015100 | 93.3% | $0.004950 | 82.0% |
| SES Inbound | $0.000570 | 3.5% | $0.000570 | 9.4% |
| SES Outbound | $0.000390 | 2.4% | $0.000390 | 6.5% |
| S3 | $0.000126 | 0.8% | $0.000126 | 2.1% |
| **Total** | **$0.016186** | **100%** | **$0.006036** | **100%** |

### Costo anual desglosado (4,000 conv/mes)

| Concepto | Gemini | GPT-OSS |
|---|---|---|
| SES (inbound + outbound) | $45.96 | $45.96 |
| S3 (storage + ops) | ~$15.20 | ~$15.20 |
| MongoDB Atlas (storage) | ~$6.10 | ~$6.10 |
| LLM | $724.80 | $237.60 |
| **Total anual** | **~$792.06** | **~$304.86** |
| **Promedio mensual** | **~$66.01** | **~$25.41** |

---

## 2. Supuestos y Parámetros

### 2.1 Escenarios de Duración de Conversación

| Escenario | Mails totales | Inbound | Outbound | Descripción |
|---|---|---|---|---|
| **Corta** | 2 | 1 | 1 | Pregunta inicial + respuesta definitiva |
| **Estándar** | 6 | 3 | 3 | Tres idas y vueltas (caso promedio) |
| **Larga** | 10 | 5 | 5 | Discusión extendida, cinco idas y vueltas |

**Reglas comunes a todos los escenarios:**
- Solo el **primer mail entrante** tiene tamaño variable (ver distribución)
- Todos los demás mails (entrantes posteriores y salientes) pesan **≤ 256 KB**
- Los mails **salientes no se almacenan** en S3 (solo pagan fee SES outbound)

### 2.2 Distribución de Tamaño del Primer Mail Entrante

| Percentil | Tamaño máximo | Chunks SES (256 KB c/u) |
|---|---|---|
| **P60** (60% de mails) | ≤ 256 KB | 1 |
| **P90** (61%-90%) | ≤ 4 MB | 16 |
| **P95** (91%-95%) | ≤ 20 MB | 80 |
| **P99** (96%-99%) | ≤ 40 MB | 160 |

### 2.3 Estrategia de Almacenamiento S3

- Cada mail inbound se almacena como **raw email** (depositado por SES)
- El raw email se **mueve a otro prefix** (COPY + DELETE)
- Cada **part del mail** se almacena como objeto independiente
- Partes por email: **2 a 5** (promedio estimado: **3 partes**)
- Total de objetos por email inbound: **1 (raw) + 3 (partes) = 4 objetos**

| Escenario | Emails almacenados | Objetos S3 |
|---|---|---|
| Corta (2 mails) | 1 inbound | 4 |
| Estándar (6 mails) | 3 inbound | 12 |
| Larga (10 mails) | 5 inbound | 20 |

### 2.4 Modelos LLM Evaluados

| Modelo | Input /1M tokens | Output /1M tokens |
|---|---|---|
| **Gemini** | $0.30 | $2.50 |
| **OpenAI gpt-oss-safeguard-120b** | $0.15 | $0.60 |

---

## 3. Detalle: Amazon SES

### 3.1 Recepción (Inbound)

**Tarifa base**: $0.10 / 1,000 emails = **$0.0001 por email**

**Tarifa por chunks**: $0.09 / 1,000 chunks = **$0.00009 por chunk** (cada 256 KB)

#### Conversación Corta (1 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (1 email) | $0.000100 | $0.000100 | $0.000100 | $0.000100 |
| Chunks email 1° | 1 | 16 | 80 | 160 |
| Costo chunks | $0.000090 | $0.001440 | $0.007200 | $0.014400 |
| **Total SES inbound** | **$0.000190** | **$0.001540** | **$0.007300** | **$0.014500** |

#### Conversación Estándar (3 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (3 emails) | $0.000300 | $0.000300 | $0.000300 | $0.000300 |
| Chunks (email 1° + 2 × 1) | 3 | 18 | 82 | 162 |
| Costo chunks | $0.000270 | $0.001620 | $0.007380 | $0.014580 |
| **Total SES inbound** | **$0.000570** | **$0.001920** | **$0.007680** | **$0.014880** |

#### Conversación Larga (5 inbound)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| Tarifa base (5 emails) | $0.000500 | $0.000500 | $0.000500 | $0.000500 |
| Chunks (email 1° + 4 × 1) | 5 | 20 | 84 | 164 |
| Costo chunks | $0.000450 | $0.001800 | $0.007560 | $0.014760 |
| **Total SES inbound** | **$0.000950** | **$0.002300** | **$0.008060** | **$0.015260** |

### 3.2 Envío (Outbound)

**Tarifa base**: $0.10 / 1,000 emails = **$0.0001 por email**

**Tarifa data**: $0.12 / GB

| Escenario | Emails | Tarifa base | Data (256KB c/u) | **Total outbound** |
|---|---|---|---|---|
| Corta (2 mails) | 1 | $0.000100 | $0.000030 | **$0.000130** |
| Estándar (6 mails) | 3 | $0.000300 | $0.000090 | **$0.000390** |
| Larga (10 mails) | 5 | $0.000500 | $0.000146 | **$0.000646** |

### 3.3 SNS - Notificaciones

| Concepto | Costo |
|---|---|
| Notificaciones SNS | **$0.00** (free tier: primer 1M requests/mes) |

---

## 4. Detalle: Amazon S3

**Storage class**: S3 Standard — **$0.023/GB/mes** (precio fijo, sin reducción por antigüedad)

> Se utiliza S3 Standard como base de cálculo para establecer el **techo máximo de costo**. No tiene cargos de monitoring ni tarifas adicionales por objeto. Ver [sección 8](#8-nota-s3-intelligent-tiering-vs-standard) para comparativa con Intelligent Tiering.

### 4.1 Volumen almacenado por conversación

El almacenamiento total es aproximadamente **2× el tamaño del raw email** (se conserva tanto el raw como las partes descompuestas).

| Escenario | Componente | P60 | P90 | P95 | P99 |
|---|---|---|---|---|---|
| **Corta** | Email 1° (raw + partes) | 512 KB | 8 MB | 40 MB | 80 MB |
| **Estándar** | Email 1° + emails 2°-3° | 1.5 MB | 9 MB | 41 MB | 81 MB |
| **Larga** | Email 1° + emails 2°-5° | 2.5 MB | 10 MB | 42 MB | 82 MB |

### 4.2 Costo de almacenamiento por conversación

| Percentil | Corta (1 email) | Estándar (3 emails) | Larga (5 emails) |
|---|---|---|---|
| P60 | $0.000012 | $0.000034 | $0.000056 |
| P90 | $0.000180 | $0.000202 | $0.000225 |
| P95 | $0.000898 | $0.000921 | $0.000943 |
| P99 | $0.001797 | $0.001819 | $0.001842 |

### 4.3 Costo de operaciones S3

| Escenario | PUTs | GETs | DELETEs | **Costo ops** |
|---|---|---|---|---|
| Corta (1 email) | ~5 | ~2 | ~1 | **$0.000031** |
| Estándar (3 emails) | ~15 | ~6 | ~3 | **$0.000092** |
| Larga (5 emails) | ~25 | ~10 | ~5 | **$0.000154** |

> Precios: PUT/COPY/DELETE = $0.005/1K ($0.000005 c/u), GET = $0.0004/1K ($0.0000004 c/u)

### 4.4 Total S3 por conversación

| Escenario | P60 | P90 | P95 | P99 |
|---|---|---|---|---|
| **Corta** | **$0.000043** | **$0.000211** | **$0.000929** | **$0.001828** |
| **Estándar** | **$0.000126** | **$0.000294** | **$0.001013** | **$0.001911** |
| **Larga** | **$0.000210** | **$0.000379** | **$0.001097** | **$0.001996** |

---

## 5. Detalle: LLM

### 5.1 Tokens por escenario

| Concepto | Corta (2 mails) | Estándar (6 mails) | Larga (10 mails) |
|---|---|---|---|
| **Input**: prompt inicial | 14,000 | 14,000 | 14,000 |
| **Input**: consultas entrantes | 1 × 1,000 | 3 × 1,000 | 5 × 1,000 |
| **Total input tokens** | 15,000 | 17,000 | 19,000 |
| **Output tokens** | ~1,333 | 4,000 | ~6,667 |

### 5.2 Costo por modelo y escenario

| Modelo | Concepto | Corta | Estándar | Larga |
|---|---|---|---|---|
| **Gemini** | Input | $0.0045 | $0.0051 | $0.0057 |
| | Output | $0.0033 | $0.0100 | $0.0167 |
| | **Total** | **$0.0078** | **$0.0151** | **$0.0224** |
| **GPT-OSS** | Input | $0.00225 | $0.00255 | $0.00285 |
| | Output | $0.0008 | $0.0024 | $0.004 |
| | **Total** | **$0.00305** | **$0.00495** | **$0.00685** |

### 5.3 Costo total por conversación (todos los componentes)

#### Conversación Corta (2 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000190 | $0.001540 | $0.007300 | $0.014500 |
| SES Outbound | $0.000130 | $0.000130 | $0.000130 | $0.000130 |
| S3 | $0.000043 | $0.000211 | $0.000929 | $0.001828 |
| LLM Gemini | $0.007800 | $0.007800 | $0.007800 | $0.007800 |
| LLM GPT-OSS | $0.003050 | $0.003050 | $0.003050 | $0.003050 |
| **TOTAL (Gemini)** | **$0.008163** | **$0.009681** | **$0.016159** | **$0.024258** |
| **TOTAL (GPT-OSS)** | **$0.003413** | **$0.004931** | **$0.011409** | **$0.019508** |

#### Conversación Estándar (6 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000570 | $0.001920 | $0.007680 | $0.014880 |
| SES Outbound | $0.000390 | $0.000390 | $0.000390 | $0.000390 |
| S3 | $0.000126 | $0.000294 | $0.001013 | $0.001911 |
| LLM Gemini | $0.015100 | $0.015100 | $0.015100 | $0.015100 |
| LLM GPT-OSS | $0.004950 | $0.004950 | $0.004950 | $0.004950 |
| **TOTAL (Gemini)** | **$0.016186** | **$0.017704** | **$0.024183** | **$0.032281** |
| **TOTAL (GPT-OSS)** | **$0.006036** | **$0.007554** | **$0.014033** | **$0.022131** |

#### Conversación Larga (10 mails)

| Componente | P60 (≤256KB) | P90 (≤4MB) | P95 (≤20MB) | P99 (≤40MB) |
|---|---|---|---|---|
| SES Inbound | $0.000950 | $0.002300 | $0.008060 | $0.015260 |
| SES Outbound | $0.000646 | $0.000646 | $0.000646 | $0.000646 |
| S3 | $0.000210 | $0.000379 | $0.001097 | $0.001996 |
| LLM Gemini | $0.022400 | $0.022400 | $0.022400 | $0.022400 |
| LLM GPT-OSS | $0.006850 | $0.006850 | $0.006850 | $0.006850 |
| **TOTAL (Gemini)** | **$0.024206** | **$0.025725** | **$0.032203** | **$0.040302** |
| **TOTAL (GPT-OSS)** | **$0.008656** | **$0.010175** | **$0.016653** | **$0.024752** |

---

## 6. Detalle: MongoDB Atlas

Además del almacenamiento en S3 (raw emails y partes), se almacena en MongoDB Atlas el contenido textual de los mails para consulta y gestión.

**Costo de storage adicional en MongoDB Atlas**: **~$0.17/GB/mes**

- **Mails salientes**: HTML multipart (tags de estructura, estilos inline, texto; imágenes alojadas en la web) → **~20 KB por mail**
- **Mails entrantes**: HTML multipart del email (sin adjuntos; imágenes referenciadas externamente) → **~20 KB por mail**

### 6.1 Volumen por conversación

| Escenario | Salientes (HTML) | Entrantes (HTML) | **Total MongoDB** |
|---|---|---|---|
| Corta (2 mails) | 1 × 20 KB = 20 KB | 1 × 20 KB = 20 KB | **40 KB** |
| Estándar (6 mails) | 3 × 20 KB = 60 KB | 3 × 20 KB = 60 KB | **120 KB** |
| Larga (10 mails) | 5 × 20 KB = 100 KB | 5 × 20 KB = 100 KB | **200 KB** |

### 6.2 Costo mensual y acumulado (4,000 conversaciones estándar/mes)

| Mes | Conv. acumuladas | Storage acum. | Costo MongoDB/mes |
|---|---|---|---|
| 1 | 4,000 | ~0.46 GB | **$0.08** |
| 3 | 12,000 | ~1.37 GB | **$0.23** |
| 6 | 24,000 | ~2.75 GB | **$0.47** |
| 12 | 48,000 | ~5.49 GB | **$0.93** |

> El costo anual total de MongoDB para almacenar el contenido HTML de todos los mails es de **~$6.10**.

---

## 7. Estimación Mensual y Anual (4,000 conv/mes)

Escenario: **4,000 conversaciones estándar (6 mails) con primer mail P60 (≤ 256 KB)**.

### 7.1 Parámetros

| Parámetro | Valor |
|---|---|
| Conversaciones por mes | 4,000 |
| Total mails/mes | 24,000 (12K in + 12K out) |
| Objetos S3 nuevos/mes | 48,000 |
| Storage S3 nuevo/mes | ~6 GB |
| Storage MongoDB nuevo/mes | ~0.46 GB |

### 7.2 Costo mensual desglosado

| Componente | Cálculo | Gemini | GPT-OSS |
|---|---|---|---|
| **SES Inbound base** | 12,000 emails × $0.0001 | $1.20 | $1.20 |
| **SES Inbound chunks** | 12,000 chunks × $0.00009 | $1.08 | $1.08 |
| **SES Outbound base** | 12,000 emails × $0.0001 | $1.20 | $1.20 |
| **SES Outbound data** | 2.93 GB × $0.12 | $0.35 | $0.35 |
| **S3 storage** | 6 GB × $0.023 | $0.14 | $0.14 |
| **S3 operaciones** | 4,000 × $0.000092 | $0.37 | $0.37 |
| **MongoDB storage** | 0.46 GB × $0.17 | $0.08 | $0.08 |
| **SNS** | Free tier | $0.00 | $0.00 |
| **LLM** | 4,000 × costo/conv | $60.40 | $19.80 |
| | | | |
| **TOTAL MES** | | **$64.82** | **$24.22** |

### 7.3 Evolución mensual con storage acumulado

| Mes | SES | S3 | MongoDB | LLM Gemini | **Total Gemini** | LLM GPT-OSS | **Total GPT-OSS** |
|---|---|---|---|---|---|---|---|
| 1 | $3.83 | $0.51 | $0.08 | $60.40 | **$64.82** | $19.80 | **$24.22** |
| 3 | $3.83 | $0.78 | $0.23 | $60.40 | **$65.24** | $19.80 | **$24.64** |
| 6 | $3.83 | $1.20 | $0.47 | $60.40 | **$65.90** | $19.80 | **$25.30** |
| 12 | $3.83 | $2.03 | $0.93 | $60.40 | **$67.19** | $19.80 | **$26.59** |

### 7.4 Evolución del storage S3 acumulado

| Mes | Conv. acumuladas | Storage acum. | Costo S3 storage/mes | Costo S3 ops/mes | **Total S3/mes** |
|---|---|---|---|---|---|
| 1 | 4,000 | 6 GB | $0.14 | $0.37 | **$0.51** |
| 2 | 8,000 | 12 GB | $0.28 | $0.37 | **$0.65** |
| 3 | 12,000 | 18 GB | $0.41 | $0.37 | **$0.78** |
| 6 | 24,000 | 36 GB | $0.83 | $0.37 | **$1.20** |
| 12 | 48,000 | 72 GB | $1.66 | $0.37 | **$2.03** |

---

## 8. Nota: S3 Intelligent Tiering vs Standard

Si se utiliza **S3 Intelligent Tiering** en lugar de S3 Standard, los objetos que no se acceden migran automáticamente a capas más baratas:

| Capa | Condición | Precio/GB/mes | Ahorro vs Standard |
|---|---|---|---|
| Frequent Access | Acceso reciente (< 30 días) | $0.023 | — |
| Infrequent Access | Sin acceso por 30+ días | $0.0125 | -46% |
| Archive Instant Access | Sin acceso por 90+ días | $0.004 | -83% |
| Deep Archive | Sin acceso por 180+ días | $0.00099 | -96% |

Sin embargo, Intelligent Tiering cobra un **fee de monitoring de $0.0025/1,000 objetos/mes** por cada objeto almacenado. Con 12 objetos por conversación y acumulación mensual, este cargo crece proporcionalmente:

| Mes | Objetos acumulados | Monitoring/mes | Storage IT/mes | **Total IT/mes** | **Total Standard/mes** |
|---|---|---|---|---|---|
| 1 | 48,000 | $0.12 | $0.14 | **$0.63** | **$0.51** |
| 6 | 288,000 | $0.72 | $0.36 | **$1.45** | **$1.20** |
| 12 | 576,000 | $1.44 | $0.40 | **$2.21** | **$2.03** |

> Para este patrón de uso (muchos objetos pequeños, ~125 KB promedio por objeto), **Intelligent Tiering resulta más caro que S3 Standard** debido a que el fee de monitoring acumulado supera el ahorro en storage. **S3 Standard es la opción recomendada**.

---

## 9. Fuentes de Precios

- [Amazon SES Pricing](https://aws.amazon.com/ses/pricing/) — Inbound: $0.10/1K emails + $0.09/1K chunks; Outbound: $0.10/1K emails + $0.12/GB data
- [Amazon S3 Pricing](https://aws.amazon.com/s3/pricing/) — Standard: $0.023/GB; PUT: $0.005/1K; GET: $0.0004/1K
- [S3 Intelligent Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) — Monitoring: $0.0025/1K objetos/mes, sin cargos de retrieval
- [OpenAI Pricing](https://openai.com/api/pricing/) — gpt-oss-safeguard-120b: $0.15/1M input, $0.60/1M output
- [MongoDB Atlas Pricing](https://www.mongodb.com/pricing) — Storage adicional: ~$0.17/GB/mes
