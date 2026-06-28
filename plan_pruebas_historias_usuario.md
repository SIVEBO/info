# Plan de Pruebas — Historias de Usuario SIVEBO

## Contexto

SIVEBO es un sistema de logística basado en microservicios (Spring Boot 4 / Java 25). El entorno de prueba levanta los 12 servicios del proyecto mediante Docker Compose y verifica la interacción real entre microservicios usando las historias de usuario definidas en `info/README.md`.

**Stack:**
- 1 base de datos MariaDB 11 (compartida, bases separadas por microservicio)
- 1 Eureka Server (registro y descubrimiento)
- 1 API Gateway (enrutamiento via `lb://MS-*`)
- 10 microservicios de negocio

**Punto de entrada:** `http://localhost:8080` (Gateway)  
**Eureka Dashboard:** `http://localhost:8761`

---

## Prerequisitos Postman

Cada microservicio incluye su propia colección Postman en la raíz de su directorio:

| Colección | Archivo |
|-----------|---------|
| ms_auth | `ms_auth/ms_auth.postman_collection.json` |
| ms_sucursales | `ms_sucursales/ms_sucursales.postman_collection.json` |
| ms_clientes | `ms_clientes/ms_clientes.postman_collection.json` |
| ms_admision | `ms_admision/ms_admision.postman_collection.json` |
| ms_tracking | `ms_tracking/ms_tracking.postman_collection.json` |
| ms_paquetes | `ms_paquetes/ms_paquetes.postman_collection.json` |
| ms_embalaje | `ms_embalaje/ms_embalaje.postman_collection.json` |
| ms_ventas | `ms_ventas/ms_ventas.postman_collection.json` |
| ms_finanzas | `ms_finanzas/ms_finanzas.postman_collection.json` |
| ms_portal | `ms_portal/ms_portal.postman_collection.json` |

**Importar en Postman:** File → Import → seleccionar cada `.json` individualmente.

**Variables de entorno necesarias** (todas se extraen automáticamente por los scripts internos de cada colección al ejecutar los requests POST en orden):

| Variable | Quién la genera | Cómo |
|----------|----------------|------|
| `{{token}}` | `ms_auth` → POST login | Script: `pm.collectionVariables.set("token", ...)` |
| `{{idSucursal}}` | `ms_sucursales` → POST sucursales | Script: guarda el `id` de la respuesta |
| `{{idCliente}}` | `ms_clientes` → POST clientes | Script: guarda el `id` de la respuesta |
| `{{idAdmision}}` | `ms_admision` → POST admisiones | Script: guarda el `id` de la respuesta |
| `{{idGuia}}` | `ms_tracking` → POST guias | Script: guarda el `id` de la respuesta |
| `{{idInventario}}` | `ms_paquetes` → POST inventario/ingreso | Script: guarda el `id` de la respuesta |
| `{{idCategoria}}` / `{{idArticulo}}` / `{{idStock}}` | `ms_embalaje` → POST respectivos | Script: guarda el `id` de cada respuesta |
| `{{idVenta}}` | `ms_ventas` → POST ventas | Script: guarda el `id` de la respuesta |
| `{{idCaja}}` / `{{idSesion}}` | `ms_finanzas` → POST cajas / aperturas | Script: guarda el `id` de cada respuesta |
| `{{idConsulta}}` / `{{idFeedback}}` | `ms_portal` → POST consultas / feedbacks | Script: guarda el `id` de cada respuesta |

> **Nota de autenticación:** Los endpoints protegidos de `ms_auth` usan `Authorization: Bearer {{token}}`. El resto de los microservicios no validan JWT en sus propios requests (la seguridad perimetral recae en el gateway y en `ms_auth`).

---

## Paso 1 — Levantar el Stack con Docker Compose

```bash
cd /home/daemunozr/github/dae.munozr@duocuc.cl/SIVEBO
docker compose up --build -d
```

Se construyeron las 12 imágenes en paralelo usando builds multi-etapa (Maven 3.9 + Eclipse Temurin 25). El archivo `docker-compose.override.yml` configura `network: host` en el contexto de build para que Maven pueda descargar dependencias desde internet en WSL2.

**Esperar a que Eureka esté disponible:**

```bash
until curl -s http://localhost:8761/actuator/health | grep -q '"status":"UP"'; do sleep 5; done
```

**Esperar a que los 10 microservicios se registren:**

```bash
until [ "$(curl -s http://localhost:8761/eureka/apps | grep -c '<application>')" -ge 10 ]; do sleep 5; done
```

**Resultado:** 13 contenedores corriendo (`mariadb`, `eureka-server`, `gateway` + 10 ms_*). Todos los microservicios registrados en Eureka.

---

## Paso 2 — Datos de Referencia (Seed)

