# Servicios del Dominio de Usuarios (User Services)

## Introducción

Este documento especifica la lógica funcional y reglas de negocio para la gestión de usuarios, identidades y roles dentro del Marketplace NexusMarket. Este dominio actúa como la base de autenticación y autorización, administrando el estado operativo de las cuentas y la asignación de permisos para todos los participantes del sistema.

---

## 1. Especificación Funcional de Usuarios

### Casos de Uso (Input Ports)

* **`AutenticarUsuarioInputPort`**
  * **Descripción:** Valida las credenciales de acceso de un usuario en la plataforma.
  * **Parámetros:** `correo_electronico: str`, `contrasena_hash: str`
  * **Respuesta:** `Usuario`

* **`CambiarEstadoUsuarioInputPort`**
  * **Descripción:** Permite al Administrador modificar la condición operativa de un usuario (Activo, Bloqueado, Pendiente).
  * **Parámetros:** `id_usuario: str`, `nuevo_estado: EstadoUsuario`
  * **Respuesta:** `Usuario`

* **`AsignarRolInputPort`**
  * **Descripción:** Define o actualiza las responsabilidades y permisos asignados a un usuario dentro de la plataforma.
  * **Parámetros:** `id_usuario: str`, `nuevo_rol: RolUsuario`
  * **Respuesta:** `Usuario`

* **`ConsultarUsuarioInputPort`**
  * **Descripción:** Obtiene la información de perfil e identidad de un usuario registrado.
  * **Parámetros:** `id_usuario: str`
  * **Respuesta:** `Usuario`

---

## 2. Reglas de Negocio Aplicables

1. **Unicidad de Identidad:** El correo electrónico y el documento de identificación de cada usuario deben ser únicos e irrepetibles en toda la plataforma.
2. **Rol Único por Usuario:** Cada participante desempeñará un único rol dentro del sistema (`Comprador`, `Vendedor`, `Operador Logístico`, `Administrador` o `Supervisor`).
3. **Bloqueo Operativo:** Un usuario en estado `BLOQUEADO` o `INACTIVO` tendrá denegado el acceso inmediato a cualquier operación del sistema.

---

## 3. Puertos de Salida Requeridos (Output Ports)

```python
from abc import ABC, abstractmethod
from typing import Optional
from domain.models import Usuario

class RepositorioUsuarioOutputPort(ABC):
    @abstractmethod
    def guardar(self, usuario: Usuario) -> Usuario: pass

    @abstractmethod
    def buscar_por_id(self, id_usuario: str) -> Optional[Usuario]: pass

    @abstractmethod
    def buscar_por_correo(self, correo: str) -> Optional[Usuario]: pass

class ServicioHashContrasenaOutputPort(ABC):
    @abstractmethod
    def verificar_hash(self, contrasena_plana: str, contrasena_hash: str) -> bool: pass