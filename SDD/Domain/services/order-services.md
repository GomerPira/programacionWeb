# Servicios del Dominio de Pedidos (Order Services)

## Introducción

Este documento especifica la lógica funcional y las reglas de negocio que rigen el ciclo de vida central de las transacciones comerciales en NexusMarket. Este dominio administra la gestión del carrito de compras, la creación y validación de pedidos, los cambios de estado operativo y la cancelación de solicitudes.

---

## 1. Especificación Funcional de Pedidos

### Casos de Uso (Input Ports)

* **`AgregarAlCarritoInputPort`**
  * **Descripción:** Permite a un comprador agregar productos o variantes específicas a su carrito de compras provisional.
  * **Parámetros:** `id_comprador: str`, `id_producto: str`, `cantidad: int`, `id_variante: Optional[str]`
  * **Respuesta:** `Carrito`

* **`CrearPedidoDesdeCarritoInputPort`**
  * **Descripción:** Transforma el carrito de compras en un pedido formal, validando precios y la disponibilidad de inventario.
  * **Parámetros:** `id_comprador: str`, `direccion_envio: str`
  * **Respuesta:** `Pedido`

* **`ConfirmarPagoPedidoInputPort`**
  * **Descripción:** Transiciona el estado del pedido de `PENDIENTE_PAGO` a `PAGADO` tras la validación financiera.
  * **Parámetros:** `id_pedido: str`, `referencia_pago: str`
  * **Respuesta:** `Pedido`

* **`CancelarPedidoInputPort`**
  * **Descripción:** Cancela un pedido que no ha sido despachado, liberando las reservas de inventario asociadas.
  * **Parámetros:** `id_pedido: str`, `motivo: str`
  * **Respuesta:** `Pedido`

---

## 2. Reglas de Negocio Aplicables

1. **Inmutabilidad de Pedidos Finalizados:** Un pedido en estado `ENTREGADO` o `CANCELADO` no podrá ser modificado bajo ninguna circunstancia.
2. **Validación de Comprador Activo:** Únicamente los compradores en estado comercial `ACTIVO` pueden crear carritos y procesar pedidos.
3. **Limpieza del Carrito:** Tras la conversión exitosa a `Pedido`, el carrito del comprador se vacía automáticamente.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Pedido, Carrito

class RepositorioPedidoOutputPort(ABC):
    @abstractmethod
    def guardar_pedido(self, pedido: Pedido) -> Pedido: pass

    @abstractmethod
    def buscar_por_id(self, id_pedido: str) -> Optional[Pedido]: pass

class RepositorioCarritoOutputPort(ABC):
    @abstractmethod
    def guardar_carrito(self, carrito: Carrito) -> Carrito: pass

    @abstractmethod
    def buscar_por_comprador(self, id_comprador: str) -> Optional[Carrito]: pass