# Servicios de Clientes (Customer Services)

## Introducción

Este documento define los servicios pertenecientes al subdominio de Gestión de Clientes del Sistema de Gestión de Información Bancaria.
Los servicios de este subdominio son responsables de la creación, consulta, actualización, cambio de estado y gestión del ciclo de vida de los clientes bancarios (Personas Naturales y Empresas).

Estos servicios operan exclusivamente con Modelos de Dominio y Objetos de Valor (*Value Objects*). No dependen de bases de datos, entidades de persistencia, repositorios concretos, frameworks, HTTP, REST ni ninguna tecnología de infraestructura. Cuando un servicio requiere información que no está en el Modelo de Dominio, debe obtenerla mediante un Puerto de Salida (*Output Port*).

---

## Principios Arquitectónicos

### Modelos de Dominio como Parámetros
Todos los Servicios de Clientes y sus Puertos de Entrada deben recibir exclusivamente Modelos de Dominio u Objetos de Valor. 

Nunca deben recibir:
* Cadenas de texto (*strings*) o identificadores primitivos como sustitutos de relaciones de dominio.
* Atributos individuales que pertenezcan a un Modelo de Dominio.
* DTOs de petición (*Request DTOs*).
* Entidades de persistencia (modelos de ORM).

* **Incorrecto:**
  ```python
  def registrar_cliente_natural(identificacion: str, nombre: str, email: str) -> None: ...
Correcto:Pythondef registrar_cliente_natural(cliente: ClienteNatural) -> ClienteNatural: ...
De igual forma, las relaciones de dominio se expresan mediante objetos:PlaintextClienteEmpresa
    │
    └── representante_legal: ClienteNatural
Información ExternaSi la validación requiere datos no presentes en el objeto de dominio (como verificar si la identificación ya existe en la base de datos), el servicio interactúa con un Puerto de Salida.PlaintextModeloDominio ──► ServicioCliente ──► PuertoSalida ──► AdaptadorSalida ──► Base de Datos
Servicios de Clientes1. Registrar Cliente NaturalDescripción: Crea un nuevo ClienteNatural validando las reglas de negocio del dominio.Entrada: cliente: ClienteNaturalValidaciones de Dominio:Mayoría de Edad: Evalúa cliente.fecha_nacimiento frente a la fecha actual para garantizar que tenga $\ge 18$ años. Esta validación ocurre en memoria dentro del dominio.Unicidad de Identificación: Verifica que la identificación no haya sido registrada previamente consultando el PuertoRepositorioCliente.Estado Inicial: Valida que el EstadoCliente inicial sea válido e independiente de EstadoUsuario.Persistencia: Persiste el objeto a través del PuertoRepositorioCliente.2. Registrar Cliente EmpresaDescripción: Crea un nuevo ClienteEmpresa representando una entidad jurídica y su relación con el representante legal.Entrada: empresa: ClienteEmpresaValidaciones de Dominio:Unicidad de Identificación Fiscal (NIT): Verifica la unicidad del NIT a través del PuertoRepositorioCliente.Representante Legal: Valida que empresa.representante_legal sea una instancia válida de ClienteNatural y cumpla con las condiciones operativas requeridas por el dominio.3. Consultar ClienteDescripción: Recupera la información de un cliente existente y retorna su Modelo de Dominio.Entrada: cliente: Cliente (o un identificador de dominio tipado)Reglas de Autorización:CAJERO_EMPLEADO, ASESOR_COMERCIAL y ANALISTA_INTERNO: Pueden consultar cualquier cliente.CLIENTE_NATURAL: Solo puede consultar su propio registro de cliente.OPERADOR_EMPRESA / SUPERVISOR_EMPRESA: Solo pueden consultar el cliente asociado a su Usuario.cliente.4. Actualizar ClienteDescripción: Aplica cambios sobre los datos bancarios de un cliente existente.Entrada: cliente: ClienteValidaciones de Dominio: Verifica la existencia previa del cliente y valida las restricciones de los nuevos datos (email válido, teléfono, dirección, etc.). Si la identificación cambia, valida su unicidad mediante el Puerto de Salida.5. Cambiar Estado del ClienteDescripción: Modifica el EstadoCliente (ACTIVO, INACTIVO, BLOQUEADO).Entrada: cliente: ClienteRegla de Independencia: El estado del cliente (EstadoCliente) es independiente del estado de acceso al sistema (EstadoUsuario). Bloquear a un cliente no bloquea automáticamente su usuario de sistema a menos que la regla explícita de negocio así lo ordene.6. Consultar Productos del ClienteDescripción: Recupera todos los productos bancarios asignados a un cliente.Entrada: cliente: ClienteInteracción: El servicio utiliza los puertos de salida PuertoRepositorioCuentaBancaria, PuertoRepositorioPrestamo y PuertoRepositorioTransferencia para consolidar los modelos de dominio.Puertos de Entrada e Interfaces en PythonDefinición de casos de uso mediante clases base abstractas (abc):Pythonfrom abc import ABC, abstractmethod
from typing import List, Optional

class CasoUsoRegistrarClienteNatural(ABC):
    
    @abstractmethod
    def ejecutar(self, cliente: ClienteNatural) -> ClienteNatural:
        """Registra un nuevo cliente persona natural."""
        pass


class CasoUsoRegistrarClienteEmpresa(ABC):
    
    @abstractmethod
    def ejecutar(self, empresa: ClienteEmpresa) -> ClienteEmpresa:
        """Registra un nuevo cliente empresa y asigna su representante legal."""
        pass


class CasoUsoConsultarCliente(ABC):
    
    @abstractmethod
    def ejecutar(self, cliente: Cliente) -> Cliente:
        """Consulta la información de un cliente validando permisos."""
        pass


class CasoUsoCambiarEstadoCliente(ABC):
    
    @abstractmethod
    def ejecutar(self, cliente: Cliente) -> None:
        """Cambia el estado operacional de un cliente."""
        pass


class CasoUsoConsultarProductosCliente(ABC):
    
    @abstractmethod
    def ejecutar(self, cliente: Cliente) -> List[ProductoBancario]:
        """Devuelve los productos bancarios asociados al cliente."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)Pythonfrom abc import ABC, abstractmethod
from typing import List, Optional

class PuertoRepositorioCliente(ABC):
    
    @abstractmethod
    def guardar(self, cliente: Cliente) -> Cliente:
        pass

    @abstractmethod
    def buscar_por_identificacion(self, identificacion: str) -> Optional[Cliente]:
        pass

    @abstractmethod
    def existe_por_identificacion(self, identificacion: str) -> bool:
        pass


class PuertoRepositorioCuentaBancaria(ABC):
    
    @abstractmethod
    def buscar_por_cliente(self, cliente: Cliente) -> List[CuentaBancaria]:
        pass


class PuertoRepositorioPrestamo(ABC):
    
    @abstractmethod
    def buscar_por_cliente(self, cliente: Cliente) -> List[Prestamo]:
        pass


class PuertoRepositorioTransferencia(ABC):
    
    @abstractmethod
    def buscar_por_cliente(self, cliente: Cliente) -> List[Transferencia]:
        pass
Flujo de Transformación de Petición EntradaLos DTOs de HTTP no deben entrar a la capa de dominio. El adaptador de entrada mapea la petición antes de invocar al puerto de entrada:PlaintextSolicitud HTTP (JSON)
        │
        ▼
DTO de Peticion (Request DTO)
        │
        ▼
Mapeador / Mapper (Pydantic / Serializador)
        │
        ▼
Modelo de Dominio ClienteNatural
        │
        ▼
Puerto de Entrada (CasoUsoRegistrarClienteNatural)
        │
        ▼
Servicio de Dominio de Clientes
Catálogo de Excepciones de DominioLas violaciones a las reglas de negocio de este subdominio deben lanzar excepciones de dominio explícitas:ExcepcionClienteYaExisteExcepcionClienteNoEncontradoExcepcionClienteMenorDeEdadExcepcionEstadoClienteInvalidoExcepcionRepresentanteLegalInvalidoExcepcionOperacionClienteNoAutorizadaRestricciones Arquitectónicas (Contexto Python)Aislamiento de Infraestructura: El subdominio de Clientes en Python no debe importar ni depender de ORMs (SQLAlchemy, Django ORM), librerías de API (FastAPI, Flask), serializadores web (Pydantic, Marshmallow) ni conectores de base de datos (psycopg2, pymongo).Uso de Type Hints: Métodos y atributos deben hacer uso estricto de las anotaciones de tipo nativas de Python (ClienteNatural, ClienteEmpresa, EstadoCliente).Pura Lógica de Dominio: La regla de mayoría de edad ($\ge 18$ años) y la validez de las relaciones entre ClienteEmpresa y ClienteNatural se ejecutan exclusivamente en memoria mediante los métodos del modelo de dominio.Pruebas Unitarias Aisladas: Todo el subdominio debe poder probarse con pytest sin requerir una instancia de MySQL, MongoDB ni un servidor HTTP activo.