Los servicios usan `ddl-auto: update`, por lo que Hibernate crea las tablas automáticamente. El usuario `admin` es creado por el `DataInitializer` de `ms_auth` al iniciar. Sin embargo, las tablas de catálogo (regiones, comunas, tipos de carga) no tienen endpoints de escritura y deben poblarse directamente en la BD.

### 2.1 Login — US-01

```bash
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}' \
  | jq -r '.token')
```

- **Servicio:** `ms_auth` (puerto 8081, enrutado por gateway)
- **Resultado:** Token JWT emitido con rol `Admin`, expiración 1 hora
- **Historia:** US-01 ✓

> **Nota:** Al primer intento el gateway devolvió 503 porque su caché de Eureka aún no tenía instancias registradas (intervalo de refresco: 30 s). El segundo intento, tras el refresco automático, fue exitoso.

#### Colección Postman: `ms_auth/ms_auth.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| Autenticación | Iniciar sesión | POST | `/api/v1/auth/login` | Extrae `{{token}}` automáticamente |
| Autenticación | Registrar usuario | POST | `/api/v1/auth/register` | Requiere `Authorization: Bearer {{token}}` |
| Usuarios | Listar usuarios | GET | `/api/v1/usuarios` | Rol ADMIN |
| Usuarios | Obtener por id | GET | `/api/v1/usuarios/{{idUsuario}}` | Público |
| Roles | Crear rol | POST | `/api/v1/roles` | Extrae `{{idRol}}`; rol ADMIN |
| Roles | Listar roles | GET | `/api/v1/roles` | Rol ADMIN |
| Roles | Obtener por id | GET | `/api/v1/roles/{{idRol}}` | Rol ADMIN |
| Roles | Actualizar rol | PUT | `/api/v1/roles/{{idRol}}` | Rol ADMIN |
| Roles | Eliminar rol | DELETE | `/api/v1/roles/{{idRol}}` | Rol ADMIN |
| Sesión (ejecutar último) | Cerrar sesión | POST | `/api/v1/auth/logout` | Invalida el token activo |

### 2.2 Seed Regiones y Comunas (vía MariaDB)

Los endpoints `/api/v1/regiones` y `/api/v1/comunas` son de solo lectura. Se insertan directamente en la base de datos:

```bash
docker exec sivebo-mariadb-1 mariadb -uroot -psivebo -e "
USE db_ms_sucursales;
INSERT IGNORE INTO regiones (nombre_region) VALUES
  ('Región Metropolitana'), ('Región de Valparaíso'), ('Región del Biobío');
INSERT IGNORE INTO comunas (nombre_comuna, id_region) VALUES
  ('Santiago', 1), ('Providencia', 1), ('Maipú', 1),
  ('Viña del Mar', 2), ('Concepción', 3);
"
```

### 2.3 Crear Sucursales — US-16

```bash
curl -s -X POST http://localhost:8080/api/v1/sucursales \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Sucursal Centro","nombreComuna":"Santiago",
       "direccionFisica":"Av. Libertador 100","telefonoContacto":"222001001"}'

curl -s -X POST http://localhost:8080/api/v1/sucursales \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Sucursal Norte","nombreComuna":"Providencia",
       "direccionFisica":"Av. Providencia 500","telefonoContacto":"222002002"}'
```

- **Servicio:** `ms_sucursales` (puerto 8082)
- **Resultado:** Sucursal Centro (id=3), Sucursal Norte (id=4), estado ACTIVA
- **Historia:** US-16 ✓

#### Colección Postman: `ms_sucursales/ms_sucursales.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| SucursalController | Crear sucursal | POST | `/api/v1/sucursales` | Extrae `{{idSucursal}}` |
| SucursalController | Listar sucursales | GET | `/api/v1/sucursales` | |
| SucursalController | Buscar por comuna | GET | `/api/v1/sucursales/buscar?comuna=Santiago` | |
| SucursalController | Actualizar sucursal | PUT | `/api/v1/sucursales/{{idSucursal}}` | |
| SucursalController | Desactivar sucursal | PATCH | `/api/v1/sucursales/{{idSucursal}}/desactivar` | |
| SucursalController | Activar sucursal | PATCH | `/api/v1/sucursales/{{idSucursal}}/activar` | |
| RegionController | Listar regiones | GET | `/api/v1/regiones` | Solo lectura |
| RegionController | Buscar región | GET | `/api/v1/regiones/buscar?nombre=Metropolitana` | |
| ComunaController | Listar comunas | GET | `/api/v1/comunas` | Solo lectura |
| ComunaController | Buscar comuna | GET | `/api/v1/comunas/buscar?nombre=Santiago` | |

### 2.4 Seed Tipos de Carga (vía MariaDB)

El microservicio `ms_admision` no expone endpoint POST para tipos de carga:

```bash
docker exec sivebo-mariadb-1 mariadb -uroot -psivebo -e "
USE db_ms_admision;
INSERT IGNORE INTO tipos_carga (nombre_tipo, descripcion) VALUES
  ('NORMAL','Carga estándar'),
  ('FRAGIL','Requiere manejo cuidadoso'),
  ('SOBREDIMENSIONADA','Carga de gran tamaño');
"
```

### 2.5 Seed Catálogo de Embalaje — US-17

```bash
curl -s -X POST http://localhost:8080/api/v1/categorias \
  -H "Content-Type: application/json" \
  -d '{"nombreCategoria":"Cajas"}'

curl -s -X POST http://localhost:8080/api/v1/articulos \
  -H "Content-Type: application/json" \
  -d '{"idCat":1,"nombre":"Caja pequeña","descripcion":"Caja 30x30x30 cm","precioVta":1500}'

curl -s -X POST http://localhost:8080/api/v1/stock \
  -H "Content-Type: application/json" \
  -d '{"idArt":1,"idSucursal":3,"cantidadDisponible":50}'
```

- **Servicio:** `ms_embalaje` (puerto 8086)
- **Resultado:** Categoría "Cajas" (id=1), Artículo "Caja pequeña" (id=1), Stock=50 en Sucursal Centro
- **Historia:** US-17 ✓

#### Colección Postman: `ms_embalaje/ms_embalaje.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| CategoriaEmbalajeController | Crear categoría | POST | `/api/v1/categorias` | Extrae `{{idCategoria}}` |
| CategoriaEmbalajeController | Listar categorías | GET | `/api/v1/categorias` | |
| CategoriaEmbalajeController | Obtener por id | GET | `/api/v1/categorias/{{idCategoria}}` | |
| CategoriaEmbalajeController | Actualizar categoría | PUT | `/api/v1/categorias/{{idCategoria}}` | |
| ArticuloEmbalajeController | Crear artículo | POST | `/api/v1/articulos` | Extrae `{{idArticulo}}` |
| ArticuloEmbalajeController | Listar artículos | GET | `/api/v1/articulos` | |
| ArticuloEmbalajeController | Obtener por id | GET | `/api/v1/articulos/{{idArticulo}}` | |
| ArticuloEmbalajeController | Por categoría | GET | `/api/v1/articulos/categoria/{{idCategoria}}` | |
| ArticuloEmbalajeController | Actualizar artículo | PUT | `/api/v1/articulos/{{idArticulo}}` | |
| StockSucursalController | Crear stock | POST | `/api/v1/stock` | Extrae `{{idStock}}` |
| StockSucursalController | Obtener por id | GET | `/api/v1/stock/{{idStock}}` | |
| StockSucursalController | Por sucursal | GET | `/api/v1/stock/sucursal/{{idSucursal}}` | US-07 |
| StockSucursalController | Actualizar stock | PATCH | `/api/v1/stock/actualizar?idArt=...&idSucursal=...&cantidad=50` | |
| StockSucursalController | Descontar stock | PATCH | `/api/v1/stock/descontar?idArt=...&idSucursal=...&cantidad=1` | |
| StockSucursalController | Verificar stock | GET | `/api/v1/stock/verificar?idArt=...&idSucursal=...&cantidadRequerida=1` | |
| Limpiar (ejecutar último) | Desactivar artículo | PATCH | `/api/v1/articulos/{{idArticulo}}/desactivar` | Destructivo |
| Limpiar (ejecutar último) | Borrar categoría | DELETE | `/api/v1/categorias/{{idCategoriaTmp}}` | Destructivo |

### 2.6 Crear Usuario Operador — US-15

```bash
curl -s -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"username":"operador1","password":"op1234","email":"op1@sivebo.cl",
       "nombreRol":"Operador","idSucursalAsignada":3}'
```

- **Servicio:** `ms_auth` (endpoint protegido con rol ADMIN)
- **Resultado:** `operador1` (id=2), activo, asignado a Sucursal Centro
- **Historia:** US-15 ✓

---

## Paso 3 — Flujos Principales de Historias de Usuario

### 3.1 Buscar Cliente — US-02

```bash
curl -s "http://localhost:8080/api/v1/clientes/buscar?tipoDocumento=RUT&nroDocumento=12345678-9"
```

- **Servicio:** `ms_clientes` (puerto 8089)
- **Resultado:** Vacío (cliente no existe aún), comportamiento esperado
- **Historia:** US-02 ✓

### 3.2 Registrar Clientes — US-03

```bash
curl -s -X POST http://localhost:8080/api/v1/clientes \
  -H "Content-Type: application/json" \
  -d '{"codigoTipoDocumento":"RUT","nroDocumento":"12345678-9",
       "nombre":"Juan","apellido":"Pérez","email":"juan@sivebo.cl","telefono":"912345678"}'

curl -s -X POST http://localhost:8080/api/v1/clientes \
  -H "Content-Type: application/json" \
  -d '{"codigoTipoDocumento":"RUT","nroDocumento":"98765432-1",
       "nombre":"María","apellido":"García","email":"maria@sivebo.cl","telefono":"987654321"}'
```

