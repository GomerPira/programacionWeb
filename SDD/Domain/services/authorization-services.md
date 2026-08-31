# Servicios de Autorización (Authorization Services)

## Introducción

Este documento define los servicios de dominio responsables de la autorización dentro del Sistema de Gestión de Información Bancaria.
La autorización determina si un `Usuario` autenticado tiene permitido realizar una operación de negocio específica según su `RolSistema` y las reglas de negocio asociadas a dicha operación.

La autenticación y la autorización son responsabilidades separadas:

```text
Autenticación
     │
     ▼
¿Quién es el Usuario?
     │
     ▼
Autorización
     │
     ▼
¿Qué puede hacer el Usuario?
Autenticación: Es responsable de validar las credenciales y establecer la identidad del usuario.

Autorización: Es responsable de determinar si ese usuario autenticado tiene permiso para ejecutar una operación solicitada.

Los servicios de autorización no ejecutan la operación de negocio en sí. Determinan si la operación puede ser iniciada por el Usuario actual.

Contexto del Modelo de Dominio
La autorización se basa principalmente en los siguientes Modelos de Dominio:

Plaintext
Usuario
├── id_usuario: int
├── nombre_usuario: str
├── rol: RolSistema
├── estado: EstadoUsuario
└── cliente: Optional[Cliente]
El rol de autorización del usuario está representado por el enum/objeto de valor Usuario.rol. El servicio debe trabajar directamente con el Modelo de Dominio Usuario en lugar de recibir roles primitivos o cadenas de texto genéricas (strings).

Principios de Autorización
El Usuario como Modelo de Dominio
Los servicios de autorización deben recibir un Modelo de Dominio Usuario.

Incorrecto:

Python
def autorizar(nombre_usuario: str, rol: str) -> bool: ...
Correcto:

Python
def autorizar(usuario: Usuario) -> bool: ...
Operación y Producto como Modelos de Dominio
Cuando se requiere autorización para una operación o producto de negocio, el servicio debe recibir el Modelo de Dominio correspondiente (CuentaBancaria, Prestamo, Transferencia).

Incorrecto:

Python
def autorizar_transferencia(id_usuario: int, id_transferencia: int) -> bool: ...
Correcto:

Python
def autorizar_transferencia(usuario: Usuario, transferencia: Transferencia) -> bool: ...
Alcance de la Autorización
Autorización de Lectura (Consulta)
Determina si un usuario tiene permitido consultar o leer información.

puede_consultar_cliente(usuario: Usuario, cliente: Cliente) -> bool

puede_consultar_prestamo(usuario: Usuario, prestamo: Prestamo) -> bool

puede_consultar_cuenta_bancaria(usuario: Usuario, cuenta: CuentaBancaria) -> bool

puede_consultar_transferencia(usuario: Usuario, transferencia: Transferencia) -> bool

puede_consultar_bitacora(usuario: Usuario) -> bool

Autorización de Ejecución (Modificación)
Determina si un usuario tiene permitido ejecutar o modificar una operación de dominio.

puede_ejecutar_deposito(usuario: Usuario, cuenta: CuentaBancaria) -> bool

puede_ejecutar_retiro(usuario: Usuario, cuenta: CuentaBancaria) -> bool

puede_aprobar_prestamo(usuario: Usuario, prestamo: Prestamo) -> bool

puede_aprobar_transferencia(usuario: Usuario, transferencia: Transferencia) -> bool

Roles del Sistema y Reglas de Autorización
El sistema utiliza el Objeto de Valor RolSistema (Enum en Python):

CLIENTE_NATURAL: Puede consultar y operar exclusivamente sobre sus propios productos bancarios.

CLIENTE_EMPRESA: Entidad titular de productos empresariales (operados a través de Operadores o Supervisores).

CAJERO_EMPLEADO: Puede consultar cualquier cliente/producto; ejecuta depósitos y retiros; no puede aprobar préstamos ni transferencias.

ASESOR_COMERCIAL: Puede consultar cualquier cliente y producto sin restricciones.

OPERADOR_EMPRESA: Realiza operaciones en nombre del ClienteEmpresa asignado.

SUPERVISOR_EMPRESA: Aprueba transferencias de alto monto en nombre del ClienteEmpresa asignado.

ANALISTA_INTERNO: Consulta cualquier entidad; es el único rol autorizado para aprobar/rechazar préstamos y gestionar personal interno.

Validación del Estado del Usuario
La autorización evalúa Usuario.estado (EstadoUsuario.ACTIVO, EstadoUsuario.INACTIVO, EstadoUsuario.BLOQUEADO).
A los usuarios que no estén en estado ACTIVO se les denegará inmediatamente cualquier permiso de ejecución:

Plaintext
Usuario ──► EstadoUsuario (¿ACTIVO?) ──► RolSistema ──► Resultado de Autorización
Puertos de Entrada e Interfaces en Python
Utilizando el módulo nativo abc de Python para definir Casos de Uso / Puertos de Entrada estrictos:

Python
from abc import ABC, abstractmethod
from typing import Optional

class CasoUsoAutorizarOperacion(ABC):
    
    @abstractmethod
    def autorizar(self, usuario: Usuario, operacion: Operacion) -> bool:
        """Determina si un usuario está autorizado para realizar una operación genérica."""
        pass


class CasoUsoAutorizarOperacionCuentaBancaria(ABC):
    
    @abstractmethod
    def autorizar(self, usuario: Usuario, cuenta: CuentaBancaria, operacion: Operacion) -> bool:
        """Valida la autorización del usuario sobre una CuentaBancaria específica."""
        pass


class CasoUsoAutorizarAprobacionPrestamo(ABC):
    
    @abstractmethod
    def autorizar(self, usuario: Usuario, prestamo: Prestamo) -> bool:
        """
        Valida si el usuario puede aprobar un préstamo.
        Regla: usuario.rol == RolSistema.ANALISTA_INTERNO 
              y usuario.estado == EstadoUsuario.ACTIVO 
              y prestamo.estado == EstadoPrestamo.EN_ESTUDIO
        """
        pass


class CasoUsoAutorizarAprobacionTransferencia(ABC):
    
    @abstractmethod
    def autorizar(self, usuario: Usuario, transferencia: Transferencia) -> bool:
        """
        Valida los derechos de aprobación de transferencias de alto monto para SUPERVISOR_EMPRESA.
        """
        pass
Puertos de Salida (Clases Base Abstractas en Python)
Cuando las reglas de autorización requieren información externa, el servicio utiliza Puertos de Salida:

Python
from abc import ABC, abstractmethod

class PuertoRepositorioUsuario(ABC):
    
    @abstractmethod
    def buscar_por_usuario(self, usuario: Usuario) -> Optional[Usuario]:
        pass


class PuertoRepositorioCuentaBancaria(ABC):
    
    @abstractmethod
    def buscar_por_numero_cuenta(self, numero_cuenta: str) -> Optional[CuentaBancaria]:
        pass


class PuertoRepositorioPrestamo(ABC):
    
    @abstractmethod
    def buscar_por_id(self, id_prestamo: int) -> Optional[Prestamo]:
        pass


class PuertoRepositorioTransferencia(ABC):
    
    @abstractmethod
    def buscar_por_id(self, id_transferencia: int) -> Optional[Transferencia]:
        pass
Flujo de Autorización
Plaintext
Solicitud Autenticada
        │
        ▼
 Adaptador de Seguridad (Extrae el Modelo de Dominio Usuario)
        │
        ▼
 Servicio de Autorización (Servicio de Dominio en Python)
        │
        ├── 1. Validar Usuario.estado == EstadoUsuario.ACTIVO
        ├── 2. Validar Usuario.rol dentro de RolSistema
        ├── 3. Validar Invariantes de Dominio y Propiedad
        └── 4. Consultar Puertos de Salida (si se requieren datos externos)
        │
        ├── Autorizado ──► Continuar al Servicio de Negocio
        └── No Autorizado ──► Lanzar ExcepcionAutorizacion
Restricciones Arquitectónicas (Contexto Python)
Dominio Puro en Python: Los servicios de autorización del dominio no deben depender de frameworks como Django, FastAPI, Flask, Pydantic, SQLAlchemy ni de módulos de HTTP/REST.

Anotaciones de Tipo (Type Hints): Todos los métodos deben usar tipado estricto en Python (Usuario, CuentaBancaria, Transferencia).

Sin Primitivos: Nunca se deben pasar variables como id_usuario: int o rol: str directamente a la lógica de autorización.

Excepciones: Los fallos de autorización deben lanzar excepciones de dominio (por ejemplo: ExcepcionOperacionNoAutorizada, ExcepcionUsuarioInactivo).
