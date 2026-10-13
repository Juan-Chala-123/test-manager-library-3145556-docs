# ADR-002 · Concurrencia optimista sobre los ejemplares

- **Estado:** aceptada
- **Contexto:** dos bibliotecarios pueden prestar el **último ejemplar** del mismo libro al mismo tiempo (RNF-2). Si ambos leen
  `available_copies = 1` y ambos escriben, el sistema presta un libro que no tiene.
- **Decisión:** control de concurrencia **optimista sin columna de versión**. El adaptador descuenta el ejemplar mediante una
  actualización atómica condicionada (`UPDATE ... SET available_copies = available_copies - 1 WHERE id = ? AND available_copies > 0`)
  y verifica las filas afectadas. Si el resultado es cero, responde **409 `BOOK_NOT_AVAILABLE`**. La operación de préstamo
  y la creación de la línea ocurren en una única transacción. La **última barrera** es el `CHECK`
  `ck_book_available_range` (`0 <= available_copies <= total_copies`).
- **Alternativas descartadas:** agregar un testigo de versión persistido sin declararlo en el modelo de datos (segunda fuente de
  verdad documental); bloqueo pesimista de la fila (`SELECT … FOR UPDATE`) en cada préstamo (serializa toda la biblioteca por
  un caso raro); contador en memoria (no sobrevive a más de un proceso).
- **Consecuencias:** si el `CHECK` salta, **algo escribió fuera del adaptador**: es un defecto, no un caso de negocio. Devolver dos veces
  la misma línea no suma dos veces porque `returned_at` ya está lleno (CA-07.2).
- **Cómo se verifica:** prueba concurrente con dos peticiones simultáneas sobre un libro con un ejemplar: exactamente una recibe 201.