- **Servicio:** `ms_clientes` (puerto 8089)
- **Resultado:** Juan Pérez (id=1), María García (id=2)
- **Historia:** US-03 ✓

#### Colección Postman: `ms_clientes/ms_clientes.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| TipoDocumentoController | Listar tipos de documento | GET | `/api/v1/tipos-documento` | Catálogo: RUT, PASAPORTE, DNI |
| TipoDocumentoController | Obtener por código | GET | `/api/v1/tipos-documento/{{codigoTipoDocumento}}` | |
| ClienteController | Crear cliente | POST | `/api/v1/clientes` | Extrae `{{idCliente}}`; US-03 |
| ClienteController | Buscar por documento | GET | `/api/v1/clientes/buscar?tipoDocumento=RUT&nroDocumento=11111111-1` | US-02 |
| ClienteController | Obtener por id | GET | `/api/v1/clientes/{{idCliente}}` | |
| ClienteController | Listar (paginado) | GET | `/api/v1/clientes?filtro=&page=0&size=10` | |
| ClienteController | Actualizar contacto | PUT | `/api/v1/clientes/{{idCliente}}/contacto` | Actualiza email y teléfono |

### 3.3 Registrar Admisión + Guía Automática — US-04 / US-05

Esta es la prueba de integración más compleja: `ms_admision` hace llamadas WebClient a otros microservicios antes de persistir y crear la guía en `ms_tracking`.

```bash
curl -s -X POST http://localhost:8080/api/v1/admisiones \
  -H "Content-Type: application/json" \
  -d '{
    "idClienteRem": 1,
    "idClienteDest": 2,
    "idSucursalOrigen": 3,
    "idSucursalDest": 4,
    "nombreTipoCarga": "NORMAL",
    "pesoKg": 3.5,
    "idUsuarioReg": 1
  }'
```

**Cadena de llamadas inter-microservicio:**

```
ms_admision
  ├── GET ms_clientes/api/v1/clientes/1        (valida remitente)
  ├── GET ms_clientes/api/v1/clientes/2        (valida destinatario)
  ├── GET ms_sucursales/api/v1/sucursales/buscar?id=3  (valida origen)
  ├── GET ms_sucursales/api/v1/sucursales/buscar?id=4  (valida destino)
  ├── GET ms_auth/api/v1/usuarios/1            (valida usuario)
  └── POST ms_tracking/api/v1/guias            (crea guía con tracking code)
```

- **Resultado:** Admisión id=1, código de tracking `C32627D89760` generado en `ms_tracking`
- **Historias:** US-04 ✓ / US-05 ✓

```bash
curl -s http://localhost:8080/api/v1/guias
# → [{"id":1,"codigoTracking":"C32627D89760","idAdmision":1,...}]
```

#### Colección Postman: `ms_admision/ms_admision.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| Admisiones | Crear admisión | POST | `/api/v1/admisiones` | Extrae `{{idAdmision}}`; dispara cadena inter-servicio |
| Admisiones | Obtener por id | GET | `/api/v1/admisiones/{{idAdmision}}` | |
| Admisiones | Listar por sucursal | GET | `/api/v1/admisiones?idSucursal={{idSucursal}}` | Filtra por sucursal de origen |

### 3.4 Actualizar Estado de Seguimiento — US-06

```bash
curl -s -X POST http://localhost:8080/api/v1/historial \
  -H "Content-Type: application/json" \
  -d '{"idGuia":1,"idEstado":2,"idSucursalActual":4,"idUsuario":2,
       "fechaHora":"2026-06-24T10:00:00","comentario":"Paquete en ruta hacia Sucursal Norte"}'

curl -s -X POST http://localhost:8080/api/v1/historial \
  -H "Content-Type: application/json" \
  -d '{"idGuia":1,"idEstado":3,"idSucursalActual":4,"idUsuario":2,
       "fechaHora":"2026-06-24T14:30:00","comentario":"Paquete entregado al destinatario"}'
```

- **Servicio:** `ms_tracking` (puerto 8084)
- **Estados maestros:** 1=RECIBIDO, 2=EN_TRANSITO, 3=ENTREGADO, 4=DEVUELTO
- **Resultado:** Historial logístico con progresión RECIBIDO → EN_TRANSITO → ENTREGADO
- **Historia:** US-06 ✓

