# SIVEBO - Sistema de Ventas y Bodega
# PROPUESTA DE PROYECTO

**Sistema POS Integral para Operadores Polifuncionales - RapidinBombin**

Fecha: Abril 2025  
Última actualización del modelo de datos: Junio 2025

---

## 1. Definición del Negocio y Problemática

### 1.1 Contexto de la Empresa

RapidinBombin es una empresa de logística con presencia en múltiples sucursales, cuya operación diaria depende de operadores polifuncionales. Estos trabajadores son responsables de ejecutar una amplia variedad de tareas críticas para el funcionamiento del negocio.

### 1.2 Funciones del Operador Polifuncional

- Admisión y recepción de paquetería entrante
- Manejo del inventario de paquetes recepcionados y despachados
- Control del inventario de materiales de embalaje
- Venta de embalaje y servicios adicionales a los clientes
- Gestión de las finanzas de la sucursal (caja, cierre, reportes)

### 1.3 Descripción del Problema

La empresa busca un sistema integrado para los operadores polifuncionales. Debido a la gran cantidad de funciones, la empresa busca que el operador interactúe a través de una única interfaz unificada, rápida y fácil de aprender para atender un alto volumen de clientes. Los problemas actuales incluyen la fragmentación de herramientas, la elevada curva de aprendizaje, la falta de trazabilidad integral y el riesgo de errores financieros.

---

## 2. Solución Propuesta

| Solución | Descripción | Ventajas | Desventajas |
| --- | --- | --- | --- |
| **Sistema POS Integral con Microservicios** | Desarrollo de un sistema web unificado basado en arquitectura de microservicios con Spring Boot. Cada funcionalidad opera como un servicio independiente con su propia base de datos Oracle, comunicados mediante API REST y WebClient. | Alta escalabilidad. Fácil mantenimiento por módulos. Alineado con directrices académicas de 3er semestre. | Mayor complejidad inicial. Requiere coordinación y API Gateway. |

---

## 3. Arquitectura de Microservicios (Solución Recomendada)

El sistema se desarrollará obligatoriamente utilizando **Spring Boot** y estará compuesto por un **API Gateway (Eureka)** para la gestión de peticiones y los siguientes 10 microservicios independientes:

- **MS-01 Auth & Usuarios:** Gestión de login, tokens de sesión, encriptación de contraseñas y roles (Admin, Operador, Cliente).
- **MS-02 Sucursales:** Configuración y administración de la red de sucursales.
- **MS-03 Admisión de Paquetes:** Registro de ingreso, sucursal de origen y destino, generación de número de tracking.
- **MS-04 Tracking & Logística:** Control de estados (recibido, en tránsito, entregado) con historial completo por guía.
- **MS-05 Inventario de Paquetes:** Stock de paquetes recepcionados y despachados por sucursal.
- **MS-06 Inventario de Embalaje:** Control de materiales de embalaje y stock por sucursal.
- **MS-07 Ventas / POS:** Venta de embalaje y servicios de envío, generación de boletas.
- **MS-08 Finanzas:** Apertura y cierre de caja, movimientos y reportes de ventas.
- **MS-09 Clientes:** Registro de remitentes y destinatarios con tipo de documento.
- **MS-10 Portal Cliente:** Consulta pública de tracking y feedback de calificación.

---

## 4. Desacoplamiento y Principios Técnicos

- **Comunicación:** Interacción mediante **API REST** y consultas de datos utilizando **WebClient**. Los microservicios estarán completamente desacoplados.
- **API Gateway:** Implementación de **Eureka** (u otro equivalente) para el enrutamiento y registro de servicios.
- **Database per Service:** Cada microservicio tiene su propia base de datos Oracle independiente. Las referencias entre servicios se implementan mediante **ID externos (Ref Ext)** sin FK formal entre bases de datos. La integridad referencial entre servicios es responsabilidad de la capa de aplicación.
- **Motor de Base de Datos:** **Oracle Database 21c**, con modelo relacional en **Tercera Forma Normal (3FN)**, sequences para autoincremento, triggers de compatibilidad y constraints de negocio declarados explícitamente.
- **Patrón de Diseño:** Implementación del patrón **CSR (Controller-Service-Repository)** y uso de **DTOs**.

