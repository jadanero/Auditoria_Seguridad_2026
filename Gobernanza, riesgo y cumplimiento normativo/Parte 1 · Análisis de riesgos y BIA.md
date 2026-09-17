### Activos tecnológicos
- 8 servidores virtualizados con Windows Server 2012 R2.
- 2 servidores físicos Dell PowerEdge R240 (1 CPU de 4 núcleos, 16 GB RAM, 2 x HDD SATA 2 TB, 1 fuente), en configuración activo-pasivo. On-prem.
- 1 firewall (sin equipo redundante).
- 1 conmutador no gestionable y un único enlace de operador (sin línea de respaldo)
- Base de datos MySQL 5.7.
- Cabina de almacenamiento al 92 % de su capacidad.
- 5 portátiles Windows 10 para el equipo de desarrollo. 
- Servicio de backup en AWS S3, región us-east-1 (Virginia, EE. UU.), gestionado desde la matriz de EE. UU. 
- Herramienta de analítica y cuadros de mando en un SaaS estadounidense que procesa datos de uso. 
- Antivirus: Windows Defender.

## Activos físicos
- 1 sede física en Tudela, con control de acceso por tarjeta. 
- CPD propio, sin SAI redundante ni climatización de respaldo.

## Personal
- 1 Gerente de sede. 
- 2 técnicos de Soporte IT. 
- 3 Desarrolladores. 
- No hay Delegado de Protección de Datos designado.

## Configuraciones IT
- 2 VLAN (IT y Wifi); el entorno de pagos no está segmentado. 
- 2 entornos de trabajo (desarrollo y producción); no hay preproducción. 
- Puertos abiertos: 80, 8080, 21, 22 y 3389. 
- Actualización automática desactivada en los servidores. 
- SLA crítico identificado 24×7 y disponibilidad comprometida del 99,9 %. 
- 2FA no forzado para las cuentas de administración.

## Configuración de AyudaPay
**Backend:**
- API REST en Java 8 con Spring Boot 2.7 (Spring Framework 5.3.39). 
- Integración y despliegue continuos (CI/CD) con Jenkins 2.426.1 LTS. 
- Gestión de sesión mediante token; 2FA opcional. 
- Base de datos MySQL 5.7; el IBAN y los datos de tarjeta se almacenan con cifrado reversible y el PAN completo aparece en los registros (logs). 
- Canal de comunicación con TLS; están habilitados TLS 1.0 y 1.1.
**Aplicación móvil:** desarrollada siguiendo las guías del sector para el tratamiento de datos financieros y el estándar **PCI-DSS** para el pago con tarjeta.