#### Colección Postman: `ms_tracking/ms_tracking.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| GuiaDespachoController | Crear guía | POST | `/api/v1/guias` | Extrae `{{idGuia}}`; normalmente lo llama ms_admision |
| GuiaDespachoController | Listar guías | GET | `/api/v1/guias` | |
| GuiaDespachoController | Obtener por id | GET | `/api/v1/guias/{{idGuia}}` | |
| GuiaDespachoController | Buscar por código | GET | `/api/v1/guias/buscar?codigoTracking={{codigoTracking}}` | |
| HistorialLogisticoController | Crear historial | POST | `/api/v1/historial` | Extrae `{{idHistorial}}`; US-06 |
| HistorialLogisticoController | Listar historial | GET | `/api/v1/historial` | |
| HistorialLogisticoController | Historial por guía | GET | `/api/v1/historial/{{idGuia}}` | US-22 |
| HistorialLogisticoController | Estado actual | GET | `/api/v1/historial/estado-actual/{{idGuia}}` | Estado más reciente |
| Limpiar (ejecutar último) | Borrar historial | DELETE | `/api/v1/historial/{{idHistorial}}` | Destructivo |
| Limpiar (ejecutar último) | Borrar guía temporal | DELETE | `/api/v1/guias/{{idGuiaTmp}}` | Destructivo |

### 3.5 Consultar Stock de Embalaje — US-07

```bash
curl -s http://localhost:8080/api/v1/stock/sucursal/3
# → [{"idStock":1,"nombreArticulo":"Caja pequeña","cantidadDisponible":50,...}]
```

- **Servicio:** `ms_embalaje` (puerto 8086)
- **Historia:** US-07 ✓

*(Ver tabla Postman en sección 2.5)*

### 3.6 Abrir Caja — US-13

```bash
curl -s -X POST http://localhost:8080/api/v1/cajas \
  -H "Content-Type: application/json" \
  -d '{"idSucursal":3,"estadoActual":"CERRADA"}'
# → {"idCaja":1,...}

curl -s -X POST http://localhost:8080/api/v1/aperturas/abrir \
  -H "Content-Type: application/json" \
  -d '{"idCaja":1,"idUsuario":2,"montoApertura":50000}'
# → {"idSesion":1,"montoApertura":50000,...}
```

- **Servicio:** `ms_finanzas` (puerto 8088)
- **Resultado:** Sesión de caja #1 abierta con $50.000 de fondo inicial
- **Historia:** US-13 ✓

#### Colección Postman: `ms_finanzas/ms_finanzas.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| CajaSucursalController | Crear caja | POST | `/api/v1/cajas` | Extrae `{{idCaja}}` |
| CajaSucursalController | Listar cajas | GET | `/api/v1/cajas` | |
| CajaSucursalController | Obtener por id | GET | `/api/v1/cajas/{{idCaja}}` | |
| CajaSucursalController | Obtener por sucursal | GET | `/api/v1/cajas/sucursal/{{idSucursal}}` | |
| AperturaCierreController | Abrir caja | POST | `/api/v1/aperturas/abrir` | Extrae `{{idSesion}}`; US-13 |
| AperturaCierreController | Obtener sesión | GET | `/api/v1/aperturas/{{idSesion}}` | |
| AperturaCierreController | Sesiones por caja | GET | `/api/v1/aperturas/caja/{{idCaja}}` | |
| AperturaCierreController | Sesión abierta | GET | `/api/v1/aperturas/caja/{{idCaja}}/abierta` | |
| MovimientoCajaController | Crear movimiento | POST | `/api/v1/movimientos` | Extrae `{{idMovimiento}}`; US-20 |
| MovimientoCajaController | Por sesión | GET | `/api/v1/movimientos/sesion/{{idSesion}}` | |
| MovimientoCajaController | Por sesión + tipo | GET | `/api/v1/movimientos/sesion/{{idSesion}}/tipo?tipo=INGRESO` | |
| Cierre y estado (ejecutar último) | Cerrar caja | PATCH | `/api/v1/aperturas/{{idSesion}}/cerrar?montoCierre=75000` | US-14 |
| Cierre y estado (ejecutar último) | Reporte cierre | GET | `/api/v1/aperturas/{{idSesion}}/reporte-cierre` | US-20 |
| Cierre y estado (ejecutar último) | Cambiar estado caja | PATCH | `/api/v1/cajas/{{idCaja}}/estado?nuevoEstado=ABIERTA` | |

### 3.7 Registrar Venta con IVA Automático — US-08 / US-09

```bash
curl -s -X POST http://localhost:8080/api/v1/ventas \
  -H "Content-Type: application/json" \
  -d '{
    "idUsuario": 2,
    "idSucursal": 3,
    "fechaVta": "2026-06-24T10:30:00",
    "detalles": [
      {"idArticulo":1,"cantidadArt":2,"precioUnitHistoricoArt":1500}
    ]
  }'
```

