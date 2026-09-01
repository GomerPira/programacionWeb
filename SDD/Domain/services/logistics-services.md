# Servicios del Dominio de Logística (Logistics Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la gestión logística, empaque, despacho y seguimiento de envíos físicos dentro del Marketplace NexusMarket. Este dominio coordina la interacción entre las bodegas, los pedidos pagados y los operadores logísticos externos.

---

## 1. Especificación Funcional de Logística

### Casos de Uso (Input Ports)

* **`GenerarOrdenEnvioInputPort`**
  * **Descripción:** Crea la orden de despacho para un pedido cuyo pago ha sido confirmado.
  * **Parámetros:** `id_pedido: str`, `id_bodega_origen: str`, `direccion_destino: str`
  * **Respuesta:** `Envio`

* **`AsignarOperadorLogisticoInputPort`**
  * **Descripción:** Asigna la empresa o personal encargado del transporte y vincula el número de guía de seguimiento.
  * **Parámetros:** `id_envio: str`, `id_operador: str`, `numero_guia: str`
  * **Respuesta:** `Envio`

* **`ActualizarEstadoEnvioInputPort`**
  * **Descripción:** Registra los hitos del transporte del paquete (En preparación, Despachado, En tránsito, Entregado).
  * **Parámetros:** `id_envio: str`, `nuevo_estado: EstadoEnvio`
  * **Respuesta:** `Envio`

* **`ConfirmarEntregaInputPort`**
  * **Descripción:** Marca el envío como concluido tras la confirmación de recepción por parte del comprador.
  * **Parámetros:** `id_envio: str`, `fecha_entrega: datetime`
  * **Respuesta:** `Envio`

---

## 2. Reglas de Negocio Aplicables

1. **Exclusividad para Productos Físicos:** Los procesos logísticos de transporte y generación de guías aplican únicamente para ítems de tipo `FISICO`. Los productos `DIGITALES` omiten completamente este dominio.
2. **Despacho Previo Pago:** No se puede generar una orden de envío si el pedido no se encuentra en estado `PAGADO`.
3. **Inmutabilidad post-entrega:** Un envío en estado `ENTREGADO` no puede cambiar de estado ni modificar sus datos de origen/destino.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional
from domain.models import Envio

class RepositorioEnvioOutputPort(ABC):
    @abstractmethod
    def guardar(self, envio: Envio) -> Envio: pass

    @abstractmethod
    def buscar_por_id(self, id_envio: str) -> Optional[Envio]: pass

    @abstractmethod
    def buscar_por_pedido(self, id_pedido: str) -> Optional[Envio]: pass

class OperadorLogisticoExternoOutputPort(ABC):
    @abstractmethod
    def solicitar_guia(self, envio: Envio) -> str: pass