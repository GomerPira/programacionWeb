# Objetos de Valor (Domain Value Objects) - NexusMarket

## Introducción

Este documento define los **Value Objects** (Objetos de Valor) y **Enums** del dominio de NexusMarket. A diferencia de las Entidades, los Objetos de Valor no poseen una identidad única (`id`), son inmutables y quedan definidos puramente por sus valores/atributos. Representan estados, catálogos fijos y estructuras de datos compuestas.

---

## Catalogos y Enumeraciones (Enums)

```python
from enum import Enum
from dataclasses import dataclass
from decimal import Decimal


# ==========================================
# USUARIOS Y ACCESOS
# ==========================================

class RolUsuario(str, Enum):
    """Define las responsabilidades y permisos asignados."""
    COMPRADOR = "COMPRADOR"
    VENDEDOR = "VENDEDOR"
    OPERADOR_LOGISTICO = "OPERADOR_LOGISTICO"
    ADMINISTRADOR = "ADMINISTRADOR"
    SUPERVISOR = "SUPERVISOR"


class EstadoUsuario(str, Enum):
    """Condición operativa general del usuario para acceder a la plataforma."""
    ACTIVO = "ACTIVO"
    BLOQUEADO = "BLOQUEADO"
    PENDIENTE_ACTIVACION = "PENDIENTE_ACTIVACION"


class EstadoComprador(str, Enum):
    """Condición comercial del comprador para realizar transacciones."""
    ACTIVO = "ACTIVO"
    SUSPENDIDO = "SUSPENDIDO"


class EstadoVendedor(str, Enum):
    """Estado operativo del vendedor en la plataforma."""
    ACTIVO = "ACTIVO"
    SUSPENDIDO = "SUSPENDIDO"
    INACTIVO = "INACTIVO"


# ==========================================
# CATÁLOGO Y PRODUCTOS
# ==========================================

class TipoProducto(str, Enum):
    """Diferencia la naturaleza de distribución del producto."""
    FISICO = "FISICO"
    DIGITAL = "DIGITAL"


class EstadoProducto(str, Enum):
    """Disponibilidad y visibilidad comercial en el catálogo público."""
    PUBLICADO = "PUBLICADO"
    SUSPENDIDO = "SUSPENDIDO"
    DESCONTINUADO = "DESCONTINUADO"
    BORRADOR = "BORRADOR"


# ==========================================
# PEDIDOS Y TRANSACCIONES
# ==========================================

class EstadoPedido(str, Enum):
    """Ciclo de vida operativo y central del pedido."""
    CARRITO = "CARRITO"
    PENDIENTE_PAGO = "PENDIENTE_PAGO"
    PAGADO = "PAGADO"
    DESPACHADO = "DESPACHADO"
    ENTREGADO = "ENTREGADO"
    CANCELADO = "CANCELADO"


# ==========================================
# LOGÍSTICA Y ENVÍOS
# ==========================================

class EstadoEnvio(str, Enum):
    """Estados del proceso físico de empaque y transporte."""
    EN_PREPARACION = "EN_PREPARACION"
    DESPACHADO = "DESPACHADO"
    EN_TRANSITO = "EN_TRANSITO"
    ENTREGADO = "ENTREGADO"
    NO_ENTREGADO = "NO_ENTREGADO"


# ==========================================
# FACTURACIÓN, DEVOLUCIONES Y REEMBOLSOS
# ==========================================

class EstadoFactura(str, Enum):
    EMITIDA = "EMITIDA"
    PAGADA = "PAGADA"
    ANULADA = "ANULADA"


class EstadoDevolucion(str, Enum):
    SOLICITADA = "SOLICITADA"
    EN_REVISION = "EN_REVISION"
    APROBADA = "APROBADA"
    RECHAZADA = "RECHAZADA"


class EstadoReembolso(str, Enum):
    PENDIENTE = "PENDIENTE"
    PROCESADO = "PROCESADO"
    FALLIDO = "FALLIDO"
Objetos de Valor Compuestos (Value Objects)
Representan atributos complejos e inmutables dentro del dominio que no requieren un ciclo de vida propio ni identificador.

Python
@dataclass(frozen=True)
class DireccionEntrega:
    """Objeto de valor inmutable para direcciones de despacho."""
    calle_principal: str
    numero: str
    ciudad: str
    departamento_estado: str
    codigo_postal: str
    referencia: str = ""


@dataclass(frozen=True)
class Dimensiones:
    """Objeto de valor para volumen físico de un producto."""
    alto_cm: float
    ancho_cm: float
    largo_cm: float

    @property
    def volumen_cc(self) -> float:
        return self.alto_cm * self.ancho_cm * self.largo_cm


@dataclass(frozen=True)
class Moneda:
    """Objeto de valor para manejo explícito de importes financieros."""
    monto: Decimal
    codigo_iso: str = "COP"  # Ejemplo: COP, USD, EUR