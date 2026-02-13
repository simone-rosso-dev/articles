# Estrategias de caché y paginación: REST vs GraphQL en producción

**Serie: Optimización de APIs (Parte 3 de 3)**

En los artículos anteriores comparamos [las arquitecturas REST y GraphQL](link1) y cómo resolver [el problema N+1](link2). En este artículo final, exploramos dos técnicas críticas para escalar APIs en producción: **caché** y **paginación**.

Estas técnicas determinan si tu API puede manejar 100 usuarios o 100,000. La implementación varía significativamente entre REST y GraphQL, y elegir la estrategia incorrecta puede limitar tu escalabilidad.

## Parte 1: Estrategias de caché

El caché no acelera consultas lentas, **evita ejecutarlas**. En sistemas con lecturas frecuentes y escrituras relativamente escasas, el impacto es inmediato: de segundos a milisegundos.

### REST: Caché HTTP nativo (la ventaja decisiva)

REST puede aprovechar toda la infraestructura de caché HTTP: navegadores, proxies intermedios, CDNs. Esta es su **mayor ventaja sobre GraphQL**.

#### Ejemplo: Caché en múltiples niveles

```javascript
// Servidor
app.get('/api/accounts/:id', async (req, res) => {
  const account = await db.query(
    'SELECT id, name, balance FROM accounts WHERE id = $1',
    [req.params.id]
  );
  
  // Headers HTTP estándar
  res.set('Cache-Control', 'public, max-age=300'); // 5 minutos
  res.set('ETag', generateETag(account));          // Validación
  res.set('Last-Modified', account.updated_at);
  
  res.json(account);
});

// Flujo de caché:
// 1. Request inicial: servidor → 50ms
// 2. Requests subsecuentes:
//    - Navegador (caché local) → 0ms ✅
//    - CDN (CloudFlare) → 5ms ✅
//    - Proxy corporativo → 10ms ✅
//    - Solo si expiró: servidor → 50ms
```

**Ventajas REST:**
- Infraestructura estándar (Varnish, nginx, CloudFlare)
- Caché geográfico sin código adicional
- Navegadores respetan automáticamente
- ETags para validación eficiente

**Limitaciones REST:**
- Granularidad limitada (solo por URL completa)
- Difícil cachear con query params variables
- Invalidación más compleja

#### Patrón: Cache-Aside con Redis

Para datos que cambian más frecuentemente:

```javascript
import Redis from 'ioredis';
const redis = new Redis();

app.get('/api/accounts/:id/balance', async (req, res) => {
  const { id } = req.params;
  const cacheKey = `balance:${id}`;
  
  // 1. Intentar caché primero
  const cached = await redis.get(cacheKey);
  if (cached) {
    res.set('X-Cache', 'HIT');
    return res.json({ balance: parseFloat(cached) });
  }
  
  // 2. Si no existe, consultar DB
  const result = await db.query(
    'SELECT balance FROM accounts WHERE id = $1',
    [id]
  );
  
  // 3. Cachear por TTL apropiado
  const balance = result.rows[0].balance;
  await redis.setex(cacheKey, 60, balance); // 1 minuto
  
  res.set('X-Cache', 'MISS');
  res.json({ balance });
});
```

#### Patrón: Write-Through para consistencia

En dominios sensibles (finanzas), actualiza caché y DB simultáneamente:

```javascript
app.post('/api/accounts/:id/transfer', async (req, res) => {
  const { id } = req.params;
  const { amount, toAccount } = req.body;
  
  await db.transaction(async (trx) => {
    // Actualizar DB
    await trx.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, id]
    );
    await trx.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toAccount]
    );
  });
  
  // Invalidar caché de ambas cuentas
  await Promise.all([
    redis.del(`balance:${id}`),
    redis.del(`balance:${toAccount}`)
  ]);
  
  res.json({ success: true });
});
```

### GraphQL: Caché a nivel de aplicación

GraphQL no puede usar caché HTTP efectivamente porque **todas las queries van al mismo endpoint** `/graphql`. Necesitas estrategias a nivel de aplicación.

#### Estrategia 1: Caché por resolver