- **Servicio:** `ms_ventas` (puerto 8087)
- **Resultado:**
  ```json
  {"nroBoleta":1,"subtotal":3000,"iva":570,"total":3570,"estado":"ACTIVA"}
  ```
- **IVA calculado automáticamente:** 3000 × 19% = 570 → total 3.570
- **Historias:** US-08 ✓ / US-09 ✓

### 3.8 Bloqueo por Stock Insuficiente — US-10

```bash
curl -s -X POST http://localhost:8080/api/v1/ventas \
  -H "Content-Type: application/json" \
  -d '{
    "idUsuario":2,"idSucursal":3,"fechaVta":"2026-06-24T11:00:00",
    "detalles":[{"idArticulo":1,"cantidadArt":9999,"precioUnitHistoricoArt":1500}]
  }'
# → {"error":"Stock insuficiente para artículo 1 en sucursal 3"}
```

- **Resultado:** Venta rechazada correctamente
- **Historia:** US-10 ✓

#### Colección Postman: `ms_ventas/ms_ventas.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| VentaController | Crear venta | POST | `/api/v1/ventas` | Extrae `{{idVenta}}`; US-08/09/10 |
| VentaController | Listar ventas | GET | `/api/v1/ventas` | |
| VentaController | Obtener por id | GET | `/api/v1/ventas/{{idVenta}}` | |
| VentaController | Buscar por boleta | GET | `/api/v1/ventas/buscar?nroBoleta=1` | |
| VentaController | Por sucursal | GET | `/api/v1/ventas/sucursal?id_sucursal={{idSucursal}}` | US-19 |
| DetalleVentaController | Listar detalles | GET | `/api/v1/detalles` | |
| DetalleVentaController | Obtener por id | GET | `/api/v1/detalles/{{idDetalle}}` | |
| DetalleVentaController | Por venta | GET | `/api/v1/detalles/venta/{{idVenta}}` | |
| Limpiar (ejecutar último) | Anular venta | PUT | `/api/v1/ventas/anular/{{idVenta}}?id_usuario={{idUsuario}}` | US-18; rol ADMIN |
| Limpiar (ejecutar último) | Borrar detalle | DELETE | `/api/v1/detalles/{{idDetalle}}` | Destructivo |
| Limpiar (ejecutar último) | Borrar venta | DELETE | `/api/v1/ventas/{{idVenta}}` | Destructivo |

### 3.9 Consulta Pública de Tracking — US-21 / US-22

El portal registra la consulta y llama internamente a `ms_tracking` para obtener el estado actual.

```bash
curl -s -X POST http://localhost:8080/api/v1/consultas \
  -H "Content-Type: application/json" \
  -d '{"codigoTrackingConsultado":"C32627D89760","ipUsuario":"127.0.0.1",
       "fechaHora":"2026-06-24T15:00:00"}'
# → {"codigoTrackingGuia":"C32627D89760","estadoActual":"ENTREGADO",...}

curl -s "http://localhost:8080/api/v1/consultas/buscar?codigoTracking=C32627D89760"
# → historial completo de consultas por ese código
```

- **Servicio:** `ms_portal` (puerto 8090) → `ms_tracking`
- **Historias:** US-21 ✓ / US-22 ✓

### 3.10 Calificación Anónima del Servicio — US-23 / US-24

```bash
curl -s -X POST http://localhost:8080/api/v1/feedbacks \
  -H "Content-Type: application/json" \
  -d '{"idGuiaTracking":1,"calificacion":5,
       "comentario":"Excelente servicio, paquete llegó en perfectas condiciones.",
       "fechaHora":"2026-06-24T16:00:00"}'

curl -s -X POST http://localhost:8080/api/v1/feedbacks \
  -H "Content-Type: application/json" \
  -d '{"idGuiaTracking":1,"calificacion":4,
       "comentario":"Buen servicio pero tardó un poco.",
       "fechaHora":"2026-06-24T16:05:00"}'
```

- **Servicio:** `ms_portal` (puerto 8090)
- **Resultado:** `"nroDocumentoCliente": null` confirma envío anónimo
- **Historias:** US-23 ✓ / US-24 ✓

#### Colección Postman: `ms_portal/ms_portal.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| ConsultaController | Crear consulta | POST | `/api/v1/consultas` | Extrae `{{idConsulta}}`; llama ms_tracking internamente; US-21 |
| ConsultaController | Listar consultas | GET | `/api/v1/consultas` | |
| ConsultaController | Obtener por id | GET | `/api/v1/consultas/{{idConsulta}}` | |
| ConsultaController | Consultas públicas | GET | `/api/v1/consultas/publica` | Vista pública sin datos sensibles |
| ConsultaController | Buscar por código | GET | `/api/v1/consultas/buscar?codigoTracking={{codigoTracking}}` | US-22 |
| ConsultaController | Borrar consulta | DELETE | `/api/v1/consultas/{{idConsulta}}` | Destructivo |
| FeedbackController | Crear feedback | POST | `/api/v1/feedbacks` | Extrae `{{idFeedback}}`; US-23/24 |
| FeedbackController | Listar feedbacks | GET | `/api/v1/feedbacks` | |
| FeedbackController | Obtener por id | GET | `/api/v1/feedbacks/{{idFeedback}}` | |
| FeedbackController | Buscar por calificación | GET | `/api/v1/feedbacks/buscar?calificacion=5` | |
| FeedbackController | Borrar feedback | DELETE | `/api/v1/feedbacks/{{idFeedback}}` | Destructivo |

