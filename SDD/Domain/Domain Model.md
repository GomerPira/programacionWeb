# Modelo de Dominio Central (Domain Model) - NexusMarket

## Introducción

Este documento define las entidades principales del sistema NexusMarket aplicando principios de Diseño Guiado por el Dominio (DDD) y Programación Orientada a Objetos. La herencia se utiliza para especializar roles y productos, mientras que las relaciones entre objetos se representan explícitamente sin usar identificadores primitivos dispersos.

---

## Jerarquía de Clases de Dominio

```text
Persona (Abstracta)
├── Comprador
├── Vendedor
├── OperadorLogistico
├── Administrador
└── Supervisor

Usuario

Producto (Abstracto)
├── ProductoFisico
└── ProductoDigital

VarianteProducto
Bodega
Inventario
Carrito / ItemCarrito
Pedido / ItemPedido
Factura
Envio
Devolucion
Reembolso
Definición de Entidades en Python (Domain Model.md)
Python
from abc import ABC
from dataclasses import dataclass, field
from datetime import date, datetime
from decimal import Decimal
from typing import List, Optional, Dict, Any

from domain.value_objects import (
    RolUsuario, EstadoUsuario, EstadoComprador, EstadoVendedor,
    TipoProducto, EstadoProducto, EstadoPedido, EstadoEnvio,
    EstadoFactura, EstadoDevolucion, EstadoReembolso
)


# ==========================================
# JERARQUÍA DE USUARIOS Y PARTICIPANTES
# ==========================================

@dataclass
class Persona(ABC):
    """Representa a cualquier persona con identidad dentro del Marketplace."""
    identificacion: str
    nombre_completo: str
    correo_electronico: str
    telefono: str


@dataclass
class Comprador(Persona):
    """Representa al usuario que realiza compras en la plataforma."""
    direccion_principal: str
    direcciones_adicionales: List[str] = field(default_factory=list)
    estado_comercial: EstadoComprador = EstadoComprador.ACTIVO


@dataclass
class Vendedor(Persona):
    """Representa al proveedor comercial de productos."""
    nombre_comercial: str
    nit_identificacion_fiscal: str
    estado_vendedor: EstadoVendedor = EstadoVendedor.ACTIVO


@dataclass
class OperadorLogistico(Persona):
    """Encargado de la operación física en bodegas y despacho de envíos."""
    codigo_operador: str


@dataclass
class Administrador(Persona):
    """Responsable del registro de vendedores y administración de bodegas."""
    nivel_acceso: str = "SUPER_ADMIN"


@dataclass
class Supervisor(Persona):
    """Perfil de consulta y seguimiento de la operación comercial."""
    departamento: str


@dataclass
class Usuario:
    """Entidad de identidad en el sistema para autenticación y autorización."""
    id_usuario: str
    nombre_usuario: str
    contrasena_hash: str
    rol: RolUsuario
    estado: EstadoUsuario
    persona_asociada: Persona  # Vincula el perfil específico (Comprador, Vendedor, etc.)


# ==========================================
# CATÁLOGO, PRODUCTOS Y VARIANTES
# ==========================================

@dataclass
class VarianteProducto:
    """Representa atributos específicos de un producto (talla, color, modelo)."""
    id_variante: str
    sku: str
    atributo: str  # Ej: "Talla"
    valor: str     # Ej: "XL"
    precio_adicional: Decimal = Decimal("0.00")


@dataclass
class Producto(ABC):
    """Representa un bien o servicio en el catálogo de NexusMarket."""
    id_producto: str
    titulo: str
    descripcion: str
    precio_base: Decimal
    vendedor: Vendedor
    tipo_producto: TipoProducto
    estado: EstadoProducto
    variantes: List[VarianteProducto] = field(default_factory=list)


@dataclass
class ProductoFisico(Producto):
    """Producto que requiere almacenamiento físico en bodega y logística de despacho."""
    peso_kg: float
    dimensiones_cm: str  # Ej: "30x20x10"
    requiere_empaque_especial: bool = False


@dataclass
class ProductoDigital(Producto):
    """Producto descargable o intangible entregado de forma inmediata tras el pago."""
    url_descarga: str
    clave_licencia: Optional[str] = None
    tamano_mb: float = 0.0


# ==========================================
# BODEGAS E INVENTARIO DISTRIBUIDO
# ==========================================

@dataclass
class Bodega:
    """Representa el espacio físico donde se almacena el inventario."""
    id_bodega: str
    nombre: str
    direccion: str
    es_bodega_marketplace: bool  # True si es propia del Marketplace, False si es del Vendedor
    administrador_cargo: Optional[Administrador] = None


@dataclass
class Inventario:
    """Registra las existencias de un producto o variante dentro de una bodega específica."""
    id_inventario: str
    producto: Producto
    bodega: Bodega
    cantidad_disponible: int
    cantidad_reservada: int
    variante: Optional[VarianteProducto] = None

    def reservar(self, cantidad: int) -> None:
        """Regla de Negocio: No permite saldo o reservas negativas."""
        if cantidad > self.cantidad_disponible:
            raise ValueError("No hay suficiente inventario disponible para reservar.")
        self.cantidad_disponible -= cantidad
        self.cantidad_reservada += cantidad


# ==========================================
# CARRITO Y PEDIDOS
# ==========================================

@dataclass
class ItemCarrito:
    producto: Producto
    cantidad: int
    precio_unitario: Decimal
    variante: Optional[VarianteProducto] = None


@dataclass
class Carrito:
    id_carrito: str
    comprador: Comprador
    items: List[ItemCarrito] = field(default_factory=list)

    @property
    def total(self) -> Decimal:
        return sum((item.precio_unitario * item.cantidad) for item in self.items)


@dataclass
class ItemPedido:
    id_item: str
    producto: Producto
    cantidad: int
    precio_unitario: Decimal
    bodega_origen: Optional[Bodega] = None
    variante: Optional[VarianteProducto] = None


@dataclass
class Pedido:
    """Representa el compromiso comercial de compra en la plataforma."""
    id_pedido: str
    comprador: Comprador
    items: List[ItemPedido]
    monto_total: Decimal
    estado: EstadoPedido
    fecha_creacion: datetime
    direccion_envio: str


# ==========================================
# FACTURACIÓN, LOGÍSTICA Y POSTVENTA
# ==========================================

@dataclass
class Factura:
    """Documento legal y comercial generado tras confirmar la compra."""
    id_factura: str
    pedido: Pedido
    numero_factura: str
    fecha_emision: datetime
    monto_subtotal: Decimal
    monto_impuestos: Decimal
    monto_total: Decimal
    estado: EstadoFactura


@dataclass
class Envio:
    """Proceso logístico asociado al transporte y entrega de productos físicos."""
    id_envio: str
    pedido: Pedido
    bodega_origen: Bodega
    operador_logistico: OperadorLogistico
    numero_guia: str
    estado: EstadoEnvio
    fecha_despacho: Optional[datetime] = None
    fecha_entrega_estimada: Optional[date] = None


@dataclass
class Devolucion:
    id_devolucion: str
    pedido: Pedido
    motivo: str
    fecha_solicitud: datetime
    estado: EstadoDevolucion
    item_afectado: ItemPedido


@dataclass
class Reembolso:
    id_reembolso: str
    devolucion: Devolucion
    monto_reembolsado: Decimal
    fecha_procesamiento: datetime
    estado: EstadoReembolso