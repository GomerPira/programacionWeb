# Puertos de Salida (Output Ports / Repository Interfaces)

## Introducción

Los Puertos de Salida (*Output Ports*) representan las interfaces abstractas que definen las necesidades de interacción del Dominio bancario hacia el exterior. 

Bajo la Arquitectura Hexagonal, el Dominio no conoce las bases de datos (PostgreSQL, MongoDB), los brokers de eventos ni los servicios externos de mensajería (Twilio, SendGrid). En su lugar, el Dominio define estos contratos (`ABC`) que deben ser implementados por los **Adaptadores de Salida** en la capa de infraestructura.

---

## Estructura de Puertos de Salida

```text
Output Ports (Interfaces / ABC)
├── RepositorioCliente
├── RepositorioUsuario
├── RepositorioCuentaBancaria
├── RepositorioPrestamo
├── RepositorioTransferencia
├── RepositorioOperacion
├── RepositorioAuditoria (NoSQL)
└── ServicioNotificaciones (Puerto de Infraestructura)
Código en Python (Output-ports.md)
Python
from abc import ABC, abstractmethod
from typing import List, Optional, Dict, Any

from domain.models import (
    Cliente, Usuario, CuentaBancaria, Prestamo, Transferencia, Operacion, BitacoraAuditoria
)
from domain.value_objects import NotificationChannel


# ==========================================
# 1. REPOSITORIO DE CLIENTES
# ==========================================

class RepositorioCliente(ABC):

    @abstractmethod
    def guardar(self, cliente: Cliente) -> Cliente:
        """Persiste o actualiza un objeto cliente en el almacenamiento."""
        pass

    @abstractmethod
    def buscar_por_identificacion(self, identificacion: str) -> Optional[Cliente]:
        """Obtiene un cliente por su documento de identidad único."""
        pass

    @abstractmethod
    def obtener_todos(self, limite: int = 100, offset: int = 0) -> List[Cliente]:
        """Recupera un listado paginado de clientes."""
        pass


# ==========================================
# 2. REPOSITORIO DE USUARIOS
# ==========================================

class RepositorioUsuario(ABC):

    @abstractmethod
    def guardar(self, usuario: Usuario) -> Usuario:
        """Persiste o actualiza las credenciales y perfil de un usuario."""
        pass

    @abstractmethod
    def buscar_por_id(self, id_usuario: int) -> Optional[Usuario]:
        """Recupera un usuario mediante su identificador numérico interno."""
        pass

    @abstractmethod
    def buscar_por_nombre_usuario(self, nombre_usuario: str) -> Optional[Usuario]:
        """Obtiene las credenciales de un usuario a partir de su username."""
        pass

    @abstractmethod
    def buscar_por_cliente_id(self, id_cliente: str) -> List[Usuario]:
        """Recupera los usuarios asociados a un cliente específico."""
        pass


# ==========================================
# 3. REPOSITORIO DE CUENTAS BANCARIAS
# ==========================================

class RepositorioCuentaBancaria(ABC):

    @abstractmethod
    def guardar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        """Persiste la apertura o modificación de estado/saldo de una cuenta."""
        pass

    @abstractmethod
    def buscar_por_numero_cuenta(self, numero_cuenta: str) -> Optional[CuentaBancaria]:
        """Obtiene una cuenta bancaria mediante su número único de cuenta."""
        pass

    @abstractmethod
    def buscar_por_cliente(self, identificacion_cliente: str) -> List[CuentaBancaria]:
        """Recupera todas las cuentas de las que es titular un cliente."""
        pass


# ==========================================
# 4. REPOSITORIO DE PRÉSTAMOS
# ==========================================

class RepositorioPrestamo(ABC):

    @abstractmethod
    def guardar(self, prestamo: Prestamo) -> Prestamo:
        """Persiste la solicitud, aprobación o amortización de un préstamo."""
        pass

    @abstractmethod
    def buscar_por_id(self, id_prestamo: str) -> Optional[Prestamo]:
        """Obtiene un préstamo mediante su identificador de contrato."""
        pass

    @abstractmethod
    def buscar_por_cliente(self, identificacion_cliente: str) -> List[Prestamo]:
        """Obtiene el histórico de créditos asignados a un cliente."""
        pass


# ==========================================
# 5. REPOSITORIO DE TRANSFERENCIAS
# ==========================================

class RepositorioTransferencia(ABC):

    @abstractmethod
    def guardar(self, transferencia: Transferencia) -> Transferencia:
        """Registra la orden o actualización del estado de una transferencia."""
        pass

    @abstractmethod
    def buscar_por_id(self, id_transferencia: str) -> Optional[Transferencia]:
        """Recupera el detalle de una transferencia bancaria."""
        pass

    @abstractmethod
    def buscar_pendientes_aprobacion(self) -> List[Transferencia]:
        """Obtiene las transferencias que requieren validación jerárquica."""
        pass


# ==========================================
# 6. REPOSITORIO DE OPERACIONES DE NEGOCIO
# ==========================================

class RepositorioOperacion(ABC):

    @abstractmethod
    def guardar(self, operacion: Operacion) -> Operacion:
        """Persiste un registro histórico de trazabilidad de negocio."""
        pass

    @abstractmethod
    def buscar_por_producto(self, id_producto: str) -> List[Operacion]:
        """Obtiene todas las operaciones efectuadas sobre un producto bancario."""
        pass


# ==========================================
# 7. REPOSITORIO DE AUDITORÍA (NoSQL Document)
# ==========================================

class RepositorioAuditoria(ABC):

    @abstractmethod
    def registrar_evento(self, bitacora: BitacoraAuditoria) -> None:
        """Persiste de forma inmutable un evento en el almacén de auditoría NoSQL."""
        pass

    @abstractmethod
    def consultar_log(self, criterios: Dict[str, Any], limite: int = 100) -> List[BitacoraAuditoria]:
        """Permite realizar búsquedas dinámicas en la bitácora según metadatos JSON/NoSQL."""
        pass


# ==========================================
# 8. PUERTO DE INFRAESTRUCTURA: NOTIFICACIONES
# ==========================================

class ServicioNotificaciones(ABC):

    @abstractmethod
    def enviar_notificacion(
        self, 
        destinatario: str, 
        mensaje: str, 
        canal: NotificationChannel,
        asunto: Optional[str] = None
    ) -> bool:
        """Envía notificaciones a usuarios o clientes mediante un canal específico (Email, SMS, Push)."""
        pass

---

### Mapa Consolidado de la Documentación Core en `Domain/`

Con los puertos de salida definidos, la estructura completa de arquitectura para el Dominio queda finalizada:

```text
Domain/
├── Domain Model.md                # Entidades en Dataclasses de Python
├── Domain Value Objects.md        # Catálogos inmutables y Enums
├── Domain Services.md             # Especificación sombrilla de Servicios
├── Input-ports.md                 # Contratos ABC de Casos de Uso (Entrada)
├── Output-ports.md                # Contratos ABC de Repositorios y Servicios (Salida)
└── services/                      # Detalle de implementaciones por Subdominio
