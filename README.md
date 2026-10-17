# Trabajo práctico 06

Laura Ivanna Toloza — Refactorización modular de reservas de salas.

## Proyecto de partida y cambios

Esta carpeta, `tp-06-salas-modular`, es una copia del TP 05 (`tp-05-salas-middleware`). La entrega anterior quedó intacta. No se copiaron `node_modules` ni `.git`.

**Línea de base (TP 05, antes de mover código):** todas las rutas respondían según el contrato, salvo un caso. Un `POST /reservas` **sin cuerpo** (o con un `Content-Type` que ningún parser interpreta) respondía **500**, por `TypeError: Cannot read properties of undefined (reading 'estudiante')` en `validarReserva`, que leía `req.body.estudiante` directamente. **Reparación previa:** el validador ahora usa `const valores = req.body ?? {}`. Con esa reparación el caso responde 400, como cualquier formulario inválido.

**Cambios de la refactorización:**

- El único `src/index.js` se dividió en arranque, configuración, aplicación, rutas, controlador, servicio y middleware (ver mapa).
- `PORT` y el formato de Morgan se leen desde el entorno, con valores predeterminados.
- `expressLayouts` se registraba antes de Morgan; ahora va después de `medirDuracion`, en el orden que pide la consigna. No cambia ninguna respuesta: solo se ejecuta al renderizar.
- Las listas de salas y turnos que el formulario necesita las pone en `res.locals` el middleware del área, en lugar de pasarlas en cada `render`.

El HTML de inicio, listado, formulario, detalle, 404 y del POST inválido es idéntico antes y después (comparado byte a byte, sin contar el número de solicitud).

## Instalación y ejecución

```bash
npm install
npm start            # puerto 3000, Morgan "dev"
npm run start:local  # lee un archivo .env local (requiere crearlo)
npm run format
npm run check
```

## Configuración del entorno

`src/configuracion.js` lee `PORT` y `NODE_ENV` de `process.env`:

- `PORT`: se convierte a número; si no existe se usa 3000; si no es un entero entre 1 y 65535 la aplicación no inicia y muestra `PORT debe ser un entero entre 1 y 65535`.
- `NODE_ENV=production` → Morgan `combined`; cualquier otro valor → `dev`. Solo cambia el formato del registro.

`.env.example` (se versiona, sin secretos):

```
PORT=3000
NODE_ENV=development
```

Para usarlo: copiarlo como `.env` y ejecutar `npm run start:local`. `.env` y `node_modules/` están en `.gitignore`. `npm start` funciona sin `.env`.

## Mapa de módulos y dependencias

| Archivo | Responsabilidad |
| --- | --- |
| `src/index.js` | Datos iniciales, lee configuración, crea el servicio y la app, escucha |
| `src/configuracion.js` | Convierte y valida `PORT` y `NODE_ENV` |
| `src/app.js` | Vistas, pipeline, `GET /`, `GET /estado`, monta `/reservas`, 404 final |
| `src/rutas/reservas.js` | Método + camino relativo → middleware y controlador |
| `src/controladores/reservas.js` | Lee parámetros, renderiza, JSON de `/estado`, 404 y redirección |
| `src/servicios/reservas.js` | Único arreglo de reservas: listar, buscar, contar, crear |
| `src/middleware/solicitudes.js` | ID `BIB-0001…` y medición hasta `finish` |
| `src/middleware/reservas.js` | Sección del área, listas del formulario y validación del POST |

```
index.js → configuracion.js
         → servicios/reservas.js   (crearServicioReservas(reservasIniciales))
         → app.js → middleware/solicitudes.js
                  → controladores/reservas.js (recibe el servicio)
                  → rutas/reservas.js → middleware/reservas.js
```

- **El servicio no usa `res`:** solo guarda y entrega datos. Decidir un estado HTTP o qué vista mostrar es trabajo del controlador; así el servicio no depende de Express.
- **El controlador** traduce una solicitud en una respuesta: convierte `req.params.id`, pide datos al servicio y elige `render`, `json`, `status(404)` o `redirect`.
- **El router declara caminos relativos** (`/`, `/nueva`, `/:id`) porque `app.js` lo monta en `/reservas`; Express une el prefijo con cada camino.
- **El único arreglo** vive dentro de `crearServicioReservas`, creado una vez en `index.js` y compartido por listado, detalle, `/estado` y creación. Al reiniciar se vuelve a crear con las 4 reservas iniciales.

