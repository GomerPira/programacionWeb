# Modelo de Dominio Central (Domain Model)

## Introducción

Este documento define las entidades principales del Sistema de Gestión de Información Bancaria. Aplica los principios de Diseño Guiado por el Dominio (DDD) y Orientación a Objetos. La herencia se utiliza para especialización genuina y las relaciones entre objetos se representan explícitamente sin usar identificadores primitivos dispersos.

---

## Jerarquía de Clases de Dominio

```text
Persona (Abstracta)
├── Cliente (Abstracto)
│   ├── ClienteNatural
│   └── ClienteJuridico
└── Usuario

ProductoBancario (Abstracto)
├── CuentaBancaria
├── Prestamo
└── Transferencia

Operacion

BitacoraAuditoria
Código de Dominio en Python
Python
from abc import ABC
from dataclasses import dataclass, field
from datetime import date, datetime
from decimal import Decimal
from typing import List, Optional, Dict, Any

from domain.value_objects import (
    RolSistema, EstadoCliente, EstadoUsuario, EstadoCuenta,
    EstadoPrestamo, EstadoTransferencia, TipoCuenta, TipoPrestamo,
    TipoOperacion, Moneda
)


# ==========================================
# JERARQUÍA DE PERSONAS
# ==========================================

@dataclass
class Persona(ABC):
    """Representa a cualquier persona identificable dentro del sistema bancario."""
    identificacion: str
    nombre: str
    email: str
    telefono: str
    direccion: str
    rol: RolSistema


@dataclass
class Cliente(Persona, ABC):
    """Representa a un cliente abstracto de la institución bancaria."""
    estado: EstadoCliente
    cuentas: List['CuentaBancaria'] = field(default_factory=list)
    prestamos: List['Prestamo'] = field(default_factory=list)
    transferencias: List['Transferencia'] = field(default_factory=list)


@dataclass
class ClienteNatural(Cliente):
    """Representa a un cliente persona natural (mínimo 18 años)."""
    fecha_nacimiento: date


@dataclass
class ClienteJuridico(Cliente):
    """Representa a un cliente corporativo/empresa."""
    representante_legal: ClienteNatural


@dataclass
class Usuario(Persona):
    """Representa la identidad en el sistema para autenticación y autorización."""
    id_usuario: int
    nombre_usuario: str
    contrasena_hash: str
    estado_usuario: EstadoUsuario
    cliente: Optional[Cliente] = None


# ==========================================
# JERARQUÍA DE PRODUCTOS BANCARIOS
# ==========================================

@dataclass
class ProductoBancario(ABC):
    """Representa cualquier producto o servicio financiero ofrecido por el banco."""
    identificador: str


@dataclass
class CuentaBancaria(ProductoBancario):
    """Representa una cuenta bancaria administrada por la institución."""
    tipo_cuenta: TipoCuenta
    propietario: Cliente
    saldo_actual: Decimal
    moneda: Moneda
    estado_cuenta: EstadoCuenta
    fecha_apertura: date


@dataclass
class Prestamo(ProductoBancario):
    """Representa un producto de crédito solicitado y gestionado por un cliente."""
    solicitante: Cliente
    tipo_prestamo: TipoPrestamo
    monto_solicitado: Decimal
    monto_aprobado: Optional[Decimal]
    tasa_interes: Decimal
    plazo_en_meses: int
    estado_prestamo: EstadoPrestamo
    fecha_aprobacion: Optional[date] = None
    fecha_desembolso: Optional[date] = None
    cuenta_destino: Optional[CuentaBancaria] = None


@dataclass
class Transferencia(ProductoBancario):
    """Representa un servicio que mueve fondos entre dos cuentas bancarias."""
    cuenta_origen: CuentaBancaria
    cuenta_destino: CuentaBancaria
    monto: Decimal
    fecha_creacion: datetime
    estado_transferencia: EstadoTransferencia
    creado_por: Usuario
    fecha_aprobacion: Optional[datetime] = None
    aprobado_por: Optional[Usuario] = None


# ==========================================
# OPERACIÓN Y AUDITORÍA
# ==========================================

@dataclass
class Operacion:
    """Representa una acción de negocio significativa sobre un producto bancario."""
    id_operacion: int
    tipo_operacion: TipoOperacion
    fecha_ejecucion: datetime
    realizado_por: Usuario
    producto_afectado: ProductoBancario


@dataclass
class BitacoraAuditoria:
    """Representa un registro inmutable de auditoría en la bitácora del sistema."""
    id_auditoria: str
    tipo_operacion: TipoOperacion
    fecha_operacion: datetime
    realizado_por: Usuario
    rol_usuario: RolSistema
    producto_afectado: ProductoBancario
    detalles: Dict[str, Any] = field(default_factory=dict)

---

# 2. `Domain Value Objects.md`

Representa los conceptos inmutables, catálogos del dominio y enumeraciones simples.

```markdown
# Objetos de Valor e Inmutables del Dominio (Domain Value Objects)

