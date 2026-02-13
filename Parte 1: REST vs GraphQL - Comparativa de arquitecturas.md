# REST vs GraphQL: ¿Cuál es más rápido? Comparativa de arquitecturas

**Serie: Optimización de APIs (Parte 1 de 3)**

Las APIs funcionan, responden y cumplen su cometido... hasta que el tráfico crece y lo que antes respondía en 100ms ahora tarda segundos. La elección entre REST y GraphQL no es solo una cuestión de preferencia sintáctica: cada arquitectura tiene características fundamentales que determinan cómo se comporta bajo carga.

En esta serie de tres artículos, compararemos REST y GraphQL desde la perspectiva del rendimiento real. En este primer artículo, analizamos las filosofías de cada arquitectura y sus implicaciones directas en el rendimiento.

## Dos filosofías, dos problemas

La diferencia fundamental entre REST y GraphQL no está en la tecnología, sino en **quién controla el acceso a los datos**.

### REST: El servidor manda

En REST, el servidor define completamente la estructura de las respuestas. Un endpoint `/api/accounts/123` siempre devuelve el mismo conjunto de campos, sin importar qué necesite el cliente.

**Ventajas para el rendimiento:**

✅ **Caché HTTP nativo**: Navegadores, proxies y CDNs pueden cachear respuestas automáticamente usando headers estándar como `Cache-Control` y `ETag`.

✅ **Predecibilidad**: Sabes exactamente cuántos recursos consume cada endpoint. Fácil de medir y optimizar.

✅ **Simplicidad operativa**: Menor overhead de servidor. No necesitas parsear queries complejas ni validar esquemas dinámicos.

✅ **Infraestructura madura**: CloudFlare, Varnish, Nginx pueden cachear y proteger tu API sin configuración especial.

**Desventajas para el rendimiento:**

❌ **Over-fetching**: El endpoint devuelve 20 campos cuando el cliente solo necesita 2. Desperdicio de bandwidth.

❌ **Under-fetching**: Necesitas hacer 3 requests HTTP para obtener datos relacionados. Más latencia.

❌ **Proliferación de endpoints**: Terminas creando endpoints especializados (`/accounts/summary`, `/accounts/with-transactions`) que duplican lógica.

**Ejemplo real - Over/Under-fetching:**

```javascript
// Cliente necesita: nombre de usuario y títulos de sus 5 posts recientes

// REST requiere 2 requests:
GET /api/users/123
// Respuesta: { id, name, email, bio, avatar, created_at, ... } 
// Devuelve 20 campos, solo usas 1

GET /api/users/123/posts?limit=5
// Respuesta: [{ id, title, content, author, tags, ... }, ...]
// Devuelve posts completos, solo usas el título

// Total transferido: ~50KB
// Latencia: 2 × 50ms = 100ms
```

### GraphQL: El cliente decide

En GraphQL, el cliente especifica exactamente qué campos necesita en cada query. Un solo endpoint `/graphql` maneja todas las peticiones.

**Ventajas para el rendimiento:**

✅ **Eliminación de over-fetching**: Solo se transfieren los datos necesarios. Ahorro significativo de bandwidth.

✅ **Una sola request para vistas complejas**: Reduces latencia al eliminar múltiples roundtrips HTTP.

✅ **Evolución sin versioning**: Añade campos nuevos sin romper clientes existentes. Menos endpoints duplicados.

✅ **Introspección**: El cliente puede descubrir qué datos están disponibles sin documentación externa.

**Desventajas para el rendimiento:**

❌ **Complejidad impredecible**: Un cliente puede hacer una query que trae 10 niveles de relaciones anidadas. Imposible predecir el costo sin análisis.

❌ **Caché HTTP limitado**: Todas las queries van al mismo endpoint `/graphql`. Los CDNs tradicionales no pueden cachear efectivamente.

❌ **Mayor overhead de servidor**: Parsing de queries, validación de schemas, resolución dinámica de campos consume más CPU.

❌ **Potencial de abuso**: Sin límites apropiados, queries complejas pueden tumbar el servidor.

**Ejemplo real - Eficiencia de transferencia:**

```graphql
# GraphQL obtiene exactamente lo necesario en 1 request:
query {
  user(id: "123") {
    name
    posts(first: 5) {
      title
    }
  }
}

# Respuesta:
{
  "data": {
    "user": {
      "name": "Alice",
      "posts": [
        { "title": "Post 1" },
        { "title": "Post 2" },
        ...
      ]
    }
  }
}

# Total transferido: ~2KB
# Latencia: 1 × 50ms = 50ms
```

## Comparativa directa: Mismo caso de uso

Veamos cómo cada arquitectura maneja el mismo requisito: un dashboard bancario que muestra información de cuenta, últimas transacciones y resumen mensual.

### Solución REST

```javascript
// Cliente necesita hacer 3 requests

// Request 1: Información básica
GET /api/accounts/123
// Respuesta: { id, name, balance, account_number, type, status, ... }

// Request 2: Transacciones
GET /api/accounts/123/transactions?limit=20
// Respuesta: [{ id, amount, description, date, category, ... }, ...]

// Request 3: Resumen
GET /api/accounts/123/summary
// Respuesta: { income: 5000, expenses: 3000, month: "January" }

// Métricas:
// - 3 requests HTTP
// - Latencia total: ~150ms (3 × 50ms)
// - Datos transferidos: ~25KB (incluye campos no usados)
```

### Solución GraphQL

