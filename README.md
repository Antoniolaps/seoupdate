# seoupdate

Sí. Eso cambia dos partes importantes del diseño y conviene incorporarlo antes de hacer los cambios anteriores, porque afecta creación de productos, búsqueda, cotización, Excel y cálculo de impuestos.

1. Reemplazar Código por Referencia globalmente

La regla será:

> referencia será el identificador comercial que se mostrará y utilizará en creación, búsqueda y cotización.



No debemos simplemente cambiar el texto visual de Código a Referencia; hay que revisar el flujo completo.

Producto

Actualmente tenemos algo parecido a:

codigo
codigo_barras
nombre
precio_venta

La estructura objetivo sería:

referencia
codigo_barras
nombre
precio_venta

Si codigo actualmente se utiliza internamente en varias partes, no recomiendo eliminarlo inmediatamente. Primero hay que identificar todas sus dependencias.

El objetivo sería:

CREAR PRODUCTO
      ↓
Referencia
      ↓
BASE DE DATOS
      ↓
BUSCAR PRODUCTO
      ↓
Referencia
      ↓
AGREGAR A COTIZACIÓN
      ↓
Referencia


---

2. Al crear producto

El formulario de creación debe mostrar:

Referencia *
Código de barras
Nombre *
Descripción
Precio de venta
...

En lugar de:

Código
Código de barras
Nombre
...

La referencia debería ser obligatoria si ese es el identificador comercial que vamos a utilizar.

También debemos revisar:

validaciones;

fillable;

búsquedas;

importación Excel;

consultas;

cotización;

mostrar producto;

edición;

eliminación;

reportes, si existen.



---

3. Al consultar/buscar productos

La búsqueda debe trabajar con referencia.

Actualmente el controlador tiene una consulta que utiliza:

select(
    'id',
    'codigo',
    'codigo_barras',
    'nombre',
    'precio_venta',
    'descripcion'
)

La nueva estructura deberá utilizar:

select(
    'id',
    'referencia',
    'codigo_barras',
    'nombre',
    'precio_venta',
    'descripcion'
)

Y la interfaz:

Referencia | Código de barras | Nombre | Precio

Así, cuando escribamos:

Buscar producto...

podremos encontrarlo por:

Referencia
Código de barras
Nombre


---

4. Cotización

En la cotización ya no deberíamos tener:

Código

sino:

Referencia

Por ejemplo:

Referencia	Código de barras	Descripción	Cant.	Precio

REF-001	745000123	Laptop Lenovo	1	$500.00


Y esto también debe mantenerse dentro de quoteItems.

{
    id,
    referencia,
    codigo_barras,
    nombre,
    cantidad,
    precio,
    garantia
}


---

5. Excel

Este cambio también afecta la importación.

Actualmente el Excel parece trabajar con:

codigo
cantidad
precio

Deberíamos pasar a:

referencia
cantidad
precio

Por ejemplo:

Referencia	Cantidad	Precio

REF-001	2	500
REF-002	1	25


Y validarLote() deberá buscar:

whereIn('referencia', $referencias)

en lugar de:

whereIn('codigo', $codigos)


---

6. Ahora el segundo cambio: ITBMS por producto

Este punto es incluso más importante para los cálculos.

No podemos asumir:

Todo producto → 7% ITBMS

porque me indicas que existen productos con:

Precio neto

y otros que:

Sí llevan ITBMS

Entonces necesitamos guardar esta condición en el producto.

Yo propondría:

productos
 ├── precio_venta
 └── aplica_itbms

donde:

aplica_itbms = 1 → el producto lleva ITBMS
aplica_itbms = 0 → el producto no lleva ITBMS


---

7. Ejemplo

Supongamos:

Producto A

Referencia: LAP-001
Precio: $500
Aplica ITBMS: Sí

Entonces:

Subtotal: $500.00
ITBMS 7%: $35.00
Total: $535.00

Producto B

Referencia: SERV-001
Precio: $500
Aplica ITBMS: No

Entonces:

Subtotal: $500.00
ITBMS: $0.00
Total: $500.00


---

8. Esto cambia el cálculo de CreateQuoteAction

Actualmente tenemos una lógica global:

$subtotal += cantidad * precio;

$itbms = round($subtotal * $itbmsRate, 2);

Eso significa que todo lo que esté en la cotización recibe ITBMS.

Ya no sería correcto.

Debemos calcularlo producto por producto:

$subtotal = 0;
$itbms = 0;

foreach ($items as $it) {

    $lineSubtotal = (float) $it['cantidad'] * (float) $it['precio'];

    $subtotal += $lineSubtotal;

    if ($producto->aplica_itbms) {
        $itbms += $lineSubtotal * $itbmsRate;
    }
}

Conceptualmente:

PRODUCTO 1
$100 × 7% = $7 ITBMS

