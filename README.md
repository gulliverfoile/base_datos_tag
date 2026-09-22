══════════════════════════════════════════════════════════════════════════
  REGISTRO LOCAL · Fase 1.5 + 2
  Base de datos personal sin propietario, portable, basada en Event Sourcing
═══════════════════════════════════════════════════════════════════════════

¿QUÉ ES?

Un único archivo HTML que funciona como base de datos local para
gestionar registros (pacientes, y en el futuro cualquier otra entidad).
No necesita servidor, no necesita instalación, no depende de ninguna
empresa. Se abre en el navegador y ya está.

No es un producto acabado. Es un esqueleto con las piezas mínimas
funcionando para poder iterar encima. La idea es que sirva como base
sobre la que añadir entidades, campos, reglas e importadores según
se necesiten.

Pensado para:
- Proyectos personales donde no quieres depender de Access ni de la nube.
- Aprender cómo funciona una base de datos por dentro sin magia.
- Tener algo portable que puedas llevar en un USB y abrir en cualquier PC
  con Chrome/Edge.


¿QUÉ HACE HOY?

- Crear, editar y archivar pacientes.
- Guardar cada cambio como un evento inmutable en un log.
- Reconstruir el estado actual a partir del log.
- Ver el historial completo de un paciente.
- Ver cómo estaba un paciente en una fecha pasada (replay).
- Deshacer el último cambio (sin borrar nada, emitiendo un evento nuevo).
- Importar JSONs de orígenes distintos, con inferencia de campos.
- Avisar cuándo un campo se ha inferido y con qué nivel de confianza.
- Buscar con filtros combinables (eq, contiene, gt, lt, entre, tieneTag).
- Agrupar pacientes por facetas declaradas (demografía, administrativa,
  clínica).
- Buscar rápido por etiquetas usando un índice inverso.
- Exportar todo (backup, pacientes actuales, log crudo).
- Registrar logs de lo que pasa, con niveles y exportables.


¿QUÉ NO HACE (todavía)?

- No tiene login ni permisos. Es local, un solo usuario.
- No cifra nada. Si quieres cifrado, va en una fase posterior.
- No sincroniza entre dispositivos.
- No tiene interfaz bonita ni pensada para uso diario.
- No tiene validación legal para datos clínicos reales.
- No soporta más de una entidad a la vez en la UI (aunque el esquema
  sí está preparado para añadir visita, nota, etc.).
- No funciona en móvil (limitación de File System Access API y del
  tamaño de la interfaz).


CÓMO SE USA

1. Abre el archivo .html en Chrome o Edge (escritorio).
2. Rellena el formulario y pulsa "Guardar". Cada guardado genera eventos.
3. Pulsa "Cargar ejemplo" en el panel de Importar y luego "Importar con
   inferencia" para ver cómo funciona la importación tolerante.
4. Selecciona un paciente en el panel de replay, pon una fecha anterior
   y pulsa "Ver estado" para ver cómo era antes.
5. Edita el teléfono de un paciente y pulsa "Deshacer". Verás que vuelve
   al valor anterior y en el log aparece un evento nuevo con actor
   "deshacer". Nada se ha borrado.
6. En el Query Builder, añade filtros y pulsa "Buscar".
7. Cambia "Agrupar por" a "demografia" para ver la lista agrupada.
8. Los tests se ejecutan al cargar y se ven en el panel inferior.


CÓMO FUNCIONA POR DENTRO

Arquitectura hexagonal, en capas. Cada capa habla con la siguiente
solo a través de contratos bien definidos. Nada de acoplamientos
ocultos.

