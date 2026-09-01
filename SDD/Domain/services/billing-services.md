# Servicios del Dominio de Facturación y Devoluciones (Billing & Refund Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para los procesos de generación de facturas comerciales, procesamiento de devoluciones y gestión de reembolsos dentro del Marketplace NexusMarket.

---

## 1. Especificación Funcional de Facturación

### Casos de Uso (Input Ports)

* **`GenerarFacturaInputPort`**
  * **Descripción:** Genera el comprobante de venta e impuestos inmediatamente después de confirmarse el pago del pedido.
  * **Parámetros:** `id_pedido: str`
  * **Respuesta:** `Factura`

* **`ConsultarFacturaInputPort`**
  * **Descripción:** Permite al Comprador o Administrador consultar los detalles contables de una compra.
  * **Parámetros:** `id_factura: str`
  * **Respuesta:** `Factura`

### Reglas de Negocio Aplicables

1. **Inmutabilidad:** Una vez emitida la factura, no se pueden modificar montos, impuestos ni ítems.
2. **Cálculo de Impuestos:** Los productos físicos y digitales aplican tasas de impuestos configuradas según su categoría comercial.
3. **Generación Automática:** Toda compra con estado `PAGADO` genera automáticamente una factura en estado `EMITIDA`.

---

## 2. Especificación Funcional de Devoluciones y Reembolsos

### Casos de Uso (Input Ports)

* **`SolicitarDevolucionInputPort`**
  * **Descripción:** Registra la solicitud de un comprador para devolver un ítem de un pedido entregado.
  * **Parámetros:** `id_pedido: str`, `id_item: str`, `motivo: str`
  * **Respuesta:** `Devolucion`

* **`ProcesarAprobacionDevolucionInputPort`**
  * **Descripción:** Evaluación ejecutada por el Administrador o Supervisor para aprobar o rechazar la devolución.
  * **Parámetros:** `id_devolucion: str`, `aprobado: bool`, `observaciones: str`
  * **Respuesta:** `Devolucion`

* **`EmitirReembolsoInputPort`**
  * **Descripción:** Inicia la transacción financiera de reembolso al comprador tras la aprobación de la devolución.
  * **Parámetros:** `id_devolucion: str`
  * **Respuesta:** `Reembolso`

### Reglas de Negocio Aplicables

1. **Restricción de Estado del Pedido:** Solo se aceptan solicitudes de devolución para pedidos en estado `ENTREGADO`.
2. **Reintegración al Inventario:** Si el producto devuelto se declara en buen estado, el servicio coordinará con el Dominio de Inventario el reingreso del stock a la bodega física.
3. **Monto Máximo de Reembolso:** El monto de un reembolso no podrá ser superior al valor unitario pagado en la factura original para ese ítem específico.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from decimal import Decimal
from domain.models import Factura, Devolucion, Reembolso

class RepositorioFacturaOutputPort(ABC):
    @abstractmethod
    def guardar(self, factura: Factura) -> Factura: pass
    
    @abstractmethod
    def buscar_por_id(self, id_factura: str) -> Factura: pass

class RepositorioDevolucionOutputPort(ABC):
    @abstractmethod
    def guardar(self, devolucion: Devolucion) -> Devolucion: pass

class PasarelaReembolsoOutputPort(ABC):
    @abstractmethod