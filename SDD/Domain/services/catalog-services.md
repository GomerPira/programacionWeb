# Servicios del Dominio de Catálogo (Catalog Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la administración del catálogo de productos dentro del Marketplace NexusMarket. Este dominio gestiona la creación de productos físicos y digitales, variantes (talla, color, modelo) y su ciclo de publicación comercial.

---

## 1. Especificación Funcional del Catálogo

### Casos de Uso (Input Ports)

* **`RegistrarProductoInputPort`**
  * **Descripción:** Permite a un Vendedor autenticado registrar un nuevo producto en el sistema, especificando si es un bien físico o digital.
  * **Parámetros:** `id_vendedor: str`, `titulo: str`, `descripcion: str`, `precio_base: Decimal`, `tipo: TipoProducto`
  * **Respuesta:** `Producto`

* **`AgregarVarianteProductoInputPort`**
  * **Descripción:** Permite asociar variaciones de un producto existente (ej. Talla XL, Color Azul) con sus respectivos impactos en costo.
  * **Parámetros:** `id_producto: str`, `sku: str`, `atributo: str`, `valor: str`, `precio_adicional: Decimal`
  * **Respuesta:** `VarianteProducto`

* **`CambiarEstadoProductoInputPort`**
  * **Descripción:** Permite al Vendedor o Administrador publicar, suspender o descontinuar la visibilidad de un producto en el catálogo.
  * **Parámetros:** `id_producto: str`, `nuevo_estado: EstadoProducto`
  * **Respuesta:** `Producto`

* **`BuscarProductosCatalogoInputPort`**
  * **Descripción:** Permite la búsqueda pública de productos activos según filtros de categoría, tipo o palabras clave.
  * **Parámetros:** `criterio_busqueda: str`, `tipo: Optional[TipoProducto]`
  * **Respuesta:** `List[Producto]`

---

## 2. Reglas de Negocio Aplicables

1. **Propiedad del Producto:** Un vendedor solo puede modificar o actualizar productos y variantes de los cuales sea el propietario directo.
2. **Diferenciación Operativa:** Los productos de tipo `DIGITAL` no requieren configuración de dimensiones ni peso, mientras que los de tipo `FISICO` obligan a ingresar sus métricas para el cálculo logístico.
3. **Visibilidad Comercial:** Un producto en estado `SUSPENDIDO`, `DESCONTINUADO` o `BORRADOR` no podrá ser seleccionado en el proceso de compra ni agregado a un carrito.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Producto, VarianteProducto

class RepositorioProductoOutputPort(ABC):
    @abstractmethod
    def guardar(self, producto: Producto) -> Producto: pass

    @abstractmethod
    def buscar_por_id(self, id_producto: str) -> Optional[Producto]: pass

    @abstractmethod
    def listar_por_vendedor(self, id_vendedor: str) -> List[Producto]: pass

class RepositorioVarianteOutputPort(ABC):
    @abstractmethod
    def guardar_variante(self, id_producto: str, variante: VarianteProducto) -> VarianteProducto: pass