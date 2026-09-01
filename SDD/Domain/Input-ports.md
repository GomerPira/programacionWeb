#Puertos de Entrada (Input Ports) - NexusMarket

## Introducción

Los **Input Ports** (Puertos de Entrada) definen la interfaz abstracta del núcleo de la aplicación. Representan los **Casos de Uso** disponibles en el sistema y actúan como el contrato que los adaptadores de entrada (API REST, GraphQL, CLI, o Controladores) deben invocar para ejecutar la lógica de negocio de NexusMarket.

---

## Interfaces de Puertos de Entrada (`Input Ports`)

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from decimal import Decimal
from datetime import datetime

# Importación de entidades y Objetos de Valor
from domain.models import (
    Usuario, Comprador, Vendedor, Producto, Bodega, Inventario, Carrito, Pedido, Factura, Envio, Devolucion, Reembolso
)
from domain.value_objects import (
    RolUsuario, EstadoUsuario, TipoProducto, EstadoPedido, EstadoProducto
)


# ==========================================
# 1. PUERTOS DE USUARIOS Y PARTICIPANTES
# ==========================================

class RegistrarVendedorInputPort(ABC):
    """Caso de Uso: Permitir únicamente al Administrador incorporar nuevos vendedores."""
    @abstractmethod
    def ejecutar(self, admin_id: str, datos_vendedor: dict) -> Vendedor:
        pass


class AdministrarUsuarioInputPort(ABC):
    """Caso de Uso: Cambiar el estado operativo de un usuario (Activo, Bloqueado, etc.)."""
    @abstractmethod
    def cambiar_estado_usuario(self, id_usuario: str, nuevo_estado: EstadoUsuario) -> Usuario:
        pass


# ==========================================
# 2. PUERTOS DE CATÁLOGO Y PRODUCTOS
# ==========================================

class RegistrarProductoInputPort(ABC):
    """Caso de Uso: Permitir a un vendedor registrar y categorizar un producto."""
    @abstractmethod
    def ejecutar(self, id_vendedor: str, datos_producto: dict, tipo: TipoProducto) -> Producto:
        pass


class CambiarEstadoProductoInputPort(ABC):
    """Caso de Uso: Publicar, suspender o descontinuar productos en el catálogo."""
    @abstractmethod
    def ejecutar(self, id_producto: str, id_vendedor: str, nuevo_estado: EstadoProducto) -> Producto:
        pass


# ==========================================
# 3. PUERTOS DE BODEGAS E INVENTARIO
# ==========================================

class RegistrarBodegaInputPort(ABC):
    """Caso de Uso: Registrar bodegas físicas del Marketplace o de Vendedores."""
    @abstractmethod
    def ejecutar(self, admin_id: str, datos_bodega: dict) -> Bodega:
        pass


class GestionarInventarioInputPort(ABC):
    """Caso de Uso: Ingresar existencias o ajustar inventario distribuido."""
    @abstractmethod
    def ingresar_stock(self, id_bodega: str, id_producto: str, cantidad: int) -> Inventario:
        pass


# ==========================================
# 4. PUERTOS DE CARRITO Y PEDIDOS
# ==========================================

class GestionarCarritoInputPort(ABC):
    """Caso de Uso: Administrar la selección de productos del comprador."""
    @abstractmethod
    def agregar_producto(self, id_comprador: str, id_producto: str, cantidad: int) -> Carrito:
        pass

    @abstractmethod
    def vaciar_carrito(self, id_comprador: str) -> None:
        pass


class RealizarPedidoInputPort(ABC):
    """Caso de Uso: Transformar el carrito en un pedido y validar disponibilidad."""
    @abstractmethod
    def checkout(self, id_comprador: str, direccion_envio: str) -> Pedido:
        pass


# ==========================================
# 5. PUERTOS DE LOGÍSTICA Y FACTURACIÓN
# ==========================================

class FacturarPedidoInputPort(ABC):
    """Caso de Uso: Generar la factura formal tras la confirmación del pago."""
    @abstractmethod
    def generar_factura(self, id_pedido: str) -> Factura:
        pass


class DespacharPedidoInputPort(ABC):
    """Caso de Uso: Asignar operador logístico y generar orden de envío."""
    @abstractmethod
    def asignar_envio(self, id_pedido: str, id_bodega: str, id_operador: str) -> Envio:
        pass


# ==========================================
# 6. PUERTOS DE DEVOLUCIONES Y REEMBOLSOS
# ==========================================

class SolicitarDevolucionInputPort(ABC):
    """Caso de Uso: Iniciar proceso de posventa para un pedido entregado."""
    @abstractmethod
    def solicitar(self, id_pedido: str, id_item: str, motivo: str) -> Devolucion:
        pass


class ProcesarReembolsoInputPort(ABC):
    """Caso de Uso: Aprobar devoluciones y autorizar el reembolso financiero."""
    @abstractmethod
    def procesar(self, id_devolucion: str, id_admin: str) -> Reembolso:
        pass
