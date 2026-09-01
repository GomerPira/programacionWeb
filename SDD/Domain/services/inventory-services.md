# Servicios del Dominio de Inventario (Inventory Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la administración del inventario distribuido y control de bodegas dentro del Marketplace NexusMarket. Este dominio gestiona las existencias físicas de productos por bodega, las reservas temporales por compra y los movimientos de stock.

---

## 1. Especificación Funcional de Inventario

### Casos de Uso (Input Ports)

* **`IngresarStockInputPort`**
  * **Descripción:** Permite registrar o incrementar existencias de un producto específico en una bodega determinada.
  * **Parámetros:** `id_bodega: str`, `id_producto: str`, `cantidad: int`, `id_variante: Optional[str]`
  * **Respuesta:** `Inventario`

* **`ReservarStockInputPort`**
  * **Descripción:** Bloquea temporalmente unidades de stock para garantizar disponibilidad durante el proceso de compra.
  * **Parámetros:** `id_producto: str`, `cantidad: int`, `id_bodega: str`
  * **Respuesta:** `bool`

* **`LiberarReservaInputPort`**
  * **Descripción:** Restaura las unidades reservadas a stock disponible si el pedido es cancelado o no se concreta el pago.
  * **Parámetros:** `id_producto: str`, `cantidad: int`, `id_bodega: str`
  * **Respuesta:** `bool`

* **`AjustarInventarioInputPort`**
  * **Descripción:** Registrar ajustes de inventario por pérdidas, mermas o productos dañados.
  * **Parámetros:** `id_bodega: str`, `id_producto: str`, `cantidad_ajuste: int`, `motivo: str`
  * **Respuesta:** `Inventario`

---

## 2. Reglas de Negocio Aplicables

1. **Prohibición de Saldos Negativos:** Bajo ninguna circunstancia se permitirá registrar existencias negativas ni reservar más unidades de las disponibles.
2. **Asociación Obligatoria:** Todo registro de inventario físico debe estar vinculado obligatoriamente a una bodega válida y a un producto activo del catálogo.
3. **Control por Variante:** Si un producto posee variantes (ej. Talla / Color), el inventario debe gestionarse a nivel de variante de forma independiente.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Inventario, Bodega

class RepositorioInventarioOutputPort(ABC):
    @abstractmethod
    def guardar(self, inventario: Inventario) -> Inventario: pass

    @abstractmethod
    def buscar_por_bodega_y_producto(self, id_bodega: str, id_producto: str) -> Optional[Inventario]: pass

    @abstractmethod
    def consultar_existencias_totales(self, id_producto: str) -> List[Inventario]: pass