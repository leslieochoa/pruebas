# CASO: API REST de Clientes en Spring Boot

## 1. Visión y Requisitos Técnicos de Arquitectura

- **Framework y Lenguaje:** Spring Boot 3.x (módulos: Spring Web, Spring Data JPA, Validation) con Java 25.
- **Persistencia Temporaria:** Base de datos H2 en memoria (`jdbc:h2:mem:clientesdb`), configurada con consola web activa para inspección de tablas durante pruebas locales.
- **Manejo Global de Excepciones:** Controlador `@RestControllerAdvice` para capturar errores de validación o entidades no encontradas y retornarlos en formato estandarizado (`ProblemDetail` / RFC 7807).

## 2. Historias de Usuario con Criterios de Aceptación (BDD)

### HU-01: Gestión CRUD de Clientes

Como Analista de Datos, quiero registrar, consultar, actualizar y eliminar clientes, para mantener actualizada la base de datos operativa.

- **Criterios de Aceptación (Formato BDD - Given / When / Then):**
  - **Escenario 1 (Creación exitosa):**
    - **Given** que envío una solicitud `POST /api/v1/clientes` con un payload válido (DNI único, Nombre, Email).
    - **When** la API procesa el registro.
    - **Then** responde con HTTP `201 Created`, incluye el header `Location` y el cliente generado con su ID.
  - **Escenario 2 (Validación de DNI duplicado):**
    - **Given** que ya existe un cliente registrado con el DNI `12345678`.
    - **When** intento registrar otro cliente con el mismo DNI `12345678`.
    - **Then** la API responde con HTTP `409 Conflict` indicando el fallo de unicidad.
  - **Escenario 3 (Actualizar un cliente inexistente):**
    - **Given** que no existe un cliente con el ID indicado.
    - **When** envío una solicitud `PUT /api/v1/clientes/{id}` para actualizarlo.
    - **Then** la API responde con HTTP `404 Not Found`.
  - **Escenario 4 (Email con formato inválido):**
    - **Given** que envío un cliente con un email que no tiene un formato válido.
    - **When** la API valida la solicitud.
    - **Then** responde con HTTP `400 Bad Request`.
  - **Escenario 5 (DNI con longitud inválida):**
    - **Given** que envío un cliente cuyo DNI no tiene exactamente 8 dígitos.
    - **When** la API valida la solicitud.
    - **Then** responde con HTTP `400 Bad Request`.
  - **Escenario 6 (Eliminar y luego consultar):**
    - **Given** que existe un cliente registrado.
    - **When** lo elimino mediante `DELETE /api/v1/clientes/{id}` y luego intento consultarlo.
    - **Then** la eliminación responde con HTTP `204 No Content` y la consulta posterior responde con HTTP `404 Not Found`.

### HU-02: Búsqueda y Filtrado por DNI o Nombre

Como Operador del Sistema, quiero realizar consultas por DNI o por coincidencia de Nombre, para ubicar rápidamente la información del cliente.

- **Criterios de Aceptación (Formato BDD):**
  - **Escenario 1 (Búsqueda exacta por DNI):**
    - **Given** que realizo una petición `GET /api/v1/clientes?dni=12345678`.
    - **When** la API ejecuta la consulta en H2.
    - **Then** retorna HTTP `200 OK` con los datos del cliente encontrado.
  - **Escenario 2 (Búsqueda parcial por Nombre):**
    - **Given** que realizo una petición `GET /api/v1/clientes?nombre=Carlos`.
    - **When** la API busca coincidencias parciales sin diferenciar mayúsculas/minúsculas (case-insensitive).
    - **Then** retorna HTTP `200 OK` con el listado de clientes coincidentes.

### HU-03: Manejo de Errores con ProblemDetail (RFC 7807)

Como consumidor de la API, quiero recibir errores con un formato uniforme, para comprender qué ocurrió cuando una solicitud falla.

