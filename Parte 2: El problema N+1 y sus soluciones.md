# El problema N+1 en REST y GraphQL: Soluciones prácticas

**Serie: Optimización de APIs (Parte 2 de 3)**

En el [artículo anterior](link), comparamos las filosofías de REST y GraphQL y sus implicaciones en el rendimiento. Ahora vamos a profundizar en el **problema de rendimiento más común y devastador** en ambas arquitecturas: el problema N+1.

Este problema puede hacer que tu API pase de responder en 50ms a tardar 5 segundos, y lo peor es que a menudo pasa desapercibido en desarrollo para manifestarse solo en producción con datos reales.

## ¿Qué es el problema N+1?

El problema N+1 ocurre cuando ejecutas **1 query inicial** para obtener una lista de N elementos, y luego ejecutas **N queries adicionales** (una por cada elemento) para obtener datos relacionados.

**Ejemplo conceptual:**
```
Query 1: Obtener 100 cuentas bancarias
Query 2: Obtener transacciones de cuenta 1
Query 3: Obtener transacciones de cuenta 2
Query 4: Obtener transacciones de cuenta 3
...
Query 101: Obtener transacciones de cuenta 100

Total: 101 queries cuando podrían ser 2
```

El problema escala linealmente: 10 cuentas = 11 queries, 1000 cuentas = 1001 queries. Con suficiente tráfico, esto puede colapsar tu base de datos.

## El N+1 en REST: El problema oculto

En REST, el N+1 suele estar **escondido dentro de la implementación** del endpoint, invisible para el cliente pero devastador para el rendimiento.

### Ejemplo problemático: API bancaria

```javascript
// Endpoint que parece inocente
app.get('/api/accounts', async (req, res) => {
  // Query 1: Obtener todas las cuentas
  const accounts = await db.query(`
    SELECT id, account_number, balance 
    FROM accounts 
    LIMIT 100
  `);
  
  // Aquí empieza el desastre: N queries adicionales
  for (let account of accounts) {
    // Query por cada cuenta (100 veces)
    account.recentTransactions = await db.query(`
      SELECT id, amount, description, date
      FROM transactions 
      WHERE account_id = $1 
      ORDER BY date DESC 
      LIMIT 5
    `, [account.id]);
  }
  
  res.json(accounts);
  // Total: 101 queries para devolver 100 cuentas con 5 transacciones
});
```

**¿Por qué es un problema?**

- Cada query tiene latencia (conexión, ejecución, red)
- 100 queries × 10ms = 1 segundo solo en queries
- Bajo carga, las conexiones a la DB se agotan
- El problema es invisible en dev con 5 cuentas de prueba

### Solución REST #1: JOIN o subconsulta

```javascript
app.get('/api/accounts', async (req, res) => {
  // Una sola query que obtiene TODO
  const result = await db.query(`
    SELECT 
      a.id as account_id,
      a.account_number,
      a.balance,
      t.id as transaction_id,
      t.amount,
      t.description,
      t.date
    FROM accounts a
    LEFT JOIN LATERAL (
      SELECT id, amount, description, date
      FROM transactions 
      WHERE account_id = a.id 
      ORDER BY date DESC 
      LIMIT 5
    ) t ON true
    LIMIT 100
  `);
  
  // Agrupar resultados en aplicación
  const accountsMap = {};
  
  result.rows.forEach(row => {
    if (!accountsMap[row.account_id]) {
      accountsMap[row.account_id] = {
        id: row.account_id,
        accountNumber: row.account_number,
        balance: row.balance,
        recentTransactions: []
      };
    }
    
    if (row.transaction_id) {
      accountsMap[row.account_id].recentTransactions.push({
        id: row.transaction_id,
        amount: row.amount,
        description: row.description,
        date: row.date
      });
    }
  });
  
  const accounts = Object.values(accountsMap);
  res.json(accounts);
  
  // Total: 1 query para todo
});
```

**Ventajas:**
- 101 queries → 1 query (100x más rápido)
- Uso eficiente de índices de DB
- Latencia predecible

