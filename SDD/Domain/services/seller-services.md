# Servicios del Dominio de Vendedores (Seller Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la administración e incorporación de vendedores (proveedores) en NexusMarket. Este dominio gestiona los perfiles comerciales, la información fiscal, las bodegas asignadas y la condición operativa de los vendedores en la plataforma.

---

## 1. Especificación Funcional de Vendedores

### Casos de Uso (Input Ports)

* **`RegistrarVendedorInputPort`**
  * **Descripción:** Permite exclusivamente al Administrador incorporar un nuevo vendedor a la plataforma.
  * **Parámetros:** `admin_id: str`, `nombre_comercial: str`, `nit_identificacion_fiscal: str`, `datos_contacto: dict`
  * **Respuesta:** `Vendedor`

* **`ActualizarEstadoVendedorInputPort`**
  * **Descripción:** Permite al Administrador activar, suspender o inactivar el perfil de un vendedor.
  * **Parámetros:** `admin_id: str`, `id_vendedor: str`, `nuevo_estado: EstadoVendedor`
  * **Respuesta:** `Vendedor`

* **`AsignarBodegaVendedorInputPort`**
  * **Descripción:** Vincula una bodega recién registrada al catálogo operativo del vendedor.
  * **Parámetros:** `id_vendedor: str`, `id_bodega: str`
  * **Respuesta:** `bool`

* **`ConsultarVendedorInputPort`**
  * **Descripción:** Obtiene los datos comerciales y el estado operativo actual de un vendedor.
  * **Parámetros:** `id_vendedor: str`
  * **Respuesta:** `Vendedor`

---

## 2. Reglas de Negocio Aplicables

1. **Restricción de Auto-registro:** Los vendedores no pueden registrarse por sí mismos; la creación del perfil debe ser ejecutada obligatoriamente por un usuario con rol `ADMINISTRADOR`.
2. **Identificación Fiscal Única:** El NIT o documento de identificación fiscal del vendedor debe ser único en la plataforma.
3. **Impacto por Suspensión:** Si un vendedor pasa a estado `SUSPENDIDO` o `INACTIVO`, todos sus productos en el catálogo cambiarán automáticamente a estado `SUSPENDIDO`, impidiendo nuevas compras.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Vendedor, Bodega

class RepositorioVendedorOutputPort(ABC):
    @abstractmethod
    def guardar(self, vendedor: Vendedor) -> Vendedor: pass

    @abstractmethod
    def buscar_por_id(self, id_vendedor: str) -> Optional[Vendedor]: pass

    @abstractmethod
    def buscar_por_nit(self, nit: str) -> Optional[Vendedor]: pass

class RepositorioBodegaVendedorOutputPort(ABC):
    @abstractmethod
    def asociar_bodega(self, id_vendedor: str, id_bodega: str) -> bool: pass