- **Criterios de Aceptación (Formato BDD):**
  - **Escenario 1 (Error de validación):**
    - **Given** que envío una solicitud con datos inválidos.
    - **When** la API detecta el error de validación.
    - **Then** responde con HTTP `400 Bad Request` y un cuerpo `ProblemDetail` que describe el error.
  - **Escenario 2 (Cliente no encontrado):**
    - **Given** que consulto, actualizo o elimino un cliente que no existe.
    - **When** la API intenta encontrarlo.
    - **Then** responde con HTTP `404 Not Found` y un cuerpo `ProblemDetail` con información del error.
  - **Escenario 3 (DNI duplicado):**
    - **Given** que intento registrar un DNI que ya pertenece a otro cliente.
    - **When** la API detecta el conflicto de unicidad.
    - **Then** responde con HTTP `409 Conflict` y un cuerpo `ProblemDetail` que identifica el conflicto.

## 3. Especificación del Contrato de API (Endpoints REST)

| Método HTTP | Endpoint | Descripción | Parámetros / Body | Código HTTP esperado |
|---|---|---|---|---|
| POST | `/api/v1/clientes` | Crear cliente | Body: `ClienteRequestDTO` | `201 Created` / `400 Bad Request` / `409 Conflict` |
| GET | `/api/v1/clientes` | Listar / Buscar | Query opcional: `dni`, `nombre` | `200 OK` |
| GET | `/api/v1/clientes/{id}` | Obtener por ID | Path: `id` | `200 OK` / `404 Not Found` |
| PUT | `/api/v1/clientes/{id}` | Actualizar cliente | Path Param: `id`; Body: `ClienteRequestDTO` | `200 OK` / `400 Bad Request` / `404 Not Found` / `409 Conflict` |
| DELETE | `/api/v1/clientes/{id}` | Eliminar cliente | Path: `id` | `204 No Content` / `404 Not Found` |

### DTOs y reglas de validación

- `ClienteRequestDTO`: datos recibidos para crear o actualizar un cliente.
  - `dni`: obligatorio y compuesto exactamente por 8 dígitos.
  - `nombre`: obligatorio y no vacío.
  - `email`: obligatorio y con formato válido.
- `ClienteResponseDTO`: datos del cliente que devuelve la API, incluido su `id`, DNI, nombre y email.
- El DNI debe ser único. Si ya existe, la API responde con `409 Conflict`.

### Estructura de paquetes sugerida

```text
com.ejemplo.clientes
├── controller       # Endpoints REST
├── dto              # DTOs de entrada y salida
├── entity           # Entidad Cliente
├── repository       # Acceso a datos con Spring Data JPA
├── service          # Lógica de negocio
├── exception        # Excepciones y manejo global de errores
└── config           # Configuración, si es necesaria
```

## 4. Lista de Cotejo Específica (DoR y DoD) para esta Tarea

- **Definition of Ready (DoR):**
  - [ ] La Historia de Usuario cumple el principio INVEST (es independiente, pequeña y testeable).
  - [ ] El contrato de API (DTOs, rutas y respuestas HTTP) está pre-diseñado y acordado.
  - [ ] Las reglas de validación (DNI obligatorio de 8 dígitos, formato de email y nombre no vacío) están documentadas.
  - [ ] La estimación en Story Points fue asignada por el equipo técnico.

- **Definition of Done (DoD):**
  - [ ] Código funcional con la base de datos H2 en memoria configurada.
  - [ ] Suite de pruebas unitarias (JUnit 5 + Mockito) y de integración (`@SpringBootTest`) ejecutadas con éxito en el pipeline de CI/CD.
  - [ ] Pull Request (PR) atómico (<300 líneas) revisado y aprobado por un desarrollador senior.
  - [ ] Análisis estático libre de vulnerabilidades y con cobertura de código mínima aprobada.
  - [ ] Documentación interactiva habilitada en Swagger / OpenAPI.