## Introducción

Los Objetos de Valor (*Value Objects*) encapsulan conceptos de negocio inmutables. Se definen por sus valores y no por una identidad única. Previenen el uso de cadenas arbitrarias y valores primitivos dispersos en el código.

---

## Catálogo Base del Dominio

```python
from abc import ABC
from dataclasses import dataclass
from enum import Enum


@dataclass(frozen=True)
class CatalogoDominio(ABC):
    """Clase base inmutable para catálogos de negocio que requieren metadata."""
    codigo: str
    nombre: str
    descripcion: str
Catálogos de Dominio (Enums con Metadata)
Python
class RolSistema(Enum):
    CLIENTE_NATURAL = CatalogoDominio("NATURAL_CUSTOMER", "Cliente Natural", "Cliente bancario individual.")
    CLIENTE_JURIDICO = CatalogoDominio("BUSINESS_CUSTOMER", "Cliente Jurídico", "Cliente bancario corporativo.")
    CAJERO_EMPLEADO = CatalogoDominio("TELLER_EMPLOYEE", "Empleado Cajero", "Realiza operaciones en ventanilla.")
    EJECUTIVO_COMERCIAL = CatalogoDominio("COMMERCIAL_EMPLOYEE", "Ejecutivo Comercial", "Gestiona clientes y préstamos.")
    OPERADOR_EMPRESARIAL = CatalogoDominio("BUSINESS_OPERATOR", "Operador Empresarial", "Opera a nombre de clientes jurídicos.")
    SUPERVISOR_EMPRESARIAL = CatalogoDominio("BUSINESS_SUPERVISOR", "Supervisor Empresarial", "Aprueba transferencias corporativas.")
    ANALISTA_INTERNO = CatalogoDominio("INTERNAL_ANALYST", "Analista Interno", "Evalúa y aprueba préstamos.")


class EstadoCliente(Enum):
    ACTIVO = CatalogoDominio("ACTIVE", "Activo", "Relación bancaria activa.")
    INACTIVO = CatalogoDominio("INACTIVE", "Inactivo", "Sin operaciones bancarias normales.")
    BLOQUEADO = CatalogoDominio("BLOCKED", "Bloqueado", "Relación bancaria suspendida.")


class EstadoUsuario(Enum):
    ACTIVO = CatalogoDominio("ACTIVE", "Activo", "Acceso permitido al sistema.")
    INACTIVO = CatalogoDominio("INACTIVE", "Inactivo", "Acceso deshabilitado.")
    BLOQUEADO = CatalogoDominio("BLOCKED", "Bloqueado", "Acceso suspendido.")


class EstadoCuenta(Enum):
    ACTIVO = CatalogoDominio("ACTIVE", "Activa", "Cuenta plenamente operativa.")
    BLOQUEADO = CatalogoDominio("BLOCKED", "Bloqueada", "Transacciones inhabilitadas temporalmente.")
    CERRADO = CatalogoDominio("CLOSED", "Cerrada", "Cuenta cerrada permanentemente.")


class EstadoPrestamo(Enum):
    EN_REVISION = CatalogoDominio("UNDER_REVIEW", "En Revisión", "Solicitud en evaluación.")
    APROBADO = CatalogoDominio("APPROVED", "Aprobado", "Préstamo aprobado.")
    RECHAZADO = CatalogoDominio("REJECTED", "Rechazado", "Solicitud rechazada.")
    DESEMBOLSADO = CatalogoDominio("DISBURSED", "Desembolsado", "Fondos entregados.")
    CERRADO = CatalogoDominio("CLOSED", "Cerrado", "Préstamo pagado totalmente.")


class EstadoTransferencia(Enum):
    PENDIENTE = CatalogoDominio("PENDING", "Pendiente", "Transferencia creada.")
    ESPERANDO_APROBACION = CatalogoDominio("WAITING_FOR_APPROVAL", "Esperando Aprobación", "Requiere autorización.")
    APROBADO = CatalogoDominio("APPROVED", "Aprobada", "Lista para ejecución.")
    EJECUTADO = CatalogoDominio("EXECUTED", "Ejecutada", "Fondos transferidos con éxito.")
    RECHAZADO = CatalogoDominio("REJECTED", "Rechazada", "Solicitud denegada.")
    EXPIRADO = CatalogoDominio("EXPIRED", "Expirada", "Tiempo de aprobación vencido.")


class TipoCuenta(Enum):
    AHOORROS = CatalogoDominio("SAVINGS", "Cuenta de Ahorros", "Cuenta con intereses.")
    CORRIENTE = CatalogoDominio("CHECKING", "Cuenta Corriente", "Para operaciones frecuentes.")
    EMPRESARIAL = CatalogoDominio("BUSINESS", "Cuenta Empresarial", "Diseñada para corporaciones.")


class TipoPrestamo(Enum):
    PERSONAL = CatalogoDominio("PERSONAL", "Préstamo Personal", "Uso personal.")
    HIPOTECARIO = CatalogoDominio("MORTGAGE", "Préstamo Hipotecario", "Garantizado por inmueble.")
    VEHICULO = CatalogoDominio("VEHICLE", "Préstamo de Vehículo", "Para adquisición de vehículos.")
    EMPRESARIAL = CatalogoDominio("BUSINESS", "Préstamo Empresarial", "Financiamiento corporativo.")


class TipoOperacion(Enum):
    APERTURA_CUENTA = CatalogoDominio("ACCOUNT_OPENING", "Apertura de Cuenta", "Creación de cuenta.")
    DEPOSITO = CatalogoDominio("DEPOSIT", "Depósito", "Ingreso de fondos.")
    RETIRO = CatalogoDominio("WITHDRAWAL", "Retiro", "Extracción de fondos.")
    BLOQUEO_CUENTA = CatalogoDominio("ACCOUNT_BLOCKING", "Bloqueo de Cuenta", "Inhabilitar cuenta.")
    DESBLOQUEO_CUENTA = CatalogoDominio("ACCOUNT_UNBLOCKING", "Desbloqueo de Cuenta", "Rehabilitar cuenta.")
    CIERRE_CUENTA = CatalogoDominio("ACCOUNT_CLOSING", "Cierre de Cuenta", "Cierre definitivo.")
    CREACION_TRANSFERENCIA = CatalogoDominio("TRANSFER_CREATION", "Creación Transferencia", "Solicitud de envío.")
    APROBACION_TRANSFERENCIA = CatalogoDominio("TRANSFER_APPROVAL", "Aprobación Transferencia", "Autorización.")
    RECHAZO_TRANSFERENCIA = CatalogoDominio("TRANSFER_REJECTION", "Rechazo Transferencia", "Denegación.")
    EJECUCION_TRANSFERENCIA = CatalogoDominio("TRANSFER_EXECUTION", "Ejecución Transferencia", "Efectivización.")
    EXPIRACION_TRANSFERENCIA = CatalogoDominio("TRANSFER_EXPIRATION", "Expiración Transferencia", "Vencimiento.")
    SOLICITUD_PRESTAMO = CatalogoDominio("LOAN_APPLICATION", "Solicitud de Préstamo", "Registro de propuesta.")
    APROBACION_PRESTAMO = CatalogoDominio("LOAN_APPROVAL", "Aprobación de Préstamo", "Aprobación de crédito.")
    RECHAZO_PRESTAMO = CatalogoDominio("LOAN_REJECTION", "Rechazo de Préstamo", "Rechazo de crédito.")
    DESEMBOLSO_PRESTAMO = CatalogoDominio("LOAN_DISBURSEMENT", "Desembolso de Préstamo", "Entrega de fondos.")
    PAGO_PRESTAMO = CatalogoDominio("LOAN_PAYMENT", "Pago de Préstamo", "Abono a cuota.")
    CIERRE_PRESTAMO = CatalogoDominio("LOAN_CLOSING", "Cierre de Préstamo", "Cancelación total.")


@dataclass(frozen=True)
class Moneda:
    codigo_iso: str
    nombre: str
    simbolo: str


MONEDAS_SOPORTADAS = {
    "COP": Moneda("COP", "Peso Colombiano", "$"),
    "USD": Moneda("USD", "Dólar Estadounidense", "$"),
    "EUR": Moneda("EUR", "Euro", "€"),
}
Enumeraciones Técnicas Primitivas
Python
class DecisionAprobacion(Enum):
    APROBADO = "APPROVED"
    RECHAZADO = "REJECTED"


class CanalNotificacion(Enum):
    EMAIL = "EMAIL"
    SMS = "SMS"
    PUSH_NOTIFICATION = "PUSH_NOTIFICATION"


class SeveridadAuditoria(Enum):
    INFORMACION = "INFORMATION"
    ADVERTENCIA = "WARNING"
    ERROR = "ERROR"
    CRITICO = "CRITICAL"

---

# 3. `Output-ports.md`

Este archivo contiene todos los contratos de infraestructura requeridos por los subdominios (repositorios, puertos de seguridad, tokens y auditoría).

```markdown
# Puertos de Salida (Output Ports)

