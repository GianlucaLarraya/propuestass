# Feature: Análisis Actualizable con Nuevas Reviews

### ¿Qué queremos lograr?

Permitir que el análisis de IA se **actualice** cuando lleguen nuevas reviews al restaurante, para que los insights siempre reflejen la opinión más reciente de los clientes.

### ¿Por qué es importante?

Actualmente el análisis se genera una sola vez (durante el onboarding) y nunca se actualiza. Si un restaurante recibe 50 reviews nuevas, el dueño no ve esos insights reflejados en su dashboard.

### ¿Cómo funciona para el usuario?

**Header actual:**
```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   Sottovoce              Rating    Reseñas            [⊕] [...] [🔄 Actualizar   │
│                           4.4      4,371                          análisis]      │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Header con la nueva feature:**
```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│   Sottovoce              Rating    Reseñas            [⊕] [...] [🔄 Actualizar   │
│                           4.4      4,371                          análisis]      │
│                                                                                  │
│                                    ┌────────────────────────────┐                │
│                                    │ 15 reviews nuevas sin ana..│                │
│                                    │ Último análisis: hace 5d   │                │
│                                    └────────────────────────────┘                │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```


El usuario ve claramente:
1. **Badge visible** con la cantidad de reviews sin analizar
2. **Botón existente** "Actualizar análisis" (ahora funcional)
3. **Contexto temporal** de cuándo fue el último análisis

---

## Decisiones de Diseño

### 1. Fuentes de Reviews

Solo consideramos dos fuentes para la actualización del análisis:

| Fuente | Descripción | Frecuencia |
|--------|-------------|------------|
| **Google Business Profile (GBP)** | Reviews sincronizadas desde la cuenta de Google del usuario | Automático cada 6h o manual |
| **Dishboard QR** | Feedback directo de clientes via código QR | Tiempo real |

> ⚠️ El scraping inicial (Apify) NO se usa para actualizaciones, solo para el onboarding.

### 2. Tipo de Actualización

**Full Regeneration** (regeneración completa)

- Se regenera TODO el análisis desde cero
- Incluye: insights, staff leaderboard, highlights
- Es la opción más simple y consistente

> Descartamos el análisis incremental por su complejidad y posible pérdida de coherencia.

### 3. Activación

**Bajo demanda** (manual)

- El usuario decide cuándo actualizar
- Ya existe un botón "Actualizar análisis" (actualmente sin funcionalidad)
- Cada actualización consume **1 crédito**

### 4. Límite de Reviews para el Análisis

**Híbrido: últimas 200 reviews O último año**

- Se analizan las **últimas 200 reviews** ordenadas por fecha
- O todas las reviews del **último año** si son menos de 200
- Esto controla el costo de tokens de IA y mantiene relevancia temporal

> Importante: La base de datos guarda TODAS las reviews (sin límite), pero el análisis solo usa las más recientes/relevantes.

**Ejemplo: ¿Qué pasa cuando llegan nuevas reviews?**

```
ESTADO INICIAL (Análisis #1)
────────────────────────────────────────────────────────────────
Base de datos: 200 reviews totales
Analizadas:    reviews 1-200 (todas)

    [Review 1] [Review 2] ... [Review 199] [Review 200]
    ◄──────────────── ANALIZADAS ─────────────────────►


LLEGAN 30 REVIEWS NUEVAS
────────────────────────────────────────────────────────────────
Base de datos: 230 reviews totales
Sin analizar:  30 reviews nuevas

    [1] [2] ... [199] [200] [201] [202] ... [229] [230]
    ◄─── ANALIZADAS ────►   ◄──── NUEVAS (sin analizar) ────►


USUARIO ACTUALIZA ANÁLISIS (Análisis #2)
────────────────────────────────────────────────────────────────
El sistema toma las últimas 200 (ordenadas por fecha):

    [1]...[30]  [31] [32] ... [229] [230]
    ◄─ SALEN ─►  ◄───── ANALIZADAS (200 más recientes) ─────►

    Reviews 1-30:    YA NO están en el análisis (muy antiguas)
    Reviews 31-200:  Siguen en el análisis
    Reviews 201-230: NUEVAS, ahora incluidas en el análisis
```

**¿Por qué funciona así?**

- El análisis siempre refleja las **200 opiniones más recientes**
- Reviews muy antiguas pierden relevancia con el tiempo
- El dueño ve insights basados en lo que los clientes dicen **ahora**

### 5. Tracking de Reviews Analizadas

Para saber qué reviews son "nuevas", agregamos un campo a la tabla de reviews:

```
reviews.last_analyzed_at = fecha cuando fue incluida en un análisis
```

- `NULL` → review nunca analizada
- Con fecha → ya fue incluida en algún análisis

---

## Flujo del Usuario

```
                    ┌─────────────────────┐
                    │  Usuario abre panel │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Sistema muestra:   │
                    │  "15 sin analizar"  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Usuario clickea    │
                    │  "Actualizar"       │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Confirmación:      │
                    │  "Costará 1 crédito"│
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Sistema:           │
                    │  1. Verifica crédito│
                    │  2. Trae reviews    │
                    │  3. Llama a Gemini  │
                    │  4. Guarda análisis │
                    │  5. Marca reviews   │
                    │  6. Consume crédito │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  UI actualizada:    │
                    │  "0 sin analizar"   │
                    │  Nuevos insights    │
                    └─────────────────────┘
```

---

## Especificación Técnica

### Cambios en Base de Datos

#### 1. Nueva columna en `reviews`

```sql
ALTER TABLE reviews
ADD COLUMN last_analyzed_at TIMESTAMPTZ NULL;

COMMENT ON COLUMN reviews.last_analyzed_at IS
  'Timestamp de cuando esta review fue incluida en un análisis. NULL si nunca fue analizada.';
```

#### 2. Modificar RPC `get_user_owned_restaurants_with_analysis`

Agregar al retorno:

```sql
-- Agregar al SELECT del RPC
(
  SELECT COUNT(*)
  FROM reviews
  WHERE restaurant_id = r.id
    AND last_analyzed_at IS NULL
) as unanalyzed_reviews_count
```

### Cambios en API

#### Endpoint: `POST /api/analysis/reviews`

Modificaciones necesarias:

1. **Agregar consumo de créditos** (actualmente no consume)
2. **Agregar límite de reviews** (últimas 200 o 1 año)
3. **Marcar reviews como analizadas** después de generar el análisis

```typescript
// Pseudo-código del flujo
async function POST(request) {
  // 1. Verificar autenticación
  const user = await requireAuth(request)

  // 2. Verificar y consumir crédito
  const creditResult = await consumeCredits(user.id, 1, 'analysis_update')
  if (!creditResult.success) return error(402, 'Sin créditos')

  // 3. Obtener reviews (con límite)
  const reviews = await getReviewsForAnalysis(restaurantId, {
    limit: 200,
    maxAge: '1 year'
  })

  // 4. Generar análisis con Gemini
  const analysis = await generateAnalysis(reviews)

  // 5. Guardar análisis
  await saveAIAnalysis(userId, restaurantId, analysis)

  // 6. Marcar reviews como analizadas
  await markReviewsAsAnalyzed(reviews.map(r => r.id))

  return { success: true, analysis }
}
```

### Cambios en Frontend

#### Componente: `PanelHeader`

1. Recibir `unanalyzedReviewsCount` como prop
2. Mostrar badge con cantidad
3. Conectar botón a función de actualización
4. Mostrar estado de carga durante análisis

```tsx
// Pseudo-código
<PanelHeader
  unanalyzedCount={15}
  onUpdateAnalysis={handleUpdateAnalysis}
  isUpdating={isLoading}
  lastAnalysisDate={analysis.created_at}
/>
```

---

## Resumen de Tareas

### Backend

| Tarea | Complejidad | Descripción |
|-------|-------------|-------------|
| Migración DB | Baja | Agregar columna `last_analyzed_at` a `reviews` |
| Modificar RPC | Baja | Agregar `unanalyzed_reviews_count` al query |
| Límite de reviews | Media | Implementar filtro en `getRestaurantReviews` |
| Consumo de créditos | Baja | Agregar lógica en `/api/analysis/reviews` |
| Marcar analizadas | Baja | UPDATE después de análisis exitoso |

### Frontend

| Tarea | Complejidad | Descripción |
|-------|-------------|-------------|
| Badge de reviews | Baja | Mostrar contador en `PanelHeader` |
| Conectar botón | Baja | onClick → llamar API con loading state |
| Confirmación | Baja | Modal/dialog antes de consumir crédito |
| Actualizar UI | Baja | Refrescar datos después de análisis |

