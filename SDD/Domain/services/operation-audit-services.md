# Servicios de Operación y Auditoría (Operation and Audit Services)

## Introducción

Este documento define los servicios encargados de gestionar las operaciones de negocio y los registros de auditoría dentro del Sistema de Gestión de Información Bancaria.
El propósito de este subdominio es garantizar la trazabilidad de las acciones de negocio significativas ejecutadas sobre los productos bancarios.

Se consideran productos bancarios a aquellos que heredan del modelo abstracto `ProductoBancario`:
* `CuentaBancaria`
* `Prestamo`
* `Transferencia`

```text
ProductoBancario
       │
       ▼
Acción de Negocio
       │
       ├── Operacion
       │
       └── BitacoraAuditoria
Este subdominio es responsable de registrar y consultar estos eventos. No implementa las reglas de negocio de los productos que los originan. Por ejemplo, el servicio de CuentaBancaria decide si un retiro es válido; el servicio de Operación y Auditoría solo registra que dicho retiro ocurrió.

Contexto del Modelo de Dominio
Los Modelos de Dominio principales son:

Plaintext
Operacion
├── id_operacion: UUID / int
├── tipo_operacion: TipoOperacion
├── fecha_ejecucion: datetime
├── ejecutado_por: Usuario
└── producto_afectado: ProductoBancario
Y el registro histórico de auditoría:

Plaintext
BitacoraAuditoria
├── id_auditoria: UUID / str
├── tipo_operacion: TipoOperacion
├── fecha_operacion: datetime
├── ejecutado_por: Usuario
├── rol_usuario: RolSistema
├── producto_afectado: ProductoBancario
└── detalles: Dict[str, Any]
Regla: El Modelo de Dominio nunca sustituye las relaciones por identificadores primitivos como id_usuario: str o id_producto: str.

Principios de Diseño del Servicio
Modelos de Dominio como Parámetros
Todos los servicios de Operación y Auditoría deben recibir exclusivamente Modelos de Dominio u Objetos de Valor (Value Objects).

Incorrecto:

Python
def registrar_operacion(id_usuario: str, id_producto: str, tipo: TipoOperacion) -> None: ...
Correcto:

Python
def registrar_operacion(operacion: Operacion) -> Operacion: ...
Inmutabilidad de la Auditoría
Los registros de BitacoraAuditoria representan eventos históricos. Una vez creados y guardados en la base de datos NoSQL (MongoDB), son estrictamente inmutables y no se permite su modificación ni actualización.

Tipos de Operación (Objeto de Valor TipoOperacion)
El enum u objeto de valor TipoOperacion define los eventos soportados por el dominio:

Cuentas Bancarias: APERTURA_CUENTA, DEPOSITO, RETIRO, BLOQUEO_CUENTA, DESBLOQUEO_CUENTA, CIERRE_CUENTA.

Transferencias: CREACION_TRANSFERENCIA, APROBACION_TRANSFERENCIA, RECHAZO_TRANSFERENCIA, EJECUCION_TRANSFERENCIA, EXPIRACION_TRANSFERENCIA.

Préstamos: SOLICITUD_PRESTAMO, APROBACION_PRESTAMO, RECHAZO_PRESTAMO, DESEMBOLSO_PRESTAMO, PAGO_PRESTAMO, CANCELACION_PRESTAMO.

Servicios del Subdominio
1. Registrar Operación
Descripción: Almacena una operación de negocio ejecutada sobre un ProductoBancario.

Entrada: operacion: Operacion

Persistencia: Se guarda mediante PuertoRepositorioOperacion (adaptador relacional MySQL).

2. Registrar Log de Auditoría
Descripción: Crea un registro histórico inmutable de un evento de negocio.

Entrada: log_auditoria: BitacoraAuditoria

Persistencia: Se persiste mediante PuertoRepositorioAuditoria (adaptador NoSQL MongoDB).

3. Registrar Operación y Auditoría (Coordinado)
Descripción: Coordina el guardado simultáneo de la Operacion y su respectiva BitacoraAuditoria dentro del mismo flujo de aplicación.

Entrada: operacion: Operacion

4. Consultas de Auditoría u Operaciones
Descripción: Permite filtrar y recuperar historiales por usuario o por producto bancario.

Entradas: usuario: Usuario o producto: ProductoBancario.

Puertos de Entrada e Interfaces en Python
Definición de casos de uso mediante clases base abstractas (abc):

Python
from abc import ABC, abstractmethod
from typing import List, Optional

class CasoUsoRegistrarOperacion(ABC):
    
    @abstractmethod
    def ejecutar(self, operacion: Operacion) -> Operacion:
        """Registra una operación de negocio en el sistema."""
        pass


class CasoUsoRegistrarBitacoraAuditoria(ABC):
    
    @abstractmethod
    def ejecutar(self, log_auditoria: BitacoraAuditoria) -> BitacoraAuditoria:
        """Persiste un registro de auditoría inmutable."""
        pass


class CasoUsoRegistrarOperacionYAuditoria(ABC):
    
    @abstractmethod
    def ejecutar(self, operacion: Operacion) -> Operacion:
        """Coordina el registro de la operación y la bitácora de auditoría asociada."""
        pass


class CasoUsoConsultarOperacionesPorProducto(ABC):
    
    @abstractmethod
    def ejecutar(self, producto: ProductoBancario) -> List[Operacion]:
        """Consulta el historial de operaciones asociadas a un producto bancario."""
        pass


class CasoUsoConsultarAuditoriaPorUsuario(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario) -> List[BitacoraAuditoria]:
        """Consulta la trazabilidad de auditoría generada por un usuario."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)
Python
from abc import ABC, abstractmethod
from typing import List, Optional

class PuertoRepositorioOperacion(ABC):
    
    @abstractmethod
    def guardar(self, operacion: Operacion) -> Operacion:
        """Persiste la operación en el almacenamiento relacional (MySQL)."""
        pass

    @abstractmethod
    def buscar_por_producto(self, producto: ProductoBancario) -> List[Operacion]:
        pass


class PuertoRepositorioAuditoria(ABC):
    
    @abstractmethod
    def guardar(self, log_auditoria: BitacoraAuditoria) -> BitacoraAuditoria:
        """Persiste el registro de auditoría en la base de datos documental (MongoDB)."""
        pass

    @abstractmethod
    def buscar_por_producto(self, producto: ProductoBancario) -> List[BitacoraAuditoria]:
        pass
Arquitectura de Persistencia Diversificada
El subdominio coordina el almacenamiento en dos tecnologías distintas de forma completamente transparente para la capa de dominio:

Plaintext
                             ServicioDominio (Python)
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
      PuertoRepositorioOperacion            PuertoRepositorioAuditoria
                    │                                     │
                    ▼                                     ▼
        AdaptadorPersistenciaMySQL           AdaptadorPersistenciaMongoDB
                    │                                     │
                    ▼                                     ▼
               Base MySQL                          Base MongoDB
          (Tabla: operaciones)               (Colección: audit_logs)
Catálogo de Excepciones de Dominio
ExcepcionOperacionInvalida

ExcepcionOperacionNoEncontrada

ExcepcionTipoOperacionInvalido

ExcepcionBitacoraAuditoriaInvalida

ExcepcionBitacoraAuditoriaNoEncontrada

ExcepcionDetallesAuditoriaInvalidos

Restricciones Arquitectónicas (Contexto Python)
Aislamiento de Persistencia: Los componentes del subdominio de Operación y Auditoría no deben importar ni hacer referencia directa a pymongo, SQLAlchemy, Django Models, Pydantic ni drivers de infraestructura.

Representación Estricta con Type Hints: Los atributos de relaciones siempre se anotan como objetos de dominio (ejecutado_por: Usuario, producto_afectado: ProductoBancario).

Estructura Flexible para Detalle: El atributo detalles del objeto BitacoraAuditoria debe mapearse usando tipos nativos (Dict[str, Any]), garantizando que la estructura del log sea dinámica pero respetando el encapsulamiento.

Pruebas Unitarias Aisladas: La capa debe poder evaluarse con pytest en memoria mockeando los puertos de salida abstractos, sin requerir instancias reales de MySQL o MongoDB.