**Desventajas:**
- Query más compleja
- Necesitas agrupar datos en aplicación
- Menos flexible (fija por endpoint)

### Solución REST #2: Endpoint agregado específico

```javascript
// Endpoint especializado que optimiza el caso de uso
app.get('/api/accounts/with-transactions', async (req, res) => {
  // Query optimizada específica para este caso
  const accounts = await db.query(`
    WITH recent_transactions AS (
      SELECT 
        account_id,
        json_agg(
          json_build_object(
            'id', id,
            'amount', amount,
            'description', description,
            'date', date
          ) ORDER BY date DESC
        ) FILTER (WHERE rn <= 5) as transactions
      FROM (
        SELECT *,
          ROW_NUMBER() OVER (PARTITION BY account_id ORDER BY date DESC) as rn
        FROM transactions
      ) t
      WHERE rn <= 5
      GROUP BY account_id
    )
    SELECT 
      a.id,
      a.account_number,
      a.balance,
      COALESCE(rt.transactions, '[]'::json) as recent_transactions
    FROM accounts a
    LEFT JOIN recent_transactions rt ON a.id = rt.account_id
    LIMIT 100
  `);
  
  res.json(accounts);
});
```

**Análisis:** En REST, resuelves el N+1 con SQL optimizado. El tradeoff es que creas endpoints específicos para cada caso de uso.

## El N+1 en GraphQL: El problema visible

En GraphQL, el N+1 aparece directamente en los **resolvers**, visible en la estructura de la query pero igual de peligroso.

### Ejemplo problemático: Resolvers naive

```typescript
const typeDefs = `#graphql
  type Account {
    id: ID!
    accountNumber: String!
    balance: Float!
    transactions: [Transaction!]!
  }
  
  type Transaction {
    id: ID!
    amount: Float!
    description: String!
    date: String!
  }
  
  type Query {
    accounts: [Account!]!
  }
`;

const resolvers = {
  Query: {
    // Query 1: obtener cuentas
    accounts: async () => {
      return await db.query('SELECT * FROM accounts LIMIT 100');
    }
  },
  
  Account: {
    // Este resolver se ejecuta 100 veces (una por cuenta)
    transactions: async (account) => {
      // Query por cada cuenta
      return await db.query(`
        SELECT * FROM transactions 
        WHERE account_id = $1 
        ORDER BY date DESC 
        LIMIT 5
      `, [account.id]);
    }
  }
};

// El cliente hace esta query:
// query {
//   accounts {
//     id
//     accountNumber
//     transactions {  ← Desencadena 100 queries
//       amount
//     }
//   }
// }

// Total: 101 queries
```

### Solución GraphQL: DataLoader

DataLoader es una librería que **agrupa y agrupa requests** automáticamente, resolviendo el N+1 de forma elegante.

```typescript
import DataLoader from 'dataloader';

// Función que recibe múltiples IDs y devuelve resultados en orden
const batchTransactions = async (accountIds) => {
  console.log('Batch loading transactions for:', accountIds);
  
  // Esta query se ejecuta UNA vez con todos los IDs
  const result = await db.query(`
    SELECT * FROM transactions 
    WHERE account_id = ANY($1)
    ORDER BY date DESC
  `, [accountIds]);
  
  // Agrupar transacciones por account_id
  const transactionsByAccount = {};
  result.rows.forEach(tx => {
    if (!transactionsByAccount[tx.account_id]) {
      transactionsByAccount[tx.account_id] = [];
    }
    if (transactionsByAccount[tx.account_id].length < 5) {
      transactionsByAccount[tx.account_id].push(tx);
    }
  });
  
  // IMPORTANTE: Devolver en el mismo orden que accountIds
  return accountIds.map(id => transactionsByAccount[id] || []);
};

// Crear DataLoader (uno por request)
const createLoaders = () => ({
  transactions: new DataLoader(batchTransactions)
});

const resolvers = {
  Query: {
    accounts: async () => {
      return await db.query('SELECT * FROM accounts LIMIT 100');
    }
  },
  
  Account: {
    transactions: async (account, _, { loaders }) => {
      // DataLoader agrupa todos los loads y ejecuta batch
      return loaders.transactions.load(account.id);
    }
  }
};

// Configurar servidor con loaders
const server = new ApolloServer({
  typeDefs,
  resolvers
});

app.use('/graphql', expressMiddleware(server, {
  context: async () => ({
    loaders: createLoaders() // Nuevo loader por request
  })
}));

// Misma query del cliente:
// query {
//   accounts {
//     id
//     transactions { amount }
//   }
// }

// Ahora solo 2 queries: 1 para cuentas, 1 batch para transacciones
```