PRODUCTO 2
$200 × 0% = $0 ITBMS

PRODUCTO 3
$300 × 7% = $21 ITBMS

-------------------------
Subtotal = $600
ITBMS    = $28
Total    = $628

Esto es mucho más correcto.


---

9. ¿El precio almacenado es neto o con ITBMS?

Aquí tenemos una decisión que debemos dejar muy clara en el sistema.

Si dices:

> "algunos llevan precio neto"



debemos distinguir entre:

Opción A — Precio siempre neto

Precio producto: $100
ITBMS: $7
Total: $107

Opción B — Precio ya incluye ITBMS

Precio producto: $107
ITBMS incluido
Total: $107

No podemos mezclar ambos criterios sin guardar la información.

Por eso recomiendo que el producto tenga dos propiedades independientes:

aplica_itbms
precio_incluye_itbms

Por ejemplo:

Producto	Aplica ITBMS	Precio incluye ITBMS

Laptop	Sí	No
Producto B	No	No
Producto C	Sí	Sí


Esto permite interpretar correctamente el precio.


---

10. Pero podemos simplificarlo

Si en tu negocio todos los precios registrados son precios netos, entonces no necesitamos el segundo campo.

Podríamos manejar solamente:

aplica_itbms

Y establecer como regla:

> precio_venta siempre representa el precio antes de ITBMS.



Entonces:

aplica_itbms = true

→ calcular impuesto.

aplica_itbms = false

→ no calcular impuesto.

Esta sería una estructura bastante limpia para el sistema.


---

11. ¿Dónde guardar el ITBMS de la cotización?

Hay otro detalle importante.

Aunque el producto tenga:

aplica_itbms

yo recomiendo guardar también el resultado en detalle_ventas.

Por ejemplo:

detalle_ventas

venta_id
producto_id
cantidad
precio_unitario
garantia
subtotal
itbms

¿Por qué?

Porque si mañana modificamos el producto:

Producto A
aplica_itbms = true

y posteriormente lo cambiamos a:

aplica_itbms = false

las cotizaciones antiguas no deberían cambiar históricamente.

La cotización debe conservar lo que ocurrió en el momento de crearla.


---

12. Estructura recomendada

Quedaría así:

PRODUCTOS
│
├── referencia
├── codigo_barras
├── nombre
├── descripcion
├── precio_venta
└── aplica_itbms
        │
        ▼
COTIZACIÓN
        │
        ├── Producto
        ├── Cantidad
        ├── Precio
        ├── Garantía
        └── ITBMS
                │
                ▼
DETALLE_VENTAS
        │
        ├── producto_id
        ├── cantidad
        ├── precio_unitario
        ├── garantia
        ├── subtotal
        └── itbms


---

13. Entonces el proceso completo cambia a esto

PRODUCTO
                    │
        ┌───────────┴────────────┐
        │                        │
   REFERENCIA                ITBMS
        │                        │
        │                 ¿Aplica ITBMS?
        │                        │
        └───────────┬────────────┘
                    ↓
              COTIZACIÓN
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Cantidad   Precio   Garantía
                    │
                    ↓
             Cálculo por línea
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
       Subtotal             ITBMS
                              │
                              ↓
                           TOTAL

14. Orden actualizado de cambios

Con estas nuevas necesidades, yo cambiaría el orden que teníamos anteriormente:

Fase 1 — Producto

1. Revisar estructura actual de productos.


2. Identificar todos los usos de codigo.


3. Preparar referencia.


4. Cambiar creación de producto.


5. Cambiar edición de producto.


6. Cambiar consultas/búsquedas.


7. Cambiar importación Excel.


8. Agregar aplica_itbms.



Fase 2 — Cotización

9. Cambiar codigo → referencia.


10. Agregar garantia por línea.


11. Agregar lógica de ITBMS por producto.


12. Modificar CotizacionController.


13. Modificar CreateQuoteAction.


14. Modificar DetalleVenta.



Fase 3 — Visualización

15. City Mall.


16. Outlet Mega.


17. Dólar Mall.


18. show.blade.php.



Fase 4 — Pruebas

19. Producto con ITBMS.


20. Producto sin ITBMS.


21. Cotización con productos mezclados.


22. Garantía independiente por producto.


23. Importación Excel.


24. Verificar totales.


25. Verificar que cotizaciones anteriores no cambien.



Y print.blade.php sigue fuera del alcance.

Antes de modificar código, el siguiente paso correcto es hacer una auditoría de todas las apariciones de codigo y de la estructura real de productos, porque reemplazarlo globalmente sin conocer esas dependencias podría romper consultas, relaciones o importaciones. Después de esa auditoría podemos hacer el cambio de una sola vez y no dejar partes del sistema usando codigo y otras usando referencia.