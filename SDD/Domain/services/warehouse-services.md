# Servicios del Dominio de Bodegas (Warehouse Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la administración de espacios físicos de almacenamiento dentro de NexusMarket. Este dominio gestiona el registro, estado operativo y clasificación de las bodegas (propias del Marketplace o pertenecientes a Vendedores) donde se resguarda el inventario físico.

---

## 1. Especificación Funcional de Bodegas

### Casos de Uso (Input Ports)

* **`RegistrarBodegaInputPort`**
  * **Descripción:** Permite al Administrador registrar un nuevo espacio físico de almacenamiento en el sistema.
  * **Parámetros:** `admin_id: str`, `nombre: str`, `direccion: str`, `es_bodega_marketplace: bool`, `id_vendedor_propietario: Optional[str]`
  * **Respuesta:** `Bodega`

* **`AsignarAdministradorBodegaInputPort`**
  * **Descripción:** Asigna o reasigna a un Administrador o Supervisor la responsabilidad operativa de una bodega.
  * **Parámetros:** `id_bodega: str`, `id_administrador: str`
  * **Respuesta:** `Bodega`

* **`ActualizarEstadoBodegaInputPort`**
  * **Descripción:** Cambia la condición operativa de una bodega (Activa, En Mantenimiento, Suspendida).
  * **Parámetros:** `id_bodega: str`, `activo: bool`
  * **Respuesta:** `Bodega`

* **`ConsultarBodegasPorVendedorInputPort`**
  * **Descripción:** Lista las bodegas autorizadas para almacenar productos de un vendedor específico.
  * **Parámetros:** `id_vendedor: str`
  * **Respuesta:** `List[Bodega]`

---

## 2. Reglas de Negocio Aplicables

1. **Clasificación Obligatoria:** Toda bodega debe quedar expresamente tipificada como `Bodega Marketplace` o `Bodega de Vendedor`.
2. **Registro Centralizado:** La creación y habilitación operativa de una bodega es facultad exclusiva del rol `ADMINISTRADOR`.
3. **Restricción de Despacho:** Una bodega inactiva o en mantenimiento no podrá ser asignada para recepcionar stock ni para despachar ítems de nuevos pedidos.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Bodega

class RepositorioBodegaOutputPort(ABC):
    @abstractmethod
    def guardar(self, bodega: Bodega) -> Bodega: pass

    @abstractmethod
    def buscar_por_id(self, id_bodega: str) -> Optional[Bodega]: pass

    @abstractmethod
    def listar_bodegas_vendedor(self, id_vendedor: str) -> List[Bodega]: pass

    @abstractmethod
    def listar_bodegas_activas(self) -> List[Bodega]: pass