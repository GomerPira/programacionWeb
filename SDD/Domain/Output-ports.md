# Puertos de Salida (Output Ports) - NexusMarket

## Introducción

Los **Output Ports** (Puertos de Salida) definen los contratos e interfaces abstractas que el núcleo del dominio necesita invocando hacia la infraestructura externa. Incluyen el acceso a la persistencia de datos (Repositorios) e integraciones con servicios externos como pasarelas de pago, servicios de facturación electrónica, operadores logísticos y envío de notificaciones.

---

## Interfaces de Puertos de Salida (`Output Ports`)

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from decimal import Decimal

from domain.models import (
    Usuario, Comprador, Vendedor, Bodega, Producto, Inventario, Carrito, Pedido, Factura, Envio, Devolucion, Reembolso
)


# ==========================================
# 1. PUERTOS DE REPOSITORIOS (PERSISTENCIA)
# ==========================================

class RepositorioUsuarioOutputPort(ABC):
    """Interfaz de persistencia para Usuarios y sus perfiles asociados."""
    @abstractmethod
    def guardar(self, usuario: Usuario) -> Usuario:
        pass

    @abstractmethod
    def buscar_por_id(self, id_usuario: str) -> Optional[Usuario]:
        pass

    @abstractmethod
    def buscar_por_correo(self, correo: str) -> Optional[Usuario]:
        pass


class RepositorioProductoOutputPort(ABC):
    """Interfaz de persistencia para el catálogo de productos."""
    @abstractmethod
    def guardar(self, producto: Producto) -> Producto:
        pass

    @abstractmethod
    def buscar_por_id(self, id_producto: str) -> Optional[Producto]:
        pass

    @abstractmethod
    def listar_por_vendedor(self, id_vendedor: str) -> List[Producto]:
        pass


class RepositorioInventarioOutputPort(ABC):
    """Interfaz de persistencia para control de existencias en bodegas."""
    @abstractmethod
    def guardar(self, inventario: Inventario) -> Inventario:
        pass

    @abstractmethod
    def buscar_por_producto_y_bodega(self, id_producto: str, id_bodega: str) -> Optional[Inventario]:
        pass

    @abstractmethod
    def obtener_existencias_totales(self, id_producto: str) -> List[Inventario]:
        pass


class RepositorioPedidoOutputPort(ABC):
    """Interfaz de persistencia para la gestión de pedidos."""
    @abstractmethod
    def guardar(self, pedido: Pedido) -> Pedido:
        pass

    @abstractmethod
    def buscar_por_id(self, id_pedido: str) -> Optional[Pedido]:
        pass

    @abstractmethod
    def listar_por_comprador(self, id_comprador: str) -> List[Pedido]:
        pass


# ==========================================
# 2. PUERTOS DE SERVICIOS EXTERNOS (ADAPTERS)
# ==========================================

class PasarelaPagoOutputPort(ABC):
    """Interfaz para procesamiento financiero con pasarelas externas (Stripe, MercadoPago, etc.)."""
    @abstractmethod
    def procesar_cobro(self, id_pedido: str, monto: Decimal, token_pago: str) -> bool:
        pass

    @abstractmethod
    def ejecutar_reembolso(self, id_reembolso: str, monto: Decimal) -> bool:
        pass


class OperadorLogisticoOutputPort(ABC):
    """Interfaz de comunicación con las API de los operadores de transporte."""
    @abstractmethod
    def solicitar_guia_despacho(self, envio: Envio) -> str:
        """Retorna el número de guía de seguimiento generado por el transportista."""
        pass


class ServicioNotificacionesOutputPort(ABC):
    """Interfaz para envío de correos, SMS o notificaciones push."""
    @abstractmethod
    def enviar_confirmacion_pedido(self, correo_comprador: str, pedido: Pedido) -> None:
        pass

    @abstractmethod
    def notificar_vendedor_nueva_venta(self, correo_vendedor: str, id_pedido: str) -> None:
        pass