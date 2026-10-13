# ADR-005 · Corrección de la inconsistencia del testigo de versión

- **Estado:** aceptada
- **Contexto:** se detectó una inconsistencia entre [`ADR-002`](adr-002-concurrencia-optimista.md),
  [`plan.md`](../plan.md) y [`tasks.md`](../tasks.md), que describían un testigo de versión por libro,
  y [`data-model.md`](../data-model.md), que no declaraba ninguna columna `version` en `book`.
  Además, `ck_book_available_range` por sí solo no detecta una escritura perdida: dos actualizaciones
  que escriban `available_copies = 1` pueden dejar el mismo valor. Eloquent tampoco ofrece bloqueo
  optimista nativo que resuelva esta discrepancia automáticamente.
- **Decisión:** se implementará la concurrencia optimista **sin testigo almacenado**. El repositorio
  hará un `UPDATE` atómico condicionado a `available_copies > 0` y verificará las filas afectadas.
  Cero filas afectadas significa que no queda ejemplar disponible y se traducirá a
  `409 BOOK_NOT_AVAILABLE`. El descuento y la creación de la línea se mantienen dentro de una única
  transacción. [`ADR-002`](adr-002-concurrencia-optimista.md) queda actualizado como decisión normativa.
- **Alternativas descartadas:** declarar `book.version` como una nueva columna y modificar el modelo,
  sus restricciones y sus consultas; confiar únicamente en `ck_book_available_range`, porque no evita
  escrituras perdidas; usar `SELECT ... FOR UPDATE`, porque introduce bloqueo pesimista innecesario
  para este caso.
- **Consecuencias:** el modelo de datos conserva únicamente las columnas requeridas y la estrategia
  funciona con Eloquent mediante una actualización condicional explícita. El repositorio debe tratar
  el conteo de filas afectadas como parte del contrato de concurrencia y cubrirlo con una prueba
  concurrente contra PostgreSQL.
- **Cómo se verifica:** dos préstamos simultáneos sobre un libro con un ejemplar producen exactamente
  una respuesta 201 y una respuesta 409; el `CHECK` sigue rechazando cualquier estado fuera de rango.