---

## 5. Seguridad Básica

El MS-01 implementará mecanismos de seguridad que incluyen:

- Encriptación de contraseñas antes del almacenamiento (`CHAR(60)`, compatible con bcrypt).
- Autenticación de usuarios y generación de tokens de sesión (JWT) en el login.
- Validación de accesos según los roles definidos (Admin, Operador, Cliente).

---

## 6. Estructura de Datos — Entidades por Microservicio

El modelo de datos fue normalizado a **Tercera Forma Normal (3FN)**. Cada microservicio tiene su propio archivo DDL Oracle en la carpeta `/ddl/`.

### Convenciones

| Sufijo | Significado |
| --- | --- |
| `PK` | Clave primaria (`NUMBER(10)` + `SEQUENCE` + `TRIGGER`) |
| `UK` | Restricción de unicidad |
| `FK` | Clave foránea interna (dentro del mismo microservicio) |
| `Ref Ext` | Referencia externa a otro microservicio (solo el ID, sin FK formal) |
| `?` | Campo nullable |

---

### MS-01 · db_ms_auth — Usuarios y Seguridad

| Entidad | Atributos |
| --- | --- |
| `ROL` | id_rol `PK`, nombre_rol `UK`, descripcion |
| `USUARIO` | id_usuario `PK`, username `UK`, password_hash `CHAR(60)`, email `UK`, id_rol `FK`, id_sucursal_asignada `Ref Ext`·`?`, activo, created_at |
| `TOKEN_SESION` | id_token `PK`, id_usuario `FK`, token `UK`, fecha_expiracion, created_at |

**Ref Ext:** `USUARIO.id_sucursal_asignada` → `db_ms_sucursales.SUCURSAL`

---

### MS-02 · db_ms_sucursales — Configuración de Red

| Entidad | Atributos |
| --- | --- |
| `REGION` | id_region `PK`, nombre_region `UK` |
| `COMUNA` | id_comuna `PK`, nombre_comuna, id_region `FK` |
| `SUCURSAL` | id_sucursal `PK`, nombre_sucursal, id_comuna `FK`, direccion_fisica, telefono_contacto`?`, activa |

**Ref Ext:** Ninguna. Fuente de verdad de la red geográfica.

---

### MS-03 · db_ms_admision — Ingreso de Carga

| Entidad | Atributos |
| --- | --- |
| `TIPO_CARGA` | id_tipo `PK`, nombre_tipo `UK` (Documento, Encomienda…), descripcion |
| `ADMISION` | id_admision `PK`, id_cliente_rem `Ref Ext`, id_cliente_dest `Ref Ext`, id_sucursal_origen `Ref Ext`, id_sucursal_dest `Ref Ext`, id_tipo `FK`, peso_kg, fecha_creacion, id_usuario_reg `Ref Ext` |

**Ref Ext:** `id_cliente_rem` / `id_cliente_dest` → `db_ms_clientes.CLIENTE` · `id_sucursal_origen` / `id_sucursal_dest` → `db_ms_sucursales.SUCURSAL` · `id_usuario_reg` → `db_ms_auth.USUARIO`

---

### MS-04 · db_ms_tracking — Logística y Estados

| Entidad | Atributos |
| --- | --- |
| `ESTADO_MAESTRO` | id_estado `PK`, nombre_estado `UK` (Recibido, En Tránsito, Entregado…), orden |
| `GUIA_DESPACHO` | id_guia `PK`, codigo_tracking `UK`, id_admision `Ref Ext`, fecha_creacion |
| `HISTORIAL_LOGISTICO` | id_hist `PK`, id_guia `FK`, id_estado `FK`, id_sucursal_actual `Ref Ext`, id_usuario `Ref Ext`, fecha_hora, comentario`?` |

**Vista:** `V_ESTADO_ACTUAL_GUIA` — estado vigente por guía (último registro del historial).

**Ref Ext:** `GUIA_DESPACHO.id_admision` → `db_ms_admision.ADMISION` · `HISTORIAL_LOGISTICO.id_sucursal_actual` → `db_ms_sucursales.SUCURSAL` · `HISTORIAL_LOGISTICO.id_usuario` → `db_ms_auth.USUARIO`

