# REST vs GraphQL: Optimización de APIs en sistemas reales

**Introducción a la serie de 3 artículos**

Tu API funciona perfectamente en desarrollo. Responde en 50ms, maneja tus casos de prueba sin problemas, y el código es limpio y elegante. Todo va bien... hasta que llega a producción.

De repente, con usuarios reales y datos de verdad, las respuestas tardan segundos. La base de datos se satura. Los usuarios se quejan de lentitud. Y tú te preguntas: **¿qué salió mal?**

## El problema invisible del rendimiento

La mayoría de problemas de rendimiento en APIs no aparecen en desarrollo porque trabajamos con datos de prueba mínimos: 5 usuarios, 20 transacciones, consultas simples. Pero en producción, con 10,000 usuarios y millones de registros, esos mismos endpoints que parecían rápidos se convierten en cuellos de botella.

Lo peor es que estos problemas suelen estar **escondidos en el diseño**, no en bugs obvios. Son decisiones arquitectónicas que tomamos al principio, pensando que funcionarían bien, pero que se convierten en limitaciones críticas cuando el sistema crece.

## REST y GraphQL: No es solo sintaxis

Cuando elegimos entre REST y GraphQL, a menudo nos enfocamos en aspectos superficiales: "GraphQL tiene un solo endpoint", "REST es más simple", "GraphQL es más moderno". Pero la realidad es que **cada arquitectura tiene características fundamentales que determinan cómo se comporta bajo carga**.

REST te obliga a definir endpoints fijos. GraphQL te da flexibilidad completa. Esa diferencia no es cosmética: afecta directamente cómo cacheas datos, cómo paginas resultados, y cómo proteges tu sistema de queries abusivas.

## Los 3 problemas que todo desarrollador enfrenta

En esta serie de artículos, vamos a explorar los tres problemas de rendimiento más comunes en APIs modernas, y cómo cada arquitectura los afronta:

### 1. La batalla del caché y la transferencia de datos

**El dilema:** ¿Cómo evitas hacer la misma consulta mil veces? ¿Cómo reduces el tamaño de las respuestas cuando el cliente solo necesita 3 campos de 20?

REST puede aprovechar toda la infraestructura de caché HTTP (CDNs, proxies, navegadores), pero sufre de over-fetching: devuelves datos que el cliente no necesita. GraphQL elimina el over-fetching, pero implementar caché efectivo es más complejo.

**[Artículo 1: REST vs GraphQL - ¿Cuál es más rápido?]**

En este artículo comparamos las arquitecturas desde cero, analizando:
- Cómo cada una maneja el acceso a datos
- Ventajas y desventajas reales de rendimiento
- Cuándo usar cada arquitectura según tu caso de uso
- Un caso práctico: mismo requisito, dos implementaciones

### 2. El problema N+1: El asesino silencioso

**El dilema:** Tu endpoint devuelve 100 cuentas con sus transacciones recientes. Parece simple, pero estás ejecutando 101 queries a la base de datos sin darte cuenta.

El problema N+1 es la causa número uno de APIs lentas. Aparece cuando cargas una lista de elementos y luego haces queries adicionales por cada elemento. En desarrollo con 5 registros de prueba no lo notas. En producción con 1,000 usuarios simultáneos, tu base de datos colapsa.

**[Artículo 2: El problema N+1 y sus soluciones prácticas]**

En este artículo profundizamos en:
- Qué es el N+1 y por qué es tan común
- Cómo se manifiesta diferente en REST y GraphQL
- Soluciones prácticas: JOINs vs DataLoader
- Cómo detectarlo antes de que llegue a producción

### 3. Caché, paginación y control: Las técnicas que escalan

**El dilema:** ¿Cómo cacheas eficientemente sin servir datos obsoletos? ¿Cómo paginas millones de registros sin que las últimas páginas tarden minutos? ¿Cómo evitas que un usuario malintencionado tumbe tu servidor con una query compleja?

Estas no son optimizaciones "nice to have", son **requisitos para producción**. Sin ellas, tu API no escala más allá de tráfico trivial.

**[Artículo 3: Estrategias de caché y paginación en producción]**

En este artículo exploramos:
- Caché HTTP vs caché de aplicación
- Por qué offset-based pagination no escala
- Cursor pagination y Relay Connections
- Rate limiting por complejidad, no solo por frecuencia

## Por qué esta serie es diferente

No vamos a darte teoría abstracta ni "mejores prácticas" genéricas. Esta serie se basa en **problemas reales con soluciones prácticas**:

- ✅ **Código funcional**, no pseudocódigo
- ✅ **Comparaciones lado a lado** de REST y GraphQL en el mismo caso de uso
- ✅ **Métricas reales**: latencia, queries ejecutadas, datos transferidos
- ✅ **Trade-offs honestos**: no hay soluciones perfectas, solo decisiones informadas

Cada artículo incluye ejemplos completos en TypeScript/Node.js que puedes adaptar a tu stack.

## Lo que aprenderás

Al final de esta serie, serás capaz de:

1. **Elegir la arquitectura apropiada** para tu caso de uso basándote en métricas reales, no en modas
2. **Identificar y resolver el problema N+1** antes de que llegue a producción
3. **Implementar estrategias de caché efectivas** según tu arquitectura
4. **Paginar correctamente** sin importar el tamaño de tu dataset
5. **Proteger tu API** de queries abusivas y tráfico excesivo

## Para quién es esta serie

Esta serie está diseñada para desarrolladores que:

- Ya tienen experiencia básica con REST o GraphQL
- Han construido APIs que funcionan, pero quieren hacerlas **escalables**
- Enfrentan (o quieren evitar) problemas de rendimiento en producción
- Necesitan tomar decisiones arquitectónicas informadas, no basadas en hype

Si estás empezando con APIs, algunos conceptos pueden ser avanzados. Te recomendamos leer primero tutoriales básicos de REST y GraphQL, y luego volver a esta serie cuando estés listo para optimizar.

## Empecemos

El rendimiento no es un problema que "arreglas después". Es una consecuencia directa de decisiones de diseño que tomas al principio. Esta serie te da las herramientas para tomar esas decisiones correctamente desde el día uno.

**Próximo artículo:** [REST vs GraphQL: ¿Cuál es más rápido? Comparativa de arquitecturas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%201:%20REST%20vs%20GraphQL%20-%20Comparativa%20de%20arquitecturas.md)

---

## Índice de la serie

1. **[Introducción: REST vs GraphQL en sistemas reales](https://github.com/simone-rosso-dev/articles/blob/main/Introducci%C3%B3n%3A%20REST%20vs%20GraphQL%20en%20sistemas%20reales.md)**
2. **[Parte 1: ¿Cuál es más rápido? Comparativa de arquitecturas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%201:%20REST%20vs%20GraphQL%20-%20Comparativa%20de%20arquitecturas.md)**
3. **[Parte 2: El problema N+1 y sus soluciones prácticas](https://github.com/simone-rosso-dev/articles/blob/main/Parte%202%3A%20El%20problema%20N%2B1%20y%20sus%20soluciones.md)**
4. **[Parte 3: Caché y paginación en producción](https://github.com/simone-rosso-dev/articles/blob/main/Parte%203:%20Cach%C3%A9%20y%20paginaci%C3%B3n%20en%20producci%C3%B3n.md)**