## Introducción

Los Puertos de Salida son clases base abstractas (`ABC`) definidas dentro del dominio que declaran las operaciones que la capa de infraestructura (bases de datos, adaptadores de red, librerías criptográficas) debe implementar obligatoriamente.

---

## Interfaces de Repositorios de Dominio

```python
from abc import ABC, abstractmethod
from typing import Optional, List
from domain.models import Usuario, Cliente, CuentaBancaria, Prestamo, Transferencia, Operacion, BitacoraAuditoria


class PuertoRepositorioUsuario(ABC):

    @abstractmethod
    def guardar(self, usuario: Usuario) -> Usuario:
        pass

    @abstractmethod
    def buscar_por_nombre_usuario(self, nombre_usuario: str) -> Optional[Usuario]:
        pass

    @abstractmethod
    def existe_nombre_usuario(self, nombre_usuario: str) -> bool:
        pass


class PuertoRepositorioCliente(ABC):

    @abstractmethod
    def buscar_por_identificacion(self, identificacion: str) -> Optional[Cliente]:
        pass

    @abstractmethod
    def guardar(self, cliente: Cliente) -> Cliente:
        pass


class PuertoRepositorioCuentaBancaria(ABC):

    @abstractmethod
    def buscar_por_identificador(self, identificador: str) -> Optional[CuentaBancaria]:
        pass

    @abstractmethod
    def guardar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        pass


class PuertoRepositorioPrestamo(ABC):

    @abstractmethod
    def buscar_por_identificador(self, identificador: str) -> Optional[Prestamo]:
        pass

    @abstractmethod
    def guardar(self, prestamo: Prestamo) -> Prestamo:
        pass


class PuertoRepositorioTransferencia(ABC):

    @abstractmethod
    def buscar_por_identificador(self, identificador: str) -> Optional[Transferencia]:
        pass

    @abstractmethod
    def guardar(self, transferencia: Transferencia) -> Transferencia:
        pass


class PuertoRepositorioOperacion(ABC):

    @abstractmethod
    def registrar(self, operacion: Operacion) -> Operacion:
        pass


class PuertoRepositorioBitacoraAuditoria(ABC):

    @abstractmethod
    def registrar_auditoria(self, bitacora: BitacoraAuditoria) -> None:
        """Persiste un registro inmutable en la bitácora de auditoría (Base de Datos NoSQL)."""
        pass
Interfaces de Infraestructura y Seguridad
Python
class PuertoSeguridadContrasena(ABC):

    @abstractmethod
    def hashear(self, contrasena_texto_plano: str) -> str:
        """Genera el hash seguro de la contraseña."""
        pass

    @abstractmethod
    def verificar(self, contrasena_texto_plano: str, contrasena_hash: str) -> bool:
        """Verifica la contraseña ingresada contra el hash persistido."""
        pass


class PuertoTokenJWT(ABC):

    @abstractmethod
    def generar(self, usuario: Usuario) -> str:
        """Crea el token JWT con los claims de nombre_usuario y rol."""
        pass


class PuertoNotificador(ABC):

    @abstractmethod
    def enviar_notificacion(self, destino: str, mensaje: str) -> bool:
        pass
