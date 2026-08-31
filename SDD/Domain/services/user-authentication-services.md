# Servicios de Autenticación de Usuarios (User Authentication Services)

## Introducción

Este documento define los servicios pertenecientes al subdominio de Autenticación de Usuarios del Sistema de Gestión de Información Bancaria.
Los servicios de este subdominio son responsables de:
* Registrar usuarios asociados a clientes (`Cliente`).
* Registrar usuarios empleados internos.
* Autenticar usuarios en el sistema.
* Validar credenciales de acceso.
* Gestionar contraseñas de usuarios.
* Administrar estados de usuario.
* Consultar información de usuarios.

Los servicios operan exclusivamente con Modelos de Dominio y Objetos de Valor (*Value Objects*).
Nunca dependen directamente de:
* Bases de datos o adaptadores relacionales/NoSQL.
* Entidades de persistencia (SQLAlchemy, Django Models).
* Frameworks web (FastAPI, Flask, Django).
* Especificaciones HTTP/REST/JSON.
* Librerías de hash de contraseñas (passlib, bcrypt, argon2).
* Librerías de generación/validación de JWT (PyJWT, python-jose).

Cualquier funcionalidad o información externa al dominio se gestiona a través de un Puerto de Salida (*Output Port*).

---

## Contexto del Modelo de Dominio

`Usuario` es un Modelo de Dominio que hereda de `Persona`:

```text
Persona
├── Cliente
│   ├── ClienteNatural
│   └── ClienteJuridico
│
└── Usuario
Por lo tanto, la información de identidad personal pertenece a Persona. El modelo Usuario representa la identidad en el sistema y su información de autenticación.

Un usuario puede estar asociado a un cliente:

Plaintext
Usuario
 │
 └── cliente: Optional[Cliente]
Regla: El Modelo de Dominio nunca sustituye esta relación por identificadores primitivos como id_cliente: str.

Principios de Diseño del Servicio
Modelos de Dominio como Parámetros
Todos los servicios de Autenticación y sus Puertos de Entrada deben recibir exclusivamente Modelos de Dominio u Objetos de Valor (Value Objects).

Incorrecto:

Python
def registrar_usuario(nombre_usuario: str, contrasena: str, id_cliente: str) -> None: ...
def login(nombre_usuario: str, contrasena: str) -> None: ...
def cambiar_estado_usuario(id_usuario: str, nuevo_estado: EstadoUsuario) -> None: ...
Correcto:

Python
def registrar_usuario(usuario: Usuario) -> Usuario: ...
def login(usuario: Usuario) -> ResultadoAutenticacion: ...
def cambiar_estado_usuario(usuario: Usuario) -> Usuario: ...
Ciclo de Vida y Roles de Usuarios
Independencia de Estados
El sistema mantiene dos conceptos de estado independientes que no deben alterarse automáticamente entre sí:

EstadoCliente: Representa el estado operacional del cliente bancario (ACTIVO, BLOQUEADO, etc.).

EstadoUsuario: Representa la capacidad de acceso al sistema del usuario (ACTIVO, INACTIVO, BLOQUEADO).

Roles de Sistema (RolSistema)
Clientes: CLIENTE_NATURAL, CLIENTE_JURIDICO.

Empleados InternOS: CAJERO_EMPLEADO, EJECUTIVO_COMERCIAL, OPERADOR_EMPRESARIAL, SUPERVISOR_EMPRESARIAL, ANALISTA_INTERNO.

Servicios de Autenticación
1. Registrar Usuario Cliente
Descripción: Crea un Usuario asociado a un Cliente existente (ClienteNatural o ClienteJuridico).

Entrada: usuario: Usuario (debe contener usuario.cliente: Cliente).

Validaciones de Dominio:

Nombre de usuario válido y único (consultado mediante PuertoRepositorioUsuario).

Contraseña conforme a reglas de complejidad de dominio.

Rol compatible con el tipo de cliente (CLIENTE_NATURAL o CLIENTE_JURIDICO).

Seguridad y Persistencia: La contraseña se procesa a través de PuertoSeguridadContrasena antes de guardarse. Nunca se almacena texto plano.

2. Registrar Usuario Empleado
Descripción: Crea un Usuario que representa a un empleado interno. No requiere asociación obligatoria a un Cliente.

Entrada: usuario_registrador: Usuario, nuevo_usuario: Usuario

Autorización Estricta: Requiere que usuario_registrador.rol == RolSistema.ANALISTA_INTERNO y esté en estado ACTIVO.

Validación de Rol: El nuevo usuario debe tener un rol correspondiente a empleados internos.

3. Autenticar Usuario (Login)
Descripción: Autentica a un usuario del sistema mediante sus credenciales.

Entrada: usuario: Usuario (contiene únicamente nombre_usuario y contrasena de entrada).

Flujo:

Busca el usuario en persistencia mediante PuertoRepositorioUsuario.buscar_por_nombre_usuario(). Si no existe, lanza ExcepcionCredencialesInvalidas.

Valida la contraseña mediante PuertoSeguridadContrasena.verificar(contrasena_ingresada, contrasena_almacenada).

Valida que usuario_almacenado.estado_usuario == EstadoUsuario.ACTIVO.

Genera el token de acceso mediante PuertoTokenJWT.generar(usuario_almacenado).

Resultado: Retorna un objeto de valor ResultadoAutenticacion(token: str).

Plaintext
  Usuario (Entrada)
        │
        ▼
PuertoRepositorioUsuario ──► Usuario Almacenado
                                   │
                                   ▼
PuertoSeguridadContrasena ──► ¿Contraseña Válida? ──► SÍ ──► PuertoTokenJWT ──► JWT
Regla de Seguridad para JWT: Los claims requeridos son únicamente nombre_usuario y rol. La contraseña jamás debe incluirse en los claims del token.

4. Cambiar Contraseña de Usuario
Descripción: Actualiza la credencial de acceso de un usuario.

Entrada: usuario: Usuario, nueva_contrasena: str (o VO Contrasena).

Procesamiento: Procesa la nueva contraseña con PuertoSeguridadContrasena y persiste los cambios.

5. Cambiar Estado de Usuario
Descripción: Modifica el EstadoUsuario (ACTIVO, INACTIVO, BLOQUEADO).

Entrada: usuario_operador: Usuario, usuario_objetivo: Usuario, nuevo_estado: EstadoUsuario

Validación: El dominio valida si la transición de estado solicitada es permitida.

6. Consultar Usuario
Descripción: Obtiene un Usuario representado como Modelo de Dominio.

Entrada: usuario: Usuario

Puertos de Entrada e Interfaces en Python
Definición de casos de uso mediante clases base abstractas (abc):

Python
from abc import ABC, abstractmethod
from typing import Optional
from dataclasses import dataclass

@dataclass(frozen=True)
class ResultadoAutenticacion:
    token: str


class CasoUsoRegistrarUsuarioCliente(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario) -> Usuario:
        """Registra un nuevo usuario vinculado a un Cliente."""
        pass


class CasoUsoRegistrarUsuarioEmpleado(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario_registrador: Usuario, nuevo_usuario: Usuario) -> Usuario:
        """
        Registra un usuario empleado interno.
        Requiere usuario_registrador.rol == RolSistema.ANALISTA_INTERNO.
        """
        pass


class CasoUsoAutenticarUsuario(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario) -> ResultadoAutenticacion:
        """Autentica las credenciales y genera un token JWT."""
        pass


class CasoUsoCambiarContrasenaUsuario(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario: Usuario, nueva_contrasena: str) -> bool:
        """Actualiza la contraseña procesada del usuario."""
        pass


class CasoUsoCambiarEstadoUsuario(ABC):
    
    @abstractmethod
    def ejecutar(self, usuario_operador: Usuario, usuario_objetivo: Usuario, nuevo_estado: EstadoUsuario) -> Usuario:
        """Actualiza el EstadoUsuario de un usuario."""
        pass
Puertos de Salida (Clases Base Abstractas en Python)
Python
from abc import ABC, abstractmethod
from typing import Optional

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
    def buscar_por_id(self, id_cliente: str) -> Optional[Cliente]:
        pass


class PuertoSeguridadContrasena(ABC):
    
    @abstractmethod
    def hashear(self, contrasena_texto_plano: str) -> str:
        """Convierte una contraseña en texto plano en su hash seguro."""
        pass

    @abstractmethod
    def verificar(self, contrasena_texto_plano: str, contrasena_hash: str) -> bool:
        """Verifica si la contraseña ingresada coincide con el hash almacenado."""
        pass


class PuertoTokenJWT(ABC):
    
    @abstractmethod
    def generar(self, usuario: Usuario) -> str:
        """Genera un token JWT firmado conteniendo los claims 'nombre_usuario' y 'rol'."""
        pass
Catálogo de Excepciones de Dominio
ExcepcionUsuarioYaExiste

ExcepcionUsuarioNoEncontrado

ExcepcionUsuarioInvalido

ExcepcionRolUsuarioInvalido

ExcepcionEstadoUsuarioInvalido

ExcepcionAsociacionClienteInvalida

ExcepcionCredencialesInvalidas

ExcepcionUsuarioNoAutorizado

ExcepcionValidacionContrasena

Restricciones Arquitectónicas (Contexto Python)
Aislamiento de Criptografía e Infraestructura: El dominio y las clases de servicio no deben importar passlib, bcrypt, hashlib, jwt ni PyJWT. Toda operabilidad criptográfica se canaliza a través de PuertoSeguridadContrasena y PuertoTokenJWT.

Prohibición de Texto Plano: Las contraseñas en texto plano solo existen temporalmente durante la invocación de los métodos y nunca deben ser almacenadas en los modelos de persistencia ni exponerse en logs, excepciones o respuestas.

Mapeo de Entradas HTTP: Los Request DTOs (p. ej., esquemas de FastAPI/Pydantic) deben mapearse a objetos Usuario de dominio antes de llamar a los casos de uso.

Desacoplamiento Total para Pruebas: Todos los flujos de autenticación deben poder ser testeados de extremo a extremo utilizando pytest y stubs/mocks en memoria de los puertos de salida, sin levantar servidores HTTP ni bases de datos.