```typescript
import Redis from 'ioredis';
const redis = new Redis();

const resolvers = {
  Query: {
    account: async (_, { id }) => {
      const cacheKey = `account:${id}`;
      
      const cached = await redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }
      
      const account = await db.query(
        'SELECT * FROM accounts WHERE id = $1',
        [id]
      );
      
      // TTL diferente según volatilidad
      await redis.setex(cacheKey, 300, JSON.stringify(account));
      
      return account;
    }
  },
  
  Account: {
    // Caché diferente por campo
    balance: async (account) => {
      const key = `balance:${account.id}`;
      
      const cached = await redis.get(key);
      if (cached) return parseFloat(cached);
      
      // Balance cambia frecuentemente: TTL corto
      await redis.setex(key, 60, account.balance);
      return account.balance;
    },
    
    accountNumber: (account) => {
      // Datos inmutables: sin caché necesario
      return account.accountNumber;
    }
  }
};
```

**Ventajas GraphQL:**
- Caché granular por campo
- TTL diferente según volatilidad de datos
- Control fino sobre qué cachear

**Desventajas GraphQL:**
- No aprovecha CDNs tradicionales
- Más código/complejidad
- Necesitas infraestructura (Redis)

#### Estrategia 2: Persisted Queries

Para recuperar caché HTTP en GraphQL:

```typescript
// Cliente envía hash en lugar de query completa
const QUERY_HASH = 'a1b2c3d4e5f6'; // SHA256 de la query

fetch('/graphql', {
  method: 'POST',
  body: JSON.stringify({
    operationId: QUERY_HASH,
    variables: { accountId: '123' }
  })
});

// Servidor
app.post('/graphql', async (req, res) => {
  const { operationId, variables } = req.body;
  const cacheKey = `query:${operationId}:${JSON.stringify(variables)}`;
  
  // Ahora SÍ puedes cachear por operationId
  const cached = await redis.get(cacheKey);
  if (cached) {
    res.set('Cache-Control', 'public, max-age=60');
    return res.json(JSON.parse(cached));
  }
  
  // Ejecutar query
  const query = persistedQueries[operationId];
  const result = await executeQuery(query, variables);
  
  await redis.setex(cacheKey, 60, JSON.stringify(result));
  res.json(result);
});
```

#### Estrategia 3: Apollo Client (caché automático)

Apollo Client normaliza y cachea automáticamente en el cliente:

```typescript
const client = new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      Account: {
        // Definir cómo cachear
        keyFields: ['id'],
        fields: {
          balance: {
            // Caché de balance por 1 minuto
            read(existing, { readField }) {
              return existing;
            }
          }
        }
      }
    }
  })
});

// Query 1
const { data } = await client.query({
  query: GET_ACCOUNT,
  variables: { id: '123' }
});

// Query 2 (mismo account) - viene del caché ✅
const { data: cached } = await client.query({
  query: GET_ACCOUNT,
  variables: { id: '123' }
});
```

### Comparativa de caché

| Aspecto | REST | GraphQL |
|---------|------|---------|
| **Caché HTTP** | ✅ Nativo | ❌ Limitado |
| **CDN** | ✅ Directo | ⚠️ Solo con Persisted Queries |
| **Granularidad** | URL completa | Por campo/entidad |
| **Complejidad** | Baja | Media-Alta |
| **Invalidación** | ETags | Custom |
| **Cliente** | Navegador automático | Apollo Client |

**Veredicto:** REST tiene ventaja decisiva en caché para APIs públicas. GraphQL mejor para caché interno complejo.

## Parte 2: Paginación escalable

Devolver 1 millón de registros de una vez colapsa cualquier sistema. La paginación es obligatoria, pero hay formas buenas y malas de implementarla.

### El problema del offset

La paginación tradicional con `OFFSET` tiene un problema oculto:

```sql
-- Página 1: rápido
SELECT * FROM transactions ORDER BY date DESC LIMIT 20 OFFSET 0;

-- Página 100: lento
SELECT * FROM transactions ORDER BY date DESC LIMIT 20 OFFSET 1980;
-- La DB debe escanear y descartar 1980 filas antes de devolver 20

-- Página 10000: muy lento
SELECT * FROM transactions ORDER BY date DESC LIMIT 20 OFFSET 199980;
-- Escanea 199,980 filas para devolver 20
```