### 3.11 Cierre de Caja — US-14

```bash
curl -s -X PATCH "http://localhost:8080/api/v1/aperturas/1/cerrar?montoCierre=53570"
# → {"montoApertura":50000,"montoCierre":53570,"saldoCalculado":53570,"diferenciaCuadre":0.00}
```

- **Resultado:** Cuadre exacto (`diferenciaCuadre: 0.00`): fondo inicial $50.000 + venta $3.570 = $53.570
- **Historia:** US-14 ✓

*(Ver tabla Postman en sección 3.6)*

---

## Paso 4 — Verificación Final

```bash
# Servicios registrados en Eureka
curl -s http://localhost:8761/eureka/apps | grep -oP '(?<=<name>)[^<]+' | sort

# Estado de todos los contenedores
docker compose ps
```

**Resultado:** 11 instancias registradas (GATEWAY + 10 microservicios). 13 contenedores en estado `Up`.

---

## Paso 5 — Endpoints Adicionales cubiertos solo por Postman

Las siguientes historias no fueron ejercidas en las pruebas curl pero sus endpoints están completamente definidos en las colecciones Postman.

### US-11 / US-12 — Inventario Físico de Paquetes en Bodega

**Colección:** `ms_paquetes/ms_paquetes.postman_collection.json`

| Carpeta | Request | Método | Endpoint | Notas |
|---------|---------|--------|----------|-------|
| InventarioPaqueteController | Ingreso a bodega | POST | `/api/v1/inventario/ingreso` | Extrae `{{idInventario}}`; US-11 |
| InventarioPaqueteController | Salida de bodega | PATCH | `/api/v1/inventario/{{idInventario}}/salida` | US-12 |
| InventarioPaqueteController | Obtener por id | GET | `/api/v1/inventario/{{idInventario}}` | |
| InventarioPaqueteController | Por guía | GET | `/api/v1/inventario/guia/{{idGuia}}` | |
| InventarioPaqueteController | En bodega | GET | `/api/v1/inventario/en-bodega` | Lista todos los paquetes en stock |
| InventarioPaqueteController | Por sucursal | GET | `/api/v1/inventario/sucursal/{{idSucursal}}` | |

**Flujo Postman para US-11/12:**
1. Ejecutar primero US-04/05 para obtener `{{idGuia}}`
2. POST `/api/v1/inventario/ingreso` con `{"idGuia": {{idGuia}}, "idSucursal": {{idSucursal}}, "fechaIngreso": "..."}`
3. PATCH `/api/v1/inventario/{{idInventario}}/salida` al despachar

### US-18 / US-19 — Anulación y Consulta de Ventas (Admin)

**Colección:** `ms_ventas/ms_ventas.postman_collection.json` → carpeta **Limpiar**

| Request | Método | Endpoint | Historia |
|---------|--------|----------|---------|
| Anular venta | PUT | `/api/v1/ventas/anular/{{idVenta}}?id_usuario={{idUsuario}}` | US-18 |
| Ventas por sucursal | GET | `/api/v1/ventas/sucursal?id_sucursal={{idSucursal}}` | US-19 |

### US-20 — Reporte de Cierre de Caja (Admin)

**Colección:** `ms_finanzas/ms_finanzas.postman_collection.json` → carpeta **Cierre y estado**

| Request | Método | Endpoint | Historia |
|---------|--------|----------|---------|
| Crear movimiento | POST | `/api/v1/movimientos` | Registra ingreso/egreso en sesión |
| Movimientos por sesión | GET | `/api/v1/movimientos/sesion/{{idSesion}}` | Detalle de caja |
| Reporte cierre | GET | `/api/v1/aperturas/{{idSesion}}/reporte-cierre` | US-20: ingresos, egresos, diferencia |

---

## Resumen de Resultados

