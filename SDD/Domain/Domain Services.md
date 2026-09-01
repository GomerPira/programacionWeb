# Servicios de Dominio (Domain Services) - NexusMarket

## Introducción

Los Servicios de Dominio encapsulan lógica y reglas de negocio puras que involucran múltiples entidades o agregados del sistema y que no pertenecen orgánicamente a una sola entidad. A diferencia de los Servicios de Aplicación, los Servicios de Dominio no manejan persistencia ni infraestructura; únicamente ejecutan la lógica del Marketplace.

---

## Servicios de Dominio Principales

### 1. `ServicioProcesamientoPedido`
Coordina la validación y transformación de un carrito de compras en un pedido formal dentro del Marketplace.

* **Responsabilidades:**
  * Verificar que el comprador tenga un estado comercial activo.
  * Validar la disponibilidad de stock en las bodegas correspondientes para cada ítem del carrito.
  * Calcular el total del pedido incluyendo precios de variantes y reglas comerciales.
* **Firma / Interfaz:**
  ```python
  class ServicioProcesamientoPedido:
      def procesar_compra(
          self, 
          comprador: Comprador, 
          carrito: Carrito, 
          inventarios_disponibles: List[Inventario]
      ) -> Pedido:
          """
          Regla de Negocio:
          1. Valida que el comprador esté en estado ACTIVO.
          2. Valida que cada producto tenga existencias suficientes en su bodega.
          3. Genera la entidad Pedido en estado PENDIENTE_PAGO.
          """
          pass
2. ServicioReservaInventario
Administra el control de existencias distribuidas entre bodegas del Marketplace y bodegas de Vendedores.

Responsabilidades:

Seleccionar la mejor bodega para despachar un producto según disponibilidad.

Ejecutar la reserva de inventario garantizando que no existan saldos negativos.

Firma / Interfaz:

Python
class ServicioReservaInventario:
    def reservar_stock_pedido(
        self, 
        pedido: Pedido, 
        inventarios: List[Inventario]
    ) -> List[Inventario]:
        """
        Regla de Negocio:
        - Asigna el ítem a la bodega con suficiente stock.
        - Descuenta del saldo disponible e incrementa el saldo reservado.
        - Lanza una ExcepcionDominio si el stock es insuficiente.
        """
        pass
3. ServicioCalculoFacturacion
Calcula los desgloses financieros, subtotales e impuestos para la emisión de facturas legales.

Responsabilidades:

Aplicar tarifas de impuestos según la categoría y tipo de producto (Físico vs Digital).

Consolidar los totales facturables por pedido.

Firma / Interfaz:

Python
class ServicioCalculoFacturacion:
    def generar_factura_pedido(
        self, 
        pedido: Pedido, 
        porcentaje_impuesto: Decimal
    ) -> Factura:
        """
        Regla de Negocio:
        - Aplica la tasa impositiva sobre el total de productos físicos/digitales.
        - Genera la Factura asociada al Pedido en estado EMITIDA.
        """
        pass
4. ServicioProcesamientoDevoluciones
Evalúa la viabilidad de la devolución de un producto y determina el cálculo del reembolso.

Responsabilidades:

Validar las condiciones del producto según las reglas de postventa.

Determinar si procede una devolución parcial o total y emitir la orden de reembolso.

Firma / Interfaz:

Python
class ServicioProcesamientoDevoluciones:
    def evaluar_y_crear_reembolso(
        self, 
        devolucion: Devolucion, 
        factura: Factura
    ) -> Reembolso:
        """
        Regla de Negocio:
        - Si la devolución es aprobada y la factura está PAGADA, calcula 
          el monto exacto a devolver al Comprador.
        """
        pass