## Pipeline y contrato de rutas

```
morgan → identificarSolicitud → medirDuracion → expressLayouts → express.static
       → express.urlencoded → express.json → GET / y GET /estado → router /reservas → 404
```

Morgan va antes de `express.static`, así que también registra el CSS y la imagen.

| Ruta | Respuesta |
| --- | --- |
| `GET /` | 200, inicio |
| `GET /estado` | 200, `{ servicio, reservas, solicitudId }` |
| `GET /reservas` | 200, listado |
| `GET /reservas/nueva` | 200, formulario (antes de `/:id`) |
| `GET /reservas/:id` | 200 detalle o 404 HTML |
| `POST /reservas` | 302 a `/reservas` si es válido; 400 con el formulario si no |
| `GET /css/estilos.css` | 200 |
| otra URL | 404 HTML |

## Matriz antes/después

Antes = TP 05; después = TP 06. Resultados observados ejecutando ambas versiones con las mismas solicitudes.

| Caso | Antes | Después | Evidencia |
| --- | --- | --- | --- |
| Inicio y CSS | 200 / 200 | 200 / 200 | Navegación, estilos y `Solicitud ID` en el pie |
| Estado inicial | 200 `reservas: 4` | 200 `reservas: 4` | `{"servicio":"activo","reservas":4,"solicitudId":"BIB-0002"}` |
| Listado | 200, 4 tarjetas | 200, 4 tarjetas | Mismo HTML |
| Detalle `/reservas/1` | 200 | 200 | "Reserva #1" |
| Detalle `/reservas/99` | 404 | 404 | "Reserva no encontrada" |
| Formulario | 200 | 200 | 3 salas y 3 turnos en los `select` |
| POST vacío | 400 | 400 | Mensaje con `role="alert"`; 4 → 4 |
| Sala no permitida | 400 | 400 | Valores conservados; 4 → 4 |
| Email sin @ | 400 | 400 | 4 → 4 |
| Personas 0, 7 y 2.5 | 400 | 400 | 4 → 4 cada uno |
| POST sin cuerpo | **500** (`TypeError`) | **400** | Reparación de la línea de base |
| POST válido | 302 → 200 | 302 → 200 | 5 tarjetas; `/reservas/5` 200 |
| URL inexistente | 404 | 404 | "Página no encontrada" |
| Reinicio | 4 reservas | 4 reservas | `/estado` vuelve a 4 y `BIB-0001` |
| Puerto alternativo | — | 200 | `PORT=4000` → "Aplicación disponible en http://localhost:4000" |
| Puerto inválido | — | No inicia | `PORT=abc`, `70000`, `0`, `3.5` → mensaje y código 1 |
| Puerto ocupado | — | No inicia | `No se pudo escuchar el puerto: EADDRINUSE` |

Morgan y la medición propia muestran 200, 302, 400 y 404 en ambas versiones, por ejemplo:

```
GET /reservas 200 2.331 ms - 3682
[BIB-0003] GET /reservas 200 4.32 ms
```

Con `NODE_ENV=production`, Morgan usa el formato `combined`:

```
127.0.0.1 - - [16/Oct/2026:01:15:11 +0000] "GET /reservas HTTP/1.1" 200 3682 "-" "curl/8.5.0"
```

## Formato y análisis estático

Dependencias de desarrollo: `prettier`, `eslint` y `@eslint/js`. `eslint.config.js` aplica las reglas recomendadas a `src/**/*.js` como CommonJS.

```bash
npm run format   # Prettier reescribe src
npm run check    # Prettier --check + ESLint: sin errores
```

## Persistencia temporal y límites

Las reservas viven en un arreglo en memoria dentro del servicio. Las altas no se escriben en ningún archivo: al reiniciar el servidor se pierden y vuelven las 4 reservas iniciales de `src/index.js`. No hay base de datos, edición, borrado ni autenticación. El email solo se valida con `@` y la fecha no se controla.