CAPAS:

  NÚCLEO (lógica pura, sin DOM ni APIs del navegador)
    - EventBus: mediador entre capas. Todos le hablan a él, nadie
      se conoce directamente.
    - Logger: registro con niveles. Los logs se pueden exportar y
      limpiar desde fuera.
    - Ok / Err: contrato único de resultado. Todo lo que puede fallar
      por negocio devuelve esto, no lanza excepciones. Las excepciones
      quedan solo para bugs de programación.
    - ESQUEMA: definición declarativa de entidades y campos. Añadir
      un campo aquí lo hace reconocible por validación, inferencia,
      UI y query automáticamente.
    - EsquemaEngine: valida registros contra el esquema.
    - EventoFactory: crea eventos y calcula diffs entre versiones.
    - Vistas: proyecciones puras del log. Cada vista tiene
      init / apply / result. No mutan el log.
    - QueryEngine: filtros declarativos sobre registros.
    - Inferidor: normaliza entradas crudas y avisa de lo que infiere.
    - Diccionario: sinónimos y typos. Se puede reemplazar por un JSON
      externo sin tocar código.

  PUERTOS (contratos documentados)
    - PersistencePort: append, all, snapshots, clear, size.

  ADAPTADORES (implementaciones del puerto)
    - MemoryAdapter: para tests.
    - IndexedDBAdapter: para producción (PC y móvil, mismo código).

  CORE (orquestador)
    - Carga el log al inicio.
    - Aplica eventos a las vistas.
    - Guarda snapshots cada 100 eventos.
    - Reconstruye vistas desde cero si se pide (rebuild).
    - Reproduce estado pasado (estadoEn).
    - Devuelve historial completo de un registro (historialDe).

  SERVICIOS DE APLICACIÓN
    - PacienteService: crear, editar, archivar, deshacer, listar,
      obtener, agrupar por faceta, buscar por etiqueta.
    - JSONAdapter: importar y exportar.

  UI
    - Un adaptador de presentación que habla con el servicio y el core
      solo por eventos.


DECISIONES DE DISEÑO (y por qué)

1. EVENT SOURCING EN LUGAR DE BORRAR Y SOBRESCRIBIR

   Cada cambio es un evento inmutable que se añade al log. Nada se
   borra, nada se sobrescribe. El estado actual es el resultado de
   aplicar todos los eventos en orden.

   Ventajas:
   - Histórico y auditoría gratis.
   - Undo/redo gratis.
   - Puedes reconstruir cualquier estado pasado.
   - El log cuenta la verdad completa.

   Inconvenientes:
   - Ocupa más espacio que guardar solo el estado actual.
   - Requiere snapshots para no ralentizarse con muchos eventos.

2. SNAPSHOTS PERIÓDICOS

   Cada 100 eventos se guarda una foto del estado consolidado de cada
   vista. Al arrancar, se carga el snapshot más reciente y se aplican
   solo los eventos posteriores. Sin esto, reconstruir sería O(n) cada
   vez.

3. DESHACER SIN BORRAR

   "Deshacer" no elimina el evento anterior. Emite un evento nuevo que
   revierte el cambio. El log queda intacto y la auditoría también.

4. INFERENCIA CON NIVELES

   Cuando se importa un JSON de fuera, no siempre los campos coinciden.
   El Inferidor prueba en este orden:
     1. Campo canónico directo → seguro, sin aviso.
     2. Alias declarado en el esquema → seguro, aviso informativo.
     3. Typo en diccionario → probable, aviso.
     4. Distancia Levenshtein ≤ 2 → probable, aviso.
     5. Nada → hueco, aviso especulativo si era obligatorio.

   Cada aviso tiene nivel: seguro, probable o especulativo. El usuario
   puede revisar solo los dudosos.

5. ÍNDICE INVERSO COMO VISTA

   El índice "etiqueta → [ids]" no es una estructura especial. Es otra
   vista derivada del log. Se reconstruye igual que las demás. Cero
   código especial.

6. FACETAS DECLARADAS

   Una faceta es un grupo de columnas que tiene sentido filtrar juntas.
   Se declaran en el esquema. La UI las ofrece como opciones de
   agrupación. No hay que escribir código por cada faceta nueva.

7. CONTRATO ÚNICO DE ERROR

   Todo lo que puede fallar por negocio devuelve { ok: true, datos } o
   { ok: false, error, razon, detalles }. Los tests comprueban el
   contrato, no excepciones. Esto hace el código predecible.


ESTRUCTURA DEL ARCHIVO

Un solo .html. Dentro, en orden:

  1. Estilos (CSS)
  2. HTML (estructura de la interfaz)
  3. JavaScript (todo el código, en un módulo)

Dentro del JavaScript, secciones claramente separadas:

  - NÚCLEO
  - VISTAS
  - PUERTOS
  - ADAPTADORES
  - DICCIONARIO + INFERENCIA
  - QUERY ENGINE
  - CORE
  - SERVICIOS
  - IMPORT/EXPORT
  - TESTS
  - UI
  - ARRANQUE


EXTENDER EL ESQUEMA