---

### MS-05 · db_ms_inv_paquetes — Stock de Envíos en Bodega

| Entidad | Atributos |
| --- | --- |
| `INVENTARIO_PAQUETE` | id_inv `PK`, id_guia `Ref Ext`, id_sucursal `Ref Ext`, fecha_ingreso, fecha_salida`?` |

> **Nota de diseño:** La entidad `UBICACION_BODEGA` fue eliminada por no corresponder a un concepto del negocio. La ubicación física de un paquete queda expresada únicamente por la sucursal donde está almacenado.

**Ref Ext:** `id_guia` → `db_ms_tracking.GUIA_DESPACHO` · `id_sucursal` → `db_ms_sucursales.SUCURSAL`

---

### MS-06 · db_ms_inv_embalaje — Materiales de Venta

| Entidad | Atributos |
| --- | --- |
| `CATEGORIA_EMBALAJE` | id_cat `PK`, nombre_categoria `UK` (Cajas, Sobres…) |
| `ARTICULO_EMBALAJE` | id_art `PK`, id_cat `FK`, nombre, descripcion`?`, precio_vta, activo |
| `STOCK_SUCURSAL` | id_stock `PK`, id_art `FK`, id_sucursal `Ref Ext`, cantidad_disponible, updated_at · `UK(id_art, id_sucursal)` |

**Ref Ext:** `STOCK_SUCURSAL.id_sucursal` → `db_ms_sucursales.SUCURSAL`

---

### MS-07 · db_ms_ventas — Transacciones POS

| Entidad | Atributos |
| --- | --- |
| `VENTA` | id_venta `PK`, nro_boleta `UK`, id_usuario `Ref Ext`, id_sucursal `Ref Ext`, fecha_vta, subtotal, iva, total, estado |
| `DETALLE_VENTA` | id_det `PK`, id_venta `FK`, id_articulo `Ref Ext`, id_admision `Ref Ext`·`?`, cantidad_art, precio_unit_historico_art, precio_admision`?` |

> **Nota de diseño:** `DETALLE_VENTA` permite registrar tanto líneas de embalaje como líneas de servicio de envío en la misma boleta. `id_admision` y `precio_admision` son nullable cuando la línea corresponde solo a embalaje.

**Ref Ext:** `VENTA.id_usuario` → `db_ms_auth.USUARIO` · `VENTA.id_sucursal` → `db_ms_sucursales.SUCURSAL` · `DETALLE_VENTA.id_articulo` → `db_ms_inv_embalaje.ARTICULO_EMBALAJE` · `DETALLE_VENTA.id_admision` → `db_ms_admision.ADMISION`

---

### MS-08 · db_ms_finanzas — Caja y Reportes

| Entidad | Atributos |
| --- | --- |
| `CAJA_SUCURSAL` | id_caja `PK`, id_sucursal `Ref Ext` `UK`, estado_actual |
| `APERTURA_CIERRE` | id_sesion `PK`, id_caja `FK`, id_usuario `Ref Ext`, monto_apertura, monto_cierre`?`, fecha_hora_ap, fecha_hora_ci`?` |
| `MOVIMIENTO_CAJA` | id_mov `PK`, id_sesion `FK`, tipo (INGRESO/EGRESO), monto, id_venta `Ref Ext`·`?`, concepto`?` |

**Ref Ext:** `CAJA_SUCURSAL.id_sucursal` → `db_ms_sucursales.SUCURSAL` · `APERTURA_CIERRE.id_usuario` → `db_ms_auth.USUARIO` · `MOVIMIENTO_CAJA.id_venta` → `db_ms_ventas.VENTA`

---

### MS-09 · db_ms_clientes — Base de Datos de Personas

| Entidad | Atributos |
| --- | --- |
| `TIPO_DOCUMENTO` | id_tipo_doc `PK`, codigo `UK` (RUT, Pasaporte…), descripcion |
| `CLIENTE` | id_cliente `PK`, id_tipo_doc `FK`, nro_documento `UK`, nombre, apellido, email`?`, telefono`?` |

**Ref Ext:** Ninguna. Fuente de verdad de personas.

