# Puertos de Entrada (Input Ports / Use Cases)

## Introducción

Los Puertos de Entrada (*Input Ports*) representan los contratos abstractos de los **Casos de Uso** del Sistema de Gestión de Información Bancaria. 

Siguiendo los principios de la Arquitectura Hexagonal y DDD, el mundo exterior (capa de infraestructura: controladores API REST, tareas programadas, CLI, etc.) interactúa con el dominio invocando únicamente estas interfaces. Esto asegura un desacoplamiento completo y garantiza que las reglas de negocio permanezcan puras e independientes de la tecnología.

---

## Estructura de Puertos de Entrada

```text
Input Ports (Interfaces / ABC)
├── CasoUsoGestionClientes
├── CasoUsoGestionUsuariosAutenticacion
├── CasoUsoGestionCuentasBancarias
├── CasoUsoGestionPrestamos
├── CasoUsoGestionTransferencias
├── CasoUsoOperacionAuditoria
└── CasoUsoAutorizacion
Código en Python (Input-ports.md)
Python
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import List, Optional, Dict, Any

from domain.models import (
    Cliente, ClienteNatural, ClienteJuridico, Usuario,
    CuentaBancaria, Prestamo, Transferencia, Operacion, BitacoraAuditoria
)
from domain.value_objects import (
    CustomerStatus, UserStatus, AccountStatus, DecisionAprobacion
)


# ==========================================
# 1. CASOS DE USO: GESTIÓN DE CLIENTES
# ==========================================

class CasoUsoGestionClientes(ABC):

    @abstractmethod
    def registrar_cliente_natural(self, datos_cliente: ClienteNatural) -> ClienteNatural:
        """Caso de uso para dar de alta un nuevo cliente persona natural."""
        pass

    @abstractmethod
    def registrar_cliente_juridico(self, datos_cliente: ClienteJuridico) -> ClienteJuridico:
        """Caso de uso para registrar una empresa y vincular su representante legal."""
        pass

    @abstractmethod
    def consultar_cliente(self, identificacion: str, usuario_solicitante: Usuario) -> Optional[Cliente]:
        """Caso de uso para consultar los datos de un cliente según los permisos del solicitante."""
        pass

    @abstractmethod
    def actualizar_cliente(self, cliente: Cliente, usuario_solicitante: Usuario) -> Cliente:
        """Caso de uso para actualizar la información de contacto o legal de un cliente."""
        pass

    @abstractmethod
    def cambiar_estado_cliente(self, identificacion: str, nuevo_estado: CustomerStatus, usuario_solicitante: Usuario) -> Cliente:
        """Caso de uso para cambiar el estado operativo de un cliente (ACTIVE, INACTIVE, BLOCKED)."""
        pass

    @abstractmethod
    def consultar_productos_cliente(self, identificacion: str, usuario_solicitante: Usuario) -> Dict[str, Any]:
        """Caso de uso para recuperar el portafolio completo del cliente (cuentas, préstamos, etc.)."""
        pass


# ==========================================
# 2. CASOS DE USO: AUTENTICACIÓN Y USUARIOS
# ==========================================

class CasoUsoGestionUsuariosAutenticacion(ABC):

    @abstractmethod
    def registrar_usuario_cliente(self, usuario: Usuario, cliente_asociado: Cliente) -> Usuario:
        """Caso de uso para crear un usuario de acceso al sistema vinculado a un cliente."""
        pass

    @abstractmethod
    def registrar_usuario_empleado(self, usuario: Usuario, usuario_creador: Usuario) -> Usuario:
        """Caso de uso para registrar credenciales de un empleado interno del banco."""
        pass

    @abstractmethod
    def iniciar_sesion(self, nombre_usuario: str, contrasena_plana: str) -> str:
        """Caso de uso para validar credenciales y generar el token JWT de acceso."""
        pass

    @abstractmethod
    def cerrar_sesion(self, token: str) -> bool:
        """Caso de uso para revocar/invalidar la sesión activa de un usuario."""
        pass

    @abstractmethod
    def consultar_usuario(self, id_usuario: int, usuario_solicitante: Usuario) -> Optional[Usuario]:
        """Caso de uso para recuperar la información del perfil de usuario."""
        pass

    @abstractmethod
    def cambiar_estado_usuario(self, id_usuario: int, nuevo_estado: UserStatus, usuario_solicitante: Usuario) -> Usuario:
        """Caso de uso para suspender, inhabilitar o activar el acceso de un usuario."""
        pass


# ==========================================
# 3. CASOS DE USO: CUENTAS BANCARIAS
# ==========================================

class CasoUsoGestionCuentasBancarias(ABC):

    @abstractmethod
    def abrir_cuenta(self, cuenta: CuentaBancaria, usuario_solicitante: Usuario) -> CuentaBancaria:
        """Caso de uso para aperturar una nueva cuenta bancaria."""
        pass

    @abstractmethod
    def consultar_cuenta(self, identificador_cuenta: str, usuario_solicitante: Usuario) -> Optional[CuentaBancaria]:
        """Caso de uso para obtener los detalles operativos de una cuenta."""
        pass

    @abstractmethod
    def consultar_saldo(self, identificador_cuenta: str, usuario_solicitante: Usuario) -> Decimal:
        """Caso de uso rápido para consultar el saldo disponible."""
        pass

    @abstractmethod
    def depositar_fondos(self, identificador_cuenta: str, monto: Decimal, ejecutado_por: Usuario) -> CuentaBancaria:
        """Caso de uso para abonar dinero a una cuenta y generar registro de operación."""
        pass

    @abstractmethod
    def retirar_fondos(self, identificador_cuenta: str, monto: Decimal, ejecutado_por: Usuario) -> CuentaBancaria:
        """Caso de uso para débito de saldo previa validación de fondos y estado."""
        pass

    @abstractmethod
    def bloquear_cuenta(self, identificador_cuenta: str, motivo: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Caso de uso para congelar/bloquear una cuenta por seguridad."""
        pass

    @abstractmethod
    def desbloquear_cuenta(self, identificador_cuenta: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Caso de uso para restituir la operatividad normal de la cuenta."""
        pass

    @abstractmethod
    def cerrar_cuenta(self, identificador_cuenta: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Caso de uso para la cancelación o cierre permanente de una cuenta bancaria."""
        pass


# ==========================================
# 4. CASOS DE USO: GESTIÓN DE PRÉSTAMOS
# ==========================================

class CasoUsoGestionPrestamos(ABC):

    @abstractmethod
    def solicitar_prestamo(self, prestamo: Prestamo, usuario_solicitante: Usuario) -> Prestamo:
        """Caso de uso para ingresar una solicitud formal de crédito."""
        pass

    @abstractmethod
    def consultar_prestamo(self, identificador_prestamo: str, usuario_solicitante: Usuario) -> Optional[Prestamo]:
        """Caso de uso para evaluar el estado actual de una solicitud o crédito activo."""
        pass

    @abstractmethod
    def aprobar_prestamo(self, identificador_prestamo: str, monto_aprobado: Decimal, evaluador: Usuario) -> Prestamo:
        """Caso de uso exclusivo para analistas aprobadores de crédito."""
        pass

    @abstractmethod
    def rechazar_prestamo(self, identificador_prestamo: str, motivo: str, evaluador: Usuario) -> Prestamo:
        """Caso de uso para denegar la solicitud de préstamo."""
        pass

    @abstractmethod
    def desembolsar_prestamo(self, identificador_prestamo: str, ejecutado_por: Usuario) -> Prestamo:
        """Caso de uso para la liquidación y transferencia de fondos a la cuenta destino."""
        pass

    @abstractmethod
    def registrar_pago_prestamo(self, identificador_prestamo: str, monto: Decimal, ejecutado_por: Usuario) -> Prestamo:
        """Caso de uso para abonar o liquidar cuotas del préstamo."""
        pass

    @abstractmethod
    def cerrar_prestamo(self, identificador_prestamo: str, ejecutado_por: Usuario) -> Prestamo:
        """Caso de uso para cerrar el ciclo de vida del crédito al ser saldado totalmente."""
        pass


# ==========================================
# 5. CASOS DE USO: TRANSFERENCIAS
# ==========================================

class CasoUsoGestionTransferencias(ABC):

    @abstractmethod
    def crear_transferencia(self, transferencia: Transferencia, creador: Usuario) -> Transferencia:
        """Caso de uso para registrar una orden de transferencia entre cuentas."""
        pass

    @abstractmethod
    def ejecutar_transferencia(self, identificador_transferencia: str, ejecutado_por: Usuario) -> Transferencia:
        """Caso de uso para realizar el débito y crédito directo de montos autorizados."""
        pass

    @abstractmethod
    def enviar_para_aprobacion(self, identificador_transferencia: str) -> Transferencia:
        """Caso de uso para colocar una transferencia en espera de visto bueno jerárquico."""
        pass

    @abstractmethod
    def aprobar_transferencia(self, identificador_transferencia: str, aprobador: Usuario) -> Transferencia:
        """Caso de uso para que un supervisor o apoderado autorice la transferencia."""
        pass

    @abstractmethod
    def rechazar_transferencia(self, identificador_transferencia: str, evaluador: Usuario) -> Transferencia:
        """Caso de uso para desestimar una transferencia retenida."""
        pass

    @abstractmethod
    def expirar_transferencia(self, identificador_transferencia: str) -> Transferencia:
        """Caso de uso (invocado por procesos batch/Cron) para cancelar transferencias caducadas."""
        pass


# ==========================================
# 6. CASOS DE USO: AUDITORÍA Y OPERACIONES
# ==========================================

class CasoUsoOperacionAuditoria(ABC):

    @abstractmethod
    def registrar_operacion(self, operacion: Operacion) -> Operacion:
        """Caso de uso para generar la traza de eventos de negocio sobre un producto."""
        pass

    @abstractmethod
    def consultar_operaciones(self, identificador_producto: str, usuario_solicitante: Usuario) -> List[Operacion]:
        """Caso de uso para obtener el histórico operacional de un producto."""
        pass

    @abstractmethod
    def registrar_evento_auditoria(self, bitacora: BitacoraAuditoria) -> None:
        """Caso de uso para escribir en la bitácora de auditoría inmutable (NoSQL)."""
        pass

    @abstractmethod
    def consultar_bitacora_auditoria(self, filtro: Dict[str, Any], usuario_solicitante: Usuario) -> List[BitacoraAuditoria]:
        """Caso de uso exclusivo para perfiles de auditoría y monitoreo."""
        pass


# ==========================================
# 7. CASOS DE USO: AUTORIZACIÓN Y REGLAS
# ==========================================

class CasoUsoAutorizacion(ABC):

    @abstractmethod
    def validar_permisos(self, usuario: Usuario, operacion_requerida: str) -> bool:
        """Caso de uso para evaluar si el rol del usuario le faculta a ejecutar una acción."""
        pass

    @abstractmethod
    def validar_acceso_cliente(self, usuario: Usuario, cliente_objetivo: Cliente) -> bool:
        """Caso de uso para verificar si el usuario puede ver/operar información del cliente."""
        pass

    @abstractmethod
    def validar_acceso_producto(self, usuario: Usuario, identificador_producto: str) -> bool:
        """Caso de uso para verificar propiedad o autorización sobre un producto específico."""
        pass

    @abstractmethod
    def validar_autorizacion_aprobacion(self, usuario: Usuario, monto: Decimal, tipo_operacion: str) -> bool:
        """Caso de uso para validar si el usuario posee el límite de atribución necesario."""
        pass

---

### Visión Actualizada de la Raíz `Domain/`

Con la adición de `Input-ports.md`, la carpeta principal de dominio en tu documentación SDD queda totalmente constituida y balanceada:

```text
Domain/
├── Domain Model.md                # Entidades en dataclasses
├── Domain Value Objects.md        # Enums y catálogos de negocio inmutables
├── Domain Services.md             # Especificación sombrilla de servicios
├── Input-ports.md                 # Contratos ABC de Casos de Uso (Este archivo)
├── Output-ports.md                # Contratos ABC de Repositorios e Infraestructura
└── services/                      # Detalle específico por subdominio
