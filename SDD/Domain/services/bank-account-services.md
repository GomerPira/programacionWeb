# Servicios de Cuentas Bancarias (Bank Account Services)

## Introducción

Este documento define los servicios pertenecientes al subdominio de Cuentas Bancarias del Sistema de Gestión de Información Bancaria.
Los servicios en este subdominio son responsables de gestionar el ciclo de vida y las operaciones de negocio asociadas con las cuentas bancarias.

Las responsabilidades principales incluyen:
* Apertura de cuentas bancarias.
* Consulta de cuentas bancarias.
* Depósito de fondos.
* Retiro de fondos.
* Bloqueo de cuentas.
* Desbloqueo de cuentas.
* Cierre de cuentas.
* Gestión de saldos de cuentas.
* Validación de propiedad de cuentas.
* Validación de estado de cuentas.
* Registro de operaciones para auditoría.

Las cuentas bancarias están representadas por el Modelo de Dominio `CuentaBancaria` y heredan de `ProductoBancario`.

```text
ProductoBancario
       │
       └── CuentaBancaria
Cada operación significativa realizada sobre una cuenta bancaria debe representarse como una Operacion de negocio y registrarse en la BitacoraAuditoria cuando lo exijan las reglas de negocio.Contexto del Modelo de DominioUna CuentaBancaria representa un producto bancario cuyo titular es un Cliente.PlaintextProductoBancario
       │
       └── CuentaBancaria
              │
              ├── tipo_cuenta: TipoCuenta
              ├── titular: Cliente
              ├── saldo_actual: Decimal
              ├── moneda: Moneda
              ├── estado_cuenta: EstadoCuenta
              └── fecha_apertura: datetime
El titular de la cuenta debe representarse utilizando el Modelo de Dominio Cliente. La relación no debe expresarse como un identificador primitivo como id_cliente: str. En su lugar, debe usarse titular: Cliente. El adaptador de persistencia se encarga de traducir esta relación de dominio hacia la base de datos.Principios de Diseño del ServicioModelos de Dominio como ParámetrosTodos los servicios de Cuentas Bancarias y Puertos de Entrada deben recibir Modelos de Dominio u Objetos de Valor (Value Objects). Nunca deben recibir identificadores primitivos, DTOs de petición o entidades de persistencia.Incorrecto:Pythondef abrir_cuenta(id_cliente: str, tipo_cuenta: TipoCuenta, moneda: Moneda) -> None: ...
Correcto:Pythondef abrir_cuenta(cuenta_bancaria: CuentaBancaria) -> None: ...
Información ExternaEl servicio valida la información directamente sobre el Modelo de Dominio siempre que sea posible. Si se requiere información externa al Dominio, el servicio debe usar un Puerto de Salida.PlaintextCuentaBancaria ──► ServicioCuentaBancaria ──► PuertoRepositorioCliente ──► AdaptadorPersistencia ──► Base de Datos
Operaciones de Dominio1. Abrir Cuenta BancariaDescripción: Crea una nueva CuentaBancaria asociada a un Cliente e inicializa su estado.Entrada: cuenta_bancaria: CuentaBancariaValidaciones: Tipo de cuenta, moneda, titular, estado inicial y saldo inicial válidos. Si se requiere validación adicional del cliente en persistencia, consulta el PuertoRepositorioCliente.Persistencia y Auditoría: Guarda la cuenta mediante PuertoRepositorioCuentaBancaria y genera el registro en el PuertoRepositorioAuditoria.2. Consultar Cuenta BancariaDescripción: Recupera la información de una cuenta bancaria existente expresada en el modelo de dominio.Entrada: cuenta_bancaria: CuentaBancaria3. Depositar FondosDescripción: Incrementa el saldo_actual de la cuenta.Entrada: cuenta_bancaria: CuentaBancaria, monto: DineroValidaciones: Estado de cuenta activo y monto de depósito mayor a cero ($monto > 0$).Procesamiento: El cambio de saldo debe ejecutarse mediante el comportamiento encapsulado en el Modelo de Dominio CuentaBancaria.4. Retirar FondosDescripción: Disminuye el saldo_actual de la cuenta.Entrada: cuenta_bancaria: CuentaBancaria, monto: DineroValidaciones:Estado de cuenta activo.Fondos suficientes: saldo_actual >= monto. Si se viola, lanza ExcepcionSaldoInsuficiente.5. Bloquear Cuenta BancariaDescripción: Transiciona el estado operacional de la cuenta a EstadoCuenta.BLOQUEADA.Entrada: cuenta_bancaria: CuentaBancaria6. Desbloquear Cuenta BancariaDescripción: Restaura una cuenta bloqueada al estado EstadoCuenta.ACTIVA.Entrada: cuenta_bancaria: CuentaBancaria7. Cerrar Cuenta BancariaDescripción: Cambia permanentemente el estado a EstadoCuenta.CERRADA. No se permite reabrir una cuenta cerrada.Entrada: cuenta_bancaria: CuentaBancaria8. Validar Propiedad de la CuentaDescripción: Comprueba si un Cliente es el titular legítimo de una CuentaBancaria comparando las relaciones entre los Modelos de Dominio.9. Consultar Saldo de CuentaDescripción: Retorna el saldo actual presente en cuenta_bancaria.saldo_actual.Puertos de Entrada e Interfaces en PythonDefinición de casos de uso mediante clases base abstractas (abc):Pythonfrom abc import ABC, abstractmethod
from decimal import Decimal
from typing import Optional

class CasoUsoAbrirCuentaBancaria(ABC):
    
    @abstractmethod
    def ejecutar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        """Abre una nueva cuenta bancaria."""
        pass


class CasoUsoDepositarFondos(ABC):
    
    @abstractmethod
    def ejecutar(self, cuenta: CuentaBancaria, monto: Dinero) -> None:
        """Realiza un depósito en una cuenta activa."""
        pass


class CasoUsoRetirarFondos(ABC):
    
    @abstractmethod
    def ejecutar(self, cuenta: CuentaBancaria, monto: Dinero) -> None:
        """Realiza un retiro verificando la disponibilidad de saldo."""
        pass


class CasoUsoBloquearCuentaBancaria(ABC):
    
    @abstractmethod
    def ejecutar(self, cuenta: CuentaBancaria) -> None:
        """Cambia el estado de la cuenta a BLOQUEADA."""
        pass


class CasoUsoCerrarCuentaBancaria(ABC):
    
    @abstractmethod
    def ejecutar(self, cuenta: CuentaBancaria) -> None:
        """Cierra permanentemente la cuenta bancaria."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)Pythonfrom abc import ABC, abstractmethod
from typing import List, Optional

class PuertoRepositorioCuentaBancaria(ABC):
    
    @abstractmethod
    def guardar(self, cuenta: CuentaBancaria) -> CuentaBancaria:
        pass

    @abstractmethod
    def buscar_por_numero(self, numero_cuenta: str) -> Optional[CuentaBancaria]:
        pass

    @abstractmethod
    def existe_numero_cuenta(self, numero_cuenta: str) -> bool:
        pass


class PuertoRepositorioCliente(ABC):
    
    @abstractmethod
    def buscar_por_identificacion(self, identificacion: str) -> Optional[Cliente]:
        pass


class PuertoRepositorioAuditoria(ABC):
    
    @abstractmethod
    def registrar_log(self, log_auditoria: RegistroAuditoria) -> None:
        """Persiste los eventos de auditoría en la base de datos NoSQL (MongoDB)."""
        pass
Flujo de Trabajo (Ejemplo: Retiro de Fondos)PlaintextSolicitud de Retiro
        │
        ▼
Adaptador de Entrada (Mapea DTO a Modelo de Dominio CuentaBancaria)
        │
        ▼
CasoUsoRetirarFondos (Servicio de Dominio)
        │
        ├── 1. Validar cuenta.estado_cuenta == EstadoCuenta.ACTIVA
        ├── 2. Validar cuenta.saldo_actual >= monto_retiro
        ├── 3. Ejecutar cuenta.decrementar_saldo(monto_retiro)
        │
        ├── PuertoRepositorioCuentaBancaria.guardar(cuenta)
        └── PuertoRepositorioAuditoria.registrar_log(evento_retiro)
Catálogo de Excepciones de DominioLas excepciones conceptuales manejadas por este subdominio son:ExcepcionCuentaBancariaNoEncontradaExcepcionEstadoCuentaInvalidoExcepcionTransicionEstadoInvalidaExcepcionSaldoInsuficienteExcepcionDepositoInvalidoExcepcionRetiroInvalidoExcepcionCuentaYaCerradaExcepcionPropiedadCuentaInvalidaRestricciones Arquitectónicas (Contexto Python)Aislado de Infraestructura: El dominio de Cuentas Bancarias no debe depender de ORMs (SQLAlchemy, Django ORM), conectores de MySQL (mysql-connector), conectores de MongoDB (pymongo) ni librerías de API (FastAPI, Flask, Pydantic).Uso Estricto de Type Hints: Todos los parámetros y retornos deben usar anotaciones de tipo nativas de Python (CuentaBancaria, Dinero, Cliente).Manejo de Transacciones Financieras: Las modificaciones sobre el saldo deben ejecutarse mediante métodos del Modelo de Dominio y nunca asignando valores directamente a atributos privados desde adaptadores externos.Separación de Persistencia: Los objetos devueltos por el repositorio de base de datos deben convertirse a Modelos de Dominio dentro del adaptador antes de llegar a la capa de servicios.
