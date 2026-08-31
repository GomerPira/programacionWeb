
# Objetos de Valor e Inmutables del Dominio (Domain Value Objects)

## Introducción

Los Objetos de Valor (*Value Objects*) representan conceptos inmutables dentro del dominio bancario. A diferencia de las entidades, los Objetos de Valor no tienen su propia identidad; están definidos enteramente por sus valores. 

Se utilizan para encapsular conceptos de negocio controlados, mejorar la expresividad del dominio y evitar el uso de primitivos dispersos (*Primitive Obsession*) o cadenas de texto arbitrarias en el código de la aplicación.

---

## Jerarquía de Objetos de Valor

```text
DomainCatalog (Abstracto)
├── SystemRole
├── CustomerStatus
├── UserStatus
├── AccountStatus
├── LoanStatus
├── TransferStatus
├── AccountType
├── LoanType
├── OperationType
└── Currency
Implementación Base en Python
Python
from abc import ABC
from dataclasses import dataclass
from enum import Enum


@dataclass(frozen=True)
class CatalogoDominio(ABC):
    """
    Representa un catálogo de negocio genérico inmutable.
    Proporciona una estructura uniforme con código único, nombre legible y descripción de negocio.
    """
    codigo: str
    nombre: str
    descripcion: str

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, CatalogoDominio):
            return False
        return self.codigo == other.codigo

    def __hash__(self) -> int:
        return hash(self.codigo)
Catálogos del Dominio (Domain Catalog Enums)
1. SystemRole (Rol del Sistema)
Python
class SystemRole(Enum):
    """Define las responsabilidades y permisos asignados a una Persona en el sistema."""
    NATURAL_CUSTOMER = CatalogoDominio(
        "NATURAL_CUSTOMER", "Natural Customer", "Individual banking customer."
    )
    BUSINESS_CUSTOMER = CatalogoDominio(
        "BUSINESS_CUSTOMER", "Business Customer", "Corporate banking customer."
    )
    TELLER_EMPLOYEE = CatalogoDominio(
        "TELLER_EMPLOYEE", "Teller Employee", "Employee responsible for performing branch operations."
    )
    COMMERCIAL_EMPLOYEE = CatalogoDominio(
        "COMMERCIAL_EMPLOYEE", "Commercial Employee", "Employee responsible for customer relationships and loan-related activities."
    )
    BUSINESS_OPERATOR = CatalogoDominio(
        "BUSINESS_OPERATOR", "Business Operator", "User authorized to perform operations on behalf of business customers."
    )
    BUSINESS_SUPERVISOR = CatalogoDominio(
        "BUSINESS_SUPERVISOR", "Business Supervisor", "User authorized to approve business transfers requiring authorization."
    )
    INTERNAL_ANALYST = CatalogoDominio(
        "INTERNAL_ANALYST", "Internal Analyst", "User responsible for reviewing and approving loan applications."
    )
2. CustomerStatus (Estado del Cliente)
Python
class CustomerStatus(Enum):
    """Representa el estado operacional del cliente dentro de la institución bancaria."""
    ACTIVE = CatalogoDominio(
        "ACTIVE", "Active", "Customer maintains an active banking relationship."
    )
    INACTIVE = CatalogoDominio(
        "INACTIVE", "Inactive", "Customer exists but is not currently active for normal banking operations."
    )
    BLOCKED = CatalogoDominio(
        "BLOCKED", "Blocked", "Customer's banking relationship has been suspended."
    )
3. UserStatus (Estado del Usuario)
Python
class UserStatus(Enum):
    """Representa el estado del acceso de un usuario al sistema bancario (independiente de CustomerStatus)."""
    ACTIVE = CatalogoDominio(
        "ACTIVE", "Active", "User can access the system normally."
    )
    INACTIVE = CatalogoDominio(
        "INACTIVE", "Inactive", "User exists but cannot perform system operations."
    )
    BLOCKED = CatalogoDominio(
        "BLOCKED", "Blocked", "User access has been suspended."
    )
4. AccountStatus (Estado de la Cuenta)
Python
class AccountStatus(Enum):
    """Representa el estado del ciclo de vida de una cuenta bancaria."""
    ACTIVE = CatalogoDominio(
        "ACTIVE", "Active", "Account is fully operational and may perform authorized transactions."
    )
    BLOCKED = CatalogoDominio(
        "BLOCKED", "Blocked", "Transactions are temporarily disabled."
    )
    CLOSED = CatalogoDominio(
        "CLOSED", "Closed", "Account has been permanently closed."
    )
5. LoanStatus (Estado del Préstamo)
Python
class LoanStatus(Enum):
    """
    Representa el estado del ciclo de vida de un préstamo.
    Flujo: UNDER_REVIEW -> (APPROVED | REJECTED | CANCELLED) -> DISBURSED -> OVERDUE / CANCELLED.
    """
    UNDER_REVIEW = CatalogoDominio(
        "UNDER_REVIEW", "Under Review", "Loan request is under evaluation."
    )
    APPROVED = CatalogoDominio(
        "APPROVED", "Approved", "Loan has been approved but funds have not yet been disbursed."
    )
    REJECTED = CatalogoDominio(
        "REJECTED", "Rejected", "Loan request has been rejected."
    )
    DISBURSED = CatalogoDominio(
        "DISBURSED", "Disbursed", "Approved funds have been transferred to the destination account."
    )
    OVERDUE = CatalogoDominio(
        "OVERDUE", "Overdue", "Loan has active obligations that have not been met on time."
    )
    CANCELLED = CatalogoDominio(
        "CANCELLED", "Cancelled", "Loan has been cancelled and is no longer active."
    )
6. TransferStatus (Estado de la Transferencia)
Python
class TransferStatus(Enum):
    """Representa el estado de ejecución del servicio de transferencias."""
    PENDING = CatalogoDominio(
        "PENDING", "Pending", "Transfer has been created and is pending processing."
    )
    WAITING_FOR_APPROVAL = CatalogoDominio(
        "WAITING_FOR_APPROVAL", "Waiting for Approval", "Transfer requires managerial or authorized approval before execution."
    )
    APPROVED = CatalogoDominio(
        "APPROVED", "Approved", "Transfer has been approved and is ready for execution."
    )
    EXECUTED = CatalogoDominio(
        "EXECUTED", "Executed", "Funds have been successfully transferred."
    )
    REJECTED = CatalogoDominio(
        "REJECTED", "Rejected", "Transfer request has been denied."
    )
    EXPIRED = CatalogoDominio(
        "EXPIRED", "Expired", "The approval or execution time window has expired."
    )
7. AccountType (Tipo de Cuenta)
Python
class AccountType(Enum):
    """Tipos de productos de cuenta ofrecidos por la institución."""
    SAVINGS = CatalogoDominio(
        "SAVINGS", "Savings Account", "Standard interest-bearing deposit account."
    )
    CHECKING = CatalogoDominio(
        "CHECKING", "Checking Account", "Transaction account intended for frequent operations."
    )
    BUSINESS = CatalogoDominio(
        "BUSINESS", "Business Account", "Account designed for corporate customers."
    )
8. LoanType (Tipo de Préstamo)
Python
class LoanType(Enum):
    """Tipos de productos de crédito provistos por el banco."""
    PERSONAL = CatalogoDominio(
        "PERSONAL", "Personal Loan", "Loan intended for personal use."
    )
    MORTGAGE = CatalogoDominio(
        "MORTGAGE", "Mortgage Loan", "Loan secured by real estate."
    )
    VEHICLE = CatalogoDominio(
        "VEHICLE", "Vehicle Loan", "Loan used to finance vehicle purchases."
    )
    BUSINESS = CatalogoDominio(
        "BUSINESS", "Business Loan", "Loan intended for business financing."
    )
9. OperationType (Tipo de Operación)
Python
class OperationType(Enum):
    """Representa el tipo de acción o evento significativo ejecutado sobre un producto bancario."""
    # Operaciones de Cuenta
    ACCOUNT_OPENING = CatalogoDominio("ACCOUNT_OPENING", "Account Opening", "Creation of a new bank account.")
    DEPOSIT = CatalogoDominio("DEPOSIT", "Deposit", "Deposit of funds into an account.")
    WITHDRAWAL = CatalogoDominio("WITHDRAWAL", "Withdrawal", "Withdrawal of funds from an account.")
    ACCOUNT_BLOCKING = CatalogoDominio("ACCOUNT_BLOCKING", "Account Blocking", "Blocking of a bank account.")
    ACCOUNT_UNBLOCKING = CatalogoDominio("ACCOUNT_UNBLOCKING", "Account Unblocking", "Removal of a block from a bank account.")
    ACCOUNT_CLOSING = CatalogoDominio("ACCOUNT_CLOSING", "Account Closing", "Permanent closure of a bank account.")

    # Operaciones de Transferencia
    TRANSFER_CREATION = CatalogoDominio("TRANSFER_CREATION", "Transfer Creation", "Creation of a transfer request.")
    TRANSFER_APPROVAL = CatalogoDominio("TRANSFER_APPROVAL", "Transfer Approval", "Approval of a transfer requiring authorization.")
    TRANSFER_REJECTION = CatalogoDominio("TRANSFER_REJECTION", "Transfer Rejection", "Rejection of a transfer request.")
    TRANSFER_EXECUTION = CatalogoDominio("TRANSFER_EXECUTION", "Transfer Execution", "Successful execution of a transfer.")
    TRANSFER_EXPIRATION = CatalogoDominio("TRANSFER_EXPIRATION", "Transfer Expiration", "Expiration of the transfer approval or execution window.")

    # Operaciones de Préstamo
    LOAN_APPLICATION = CatalogoDominio("LOAN_APPLICATION", "Loan Application", "Submission of a loan request.")
    LOAN_APPROVAL = CatalogoDominio("LOAN_APPROVAL", "Loan Approval", "Approval of a loan request.")
    LOAN_REJECTION = CatalogoDominio("LOAN_REJECTION", "Loan Rejection", "Rejection of a loan request.")
    LOAN_DISBURSEMENT = CatalogoDominio("LOAN_DISBURSEMENT", "Loan Disbursement", "Transfer of approved loan funds to the destination account.")
    LOAN_PAYMENT = CatalogoDominio("LOAN_PAYMENT", "Loan Payment", "Registration of a payment made against a loan.")
    LOAN_OVERDUE = CatalogoDominio("LOAN_OVERDUE", "Loan Overdue", "Loan marked as overdue due to unmet obligations.")
    LOAN_CANCELLATION = CatalogoDominio("LOAN_CANCELLATION", "Loan Cancellation", "Cancellation of a loan in an eligible state.")
10. Currency (Moneda)
Python
@dataclass(frozen=True)
class Currency(CatalogoDominio):
    """Moneda soportada por la institución bancaria (especialización con ISO code y símbolo)."""
    iso_code: str = ""
    symbol: str = ""


class Currencies(Enum):
    COP = Currency("COP", "Colombian Peso", "Moneda oficial de Colombia", iso_code="COP", symbol="$")
    USD = Currency("USD", "United States Dollar", "Moneda oficial de EE. UU.", iso_code="USD", symbol="$")
    EUR = Currency("EUR", "Euro", "Moneda oficial de la Eurozona", iso_code="EUR", symbol="€")
Enumeraciones Técnicas Primitivas
Conceptos que contienen valores fijos puramente técnicos y no requieren metadatos adicionales del catálogo de negocio:

Python
class ApprovalDecision(Enum):
    APPROVED = "APPROVED"
    REJECTED = "REJECTED"


class NotificationChannel(Enum):
    EMAIL = "EMAIL"
    SMS = "SMS"
    PUSH_NOTIFICATION = "PUSH_NOTIFICATION"


class AuditSeverity(Enum):
    INFORMATION = "INFORMATION"
    WARNING = "WARNING"
    ERROR = "ERROR"
    CRITICAL = "CRITICAL"
