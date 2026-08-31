# Servicios de Dominio (Domain Services)

## Introducción

Este documento proporciona una visión general de los servicios que componen el Sistema de Gestión de Información Bancaria. Los servicios descritos aquí definen las capacidades clave de negocio expuestas por el sistema a alto nivel. 

Cada servicio se especifica mediante contratos de interfaz en Python (`ABC`), declarando su propósito, entradas y salidas sin depender de librerías de infraestructura.

---

## Organización por Subdominios

Los servicios del dominio se organizan en los siguientes submódulos dentro del directorio `Domain/services/`:

```text
Domain/
├── Domain Services.md (Este archivo - Contratos e Índice Global)
└── services/
    ├── customer-services.md
    ├── user-authentication-services.md
    ├── bank-account-services.md
    ├── loan-services.md
    ├── transfer-services.md
    ├── operation-audit-services.md
    └── authorization-services.md
Interfaces Abstratas de los Servicios de Dominio (Python)
Python
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import List, Optional, Dict, Any

from domain.models import (
    Cliente, ClienteNatural, ClienteJuridico, Usuario,
    CuentaBancaria, Prestamo, Transferencia, Operacion, BitacoraAuditoria
)
from domain.value_objects import (
    EstadoCliente, EstadoUsuario, EstadoCuenta, DecisionAprobacion
)


# ==========================================
# 1. GESTIÓN DE CLIENTES (Customer Management)
# ==========================================

class ServicioGestionClientes(ABC):

    @abstractmethod
    def registrar_cliente_natural(self, datos_cliente: ClienteNatural) -> ClienteNatural:
        """Crea un nuevo cliente persona natural."""
        pass

    @abstractmethod
    def registrar_cliente_juridico(self, datos_cliente: ClienteJuridico) -> ClienteJuridico:
        """Crea un nuevo cliente persona jurídica y le asocia su representante legal."""
        pass

    @abstractmethod
    def consultar_cliente(self, identificacion: str, usuario_solicitante: Usuario) -> Optional[Cliente]:
        """Recupera la información de un cliente según los permisos del usuario solicitante."""
        pass

    @abstractmethod
    def actualizar_cliente(self, cliente: Cliente, usuario_solicitante: Usuario) -> Cliente:
        """Actualiza la información de un cliente existente."""
        pass

    @abstractmethod
    def cambiar_estado_cliente(self, identificacion: str, nuevo_estado: EstadoCliente, usuario_solicitante: Usuario) -> Cliente:
        """Cambia el estado operacional de un cliente (Activo, Inactivo, Bloqueado)."""
        pass

    @abstractmethod
    def consultar_productos_cliente(self, identificacion: str, usuario_solicitante: Usuario) -> Dict[str, Any]:
        """Recupera los productos bancarios asociados (cuentas, préstamos, transferencias)."""
        pass


# ==========================================
# 2. AUTENTICACIÓN Y USUARIOS (User & Auth)
# ==========================================

class ServicioGestionUsuariosAutenticacion(ABC):

    @abstractmethod
    def registrar_usuario_cliente(self, usuario: Usuario, cliente_asociado: Cliente) -> Usuario:
        """Crea un usuario del sistema asociado a un cliente existente."""
        pass

    @abstractmethod
    def registrar_usuario_empleado(self, usuario: Usuario, usuario_creador: Usuario) -> Usuario:
        """Crea un usuario de tipo empleado interno. Restringido a ANALISTA_INTERNO."""
        pass

    @abstractmethod
    def iniciar_sesion(self, nombre_usuario: str, contrasena_plana: str) -> str:
        """Autentica credenciales y retorna el token JWT de sesión."""
        pass

    @abstractmethod
    def cerrar_sesion(self, token: str) -> bool:
        """Invalida la sesión actual del usuario."""
        pass

    @abstractmethod
    def consultar_usuario(self, id_usuario: int, usuario_solicitante: Usuario) -> Optional[Usuario]:
        """Recupera la información de un usuario del sistema."""
        pass

    @abstractmethod
    def cambiar_estado_usuario(self, id_usuario: int, nuevo_estado: EstadoUsuario, usuario_solicitante: Usuario) -> Usuario:
        """Cambia el estado de acceso de un usuario (Activo, Inactivo, Bloqueado)."""
        pass


# ==========================================
# 3. CUENTAS BANCARIAS (Bank Account)
# ==========================================

class ServicioGestionCuentasBancarias(ABC):

    @abstractmethod
    def abrir_cuenta(self, cuenta: CuentaBancaria, usuario_solicitante: Usuario) -> CuentaBancaria:
        """Crea una nueva cuenta bancaria asociada a un cliente."""
        pass

    @abstractmethod
    def consultar_cuenta(self, identificador_cuenta: str, usuario_solicitante: Usuario) -> Optional[CuentaBancaria]:
        """Recupera la información de una cuenta bancaria."""
        pass

    @abstractmethod
    def consultar_saldo(self, identificador_cuenta: str, usuario_solicitante: Usuario) -> Decimal:
        """Consulta el saldo disponible en la cuenta."""
        pass

    @abstractmethod
    def depositar_fondos(self, identificador_cuenta: str, monto: Decimal, ejecutado_por: Usuario) -> CuentaBancaria:
        """Ingresa fondos a una cuenta y registra la operación de negocio y auditoría."""
        pass

    @abstractmethod
    def retirar_fondos(self, identificador_cuenta: str, monto: Decimal, ejecutado_por: Usuario) -> CuentaBancaria:
        """Retira fondos de una cuenta tras validar fondos y restricciones de estado."""
        pass

    @abstractmethod
    def bloquear_cuenta(self, identificador_cuenta: str, motivo: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Cambia el estado de la cuenta a Bloqueada y registra el evento."""
        pass

    @abstractmethod
    def desbloquear_cuenta(self, identificador_cuenta: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Restaura una cuenta bloqueada a estado Activo."""
        pass

    @abstractmethod
    def cerrar_cuenta(self, identificador_cuenta: str, ejecutado_por: Usuario) -> CuentaBancaria:
        """Cierra permanentemente una cuenta bancaria."""
        pass


# ==========================================
# 4. GESTIÓN DE PRÉSTAMOS (Loan Management)
# ==========================================

class ServicioGestionPrestamos(ABC):

    @abstractmethod
    def solicitar_prestamo(self, prestamo: Prestamo, usuario_solicitante: Usuario) -> Prestamo:
        """Crea una solicitud de crédito e inicia el ciclo de vida del préstamo."""
        pass

    @abstractmethod
    def consultar_prestamo(self, identificador_prestamo: str, usuario_solicitante: Usuario) -> Optional[Prestamo]:
        """Recupera los detalles de un préstamo o solicitud activa."""
        pass

    @abstractmethod
    def aprobar_prestamo(self, identificador_prestamo: str, monto_aprobado: Decimal, evaluador: Usuario) -> Prestamo:
        """Aprueba una solicitud de préstamo tras validaciones de crédito."""
        pass

    @abstractmethod
    def rechazar_prestamo(self, identificador_prestamo: str, motivo: str, evaluador: Usuario) -> Prestamo:
        """Rechaza una solicitud de préstamo y registra la decisión."""
        pass

    @abstractmethod
    def desembolsar_prestamo(self, identificador_prestamo: str, ejecutado_por: Usuario) -> Prestamo:
        """Transfiere los fondos aprobados a la cuenta de destino asociada."""
        pass

    @abstractmethod
    def registrar_pago_prestamo(self, identificador_prestamo: str, monto: Decimal, ejecutado_por: Usuario) -> Prestamo:
        """Registra el pago de una cuota sobre el préstamo activo."""
        pass

    @abstractmethod
    def cerrar_prestamo(self, identificador_prestamo: str, ejecutado_por: Usuario) -> Prestamo:
        """Finaliza el ciclo de vida del préstamo una vez saldado por completo."""
        pass


# ==========================================
# 5. TRANSFERENCIAS (Transfer Management)
# ==========================================

class ServicioGestionTransferencias(ABC):

    @abstractmethod
    def crear_transferencia(self, transferencia: Transferencia, creador: Usuario) -> Transferencia:
        """Crea una solicitud de transferencia entre cuentas."""
        pass

    @abstractmethod
    def ejecutar_transferencia(self, identificador_transferencia: str, ejecutado_por: Usuario) -> Transferencia:
        """Mueve los fondos de la cuenta origen a la destino."""
        pass

    @abstractmethod
    def enviar_para_aprobacion(self, identificador_transferencia: str) -> Transferencia:
        """Coloca la transferencia en espera de autorización si supera los límites de monto."""
        pass

    @abstractmethod
    def aprobar_transferencia(self, identificador_transferencia: str, aprobador: Usuario) -> Transferencia:
        """Aprueba una transferencia corporativa o de alto monto."""
        pass

    @abstractmethod
    def rechazar_transferencia(self, identificador_transferencia: str, evaluador: Usuario) -> Transferencia:
        """Rechaza una transferencia en espera de aprobación."""
        pass

    @abstractmethod
    def expirar_transferencia(self, identificador_transferencia: str) -> Transferencia:
        """Marca como expirada una transferencia fuera de la ventana de tiempo límite."""
        pass


# ==========================================
# 6. OPERACIÓN Y AUDITORÍA (Operation & Audit)
# ==========================================

class ServicioOperacionAuditoria(ABC):

    @abstractmethod
    def registrar_operacion(self, operacion: Operacion) -> Operacion:
        """Crea una operación de negocio para dar trazabilidad sobre un producto bancario."""
        pass

    @abstractmethod
    def consultar_operaciones(self, identificador_producto: str, usuario_solicitante: Usuario) -> List[Operacion]:
        """Obtiene las operaciones ejecutadas sobre un producto bancario."""
        pass

    @abstractmethod
    def registrar_evento_auditoria(self, bitacora: BitacoraAuditoria) -> None:
        """Persiste un registro de auditoría inmutable en el almacén NoSQL."""
        pass

    @abstractmethod
    def consultar_bitacora_auditoria(self, filtro: Dict[str, Any], usuario_solicitante: Usuario) -> List[BitacoraAuditoria]:
        """Permite consultar el historial de auditoría de la plataforma."""
        pass


# ==========================================
# 7. AUTORIZACIÓN (Authorization Services)
# ==========================================

class ServicioAutorizacion(ABC):

    @abstractmethod
    def validar_permisos(self, usuario: Usuario, operacion_requerida: str) -> bool:
        """Evalúa si el rol del usuario permite ejecutar la operación deseada."""
        pass

    @abstractmethod
    def validar_acceso_cliente(self, usuario: Usuario, cliente_objetivo: Cliente) -> bool:
        """Determina si un usuario tiene acceso a los datos de un cliente particular."""
        pass

    @abstractmethod
    def validar_acceso_producto(self, usuario: Usuario, identificador_producto: str) -> bool:
        """Determina si el usuario puede operar sobre un producto específico."""
        pass

    @abstractmethod
    def validar_autorizacion_aprobacion(self, usuario: Usuario, monto: Decimal, tipo_operacion: str) -> bool:
        """Valida si el usuario posee la jerarquía suficiente para aprobar una operación."""
        pass

---

### ¿Cómo encaja en tu estructura de archivos?

Con esta definición agregada, el mapa completo de archivos en la carpeta **`Domain/`** queda estructurado de la siguiente forma:

```text
Domain/
├── Domain Model.md                # Entidades en dataclasses (Cliente, Cuenta, Prestamo)
├── Domain Value Objects.md        # Enums y catálogos (EstadoCuenta, RolSistema, Moneda)
├── Domain Services.md             # Contratos globales ABC de Servicios (Este archivo)
├── Output-ports.md                # Contratos abstractos de Repositorios e Infraestructura
└── services/                      # Detalle de implementaciones por subdominio
    ├── bank-account-services.md
    ├── customer-management-services.md
    ├── loan-services.md
    ├── operation-and-audit-services.md
    ├── transfer-services.md
    └── user-authentication-services.md
