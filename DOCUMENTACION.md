# Inventario Ferrosoldaduras — Documentación del proyecto

> Aplicación de escritorio para Windows 11 que permite a **Ferrosoldaduras**, una ferretería colombiana, llevar su inventario, proveedores, compras y ventas, con un resumen del estado del negocio.
>
> **Carpeta del proyecto:** `winapp-inventario-ferrosoldaduras` · **Identificador Tauri:** `com.andresg09.inventarioferrosoldaduras`
> **Idioma:** documento e interfaz en español; código (tablas, columnas, variables, funciones, comentarios) en inglés.

**Contenido:** [1 Resumen](#1-resumen) · [2 Decisiones críticas](#2-decisiones-críticas) · [3 Alcance](#3-alcance) · [4 Stack](#4-stack) · [5 Convenciones](#5-convenciones) · [6 Configuración del proyecto](#6-configuración-del-proyecto) · [7 Base de datos](#7-base-de-datos) · [8 Reglas de negocio](#8-reglas-de-negocio) · [9 Interfaz](#9-interfaz) · [10 Módulos](#10-módulos) · [11 Excel](#11-excel) · [12 Copias de seguridad](#12-copias-de-seguridad) · [13 Diseño e íconos](#13-diseño-e-íconos) · [14 Empaquetado](#14-empaquetado-e-instalación) · [15 Ideas futuras](#15-ideas-futuras)

---

## 1. Resumen

La app es **100 % local**: un solo usuario, un solo computador, sin login, sin servidor y sin internet. Los datos viven en un archivo SQLite.

| Módulo | Para qué sirve |
|---|---|
| **Inventario** | Productos por marca, con stock, precios y proveedor. Es el módulo por defecto al abrir la app. |
| **Proveedores** | Proveedores y teléfonos de contacto. |
| **Compra de mercancía** | Cada compra **suma** al stock. |
| **Ventas realizadas** | Cada venta **resta** del stock. |
| **Resumen general** | Indicadores del estado actual (solo lectura). |

**Conceptos clave**

- **Producto**: el *nombre* de lo que se vende (ej. "cemento gris"). Puede existir en varias marcas.
- **Registro de inventario** (`inventory_item`): la pareja **producto + marca** (ej. "cemento gris – argos"). Tiene su propio stock, precios y proveedor.
- **Precio registrado**: precio de venta vigente del registro de inventario. **Precio vendido**: el de una venta concreta (si no se indica, es el registrado).
- **Contar productos**: para el resumen, todas las marcas de un mismo nombre cuentan como **un** producto.

---

## 2. Decisiones críticas

Lo esencial para entender el proyecto:

1. **El stock es una columna real** (`inventory_items.stock`) que **nadie edita a mano**. Lo ajustan *triggers* de SQLite: cada compra suma, cada venta resta (también al editar o eliminar). Se corrige registrando una compra o venta con una observación que explique el motivo.
2. **Dinero, cantidades y stock son `REAL` con máximo 2 decimales.** Todo resultado de un cálculo se redondea con `round2()` (TypeScript) o `ROUND(..., 2)` (SQL). El usuario escribe decimales solo con coma y sin puntos de miles.
3. **Fechas siempre `YYYY-MM-DD`**, en base de datos, código, Excel y pantalla, en zona horaria `America/Bogota`. Se obtienen con `getCurrentDate()` de `lib/date.utils.ts`. Son opcionales (vacía = `NULL`).
4. **Los nombres se guardan normalizados** (minúsculas, sin espacios sobrantes, **con tildes**) y se muestran tal cual. Las búsquedas se normalizan igual.
5. **Solo se pueden eliminar compras y ventas.** Registros de inventario y proveedores no se eliminan; se corrigen editándolos.
6. **Excel**: la exportación copia las 4 tablas; la importación (carga inicial) solo carga **proveedores e inventario**, es estricta y genera un reporte `.txt` si falla.
7. **Los `id` son internos**: no se muestran en pantalla ni se exportan. No hay columnas de auditoría (`created_at`, `updated_at`).
8. **No cambiar el `identifier` de Tauri** después de instalar en el cliente: define dónde vive la base de datos.

---

## 3. Alcance

**Incluye:** crear y editar en los 4 módulos; eliminar compras y ventas (con confirmación); búsqueda, filtros, orden multicriterio y paginación; resumen general; menú nativo de Windows (Archivo, Ver); importar/exportar Excel; copia de seguridad y restauración; alerta de stock negativo al iniciar; instalador `.exe`.

**No incluye:** eliminar productos o proveedores; historiales o auditoría; clientes, métodos de pago, descuentos, fiado; facturación; múltiples usuarios; nube o actualizaciones automáticas; hardware (lector de códigos); stock mínimo; NIT o dirección de proveedores; modo oscuro.

---

## 4. Stack

| Capa | Tecnología |
|---|---|
| Gestor de paquetes | **pnpm** |
| Escritorio | **Tauri 2** (WebView2, ya incluido en Windows 11) |
| Frontend | **Next.js** (App Router, exportación estática, sin carpeta `src/`) + React + TypeScript |
| Estilos | **Tailwind CSS** con colores como variables globales |
| Componentes | **shadcn/ui** (modales, selects, combobox, botones, inputs, avisos; ver [6.7](#67-shadcnui)) |
| Íconos | **lucide-react** (viene con shadcn/ui; componentes SVG con tree-shaking, locales) |
| Base de datos | **SQLite** vía `tauri-plugin-sql`, con migraciones |
| Validación | **zod** (mismos esquemas en formularios e importación) |
| Estado global | **zustand**, solo donde sea necesario |
| Excel | `exceljs` |

Comandos: `pnpm tauri dev` (desarrollo) · `pnpm tauri build` (instalador). Requiere Windows, Node.js, Rust y *Microsoft C++ Build Tools*.

---

## 5. Convenciones

### 5.1 Números

- **Almacenamiento**: `REAL`, máximo 2 decimales. (`DECIMAL(10,2)` no sirve en SQLite: acepta la declaración pero ignora la precisión y guarda un número flotante igual que `REAL`.)
- **Por qué redondear**: en punto flotante `0.1 + 0.2` da `0.30000000000000004`. El error aparece *después* de sumar o multiplicar, así que se redondea **el resultado de cada cálculo**, no el dato al leerlo.

  ```ts
  // lib/numbers.ts
  export function round2(value: number): number {
    const rounded = Math.round((value + Number.EPSILON) * 100) / 100;
    return rounded === 0 ? 0 : rounded; // evita mostrar "-0"
  }
  // Ejemplo: const total = round2(quantity * soldPrice);
  ```
  En SQL: todo `SUM`, producto de columnas o actualización de stock va dentro de `ROUND(..., 2)` (los triggers ya lo hacen).
- **En pantalla**: dinero siempre `$12.500,00` (punto de miles, coma decimal, 2 decimales). No usar `style: 'currency'` de `Intl` (en `es-CO` mete un espacio tras el `$`): formatear con `Intl.NumberFormat('es-CO', { minimumFractionDigits: 2, maximumFractionDigits: 2, useGrouping: 'always' })` y anteponer `$`. Cantidades y stock muestran decimales solo si existen (`12`, `2,5`). Stock negativo en rojo.
- **Lo que escribe el usuario**: parte entera sin puntos, decimales con coma. Patrón `^\d+(,\d{1,2})?$`.

  | Entrada | Resultado |
  |---|---|
  | `12500` · `12500,5` · `12500,50` | 12500 · 12500,50 · 12500,50 |
  | `12.500` | Error: "Escribe solo números, sin puntos. Usa coma para los decimales." |
  | `12500,505` | Error: máximo 2 decimales |

  ```ts
  // lib/validation.ts
  export const decimalInput = z.string().trim()
    .regex(/^\d+(,\d{1,2})?$/, 'Escribe solo números, sin puntos. Usa coma para los decimales (ej. 12500,50).')
    .transform((s) => Number(s.replace(',', '.')));
  ```
  Los campos son `<input type="text" inputmode="decimal">`. Una variante con signo (`signedDecimalInput`, patrón `^-?\d+(,\d{1,2})?$`) se usa **solo** para el stock al importar desde Excel, porque puede ser negativo.

### 5.2 Fechas

- **Formato**: `YYYY-MM-DD` (ISO 8601) exclusivamente, con año completo y ceros iniciales, **en todas partes, incluida la pantalla**. No hay conversión de formato para mostrar: lo guardado es lo que se ve. Las columnas y campos lo indican: **`Fecha (aaaa-mm-dd)`**.
- **Zona horaria**: `America/Bogota` (definida en un solo lugar). **Prohibido** `new Date().toISOString().slice(0, 10)`: convierte a UTC y después de las 7 p. m. devuelve el día siguiente.
- **Centralización**: toda la lógica de fechas vive en `lib/date.utils.ts`; ningún otro archivo convierte o formatea fechas.

  ```ts
  // lib/date.utils.ts
  export const APP_TIME_ZONE = 'America/Bogota';

  export function getCurrentDate(): string {
    return new Intl.DateTimeFormat('en-CA', {
      timeZone: APP_TIME_ZONE, year: 'numeric', month: '2-digit', day: '2-digit',
    }).format(new Date());
  }
  ```
  Otras funciones del módulo: `getCurrentMonth()` (`YYYY-MM`, para el resumen), `getCurrentDateTime()` (`YYYY-MM-DD HH:mm:ss`, solo para el reporte de importación), `isValidISODate(value)` (formato y fecha real; rechaza `2026-02-30`) y `dateToExcel(iso)` (`new Date(Date.UTC(y, m - 1, d))`, para escribir fechas en Excel sin que la zona horaria mueva el día).
- **Campo de fecha en pantalla**: `<input type="text" placeholder="aaaa-mm-dd">`, no `type="date"` (este muestra el formato regional de Windows, `dd/mm/aaaa`, y no se puede forzar). Valor por defecto en compras y ventas nuevas: `getCurrentDate()`. Los filtros "Desde" y "Hasta" usan el mismo tipo de campo.
- **Opcional**: un campo vacío se guarda como `NULL` (esquema `isoDateInput`: vacío → `null`, lleno → debe pasar `isValidISODate`).

### 5.3 Normalización de texto

Producto, marca, nombre de proveedor y tipo de unidad se **guardan normalizados** y se **muestran tal cual**: sin espacios al inicio/fin, espacios repetidos colapsados y en minúsculas. **Las tildes y la ñ se conservan.** No hay columnas `*_normalized`.

```ts
// lib/normalize.ts
export const normalizeText = (v: string) => v.trim().replace(/\s+/g, ' ').toLocaleLowerCase('es-CO');
```

- Se aplica dentro de los esquemas `zod` (`.transform(normalizeText)`), así cualquier dato validado ya sale normalizado (interfaz e importación).
- Referencia, número de factura, observación y teléfonos solo reciben `trim`.
- **El texto de las búsquedas también se normaliza** antes de consultar, para que concuerde con lo guardado.
- Consecuencia: como las tildes se conservan, `tubería` y `tuberia` son productos distintos y la búsqueda las distingue. Hay que escribir de forma consistente.

### 5.4 SQL

- **Consultas parametrizadas** (`$1`, `$2`…): el valor viaja aparte del texto SQL. Nunca concatenar texto del usuario dentro de la consulta.
- **`ORDER BY` con lista blanca**: los parámetros solo reemplazan *valores*, no nombres de columna; por eso el orden se arma como texto. El usuario elige una clave de interfaz (`"brand"`) y el código la traduce con un mapa fijo; si la clave no existe, se usa el orden por defecto. Nunca se pega el valor recibido en el SQL.

  ```ts
  const SORT_COLUMNS = { product: 'name', brand: 'brand', stock: 'stock' } as const;
  const column = SORT_COLUMNS[sortKey] ?? 'name';   // ORDER BY ${column}
  ```
- **Comodines de `LIKE`**: en `LIKE '%texto%'`, los caracteres `%` y `_` son comodines (`_` = cualquier carácter). Antes de buscar, se escapan `\`, `%` y `_` en el texto del usuario (anteponiendo `\`) y se agrega `ESCAPE '\'`; así buscar `50%` o `m_8` busca esos caracteres literales.
- **Nunca escribir `stock` desde la app**: solo los triggers lo modifican. Única excepción: el `INSERT` de la importación inicial desde Excel ([11.3](#113-importar-desde-excel)).

---

## 6. Configuración del proyecto

El proyecto ya está creado. Esto es lo que debe quedar configurado en cada archivo.

### 6.1 Dependencias

```bash
pnpm add zod zustand exceljs
pnpm tauri add dialog fs process single-instance   # el plugin sql ya está instalado
```

En `src-tauri/Cargo.toml`, el plugin SQL necesita activar el driver SQLite:

```toml
tauri-plugin-sql = { version = "2", features = ["sqlite"] }
```

### 6.2 `next.config.ts` — exportación estática

```ts
const nextConfig = {
  output: 'export',          // genera /out con HTML/JS estático
  images: { unoptimized: true },
  trailingSlash: true,
};
export default nextConfig;
```

- Tauri carga archivos estáticos: **no hay** API routes, server actions, middleware ni `redirects()`. La ruta `/` redirige a `/inventory` desde el cliente.
- Las APIs `@tauri-apps/*` solo existen dentro de la ventana de la app: usarlas en componentes `"use client"` y dentro de efectos o manejadores de eventos, nunca en el nivel superior de un módulo (se ejecutaría durante el build).

### 6.3 `tsconfig.json`

Alias sin `src/`: `"paths": { "@/*": ["./*"] }`.

### 6.4 `src-tauri/src/lib.rs` — plugins y migraciones

```rust
use tauri_plugin_sql::{Builder as SqlBuilder, Migration, MigrationKind};

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    let migrations = vec![Migration {
        version: 1,
        description: "create_initial_schema",
        sql: include_str!("../migrations/0001_init.sql"),
        kind: MigrationKind::Up,
    }];

    tauri::Builder::default()
        .plugin(tauri_plugin_single_instance::init(|_app, _args, _cwd| {}))
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_process::init())
        .plugin(
            SqlBuilder::default()
                .add_migrations("sqlite:inventory.db", migrations)
                .build(),
        )
        .run(tauri::generate_context!())
        .expect("error while running application");
}
```

`single-instance` evita abrir dos ventanas (dos instancias sobre el mismo SQLite).

### 6.5 `src-tauri/capabilities/default.json` — permisos

Verificar nombres exactos en la documentación de Tauri 2: `core:default`, `core:menu:default`, `sql:default`, `sql:allow-execute` (el `default` del plugin SQL **no** permite escribir), `dialog:default`, permisos de `fs` para leer/escribir archivos elegidos con el diálogo, `process:allow-restart`.

### 6.6 `src-tauri/tauri.conf.json`

```json
{
  "productName": "Inventario Ferrosoldaduras",
  "version": "0.1.0",
  "identifier": "com.andresg09.inventarioferrosoldaduras",
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "devUrl": "http://localhost:3000",
    "frontendDist": "../out"
  },
  "app": {
    "windows": [
      { "title": "Inventario Ferrosoldaduras", "minWidth": 1024, "minHeight": 600, "maximized": true }
    ]
  },
  "bundle": {
    "active": true,
    "targets": ["nsis"],
    "windows": { "nsis": { "installMode": "currentUser", "languages": ["Spanish"] } }
  }
}
```

### 6.7 shadcn/ui

shadcn/ui **no es una librería que se instala y se importa**: es una colección de componentes (construidos sobre primitivos accesibles, Base UI o Radix UI según lo elegido en `init`, y Tailwind) cuyo **código se copia a `components/ui/`** con un comando de su CLI. Desde ese momento son archivos del proyecto y se pueden editar. Documentación: https://ui.shadcn.com

1. **Inicializar** (una sola vez):

   ```bash
   pnpm dlx shadcn@latest init
   ```

   Crea `components.json` (su configuración) y `lib/utils.ts` (la función `cn()` para combinar clases), agrega variables de color a `app/globals.css` e instala sus dependencias, entre ellas `lucide-react`. Se puede elegir cualquier color base: después se reemplazan los valores por la paleta de la sección [13](#13-diseño-e-íconos).
2. **Ajustar `app/globals.css`** con la paleta de la sección 13.
3. **Agregar los componentes** que usa esta app:

   ```bash
   pnpm dlx shadcn@latest add button input textarea checkbox label select table badge tooltip scroll-area dialog alert-dialog popover command sonner
   ```

   Cada uno aparece en `components/ui/` y se usa con `import { Button } from '@/components/ui/button'`.
4. **Montar `<Toaster />`** (de `sonner`) una sola vez en `app/layout.tsx`.

| Necesidad de la app | Componente |
|---|---|
| Botones (Guardar, Descartar, + Nuevo) | `button` |
| Campos de las filas editables | `input`, `textarea`, `label`, `checkbox` |
| Selector de proveedor y tamaño de página | `select` |
| Producto y Marca con búsqueda por escritura | `popover` + `command` (patrón "combobox") |
| Confirmar eliminación, stock negativo, edición en curso | `alert-dialog` |
| Resultado de importación | `dialog` |
| Aviso breve "Registro guardado" | `sonner` |
| Tablas (solo estilos; la edición en fila es propia) | `table` |
| "Chips" de criterios de orden | `badge` |
| Texto completo de observaciones | `tooltip` |
| Lista con scroll del popup de stock negativo | `scroll-area` |

Notas:

- **No usar** los componentes `sidebar` ni `form` de shadcn: la barra lateral es simple y propia, y la validación se hace con `zod` directamente.
- **Botón Guardar verde**: agregar una variante `success` en `components/ui/button.tsx` (es código del proyecto).
- **Íconos**: `init` instala `lucide-react`, que usan tanto los componentes de shadcn por dentro (la X del modal, la flecha del select) como la propia app (ver [13](#13-diseño-e-íconos)). No hay un segundo set de íconos.
- **Modo oscuro**: `init` agrega un bloque `.dark` en `globals.css`; se puede dejar sin usar o borrar.
- Antes de volver a ejecutar `add` sobre un componente que ya editaste, revisar: el CLI pide confirmación antes de sobrescribir.

### 6.8 Estructura de carpetas

```
winapp-inventario-ferrosoldaduras/
├─ app/                  # layout.tsx (sidebar + menú nativo), page.tsx (→ /inventory), globals.css,
│                        # inventory/ suppliers/ purchases/ sales/ summary/  (cada una con page.tsx)
├─ components/           # layout/ (Sidebar, NativeMenu, FloatingEditActions), data-table/,
│                        # ui/ (componentes generados por shadcn)
├─ features/             # inventory/ suppliers/ purchases/ sales/ summary/
│                        #   cada una: queries.ts (TODO el SQL del módulo), schema.ts (zod), componentes
├─ lib/                  # db.ts, numbers.ts, validation.ts, date.utils.ts, normalize.ts, utils.ts (cn, de shadcn),
│                        # excel/ (export, import, import-report, columns), backup.ts
├─ store/                # stores de zustand
└─ src-tauri/            # migrations/0001_init.sql, capabilities/, src/lib.rs, tauri.conf.json
```

Stores de zustand: `useEditingStore` (qué fila está en edición; alimenta los botones flotantes y los bloqueos) y `useStartupStore` (si ya se mostró la alerta de stock negativo en esta apertura).

---

## 7. Base de datos

### 7.1 Esquema

Archivo: `src-tauri/migrations/0001_init.sql`.

```
suppliers 1 ──── N inventory_items 1 ──── N purchases   (cada compra SUMA al stock)
                         └──────────── N sales           (cada venta RESTA del stock)
```

Todas las claves foráneas son `ON DELETE RESTRICT` (red de seguridad: la app no elimina productos ni proveedores).

```sql
CREATE TABLE suppliers (
  id       INTEGER PRIMARY KEY AUTOINCREMENT,
  name     TEXT NOT NULL UNIQUE,              -- normalizado
  phone_1  TEXT,                              -- solo dígitos, 7 a 10
  phone_2  TEXT
);

CREATE TABLE inventory_items (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  reference       TEXT,                       -- opcional, no único
  name            TEXT NOT NULL,              -- normalizado
  brand           TEXT NOT NULL,              -- normalizado
  supplier_id     INTEGER NOT NULL REFERENCES suppliers(id) ON DELETE RESTRICT,
  unit_type       TEXT NOT NULL,              -- normalizado: "unidad", "metro", "kg"...
  stock           REAL NOT NULL DEFAULT 0,    -- SOLO lo modifican los triggers; puede ser negativo
  purchase_price  REAL CHECK (purchase_price IS NULL OR purchase_price >= 0),
  sale_price      REAL NOT NULL CHECK (sale_price > 0),
  UNIQUE (name, brand)
);

CREATE TABLE purchases (
  id                 INTEGER PRIMARY KEY AUTOINCREMENT,
  purchase_date      TEXT,                    -- YYYY-MM-DD; NULL si no se indicó
  inventory_item_id  INTEGER NOT NULL REFERENCES inventory_items(id) ON DELETE RESTRICT,
  quantity           REAL NOT NULL CHECK (quantity > 0),
  unit_price         REAL CHECK (unit_price IS NULL OR unit_price >= 0),
  invoice_number     TEXT,
  observation        TEXT                     -- máx. 500 caracteres (validado en la app)
);

CREATE TABLE sales (
  id                 INTEGER PRIMARY KEY AUTOINCREMENT,
  sale_date          TEXT,                    -- YYYY-MM-DD; NULL si no se indicó
  inventory_item_id  INTEGER NOT NULL REFERENCES inventory_items(id) ON DELETE RESTRICT,
  quantity           REAL NOT NULL CHECK (quantity > 0),
  registered_price   REAL NOT NULL,           -- precio de inventario capturado
  sold_price         REAL NOT NULL CHECK (sold_price >= 0),  -- si el usuario no lo indica = registered_price
  total              REAL NOT NULL,           -- round2(quantity * sold_price)
  observation        TEXT
);

CREATE INDEX idx_items_supplier ON inventory_items(supplier_id);
CREATE INDEX idx_items_name     ON inventory_items(name);
CREATE INDEX idx_purchases_item ON purchases(inventory_item_id);
CREATE INDEX idx_purchases_date ON purchases(purchase_date);
CREATE INDEX idx_sales_item     ON sales(inventory_item_id);
CREATE INDEX idx_sales_date     ON sales(sale_date);
```

**Triggers que mantienen el stock** (al editar o eliminar, se revierte el efecto anterior y se aplica el nuevo):

```sql
-- COMPRAS: suman
CREATE TRIGGER trg_purchases_after_insert AFTER INSERT ON purchases
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock + NEW.quantity, 2) WHERE id = NEW.inventory_item_id;
END;

CREATE TRIGGER trg_purchases_after_delete AFTER DELETE ON purchases
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock - OLD.quantity, 2) WHERE id = OLD.inventory_item_id;
END;

CREATE TRIGGER trg_purchases_after_update AFTER UPDATE OF quantity, inventory_item_id ON purchases
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock - OLD.quantity, 2) WHERE id = OLD.inventory_item_id;
  UPDATE inventory_items SET stock = ROUND(stock + NEW.quantity, 2) WHERE id = NEW.inventory_item_id;
END;

-- VENTAS: restan
CREATE TRIGGER trg_sales_after_insert AFTER INSERT ON sales
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock - NEW.quantity, 2) WHERE id = NEW.inventory_item_id;
END;

CREATE TRIGGER trg_sales_after_delete AFTER DELETE ON sales
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock + OLD.quantity, 2) WHERE id = OLD.inventory_item_id;
END;

CREATE TRIGGER trg_sales_after_update AFTER UPDATE OF quantity, inventory_item_id ON sales
BEGIN
  UPDATE inventory_items SET stock = ROUND(stock + OLD.quantity, 2) WHERE id = OLD.inventory_item_id;
  UPDATE inventory_items SET stock = ROUND(stock - NEW.quantity, 2) WHERE id = NEW.inventory_item_id;
END;
```

**Por qué triggers y no código de la app:** si la app hiciera el `INSERT` de la compra y luego un segundo `UPDATE` del stock, un cierre o error entre ambos dejaría el stock desincronizado, y `tauri-plugin-sql` (que usa un *pool* de conexiones) no permite transacciones confiables repartidas en varias llamadas. El trigger corre dentro de la misma sentencia, de forma atómica, y cubre todos los casos en un solo lugar.

- No hay `CHECK (stock >= 0)`: el stock puede quedar negativo (ventas antes de registrar la compra).
- Editar solo fecha, precio, factura u observación no toca el stock (los triggers de `UPDATE` solo se disparan al cambiar `quantity` o `inventory_item_id`).

### 7.2 Vistas

Una **vista** es una consulta `SELECT` guardada en la base de datos que se usa como si fuera una tabla de **solo lectura**. No guarda datos propios: cada vez que se consulta, se ejecuta y devuelve información al día. Las compras y ventas solo guardan el número `inventory_item_id`; las vistas hacen el `JOIN` que lo traduce a producto, marca y proveedor, para no repetirlo en cada consulta del listado, la búsqueda, el conteo de la paginación y la exportación. Se leen de las vistas (`SELECT … FROM v_purchases WHERE … ORDER BY … LIMIT 50`) pero se escribe siempre en las tablas reales.

```sql
CREATE VIEW v_inventory AS
SELECT i.id, i.reference, i.name, i.brand, i.supplier_id, s.name AS supplier_name,
       i.unit_type, i.stock, i.purchase_price, i.sale_price
FROM inventory_items i
JOIN suppliers s ON s.id = i.supplier_id;

CREATE VIEW v_purchases AS
SELECT p.id, p.purchase_date, p.inventory_item_id,
       i.name AS product_name, i.brand, s.name AS supplier_name,
       p.quantity, p.unit_price, p.invoice_number, p.observation
FROM purchases p
JOIN inventory_items i ON i.id = p.inventory_item_id
JOIN suppliers s       ON s.id = i.supplier_id;

CREATE VIEW v_sales AS
SELECT sa.id, sa.sale_date, sa.inventory_item_id,
       i.name AS product_name, i.brand,
       sa.quantity, sa.registered_price, sa.sold_price, sa.total, sa.observation
FROM sales sa
JOIN inventory_items i ON i.id = sa.inventory_item_id;
```

Si una migración futura cambia columnas de las tablas, hay que borrar (`DROP VIEW`) y recrear las vistas afectadas en esa misma migración: las vistas no se actualizan solas.

### 7.3 Migraciones y ubicación de la base de datos

Una **migración** es un archivo SQL numerado que cambia la *estructura* de la base de datos. Existen porque la base del cliente tendrá datos reales: más adelante no podrás borrarla y recrearla para agregar una columna.

- Se registran en `lib.rs` con un número de `version` (1, 2, 3…). Al ejecutar `Database.load('sqlite:inventory.db')`, el plugin consulta su tabla interna `_sqlx_migrations`, aplica **solo las versiones que faltan** y las anota. Instalación nueva → se ejecuta la 1; actualización posterior con una migración 2 → se ejecuta solo la 2 y los datos siguen intactos.
- **Nunca editar una migración ya aplicada**: el plugin guarda su huella (*checksum*) y la app falla al arrancar si el archivo cambia. Todo cambio de estructura es una migración nueva (`0002_….sql`, `version: 2`).
- Si una migración cambia una vista o un trigger, incluir `DROP VIEW` / `DROP TRIGGER` y recrearlo.
- Durante el desarrollo puedes empezar de cero borrando `inventory.db` (con la app cerrada). Nunca en el computador del cliente.

**Dónde vive el archivo.** `sqlite:inventory.db` es relativo a `app_config_dir`, normalmente `C:\Users\<usuario>\AppData\Roaming\com.andresg09.inventarioferrosoldaduras\inventory.db`. **No hay que crear nada a mano:** en el primer `Database.load` el plugin crea la carpeta y el archivo si no existen y ejecuta las migraciones (comprobarlo la primera vez que se corra `pnpm tauri dev`). En el equipo de desarrollo, `tauri dev` y la app instalada comparten ese mismo archivo, porque comparten el `identifier`.

```ts
// lib/db.ts
import Database from '@tauri-apps/plugin-sql';
let dbPromise: Promise<Database> | null = null;
export const getDb = () => (dbPromise ??= Database.load('sqlite:inventory.db'));
```

Comprobaciones en desarrollo: `PRAGMA foreign_keys;` debe devolver `1` (el plugin lo activa por defecto) y los triggers deben probarse a mano (insertar, editar cantidad, cambiar de producto y eliminar una compra y una venta, verificando el stock en cada paso).

---

## 8. Reglas de negocio

| # | Regla |
|---|---|
| **RN-01** | **Stock**: columna real, solo la modifican los triggers (compra suma, venta resta; también al editar cantidad o producto y al eliminar). Para corregirlo se registra una compra o una venta con una observación que explique el motivo. Un producto nuevo empieza en 0. |
| **RN-02** | **Producto + marca es único.** El mismo producto en otra marca es otro registro de inventario. |
| **RN-03** | **Contar productos** agrupa por nombre (varias marcas = un producto). |
| **RN-04** | **Proveedor**: un registro de inventario tiene un solo proveedor, y el nombre de proveedor es único. En el selector, la opción **"Otro"** muestra un campo de texto; al guardar crea el proveedor (o reutiliza el existente si el nombre ya existe). Una compra no pide proveedor: se deduce del registro de inventario. |
| **RN-05** | **Stock negativo**: está permitido (se puede vender antes de registrar la compra). Tras crear, editar o eliminar una compra o venta, si el stock del registro afectado queda negativo, aparece un popup de advertencia (la app lee el stock ya actualizado por los triggers). |
| **RN-06** | **Eliminación**: solo compras y ventas, siempre con modal de confirmación; el stock se ajusta solo. Registros de inventario y proveedores no se eliminan. |
| **RN-07** | **Precio de compra**: opcional en el inventario. Si una compra trae "precio de compra unitario", al **crearla** actualiza el precio de compra del registro de inventario (editar una compra antigua no lo cambia). |
| **RN-08** | **Precio en ventas**: al crear una venta se guarda `registered_price` (precio de venta actual del inventario). "Precio vendido" es **opcional**: vacío → `sold_price = registered_price`. `total = round2(cantidad × sold_price)`. Cambiar después el precio del inventario no altera ventas anteriores. |
| **RN-09** | **Fechas** opcionales: vacía → `NULL`. Esos registros no aparecen en filtros por rango de fechas ni cuentan en los indicadores del mes. |
| **RN-10** | **Editar compras y ventas**: se puede cambiar todo lo editable, incluido producto y marca (el trigger devuelve la cantidad al producto anterior y la aplica al nuevo). En ventas, el precio registrado se vuelve a capturar **solo si cambia el producto o la marca**; en ese caso "Precio vendido" se vacía para que el usuario decida. |
| **RN-11** | **Resumen**: los registros con stock negativo **no entran** en los totales (unidades ni valores); se cuentan aparte. |

---

## 9. Interfaz

### 9.1 Estructura

```
┌───────────────────────────────────────────────────────────────┐
│ Archivo   Ver                                                  │  ← menú nativo de Windows
├──────────────┬────────────────────────────────────────────────┤
│ Inventario   │  Título del módulo                [+ Nuevo]    │
│ Ferrosoldadu-│  ┌────────────────────────────────────────────┐ │
│ ras          │  │ Búsqueda · Filtros · Ordenar por · Invertir │ │
│              │  ├────────────────────────────────────────────┤ │
│ ▣ Inventario │  │ Tabla (una fila por registro)         [🗑] │ │   [Guardar]    ← flotantes,
│ ▢ Proveedores│  │ ...                                         │ │   [Descartar]    solo al editar
│ ▢ Compras    │  ├────────────────────────────────────────────┤ │
│ ▢ Ventas     │  │ Paginación                                  │ │   [🗑] solo en
│ ▢ Resumen    │  └────────────────────────────────────────────┘ │   Compras y Ventas
│  v0.1.0      │                                                 │
└──────────────┴────────────────────────────────────────────────┘
```

**Barra lateral**: ícono + etiqueta por módulo; el módulo activo se resalta (fondo `accent`, texto e ícono `primary`); al abrir la app se carga **Inventario**; el pie muestra la versión (`getVersion()` de `@tauri-apps/api/app`). Los módulos salen de una lista de configuración (`navItems`) para poder agregar más.

| Módulo | Etiqueta | Ícono (lucide-react) |
|---|---|---|
| Inventario | Inventario | `Package` |
| Proveedores | Proveedores | `Truck` |
| Compras | Compra de mercancía | `ShoppingCart` |
| Ventas | Ventas realizadas | `Receipt` |
| Resumen | Resumen general | `LayoutDashboard` |

**Menú nativo** (Tauri 2, construido con `@tauri-apps/api/menu` en `NativeMenu`; se ve y se comporta como una aplicación de Windows real). Sin menús "Ayuda" ni "Herramientas":

| Menú | Opciones |
|---|---|
| **Archivo** | Importar desde Excel… (`Ctrl+I`) · Exportar a Excel… (`Ctrl+E`) · — · Crear copia de seguridad… · Restaurar copia de seguridad… · — · Salir (`Alt+F4`) |
| **Ver** | Inventario (`Ctrl+1`) · Proveedores (`Ctrl+2`) · Compra de mercancía (`Ctrl+3`) · Ventas realizadas (`Ctrl+4`) · Resumen general (`Ctrl+5`) |

Importar y restaurar se bloquean con el aviso de "edición en curso" si hay una fila en edición.

### 9.2 Tablas de registros

Inventario, Proveedores, Compras y Ventas comparten el componente `DataTable`:

- **Carga** desde la base de datos cada vez que se entra al módulo. Sin columna ID.
- **Búsqueda** con *debounce* (~300 ms); el texto se normaliza (5.3).
- **Filtros** propios de cada módulo (proveedor, estado de stock, rango de fechas `aaaa-mm-dd`).
- **Ordenar por**: selector multicriterio (el usuario agrega criterios por prioridad, como "chips") y checkbox **"Invertir orden"**. Orden por defecto **ascendente** (A→Z, menor→mayor, fecha antigua→reciente; las filas sin fecha quedan primero).
- **Paginación en SQL** (`LIMIT`/`OFFSET` + `COUNT(*)` con los mismos filtros): 50 por página por defecto (opciones 25/50/100). Al cambiar búsqueda, filtros u orden vuelve a la página 1.
- **Eliminar**: botón con ícono `Trash2` al final de la fila, **solo en Compras y Ventas**.
- Estado vacío con mensaje; observaciones largas se truncan con *tooltip*; las fechas se muestran tal cual se guardan.

### 9.3 Edición en la fila

- **Editar**: clic en una celda editable → toda la fila pasa a modo edición. Aparecen dos botones flotantes arriba a la derecha: **Guardar** (verde) y **Descartar** (rojo, debajo). Guardar valida con `zod` (errores bajo cada campo) y escribe; Descartar restaura. Atajos: `Enter` guarda, `Esc` descarta.
- **Crear**: el botón "+ Nuevo" inserta una fila vacía al inicio de la tabla, en modo edición, con valores por defecto (fecha de hoy) y los mismos botones flotantes.
- **Eliminar** (Compras y Ventas): modal de confirmación con **Cancelar** y **Eliminar** (rojo): "¿Eliminar esta compra/venta de *{producto – marca}* del *{fecha}*? Esta acción no se puede deshacer." Después se recarga la tabla y, si corresponde, aplica el popup de stock negativo.
- **Una sola acción a la vez**: si hay una fila en edición y se intenta editar otra, eliminar o cambiar de módulo, aparece un popup con solo el botón **Cerrar**: *"Se está modificando la información de otro producto. Por favor guarda o descarta los cambios antes de editar otro producto."* (en módulos distintos de inventario, "registro" en lugar de "producto").

### 9.4 Popups

| Popup | Cuándo | Contenido |
|---|---|---|
| **Stock negativo al iniciar** | Una vez por apertura de la app, tras cargar el inventario, si algún registro tiene stock < 0 | Tabla con **producto, marca y stock** de cada registro afectado (con scroll). Botón "Entendido". |
| **Stock negativo tras un cambio** | Después de crear, editar o eliminar una compra o venta que deja el stock del registro < 0 | "El stock de *{producto} – {marca}* quedó en *{stock}*. Registra una compra para regularizarlo." |
| **Confirmar eliminación** | Al pulsar eliminar | Ver 9.3. |
| **Edición en curso** | Ver 9.3 | Solo botón "Cerrar". |
| **Resultado de importación** | Al terminar o fallar una importación | Éxito: registros importados por hoja. Fallo: cantidad de errores, primeros errores y ruta del reporte `.txt` (11.4). |

---

## 10. Módulos

Los campos de texto "nombre" se guardan normalizados (5.3). Todos los decimales: máximo 2 (5.1).

### 10.1 Inventario (`v_inventory`)

| Columna | Obligatorio | Editable | Notas |
|---|---|---|---|
| Referencia | No | Sí | No único; solo `trim`. |
| Producto | Sí | Sí | Repetible entre marcas. |
| Marca | Sí | Sí | Producto + marca único (RN-02). |
| Proveedor | Sí | Sí | Selector de proveedores + "Otro" (RN-04). |
| Tipo de unidad | Sí | Sí | Texto con sugerencias (`<datalist>`) de las unidades ya usadas ("unidad", "metro", "kg"…). |
| Stock | — | **No** | Lo ajustan los triggers. Rojo si < 0; resaltado suave si = 0. |
| Precio de compra | No | Sí | |
| Precio de venta | Sí | Sí | > 0. |

- **Búsqueda**: producto, marca, referencia y proveedor. **Filtros**: proveedor; estado de stock (todos / en cero / negativo / con existencias).
- **Orden**: el **producto es siempre el primer nivel de agrupación**, para que las marcas de un mismo producto aparezcan juntas; los criterios del usuario (marca, proveedor, stock, precios…) ordenan *dentro* de cada producto. "Invertir orden" invierte todos los niveles, manteniendo los grupos juntos. Por defecto: producto A→Z, luego marca A→Z.
- No hay "stock inicial" en el formulario ni eliminación. Un producto mal creado se corrige editándolo.

### 10.2 Proveedores

| Columna | Obligatorio | Editable | Notas |
|---|---|---|---|
| Nombre | Sí | Sí | Único. |
| Teléfono 1 | No | Sí | Texto, solo dígitos, 7 a 10. |
| Teléfono 2 | No | Sí | Igual. |

**Búsqueda**: nombre y teléfonos. Orden por defecto: nombre A→Z. Sin eliminación.

### 10.3 Compra de mercancía (`v_purchases`)

| Columna | Obligatorio | Editable | Notas |
|---|---|---|---|
| Fecha (aaaa-mm-dd) | No | Sí | Por defecto hoy; vacía = `NULL`. |
| Producto | Sí | Sí | Combobox con búsqueda por escritura (pueden ser cientos); en shadcn: `popover` + `command`. |
| Marca | Sí | Sí | Solo las marcas existentes del producto elegido. |
| Proveedor | — | No | Deducido del registro de inventario; solo se muestra. |
| Cantidad | Sí | Sí | > 0. |
| Precio de compra unitario | No | Sí | RN-07. |
| N.º de factura | No | Sí | Solo `trim`. |
| Observación | No | Sí | Hasta 500 caracteres. |

**Búsqueda**: producto, marca y factura. **Filtros**: rango de fechas. Orden por defecto: fecha ascendente (se recomienda activar "Invertir orden" para ver primero lo más reciente). Eliminar: permitido.

### 10.4 Ventas realizadas (`v_sales`)

| Columna | Obligatorio | Editable | Notas |
|---|---|---|---|
| Fecha (aaaa-mm-dd) | No | Sí | Por defecto hoy; vacía = `NULL`. |
| Producto / Marca | Sí | Sí | Igual que en compras. |
| Cantidad | Sí | Sí | > 0. |
| Precio registrado | — | **No** | Capturado automáticamente (RN-08, RN-10). |
| Precio vendido | **No** | Sí | Vacío → se usa el registrado (el campo lo muestra como indicación). |
| Total de venta | — | **No** | `round2(cantidad × precio vendido)`; en edición se recalcula en vivo. |
| Observación | No | Sí | Hasta 500 caracteres. |

**Búsqueda**: producto y marca. **Filtros**: rango de fechas. Eliminar: permitido.

### 10.5 Resumen general

Solo lectura; se recalcula al entrar. Los registros con stock negativo no entran en los totales (RN-11).

| Indicador | Definición |
|---|---|
| Productos en el negocio | Nombres distintos (RN-03). |
| Unidades totales en inventario | Σ stock de registros con stock > 0. Mezcla unidades de medida distintas: cifra aproximada. |
| Valor del inventario a precio de venta | Σ (stock × precio de venta), stock > 0. |
| Valor del inventario a precio de compra | Σ (stock × precio de compra), stock > 0 y con precio de compra; indica cuántos registros se excluyeron por no tenerlo. |
| Compras y ventas del mes | Cantidad de registros del mes actual (`getCurrentMonth()`); no cuentan los registros sin fecha. Permite ver si se compra mucho y se vende poco. |
| Registros con stock en 0 / negativo | Cantidad de registros (producto + marca) en cada caso. |

---

## 11. Excel

### 11.1 Formato

Libro de **4 hojas**, encabezados en la fila 1, una fila por registro, **sin columna ID**. Cada hoja corresponde a una tabla:

| Hoja | Columnas |
|---|---|
| **Proveedores** | Nombre · Teléfono 1 · Teléfono 2 |
| **Inventario** | Referencia · Producto · Marca · Proveedor · Tipo de unidad · Stock · Precio de compra · Precio de venta |
| **Compras** | Fecha · Producto · Marca · Cantidad · Precio de compra unitario · N.º de factura · Observación |
| **Ventas** | Fecha · Producto · Marca · Cantidad · Precio registrado · Precio vendido · Total de venta · Observación |

- Las relaciones van por **nombre** (Proveedor en Inventario; Producto y Marca en Compras y Ventas), no por ID.
- Las columnas que solo existen para mostrar en pantalla (como "Proveedor" en Compras) no se exportan.
- Fechas: celdas de fecha reales con formato `yyyy-mm-dd`. Dinero, cantidades y stock: números reales.

### 11.2 Exportar

`Archivo → Exportar a Excel…` sirve para **compartir el estado actual**: copia fiel de las 4 tablas, sin filtros ni extras. Nombre sugerido `inventario-ferrosoldaduras_AAAA-MM-DD.xlsx`; fechas escritas con `dateToExcel`.

### 11.3 Importar

`Archivo → Importar desde Excel…` es la **carga inicial** (la hace el desarrollador con un Excel ya verificado).

- **Solo se importan las hojas Proveedores e Inventario.** Las hojas Compras y Ventas, si existen en el archivo, se ignoran.
- **Precondición**: base de datos vacía; si ya tiene datos, se rechaza con un mensaje.
- **El Stock del Excel se guarda tal cual** en `inventory_items.stock` (obligatorio, hasta 2 decimales, puede ser negativo; se valida con `signedDecimalInput`). Es el único caso en que el código escribe el stock directamente; no hay conflicto con los triggers porque no se importan compras ni ventas.
- **Estricta**: cualquier error cancela todo y genera un reporte `.txt`.

**Flujo**

1. Elegir el `.xlsx` y leerlo con `exceljs`.
2. **Validar todo antes de escribir**, con los mismos esquemas `zod` de la interfaz: hojas y encabezados presentes, campos obligatorios, decimales, teléfonos, unicidad (nombre de proveedor; producto + marca), y que el Proveedor de cada fila de Inventario exista en la hoja Proveedores. Los textos de nombre se normalizan. Se recopilan **todos** los errores (no se detiene en el primero).
3. Si hay errores: no se escribe nada y se genera el reporte.
4. Insertar Proveedores → Inventario (con su stock).
5. Mostrar el resultado (registros importados por hoja).

**Atomicidad**: no hay transacciones multi-llamada confiables con `tauri-plugin-sql`; por eso se valida todo antes y, si algún `INSERT` fallara a mitad de camino, se **vacían las tablas** (válido porque la precondición garantiza que estaban vacías) y se genera el reporte.

### 11.4 Reporte de fallo (`.txt`)

Si la importación falla, se abre automáticamente el diálogo "Guardar como" con el nombre `reporte-importacion_AAAA-MM-DD_HH-mm-ss.txt` (de `getCurrentDateTime()`), en la carpeta del Excel. Si el usuario cancela, el popup de resultado ofrece el botón **Guardar reporte**. Lo genera `lib/excel/import-report.ts`. Cada error indica **hoja, fila (como se ve en Excel), columna, descripción y valor encontrado**; si el fallo ocurrió durante la inserción, indica también el mensaje de la base de datos y si las tablas quedaron vacías.

```
REPORTE DE IMPORTACIÓN — Inventario Ferrosoldaduras
Fecha y hora: 2026-10-30 14:05:09 (America/Bogota)
Archivo: C:\Users\...\inventario.xlsx
Resultado: FALLÓ — no se importó ningún dato.
Filas leídas: Proveedores 12 · Inventario 340
Errores encontrados: 2

[Inventario] Fila 14, columna "Proveedor": el proveedor "argos sa" no existe en la hoja Proveedores.
[Inventario] Fila 88, columna "Stock": valor inválido "2.5" (usa coma para decimales, sin puntos).
```

---

## 12. Copias de seguridad

- **Crear** (`Archivo → Crear copia de seguridad…`): diálogo "Guardar como" (`respaldo-ferrosoldaduras_AAAA-MM-DD.db`) y `VACUUM INTO '<ruta>'`, que genera una copia **consistente** aun con la app abierta (copiar el `.db` directamente no es seguro: puede haber escrituras recientes en los archivos `-wal`/`-shm`). `VACUUM INTO` falla si el destino existe; si el usuario eligió sobrescribir, se borra antes.
- **Restaurar** (`Archivo → Restaurar copia de seguridad…`): elegir el `.db` → validar que sea una base de esta app → confirmar que se reemplazarán todos los datos → copia automática del estado actual → cerrar la conexión (`db.close()`), reemplazar `inventory.db` y borrar `-wal`/`-shm` → reiniciar la app (`relaunch`).
- Recomendación para el cliente: copias periódicas guardadas **fuera del computador** (USB, nube). Todo vive en un solo archivo en un solo equipo y la copia es la única forma de recuperar compras o ventas eliminadas por error.

---

## 13. Diseño e íconos

**Colores**: neutros con color principal morado azulado **`#3E23AE`**. Todos son variables CSS globales conectadas a Tailwind (`app/globals.css`); los componentes solo usan clases semánticas (`bg-primary`, `text-muted-foreground`, `bg-accent`…), nunca colores "a mano". Los nombres son los de shadcn/ui. Después de `shadcn init` ([6.7](#67-shadcnui)), reemplazar en `:root` los valores por los de abajo y agregar los dos propios del proyecto (`success` y `warning`); el resto de lo que genere el CLI (por ejemplo `@theme inline`) se deja. `accent` hace de "morado suave" (módulo activo en la barra lateral, elementos seleccionados). Solo modo claro por ahora.

```css
/* app/globals.css (Tailwind v4) — valores de :root */
:root {
  --background: #F6F6F9;
  --foreground: #1C1B22;
  --card: #FFFFFF;
  --card-foreground: #1C1B22;
  --popover: #FFFFFF;
  --popover-foreground: #1C1B22;
  --primary: #3E23AE;
  --primary-foreground: #FFFFFF;
  --secondary: #EFEFF4;
  --secondary-foreground: #1C1B22;
  --muted: #EFEFF4;
  --muted-foreground: #6B6A78;
  --accent: #EAE7F9;
  --accent-foreground: #3E23AE;
  --destructive: #DC2626;   /* Descartar, Eliminar, stock negativo */
  --border: #E2E1EA;
  --input: #E2E1EA;
  --ring: #3E23AE;

  /* propios del proyecto */
  --success: #16A34A;       /* botón Guardar */
  --warning: #D97706;       /* stock en cero */
}

/* En el bloque @theme inline generado por shadcn, agregar solo estos dos: */
@theme inline {
  --color-success: var(--success);
  --color-warning: var(--warning);
}
```

**Íconos (lucide-react)**: cada ícono es un componente de React que se importa **por nombre**; solo se empaqueta lo que se usa y funciona sin internet (son SVG dentro del código, sin archivos de fuente). `lucide-react` lo instala `shadcn init`, y es el mismo set que usan los componentes de shadcn por dentro.

```tsx
import { Trash2, Plus } from 'lucide-react';

<Trash2 className="size-4" />        {/* tamaño con size-*, color con text-* */}
<Plus className="size-5 text-primary" />
```

- Importar siempre **por nombre**. No usar `import * as Icons` ni buscar íconos por texto (anula el tree-shaking).
- Los íconos del sidebar se guardan como componentes en la lista `navItems`: `{ label: 'Inventario', href: '/inventory', icon: Package }`, con el tipo `LucideIcon` (`import type { LucideIcon } from 'lucide-react'`).
- Acciones sugeridas: `Trash2` (eliminar), `Plus` (nuevo), `Search`, `ListFilter` (filtros), `ArrowUpDown` (invertir orden), `Save`, `X`, `TriangleAlert` (avisos).
- Los nombres se buscan en https://lucide.dev/icons. Algunos se renombran entre versiones (por ejemplo `AlertTriangle` pasó a `TriangleAlert`); si un nombre no existe, buscarlo ahí.

---

## 14. Empaquetado e instalación

- **Generar**: `pnpm tauri build` (en Windows) → `src-tauri/target/release/bundle/nsis/Inventario Ferrosoldaduras_0.1.0_x64-setup.exe`. Es un instalador `.exe` (NSIS) por usuario, **sin permisos de administrador** (`installMode: "currentUser"`).
- **Instalar en el cliente**: copiar y ejecutar el `-setup.exe`. Al no estar firmado, Windows muestra "Windows protegió su PC": **Más información → Ejecutar de todas formas** (solo al instalar). WebView2 ya viene en Windows 11. Luego, `Archivo → Importar desde Excel…` con el archivo verificado y una copia de seguridad inicial.
- **Actualizar**: subir `version` en `tauri.conf.json` y `package.json`, generar el instalador y ejecutarlo **sobre la instalación existente**. Los datos se conservan (viven en `AppData\Roaming\com.andresg09.inventarioferrosoldaduras\`, fuera de la carpeta del programa) y las migraciones nuevas se aplican solas al primer arranque. No hay actualización automática.
- ⚠️ Al **desinstalar**, NSIS ofrece una casilla "borrar los datos de la aplicación": **dejarla desmarcada** o se pierde la base de datos.

---

## 15. Ideas futuras

- Copia de seguridad automática (por ejemplo, al cerrar la app, conservando las últimas N).
- Marcar productos y proveedores como "inactivos" (ocultarlos de listas y selectores sin borrarlos), ya que no se pueden eliminar.
- Búsqueda insensible a tildes y selector de calendario para las fechas.
- Stock mínimo con alertas, modo oscuro, clientes y métodos de pago.