---

### MS-10 · db_ms_portal — Consultas Públicas y Feedback

| Entidad | Atributos |
| --- | --- |
| `CONSULTA_PUBLICA` | id_cons `PK`, codigo_tracking_consultado, id_guia `Ref Ext`·`?`, ip_usuario, fecha_hora |
| `FEEDBACK_CLIENTE` | id_feed `PK`, id_guia `Ref Ext`, id_cliente `Ref Ext`·`?`, calificacion (1–5), comentario`?`, fecha_hora |

**Vista:** `V_CONSULTA_PUBLICA` — expone `guia_encontrada` como booleano derivado de `id_guia IS NOT NULL`.

**Ref Ext:** `id_guia` → `db_ms_tracking.GUIA_DESPACHO` · `FEEDBACK_CLIENTE.id_cliente` → `db_ms_clientes.CLIENTE`

---

## 7. Reglas de Negocio

- **Control de Stock:** No se puede realizar una venta en MS-07 si MS-06 indica stock insuficiente del artículo de embalaje solicitado.
- **Generación de Tracking:** Todo paquete admitido en MS-03 debe generar automáticamente una `GUIA_DESPACHO` con un estado inicial en MS-04.
- **Registro de Destino:** Toda admisión debe registrar explícitamente la sucursal de origen (`id_sucursal_origen`) y la sucursal de destino (`id_sucursal_dest`).
- **Cierre de Caja:** MS-08 debe cuadrar las ventas registradas en MS-07 antes de permitir el cierre de sesión de caja.
- **Validación de Roles:** Solo un usuario con rol `Admin` o `Supervisor` puede anular una venta (cambiar `estado` a `ANULADA`).
- **Feedback anónimo:** MS-10 permite registrar feedback sin identificar al cliente (`id_cliente` nullable). Si el `codigo_tracking_consultado` no existe, `id_guia` queda `NULL` en `CONSULTA_PUBLICA`.
- **Una caja por sucursal:** La restricción `UNIQUE(id_sucursal)` en `CAJA_SUCURSAL` garantiza que cada sucursal tenga exactamente una caja asignada.

---

## 8. Control de Versiones, Pruebas y Despliegue

- **Repositorio:** Uso obligatorio de **GitHub** para evidenciar el avance progresivo, la participación del equipo, el historial de cambios, y el uso de ramas de desarrollo con commits frecuentes.
- **Pruebas y Documentación:** Se desarrollará documentación técnica del sistema y pruebas unitarias de los módulos principales.
- **Despliegue:** Despliegue en entorno local exponiendo los servicios a través del API Gateway, dejando el sistema disponible mediante una URL para el consumo de las APIs.

---

## 9. Archivos DDL

Cada microservicio tiene su propio script DDL Oracle ubicado en `/ddl/`:

| Archivo | Microservicio | Tablas |
| --- | --- | --- |
| `01_ms_auth.sql` | db_ms_auth | ROL, USUARIO, TOKEN_SESION |
| `02_ms_sucursales.sql` | db_ms_sucursales | REGION, COMUNA, SUCURSAL |
| `03_ms_clientes.sql` | db_ms_clientes | TIPO_DOCUMENTO, CLIENTE |
| `04_ms_admision.sql` | db_ms_admision | TIPO_CARGA, ADMISION |
| `05_ms_tracking.sql` | db_ms_tracking | ESTADO_MAESTRO, GUIA_DESPACHO, HISTORIAL_LOGISTICO |
| `06_ms_inv_paquetes.sql` | db_ms_inv_paquetes | INVENTARIO_PAQUETE |
| `07_ms_inv_embalaje.sql` | db_ms_inv_embalaje | CATEGORIA_EMBALAJE, ARTICULO_EMBALAJE, STOCK_SUCURSAL |
| `08_ms_ventas.sql` | db_ms_ventas | VENTA, DETALLE_VENTA |
| `09_ms_finanzas.sql` | db_ms_finanzas | CAJA_SUCURSAL, APERTURA_CIERRE, MOVIMIENTO_CAJA |
| `10_ms_portal.sql` | db_ms_portal | CONSULTA_PUBLICA, FEEDBACK_CLIENTE |