El costo crece linealmente. Con 1 millón de registros, las últimas páginas son inutilizables.

### REST: Paginación por cursor

```javascript
app.get('/api/transactions', async (req, res) => {
  const cursor = req.query.cursor; // ID de última transacción vista
  const limit = Math.min(parseInt(req.query.limit) || 20, 100);
  
  let query = `
    SELECT id, amount, description, date
    FROM transactions
  `;
  
  const params = [];
  
  if (cursor) {
    // Continuar desde el cursor
    query += ' WHERE id < $1';
    params.push(cursor);
  }
  
  query += ' ORDER BY id DESC LIMIT $' + (params.length + 1);
  params.push(limit + 1); // +1 para saber si hay más
  
  const result = await db.query(query, params);
  const hasMore = result.rows.length > limit;
  const transactions = result.rows.slice(0, limit);
  
  res.json({
    data: transactions,
    pagination: {
      nextCursor: hasMore ? transactions[transactions.length - 1].id : null,
      hasMore
    }
  });
});

// Cliente
// Request 1: GET /api/transactions?limit=20
// Response: { data: [...], pagination: { nextCursor: "tx_999", hasMore: true } }

// Request 2: GET /api/transactions?cursor=tx_999&limit=20
// Response: { data: [...], pagination: { nextCursor: "tx_979", hasMore: true } }
```

**Ventajas del cursor:**
- Performance constante (siempre usa índice)
- Escala a millones de registros
- No hay "páginas vacías" si se insertan/eliminan datos

**Limitaciones:**
- No puedes saltar a página arbitraria
- Necesitas índice en columna de cursor
- Más complejo que offset

### GraphQL: Relay Connection Standard

GraphQL tiene un estándar establecido para paginación (Relay Cursor Connections):

```graphql
type Query {
  transactions(
    first: Int      # Número de resultados
    after: String   # Cursor para continuar
  ): TransactionConnection!
}

type TransactionConnection {
  edges: [TransactionEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type TransactionEdge {
  cursor: String!  # Identificador opaco
  node: Transaction!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Transaction {
  id: ID!
  amount: Float!
  description: String!
  date: String!
}
```

Implementación:

```typescript
const resolvers = {
  Query: {
    transactions: async (_, { first = 20, after }) => {
      first = Math.min(first, 100); // Límite máximo
      
      let query = `
        SELECT id, amount, description, date
        FROM transactions
      `;
      
      const params = [];
      
      if (after) {
        // Decodificar cursor (base64)
        const decodedCursor = Buffer.from(after, 'base64').toString();
        query += ' WHERE id < $1';
        params.push(decodedCursor);
      }
      
      query += ' ORDER BY id DESC LIMIT $' + (params.length + 1);
      params.push(first + 1);
      
      const result = await db.query(query, params);
      const hasMore = result.rows.length > first;
      const transactions = result.rows.slice(0, first);
      
      return {
        edges: transactions.map(tx => ({
          cursor: Buffer.from(tx.id).toString('base64'),
          node: tx
        })),
        pageInfo: {
          hasNextPage: hasMore,
          hasPreviousPage: !!after,
          startCursor: transactions.length > 0
            ? Buffer.from(transactions[0].id).toString('base64')
            : null,
          endCursor: transactions.length > 0
            ? Buffer.from(transactions[transactions.length - 1].id).toString('base64')
            : null
        },
        totalCount: await getTotalCount() // Opcional, puede ser costoso
      };
    }
  }
};

// Cliente
query {
  transactions(first: 20) {
    edges {
      cursor
      node {
        id
        amount
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

**Ventajas Relay:**
- Estándar bien documentado
- Bidireccional (forward/backward)
- Metadatos ricos (cursors, hasNext, etc.)
- Compatible con Apollo Client

**Limitaciones:**
- Schema más complejo (edges, nodes, pageInfo)
- Curva de aprendizaje mayor
- Overhead de tipos adicionales

### Comparativa de paginación

| Aspecto | REST Cursor | GraphQL Relay |
|---------|-------------|---------------|
| **Simplicidad** | Media | Baja |
| **Estándar** | Varía | Relay (oficial) |
| **Performance** | Excelente | Excelente |
| **Metadatos** | Custom | Rico (pageInfo) |
| **Bidireccional** | Manual | Built-in |
| **Apollo Client** | No integrado | Integrado |

## Parte 3: Rate Limiting y control de complejidad

### REST: Rate limiting por endpoint

```javascript
import rateLimit from 'express-rate-limit';

