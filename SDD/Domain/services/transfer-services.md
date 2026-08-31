# Servicios de Transferencias (Transfer Services)

## Introducción

Este documento define los servicios pertenecientes al subdominio de Gestión de Transferencias del Sistema de Gestión de Información Bancaria.
Los servicios de este subdominio gestionan el ciclo de vida completo de las transferencias de fondos entre cuentas bancarias: desde su creación, evaluación de montos, aprobación o rechazo, hasta su ejecución final o expiración.

Las transferencias están representadas por el Modelo de Dominio `Transferencia` y heredan de `ProductoBancario`.

```text
ProductoBancario
       │
       └── Transferencia
Cada acción de negocio en el ciclo de vida de una transferencia genera una Operacion y su correspondiente registro en BitacoraAuditoria.Contexto del Modelo de DominioUna Transferencia representa la movilización de fondos entre dos cuentas bancarias:PlaintextProductoBancario
       │
       └── Transferencia
              │
              ├── cuenta_origen: CuentaBancaria
              ├── cuenta_destino: CuentaBancaria
              ├── monto: Decimal
              ├── fecha_creacion: datetime
              ├── fecha_aprobacion: Optional[datetime]
              ├── estado_transferencia: EstadoTransferencia
              ├── creado_por: Usuario
              └── aprobado_por: Optional[Usuario]
Las relaciones se representan directamente utilizando los objetos del Modelo de Dominio:Transferencia.cuenta_origen: CuentaBancariaTransferencia.cuenta_destino: CuentaBancariaTransferencia.creado_por: UsuarioRegla: El Modelo de Dominio nunca sustituye estas relaciones por identificadores primitivos como id_cuenta_origen: str o id_usuario: str.Principios de Diseño del ServicioModelos de Dominio como ParámetrosTodos los servicios de Transferencias y sus Puertos de Entrada deben recibir exclusivamente Modelos de Dominio u Objetos de Valor (Value Objects).Incorrecto:Pythondef crear_transferencia(id_origen: str, id_destino: str, monto: Decimal) -> None: ...
Correcto:Pythondef crear_transferencia(transferencia: Transferencia) -> Transferencia: ...
Información ExternaSi una regla de negocio requiere datos no presentes en el objeto de dominio recibido (como el umbral de aprobación o límites globales), debe obtenerse a través de un Puerto de Salida (Output Port). Nunca directamente desde bases de datos o servicios REST externos.Ciclo de Vida de la TransferenciaEl estado operacional se gestiona mediante el enum/objeto de valor EstadoTransferencia:PlaintextPENDIENTE
   │
   ├──────────────► RECHAZADA
   │
   ▼
EN_ESPERA_DE_APROBACION
   │
   ├──────────────► RECHAZADA
   │
   ├──────────────► EXPIRADA
   │
   ▼
APROBADA
   │
   ▼
EJECUTADA
Umbral de AprobaciónEl sistema determina si una transferencia requiere aprobación consultando PuertoConfiguracionTransferencia:PlaintextServicioTransferencia ──► PuertoConfiguracionTransferencia ──► Umbral de Aprobación
                                                                     │
                                                 monto > umbral? ────┤
                                                                     ├── SÍ ──► EN_ESPERA_DE_APROBACION
                                                                     └── NO ──► APROBADA
Servicios de Transferencias1. Crear TransferenciaDescripción: Registra una solicitud de transferencia entre dos cuentas y asigna su estado inicial (APROBADA o EN_ESPERA_DE_APROBACION).Entrada: transferencia: TransferenciaValidaciones:Cuentas de origen y destino válidas y en estado ACTIVA.La cuenta de origen y la de destino deben ser distintas.Monto estrictamente positivo ($> 0$).Saldo suficiente en la cuenta de origen.Usuario creador autorizado.Persistencia y Auditoría: Se guarda mediante PuertoRepositorioTransferencia y genera la operación CREACION_TRANSFERENCIA.2. Ejecutar TransferenciaDescripción: Efectúa el débito en la cuenta de origen y el crédito en la cuenta de destino para transferencias en estado APROBADA.Entrada: usuario: Usuario, transferencia: TransferenciaEfecto de Dominio: Modifica los saldos invocando los métodos del objeto de dominio CuentaBancaria (debitar_saldo() y acreditar_saldo()).Persistencia: Guarda las cuentas actualizadas y la transferencia con estado EJECUTADA. Genera la operación EJECUCION_TRANSFERENCIA.3. Aprobar TransferenciaDescripción: Autoriza una transferencia que requería aprobación previa.Entrada: usuario: Usuario, transferencia: TransferenciaAutorización: El usuario debe estar activo y poseer el rol RolSistema.SUPERVISOR_EMPRESARIAL.Resultado: Establece fecha_aprobacion, aprobado_por y transiciona el estado a APROBADA. Genera la operación APROBACION_TRANSFERENCIA.4. Rechazar TransferenciaDescripción: Rechaza una transferencia en estado EN_ESPERA_DE_APROBACION o PENDIENTE.Entrada: usuario: Usuario, transferencia: TransferenciaResultado: Transiciona el estado a RECHAZADA. Genera la operación RECHAZO_TRANSFERENCIA.5. Expirar TransferenciaDescripción: Marca como expirada una transferencia que superó el tiempo máximo en EN_ESPERA_DE_APROBACION.Entrada: transferencia: TransferenciaRegla: No modifica el saldo de ninguna cuenta bancaria. Genera la operación EXPIRACION_TRANSFERENCIA.6. Consultar TransferenciaDescripción: Obtiene la representación en Modelo de Dominio de una transferencia validando los permisos del usuario consultante.Entrada: transferencia: Transferencia, usuario: UsuarioPuertos de Entrada e Interfaces en PythonDefinición de casos de uso mediante clases base abstractas (abc):Pythonfrom abc import ABC, abstractmethod
from typing import Optional

class CasoUsoCrearTransferencia(ABC):
    
    @abstractmethod
    def ejecutar(self, transferencia: Transferencia) -> Transferencia:
        """Crea y evalúa el flujo inicial de una transferencia."""
        pass


class CasoUsoEjecutarTransferencia(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario, transferencia: Transferencia) -> Transferencia:
        """Afecta los saldos de las cuentas e incrementa el historial a EJECUTADA."""
        pass


class CasoUsoAprobarTransferencia(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario, transferencia: Transferencia) -> Transferencia:
        """
        Aprueba una transferencia en espera.
        Requiere usuario.rol == RolSistema.SUPERVISOR_EMPRESARIAL
        """
        pass


class CasoUsoExpirarTransferencia(ABC):
    
    @abstractmethod
    def ejecutar(self, transferencia: Transferencia) -> Transferencia:
        """Expiración por tiempo límite sin mover fondos."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)Pythonfrom abc import ABC, abstractmethod
from decimal import Decimal
from datetime import timedelta
from typing import Optional

class PuertoRepositorioTransferencia(ABC):
    
    @abstractmethod
    def guardar(self, transferencia: Transferencia) -> Transferencia:
        pass

    @abstractmethod
    def buscar_por_id(self, id_transferencia: str) -> Optional[Transferencia]:
        pass


class PuertoConfiguracionTransferencia(ABC):
    
    @abstractmethod
    def obtener_umbral_aprobacion(self) -> Decimal:
        """Obtiene el monto límite a partir del cual se requiere aprobación."""
        pass

    @abstractmethod
    def obtener_tiempo_expiracion(self) -> timedelta:
        """Obtiene el tiempo máximo permitido para aprobación."""
        pass


class PuertoRepositorioCuentaBancaria(ABC):
    
    @abstractmethod
    def guardar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        pass
Flujo de Ejecución de TransferenciaPlaintextTransferencia (Estado: APROBADA)
        │
        ▼
CasoUsoEjecutarTransferencia (Servicio de Dominio)
        │
        ├── 1. Validar transferencia.estado == EstadoTransferencia.APROBADA
        ├── 2. Validar estado de cuenta_origen y cuenta_destino == ACTIVA
        ├── 3. Validar saldo suficiente en cuenta_origen
        │
        ├── 4. transferencia.cuenta_origen.debitar_saldo(transferencia.monto)
        ├── 5. transferencia.cuenta_destino.acreditar_saldo(transferencia.monto)
        ├── 6. transferencia.marcar_como_ejecutada()
        │
        ├── PuertoRepositorioCuentaBancaria.guardar(cuenta_origen)
        ├── PuertoRepositorioCuentaBancaria.guardar(cuenta_destino)
        ├── PuertoRepositorioTransferencia.guardar(transferencia)
        ├── PuertoRepositorioOperacion.guardar(operacion_ejecucion)
        └── PuertoRepositorioAuditoria.guardar(log_auditoria_ejecucion)
Catálogo de Excepciones de DominioExcepcionTransferenciaNoEncontradaExcepcionEstadoTransferenciaInvalidoExcepcionTransicionEstadoTransferenciaInvalidaExcepcionSaldoInsuficienteParaTransferenciaExcepcionCuentasTransferenciaIdenticasExcepcionUsuarioNoAutorizadoParaAprobarTransferenciaExcepcionTransferenciaExpiradaRestricciones Arquitectónicas (Contexto Python)Aislamiento de Infraestructura: El servicio de transferencias no debe importar Django, FastAPI, SQLAlchemy ni Pydantic.Sin Umbrales en Duro (No Hardcoding): Ningún valor límite de aprobación o tiempo de expiración debe estar predefinido en código. Deben consultarse mediante PuertoConfiguracionTransferencia.Modificación de Saldos Segura: La transferencia nunca altera directamente los campos numéricos de las cuentas en la base de datos; invoca los métodos explícitos del objeto de dominio CuentaBancaria.Pruebas de Dominio Aisladas: Todo el subdominio de transferencias debe ser testable unitariamente con pytest mockeando los puertos de salida en memoria.