Para añadir un campo a "paciente":

  1. Abre el archivo.
  2. Busca "const ESQUEMA".
  3. Añade el campo dentro de "paciente.campos":
       nuevoCampo: { tipo: 'text', required: false, indexed: true,
                     faceta: 'clinica', aliases: ['nuevo','nuev'] }
  4. Guarda y recarga.

Automáticamente:
  - La validación lo reconoce.
  - La inferencia lo busca en imports.
  - El Query Builder lo ofrece.
  - El formulario lo pide si añades el input en el HTML.

Para añadir una entidad nueva ("visita", "nota"):

  1. Añade la entidad en ESQUEMA.entidades con sus campos y sus
     eventos (creado, archivado).
  2. Añade su service correspondiente (o generaliza el actual).
  3. Añade su UI.

El Core, las vistas, el log y los adaptadores no cambian. Solo se
añade esquema y UI.


TESTS

Al cargar la página se ejecutan 22 tests que cubren:

  - Validación contra esquema.
  - Generación y diff de eventos.
  - Aplicación de eventos en vistas.
  - Índice inverso de etiquetas.
  - Distancia de Levenshtein.
  - Inferencia con alias, typos y claves desconocidas.
  - QueryEngine con todos los operadores.
  - Core: append, derive, rebuild, replay, historial.
  - Servicio: crear, editar, archivar, deshacer.
  - Facetas.
  - Import con inferencia.

Los resultados aparecen en el panel inferior. Si algún test falla,
algo se ha roto. No sigas sin arreglarlo.


LOGS

El Logger registra todo lo que pasa con niveles debug / info / warn /
error. Por defecto solo muestra info hacia arriba. Para ver todo:

  Logger.setNivel('debug')

Para exportar:

  Logger.exportText()

Para limpiar:

  Logger.clear()

Los logs se pueden desactivar por completo con:

  Logger.setNivel('error')

Pensado para que un programa externo los procese, filtre o elimine.


PERSISTENCIA

Los datos viven en IndexedDB del navegador, bajo la base "fase12_db".
Dos almacenes:
  - eventos: el log completo.
  - snapshots: las fotos periódicas.

Si abres el mismo HTML en otro navegador u otro PC, empieza vacío.
Para mover datos entre equipos, usa Exportar / Importar.

Si borras la caché del navegador, pierdes los datos. Por eso conviene
hacer backups periódicos con "Backup completo (JSON)".


LIMITACIONES CONOCIDAS

- No funciona en móvil por ahora.
- No cifra nada.
- No sincroniza entre dispositivos.
- No tiene autenticación.
- Los snapshots se guardan por número de eventos, no por tiempo.
- El undo solo deshace el último cambio, no varios seguidos.
- La UI es funcional, no bonita.
- No hay validación legal para datos clínicos reales. Esto es un
  proyecto personal, no un producto sanitario.


AVISO LEGAL

Si vas a guardar datos de personas reales, ten en cuenta que:

  - En España y la UE, los datos de salud son categoría especial
    según el RGPD.
  - Este proyecto no implementa cifrado, control de acceso, auditoría
    de accesos ni las demás medidas que la ley exige.
  - No lo subas a una URL pública con datos reales.
  - No lo uses como historia clínica oficial.

Para uso personal, con datos de prueba o con datos propios, sin
problema. Para datos de terceros, consulta antes.


ESTADO DEL PROYECTO

Versión actual: esqueleto funcional con Fase 1.5 y Fase 2 integradas.

Funciona:
  - Todo lo listado arriba en "¿QUÉ HACE HOY?".

Pendiente (fases futuras):
  - Cifrado de exportaciones.
  - Snapshot por tiempo.
  - Multi-entidad completa.
  - Import CSV.
  - Undo múltiple.
  - Modo estricto vs tolerante al importar.
  - Índices reales sobre columnas para búsquedas grandes.


FILOSOFÍA

- Un solo archivo, sin dependencias, sin servidor, sin propietario.
- El log es la fuente de verdad. Todo lo demás se deriva.
- Cada capa habla por contratos, no por acoplamiento.
- Si algo puede fallar, devuelve Result. No lanza.
- Si algo se infiere, se avisa. Nunca se inventa en silencio.
- Se itera pequeño. Primero funciona, luego se mejora.
- Nada se borra. Todo se compensa.


LICENCIA

agpl3. Uso personal. Si te sirve, bien. Si quieres
contribuir, mejor. Si quieres adaptarlo, hazlo. Pero no metas datos
de terceros sin cumplir la ley.


FIN
