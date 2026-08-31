# Servicios de Préstamos (Loan Services)

## Introducción

Este documento define los servicios pertenecientes al subdominio de Préstamos del Sistema de Gestión de Información Bancaria.
Los servicios de este subdominio gestionan el ciclo de vida completo de las solicitudes de crédito: desde la solicitud inicial, evaluación, aprobación o rechazo, desembolso, hasta el cierre final.

Los préstamos están representados por el Modelo de Dominio `Prestamo` y heredan de `ProductoBancario`.

```text
ProductoBancario
       │
       └── Prestamo
El ciclo de vida del préstamo genera operaciones de negocio que deben ser rastreables mediante los Modelos de Dominio Operacion y BitacoraAuditoria.

Contexto del Modelo de Dominio
Un Prestamo representa un producto de crédito solicitado por un cliente.

Plaintext
ProductoBancario
       │
       └── Prestamo
              │
              ├── solicitante: Cliente
              ├── tipo_prestamo: TipoPrestamo
              ├── monto_solicitado: Decimal
              ├── monto_aprobado: Decimal
              ├── tasa_interes: Decimal
              ├── plazo_meses: int
              ├── estado_prestamo: EstadoPrestamo
              ├── fecha_aprobacion: Optional[date]
              ├── fecha_desembolso: Optional[date]
              └── cuenta_destino: BankAccount
El solicitante y la cuenta destino se representan directamente como objetos del dominio:

Prestamo.solicitante: Cliente

Prestamo.cuenta_destino: CuentaBancaria

Regla: El Modelo de Dominio nunca sustituye estas relaciones por identificadores primitivos como id_cliente: str o id_cuenta: str.

Principios de Diseño del Servicio
Modelos de Dominio como Parámetros
Todos los servicios de Préstamos y sus Puertos de Entrada deben recibir exclusivamente Modelos de Dominio u Objetos de Valor (Value Objects).

Incorrecto:

Python
def solicitar_prestamo(id_cliente: str, tipo_prestamo: str, monto: Decimal) -> None: ...
Correcto:

Python
def solicitar_prestamo(prestamo: Prestamo) -> Prestamo: ...
Información Externa
El servicio valida la información disponible en el objeto de dominio. Si requiere datos persistidos adicionales, consulta un Puerto de Salida (Output Port).

Plaintext
ServicioPrestamos ──► PuertoSalida ──► AdaptadorSalida ──► Base de Datos
Ciclo de Vida del Préstamo
El estado operacional se gestiona mediante el enum/objeto de valor EstadoPrestamo:

Plaintext
EN_ESTUDIO
    │
    ├──────────────► RECHAZADO
    │
    ├──────────────► CANCELADO
    │
    ▼
APROBADO
    │
    ├──────────────► CANCELADO
    │
    ▼
DESEMBOLSADO
    │
    ▼
VENCIDO ──► CANCELADO
El dominio es el único responsable de validar si una transición de estado es válida.

Servicios de Préstamos
1. Solicitar Préstamo
Descripción: Registra una nueva solicitud de crédito para un Cliente en estado EstadoPrestamo.EN_ESTUDIO.

Entrada: prestamo: Prestamo

Validaciones: Monto solicitado, plazo, tipo de préstamo y cuenta destino válidos.

Persistencia y Auditoría: Se guarda mediante PuertoRepositorioPrestamo y genera un evento de auditoría (LOAN_APPLICATION).

2. Consultar Préstamo
Descripción: Obtiene la representación en modelo de dominio de un préstamo existente.

Entrada: prestamo: Prestamo

3. Evaluar Préstamo
Descripción: Coordina la evaluación crediticia para determinar si cumple las condiciones de aprobación.

Entrada: prestamo: Prestamo

Resultado: Determina si el estado transiciona a APROBADO o RECHAZADO.

4. Aprobar Préstamo
Descripción: Aprueba la solicitud de préstamo y establece el monto aprobado y la fecha de aprobación.

Entrada: usuario: Usuario, prestamo: Prestamo

Autorización Estricta: Requiere que usuario.rol == RolSistema.ANALISTA_INTERNO. De lo contrario, lanza una excepción de dominio.

Persistencia: Guarda los cambios y registra la operación LOAN_APPROVAL.

5. Rechazar Préstamo
Descripción: Rechaza una solicitud bajo estudio.

Entrada: usuario: Usuario, prestamo: Prestamo

Autorización: Requiere rol RolSistema.ANALISTA_INTERNO.

Resultado: Transiciona el estado a RECHAZADO.

6. Desembolsar Préstamo
Descripción: Transfiere el monto aprobado a la cuenta_destino asociada al préstamo.

Entrada: prestamo: Prestamo

Validaciones:

El préstamo debe estar en estado APROBADO.

La cuenta destino debe estar activa y apta para recibir abonos.

Efecto: Invoca el comportamiento del dominio CuentaBancaria para acreditar el saldo, actualiza la fecha de desembolso (fecha_desembolso) y cambia el estado a DESEMBOLSADO.

7. Cancelar Préstamo
Descripción: Cancela un préstamo que esté en un estado permisible (EN_ESTUDIO, APROBADO o VENCIDO).

Entrada: usuario: Usuario, prestamo: Prestamo

8. Consultar Elegibilidad de Préstamo
Descripción: Evalúa de forma previa si un cliente y una solicitud cumplen los criterios de riesgo del dominio antes de iniciar la solicitud formal.

Puertos de Entrada e Interfaces en Python
Definición de casos de uso mediante clases base abstractas (abc):

Python
from abc import ABC, abstractmethod
from typing import Optional

class CasoUsoSolicitarPrestamo(ABC):
    
    @abstractmethod
    def ejecutar(self, prestamo: Prestamo) -> Prestamo:
        """Registra una nueva solicitud de préstamo."""
        pass


class CasoUsoAprobarPrestamo(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario, prestamo: Prestamo) -> Prestamo:
        """
        Aprueba un préstamo en estudio.
        Requiere usuario.rol == RolSistema.ANALISTA_INTERNO
        """
        pass


class CasoUsoRechazarPrestamo(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario, prestamo: Prestamo) -> Prestamo:
        """Rechaza un préstamo en estudio."""
        pass


class CasoUsoDesembolsarPrestamo(ABC):
    
    @abstractmethod
    def ejecutar(self, prestamo: Prestamo) -> Prestamo:
        """Acredita el dinero en la cuenta bancaria destino y actualiza el préstamo."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)
Python
from abc import ABC, abstractmethod
from typing import Optional

class PuertoRepositorioPrestamo(ABC):
    
    @abstractmethod
    def guardar(self, prestamo: Prestamo) -> Prestamo:
        pass

    @abstractmethod
    def buscar_por_id(self, id_prestamo: str) -> Optional[Prestamo]:
        pass


class PuertoRepositorioCuentaBancaria(ABC):
    
    @abstractmethod
    def guardar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        pass


class PuertoRepositorioAuditoria(ABC):
    
    @abstractmethod
    def registrar_log(self, log_auditoria: RegistroAuditoria) -> None:
        pass
Flujo de Desembolso de Préstamo
Plaintext
Solicitud de Desembolso
        │
        ▼
CasoUsoDesembolsarPrestamo (Servicio de Dominio)
        │
        ├── 1. Validar prestamo.estado_prestamo == EstadoPrestamo.APROBADO
        ├── 2. Validar prestamo.cuenta_destino esta ACTIVA
        │
        ├── 3. prestamo.cuenta_destino.acreditar_saldo(prestamo.monto_aprobado)
        ├── 4. prestamo.marcar_como_desembolsado()
        │
        ├── PuertoRepositorioCuentaBancaria.guardar(prestamo.cuenta_destino)
        ├── PuertoRepositorioPrestamo.guardar(prestamo)
        └── PuertoRepositorioAuditoria.registrar_log(evento_desembolso)
Catálogo de Excepciones de Dominio
ExcepcionPrestamoNoEncontrado

ExcepcionEstadoPrestamoInvalido

ExcepcionTransicionEstadoPrestamoInvalida

ExcepcionPrestamoYaAprobado

ExcepcionPrestamoYaRechazado

ExcepcionPrestamoYaDesembolsado

ExcepcionMontoPrestamoInvalido

ExcepcionUsuarioNoAutorizadoParaAprobar

ExcepcionDesembolsoPrestamo

Restricciones Arquitectónicas (Contexto Python)
Sin Dependencias Tecnológicas: Las clases de servicios y casos de uso de préstamos no deben importar Django, FastAPI, SQLAlchemy, MongoEngine ni Pydantic.

Uso de Annotations/Type Hints: Uso obligatorio de tipos estáticos (Prestamo, Usuario, CuentaBancaria).

Desembolso Seguro: El incremento del saldo en la cuenta de destino durante el desembolso debe realizarse mediante los métodos del objeto de dominio CuentaBancaria y persistirse usando el PuertoRepositorioCuentaBancaria.

Pruebas Unitarias Aisladas: Todo el ciclo de vida del préstamo debe ser testable con unittest o pytest en memoria sin requerir conexiones a MySQL o MongoDB.