| ID | Historia | Microservicio(s) involucrado(s) | Curl | Postman |
|----|----------|---------------------------------|------|---------|
| US-01 | Login con credenciales | ms_auth | ✓ | `ms_auth` → Autenticación → Iniciar sesión |
| US-02 | Buscar cliente por RUT | ms_clientes | ✓ | `ms_clientes` → ClienteController → Buscar por documento |
| US-03 | Registrar nuevo cliente | ms_clientes | ✓ | `ms_clientes` → ClienteController → Crear cliente |
| US-04 | Registrar admisión de paquete | ms_admision → ms_clientes, ms_sucursales, ms_auth | ✓ | `ms_admision` → Admisiones → Crear admisión |
| US-05 | Generar código de tracking y guía | ms_admision → ms_tracking | ✓ | Automático al crear admisión |
| US-06 | Cambiar estado de guía | ms_tracking | ✓ | `ms_tracking` → HistorialLogisticoController → Crear historial |
| US-07 | Consultar stock de embalaje | ms_embalaje | ✓ | `ms_embalaje` → StockSucursalController → Por sucursal |
| US-08 | Registrar venta con artículos | ms_ventas | ✓ | `ms_ventas` → VentaController → Crear venta |
| US-09 | Cálculo automático de IVA 19% | ms_ventas | ✓ | Automático en respuesta de crear venta |
| US-10 | Bloqueo por stock insuficiente | ms_ventas → ms_embalaje | ✓ | `ms_ventas` → VentaController → Crear venta (cantidad > stock) |
| US-11 | Ingreso de paquete a bodega | ms_paquetes | — | `ms_paquetes` → InventarioPaqueteController → Ingreso |
| US-12 | Salida de paquete de bodega | ms_paquetes | — | `ms_paquetes` → InventarioPaqueteController → Salida |
| US-13 | Abrir caja al inicio del turno | ms_finanzas | ✓ | `ms_finanzas` → AperturaCierreController → Abrir caja |
| US-14 | Cerrar caja con cuadre | ms_finanzas | ✓ | `ms_finanzas` → Cierre y estado → Cerrar caja |
| US-15 | Registrar usuario con rol y sucursal | ms_auth | ✓ | `ms_auth` → Autenticación → Registrar usuario |
| US-16 | Crear y gestionar sucursales | ms_sucursales | ✓ | `ms_sucursales` → SucursalController → Crear / Actualizar / Desactivar |
| US-17 | Gestionar catálogo de embalaje | ms_embalaje | ✓ | `ms_embalaje` → todas las carpetas |
| US-18 | Anular venta registrada | ms_ventas | — | `ms_ventas` → Limpiar → Anular venta |
| US-19 | Consultar ventas por sucursal | ms_ventas | — | `ms_ventas` → VentaController → Por sucursal |
| US-20 | Reporte de cierre de caja | ms_finanzas | — | `ms_finanzas` → Cierre y estado → Reporte cierre |
| US-21 | Consulta pública de tracking | ms_portal → ms_tracking | ✓ | `ms_portal` → ConsultaController → Crear consulta |
| US-22 | Historial completo de estados | ms_portal → ms_tracking | ✓ | `ms_portal` → ConsultaController → Buscar por código |
| US-23 | Calificar servicio (1–5) | ms_portal | ✓ | `ms_portal` → FeedbackController → Crear feedback |
| US-24 | Calificación anónima | ms_portal | ✓ | `ms_portal` → FeedbackController → Crear feedback (sin idCliente) |

**19 historias verificadas con curl. 24 de 24 historias cubiertas por las colecciones Postman.**

---

## Observaciones Técnicas

- **Cache de Eureka:** El gateway tiene un intervalo de refresco de 30 s. Al levantar en frío, la primera solicitud puede devolver 503 si los microservicios aún no están en la caché local. Solución: reintentar tras el primer ciclo.
- **Seed de catálogos:** Regiones, comunas y tipos de carga no tienen endpoints de escritura; deben insertarse directamente en MariaDB mediante `docker exec`.
- **Comunicación inter-servicio:** `ms_admision` usa `@LoadBalanced WebClient` con URIs `http://ms-nombre` resueltas por Eureka. Las 6 llamadas (2 a clientes, 2 a sucursales, 1 a auth, 1 a tracking) se ejecutan de forma síncrona (`.block()`).
- **JWT:** La validación de token solo aplica dentro de `ms_auth`. Los demás microservicios confían en el tráfico interno de la red Docker `sivebo-net`; no propagan ni validan JWT en las llamadas inter-servicio.
- **Variables Postman:** Cada colección es autónoma — los scripts internos encadenan automáticamente los ids entre requests (`pm.collectionVariables.set`). Para pruebas que cruzan colecciones (p.ej. usar `{{idGuia}}` de ms_tracking en ms_paquetes), es necesario copiar la variable manualmente o usar un entorno Postman compartido.
- **Orden de ejecución Postman:** Las carpetas marcadas "Limpiar" o "ejecutar último" contienen operaciones destructivas (DELETE, anulaciones). Ejecutarlas al final garantiza que las lecturas y creaciones previas queden registradas para revisión.