// Rate limiting simple
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutos
  max: 100, // 100 requests
  message: 'Too many requests'
});

// Aplicar por ruta
app.use('/api/transactions', limiter);

// O diferente por endpoint
const strictLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10 // Solo 10 para operaciones sensibles
});

app.use('/api/transfers', strictLimiter);
```

**Ventajas REST:**
- Simple de implementar
- Predecible por ruta
- Fácil de entender

**Limitaciones:**
- No considera complejidad real
- Trata igual request simple y compleja

### GraphQL: Límite por complejidad de query

```typescript
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    createComplexityLimitRule(1000, {
      // Asignar costos
      scalarCost: 1,
      objectCost: 5,
      listFactor: 10,
      onCost: (cost) => {
        console.log('Query cost:', cost);
      }
    })
  ]
});

// Ejemplo de análisis:
// query {
//   accounts(first: 10) {        // 10 accounts × 5 = 50
//     transactions(first: 20) {  // 10 × 20 × 5 = 1000
//       amount                   // Costo total = 1050
//     }
//   }
// }
// Si el límite es 1000, esta query sería rechazada
```

También limitar profundidad:

```typescript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  typeDefs,
  resolvers,
  validationRules: [
    depthLimit(5) // Máximo 5 niveles de anidación
  ]
});

// Esta query sería rechazada (6 niveles):
// query {
//   user {              // 1
//     posts {           // 2
//       comments {      // 3
//         author {      // 4
//           posts {     // 5
//             likes {   // 6 ❌ Rechazada
```

**Ventajas GraphQL:**
- Limita por complejidad real
- Protege contra queries abusivas
- Más justo que count de requests

**Limitaciones:**
- Más complejo de configurar
- Difícil calcular costos apropiados
- Puede rechazar queries legítimas

## Conclusión: Estrategias por arquitectura

### REST - Receta de optimización

1. **Caché HTTP agresivo** con ETags y Cache-Control
2. **CDN** para recursos estáticos y endpoints públicos
3. **Redis cache-aside** para datos dinámicos
4. **Paginación por cursor** para listas grandes
5. **Rate limiting por endpoint** con express-rate-limit

### GraphQL - Receta de optimización

1. **DataLoader** para resolver N+1 (del artículo 2)
2. **Caché por resolver** con Redis y TTLs apropiados
3. **Apollo Client** con caché normalizado
4. **Relay Cursor Connections** para paginación
5. **Límites de complejidad** y profundidad
6. **Persisted Queries** para recuperar caché HTTP

### El veredicto final

No existe un ganador absoluto. Cada arquitectura brilla en contextos diferentes:

**Elige REST cuando:**
- Necesitas máximo cacheo público (CDN, proxies)
- Simplicidad operativa es prioritaria
- API pública con casos de uso conocidos

**Elige GraphQL cuando:**
- Múltiples clientes con necesidades variadas
- Bandwidth limitado (móvil)
- Relaciones complejas entre entidades
- Desarrollo frontend/backend desacoplado

La mayoría de sistemas reales se benefician de un **enfoque híbrido**: REST para operaciones simples y públicas, GraphQL para queries complejas internas.

---

**Esto concluye nuestra serie de 3 artículos sobre optimización de APIs. ¿Qué técnicas has aplicado? ¿Qué resultados obtuviste? Comparte en los comentarios.**

**Lee la serie completa:**
- [Parte 1: REST vs GraphQL - Comparativa de arquitecturas](link1)
- [Parte 2: El problema N+1 y sus soluciones](link2)
- [Parte 3: Caché y paginación en producción](link3)
