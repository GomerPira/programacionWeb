# Servicios del Dominio de Compradores (Buyer Services)

## Introducción

Este documento define la especificación funcional y reglas de negocio para la gestión de compradores dentro del Marketplace NexusMarket. Este dominio administra las direcciones de entrega, la información comercial del comprador y su estado para interactuar en la plataforma.

---

## 1. Especificación Funcional de Compradores

### Casos de Uso (Input Ports)

* **`RegistrarCompradorInputPort`**
  * **Descripción:** Permite a un nuevo comprador crear su perfil comercial en la plataforma para realizar compras.
  * **Parámetros:** `id_usuario: str`, `direccion_principal: str`, `telefono: str`
  * **Respuesta:** `Comprador`

* **`GestionarDireccionesInputPort`**
  * **Descripción:** Permite al comprador agregar, actualizar o eliminar direcciones adicionales para el despacho de sus pedidos.
  * **Parámetros:** `id_comprador: str`, `nueva_direccion: str`
  * **Respuesta:** `List[str]`

* **`ConsultarHistorialComprasInputPort`**
  * **Descripción:** Permite al comprador consultar la lista de sus pedidos realizados en la plataforma.
  * **Parámetros:** `id_comprador: str`
  * **Respuesta:** `List[Pedido]`

* **`ActualizarEstadoComercialInputPort`**
  * **Descripción:** Permite al Administrador cambiar la condición comercial de un comprador (ej. Activo o Suspendido).
  * **Parámetros:** `id_comprador: str`, `nuevo_estado: EstadoComprador`
  * **Respuesta:** `Comprador`

---

## 2. Reglas de Negocio Aplicables

1. **Aislamiento de Información:** Un comprador nunca podrá visualizar, administrar ni modificar la información personal o comercial de otros compradores.
2. **Dirección Obligatoria:** Todo comprador debe tener configurada al menos una dirección principal de entrega activa para iniciar el proceso de checkout.
3. **Bloqueo de Transacciones:** Si el `EstadoComprador` se encuentra en estado `SUSPENDIDO`, el dominio rechazará cualquier intento de agregar productos al carrito o procesar un pedido.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Comprador, Pedido

class RepositorioCompradorOutputPort(ABC):
    @abstractmethod
    def guardar(self, comprador: Comprador) -> Comprador: pass

    @abstractmethod
    def buscar_por_id(self, id_comprador: str) -> Optional[Comprador]: pass

    @abstractmethod
    def obtener_historial_pedidos(self, id_comprador: str) -> List[Pedido]: pass