**Cómo funciona DataLoader:**

1. Durante una request, múltiples resolvers llaman `loader.load(id)`
2. DataLoader **agrupa** todos los IDs pendientes
3. Al final del tick del event loop, ejecuta la función batch UNA vez
4. Distribuye los resultados a cada resolver que esperaba

**Ventajas:**
- Reutilizable para cualquier relación
- Automático (no cambias queries)
- Incluye caché por request

**Desventajas:**
- Requiere entender async/await bien
- Debes devolver resultados en orden correcto
- Un loader nuevo por request (no global)

## Comparativa de soluciones

| Aspecto | REST (JOIN) | GraphQL (DataLoader) |
|---------|-------------|----------------------|
| **Complejidad SQL** | Alta | Baja |
| **Complejidad código** | Media | Media |
| **Reutilizable** | No (por endpoint) | Sí (cualquier relación) |
| **Flexibilidad** | Baja | Alta |
| **Performance** | Excelente | Excelente |
| **Aprende una vez** | No | Sí |

## Detectando el N+1 en producción

### En REST: Logging de queries

```javascript
// Middleware para contar queries por request
let queryCount = 0;

app.use((req, res, next) => {
  queryCount = 0;
  
  res.on('finish', () => {
    if (queryCount > 10) {
      console.warn(`⚠️ N+1 detected: ${req.path} executed ${queryCount} queries`);
    }
  });
  
  next();
});

// Wrapper de db.query que cuenta
const originalQuery = db.query;
db.query = function(...args) {
  queryCount++;
  return originalQuery.apply(this, args);
};
```

### En GraphQL: Apollo Studio

Apollo Studio muestra automáticamente:
- Cuántas queries ejecuta cada resolver
- Qué resolvers son lentos
- Sugerencias de DataLoader

También puedes usar plugins:

```typescript
const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [
    {
      async requestDidStart() {
        let queryCount = 0;
        
        return {
          async executionDidStart() {
            const originalQuery = db.query;
            db.query = function(...args) {
              queryCount++;
              return originalQuery.apply(this, args);
            };
          },
          
          async willSendResponse() {
            if (queryCount > 10) {
              console.warn(`⚠️ Query executed ${queryCount} DB queries`);
            }
          }
        };
      }
    }
  ]
});
```

## Mejores prácticas

### Para REST:

1. **Revisa queries en desarrollo** con `EXPLAIN ANALYZE`
2. **Crea endpoints agregados** para casos comunes (`/accounts/with-transactions`)
3. **Usa JOINs o subconsultas** en lugar de loops
4. **Implementa eager loading** en tu ORM (si usas uno)

### Para GraphQL:

1. **Usa DataLoader siempre** para relaciones
2. **Un loader por tipo de entidad** (accounts, transactions, users)
3. **Nuevos loaders por request** (no globales)
4. **Monitorea con Apollo Studio** o plugins custom

## Conclusión

El problema N+1 afecta a ambas arquitecturas pero se resuelve de formas diferentes:

- **REST**: Lo resuelves con SQL optimizado en cada endpoint
- **GraphQL**: Lo resuelves con DataLoader reutilizable

Ninguna arquitectura es inmune, pero GraphQL ofrece una solución más sistemática que puedes aplicar una vez y reutilizar en todas las relaciones.

En el **próximo artículo** (Parte 3), exploraremos estrategias avanzadas de caché, paginación y rate limiting específicas para cada arquitectura.

---

**¿Has sufrido el N+1 en tu API? ¿Cómo lo detectaste? Comparte tu experiencia en los comentarios.**