```graphql
# Cliente hace 1 request con query específica
query GetDashboard($accountId: ID!) {
  account(id: $accountId) {
    name
    balance
    accountNumber
    transactions(limit: 20) {
      id
      amount
      description
      date
    }
    summary {
      income
      expenses
    }
  }
}

# Métricas:
# - 1 request HTTP
# - Latencia total: ~50ms
# - Datos transferidos: ~8KB (solo campos solicitados)
```

### Análisis de métricas

| Métrica | REST | GraphQL | Ganador |
|---------|------|---------|---------|
| **Requests HTTP** | 3 | 1 | GraphQL |
| **Latencia de red** | ~150ms | ~50ms | GraphQL |
| **Datos transferidos** | ~25KB | ~8KB | GraphQL |
| **Complejidad cliente** | Baja | Media | REST |
| **Caché HTTP** | ✅ Sí | ❌ Limitado | REST |
| **Predecibilidad** | ✅ Alta | ⚠️ Variable | REST |

**Conclusión del caso:** GraphQL gana en eficiencia de red, REST gana en simplicidad y cacheo.

## ¿Cuándo usar cada arquitectura?

No hay una respuesta universal. La elección depende de tus necesidades específicas.

### Usa REST cuando:

**1. Tu API es pública y necesitas máximo cacheo**
- APIs de productos (e-commerce)
- APIs de contenido (blogs, noticias)
- Webhooks y notificaciones

**2. Los casos de uso son limitados y conocidos**
- CRUD básico (crear, leer, actualizar, eliminar)
- Operaciones simples sin relaciones complejas
- Pocos endpoints necesarios

**3. Simplicidad > Flexibilidad**
- Equipos pequeños con menos experiencia
- Proyectos con timeline ajustado
- No necesitas evolución constante

**4. El over-fetching no es crítico**
- Conexiones rápidas (no móvil)
- Datos pequeños (< 10KB por request)
- Usuarios en regiones con buen internet

**Ejemplo ideal REST:**
```javascript
// API de autenticación - simple y cacheable
POST /api/auth/login
GET  /api/auth/verify
POST /api/auth/logout

// Endpoints predecibles, fáciles de proteger
```

### Usa GraphQL cuando:

**1. Múltiples clientes con necesidades diferentes**
- Web dashboard (necesita mucha data)
- App móvil (necesita poca data)
- Smart TV (necesita data específica)

**2. Relaciones complejas entre entidades**
- Redes sociales (usuarios → posts → comentarios → likes)
- E-commerce con variantes (productos → categorías → reviews → vendedores)
- CRM con múltiples entidades relacionadas

**3. Bandwidth es limitado**
- Aplicaciones móviles
- Zonas con internet lento
- APIs que se cobran por transferencia de datos

**4. Desarrollo frontend/backend desacoplado**
- Frontend puede iterar sin cambiar backend
- Múltiples equipos trabajando en paralelo
- Necesitas evolución rápida sin breaking changes

**Ejemplo ideal GraphQL:**
```graphql
# Dashboard que combina datos de múltiples fuentes
query AdminDashboard {
  me {
    name
    role
  }
  analytics {
    activeUsers
    revenue
  }
  recentOrders(limit: 10) {
    id
    total
    customer { name }
  }
  alerts {
    type
    message
  }
}

# Una query, toda la data necesaria
```

## Enfoque híbrido: Lo mejor de ambos mundos

En la práctica, muchos sistemas se benefician de **usar ambas arquitecturas**:

```javascript
// REST para operaciones simples y públicas
POST /api/auth/login          // Autenticación
POST /api/webhooks/stripe     // Webhooks
GET  /api/health              // Health check

// GraphQL para queries complejas internas
POST /graphql                 // Dashboard, reportes, vistas complejas
```

**Ventajas del enfoque híbrido:**
- REST para cacheo público y webhooks
- GraphQL para aplicaciones interactivas
- Menor complejidad que "todo GraphQL"
- Mejor performance que "todo REST"

## Conclusión

**No existe un "ganador" absoluto.** REST y GraphQL resuelven problemas diferentes:

- **REST es predecible y simple**, ideal cuando el cacheo y la simplicidad operativa importan más que la flexibilidad.

- **GraphQL es flexible y eficiente**, ideal cuando tienes clientes diversos y relaciones complejas que justifican la complejidad adicional.

La pregunta no es "¿cuál es mejor?", sino "¿cuál se adapta mejor a mi caso de uso?".

### Próximos artículos de la serie

En el **Artículo 2**, profundizaremos en el problema N+1, el cuello de botella más común en ambas arquitecturas, y cómo resolverlo en REST y GraphQL con ejemplos prácticos de código. **[Parte 2: El problema N+1 y sus soluciones prácticas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%202%3A%20El%20problema%20N%2B1%20y%20sus%20soluciones.md)**

---

## Índice de la serie

1. **[Introducción: REST vs GraphQL en sistemas reales](https://github.com/simone-rosso-dev/articles/blob/main/Introducci%C3%B3n%3A%20REST%20vs%20GraphQL%20en%20sistemas%20reales.md)**
2. **[Parte 1: ¿Cuál es más rápido? Comparativa de arquitecturas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%201:%20REST%20vs%20GraphQL%20-%20Comparativa%20de%20arquitecturas.md)**
3. **[Parte 2: El problema N+1 y sus soluciones prácticas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%202%3A%20El%20problema%20N%2B1%20y%20sus%20soluciones.md)**
4. **[Parte 3: Caché y paginación en producción](https://github.com/simone-rosso-dev/articles/blob/main/Parte%203:%20Cach%C3%A9%20y%20paginaci%C3%B3n%20en%20producci%C3%B3n.md)**
