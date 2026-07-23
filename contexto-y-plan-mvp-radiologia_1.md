# Sistema de Gestión Administrativa para Radiología — Contexto y Plan de MVP

## 1. Contexto del negocio

La empresa cuenta con **dos sedes ubicadas en ciudades distintas**, ambas dedicadas a la prestación de servicios de radiología. La operación diaria incluye la recepción de pacientes, la carga de órdenes de estudios solicitadas por médicos derivantes, y la gestión de los cobros correspondientes, ya sea a pacientes particulares o a través de obras sociales y mutuales.

El desafío central que motiva este proyecto no es funcional sino de infraestructura: **el servicio de internet en ambas sedes es inestable**. Esto descarta de plano cualquier solución que dependa de conectividad permanente (una aplicación web tradicional, por ejemplo) y obliga a diseñar el sistema bajo un enfoque **offline-first**: la operación diaria —carga de pacientes, órdenes, cobros— debe poder realizarse sin conexión, y la sincronización con el resto del sistema debe ocurrir de forma automática y transparente cuando la red esté disponible, sin intervención manual del personal administrativo.

A esto se suma un requisito de **acceso multiplataforma**: el sistema debe funcionar tanto en las cinco computadoras de escritorio que utiliza el personal administrativo (conectadas a la red local de cada sede) como en teléfonos celulares y tablets, que probablemente se usen para consultas rápidas o carga de datos fuera del puesto fijo de trabajo.

Por decisión de alcance, en esta primera etapa **el sistema no procesará ni almacenará las imágenes médicas** (estudios radiológicos en sí, que son archivos pesados y requieren infraestructura de almacenamiento específica). El foco está puesto exclusivamente en la **gestión administrativa**: pacientes, órdenes, estudios solicitados, médicos derivantes, obras sociales y cobros.

## 2. Problema a resolver

En síntesis, el sistema debe garantizar que el personal de cada sede pueda seguir trabajando con normalidad —registrar pacientes, cargar órdenes, cobrar estudios— **incluso si la sede pierde la conexión a internet por un tiempo prolongado**, y que al recuperar la conexión, todos los datos cargados localmente se integren correctamente al sistema central sin duplicados, sin pérdida de información, y sin necesidad de que el personal haga nada manual para "subir" los datos.

Además, dado que existen dos sedes operando de forma independiente pero sobre el mismo negocio, el sistema debe eventualmente reflejar en ambas sedes la información cargada en la otra (por ejemplo, si un paciente ya fue cargado en la Sede A, el personal de la Sede B debería poder encontrarlo sin tener que cargarlo de nuevo, una vez que ambas sedes hayan sincronizado).

## 3. Alcance del MVP

El objetivo del MVP es contar con un sistema funcional mínimo que cubra el flujo administrativo central del negocio, priorizando la validación de la arquitectura offline-first por sobre la cobertura completa de funcionalidades. El MVP incluye:

- Alta y gestión de las entidades maestras: sedes, pacientes, médicos derivantes, estudios (catálogo) y obras sociales/mutuales.
- Carga de órdenes de estudio (`MedicalTest`) con sus estudios individuales asociados (`MedicalTestDetail`), incluyendo el flujo de estados (pendiente, en curso, completado, cancelado).
- Registro de cobros asociados a una orden, contemplando tanto pacientes particulares como con cobertura de obra social, incluyendo cobros manuales (coseguros, diferencias).
- Operación 100% funcional sin conexión a internet en cualquiera de las cinco PCs de administración, con sincronización automática al recuperar la red.
- Filtro de visualización por sede (ver todo el sistema o solo lo cargado en la sede propia), configurable por el usuario.
- Cliente funcional en al menos una plataforma de escritorio y una plataforma móvil, para validar que el enfoque MAUI + Blazor Hybrid cumple con el objetivo multiplataforma antes de invertir en pulir la experiencia en todas las plataformas.

Quedan fuera del MVP (ver sección 9) el manejo de imágenes médicas, reportes avanzados, y refinamientos de UX que no sean críticos para validar el flujo administrativo básico.

## 4. Arquitectura de la solución

La solución se estructura siguiendo **Clean Architecture** de forma estricta, con las siguientes capas:

- **Domain**: entidades de negocio, enums, reglas de negocio puras. No depende de ninguna otra capa ni de tecnología externa.
- **Application**: casos de uso, interfaces de repositorios y servicios (`IUnitOfWork`, `IMedicalTestQueryService`, etc.), DTOs de lectura y escritura. Depende únicamente de Domain.
- **Infrastructure.Local**: implementación concreta de persistencia local con SQLite y Entity Framework Core, pensada para funcionar en cada dispositivo (PC, celular, tablet) de forma autónoma.
- **Infrastructure.Cloud**: implementación concreta de persistencia central con PostgreSQL o SQL Server, y la lógica de recepción/fusión de datos sincronizados desde los dispositivos.
- **WebAPI**: expone la API central en la nube, consumida por los dispositivos al sincronizar.
- **App.UI**: cliente construido en .NET 10 con .NET MAUI + Blazor Hybrid, que permite reutilizar la interfaz Razor tanto como aplicación de escritorio nativa como aplicación móvil nativa, sin depender de un navegador.

La regla de dependencia se mantiene en todo momento: las capas externas (Infrastructure, WebAPI, UI) dependen de las internas (Application, Domain), nunca al revés. Esto permite que el dominio del negocio quede completamente desacoplado de las decisiones tecnológicas (SQLite vs. Postgres, MAUI vs. cualquier otro framework de UI).

## 5. Modelo de datos

### Entidades maestras
- **Location**: sede (Id, Name, City).
- **Patient**: paciente (Id, Dni, FirstName, LastName).
- **Doctor**: médico solicitante (Id, FirstName, LastName).
- **Study**: estudio radiológico disponible (Id, Name, BaseCost).
- **Healthcare**: obra social o mutual (Id, Name).

### Entidades transaccionales
- **MedicalTest**: orden u orden de estudio general (Id, PatientId, DoctorId, HealthcareId nullable, LocationId, AuthCode, Observations, OrderDate, Status).
- **MedicalTestDetail**: estudios individuales dentro de una orden, relación 1 a N con `MedicalTest` y 1 a 1 con `Study`. Incluye un campo de costo aplicado (`AppliedCost`) que se copia del `BaseCost` del estudio en el momento de la creación, para que cambios futuros en el precio del catálogo no alteren órdenes ya facturadas.
- **Payment**: entidad separada para registrar cobros (Id, MedicalTestId, Amount, PaymentMethod, PaidAtUtc), permitiendo múltiples pagos parciales o combinados sobre una misma orden.

### Convenciones transversales
Todas las entidades principales utilizan `Guid` como clave primaria (generado de forma secuencial, tipo UUID v7, para minimizar la fragmentación de índices en la base central), junto con campos de control `IsSynced` (bool) y `LastModifiedUtc` (DateTime) para soportar la sincronización offline-first. Los enums (`Status`, `PaymentMethod`) se almacenan como string en la base de datos, no como entero, para evitar ambigüedades si se reordenan o agregan valores en el futuro.

Solo las entidades transaccionales (`MedicalTest`) llevan `LocationId`; las entidades maestras (`Patient`, `Doctor`, `Study`, `Healthcare`) son globales al negocio y no están atadas a una sede particular, dado que un mismo paciente o médico derivante puede operar en ambas sedes.

## 6. Reglas de negocio

**Lógica de pagos**: si el paciente es particular (sin obra social asociada), se cobra el `BaseCost` de cada estudio incluido en la orden. Si el paciente tiene una obra social o mutual asociada, el costo de los estudios arranca en $0, pero el sistema permite cargar cobros manuales sobre la orden (coseguros, diferencias de valor, etc.) a través de la entidad `Payment`, que admite múltiples registros de pago con distinto método (efectivo, transferencia, débito, crédito).

**Separación de DTOs**: se mantiene una separación estricta entre los objetos de lectura (por ejemplo, `StudyDTO`) y los objetos de escritura/creación (por ejemplo, `MedicalTestCreateDTO`, que agrupa las referencias por Id y los datos de pago necesarios para crear una orden completa en una sola operación).

## 7. Estrategia de sincronización offline-first

Cada dispositivo (PC de administración, celular o tablet) cuenta con su propia base de datos SQLite local, sobre la cual opera de forma completamente autónoma cuando no hay conexión. Un **Worker Service local** monitorea el estado de la red de forma continua; mientras no haya conexión, todas las operaciones (altas, modificaciones) se persisten únicamente en la base local.

Al detectar que la conexión se restableció, el Worker Service sincroniza los cambios pendientes contra la API central, aplicando el patrón **Unit of Work** para garantizar que cada transacción se aplique de forma atómica. Para hacer este proceso más robusto ante reintentos y fallas parciales, los cambios pendientes se registran mediante un **patrón Outbox**: en lugar de simplemente marcar registros como no sincronizados y volver a escanearlos, cada operación se encola como un evento ordenado en una tabla de sincronización dedicada, lo que permite reproducir el envío de forma confiable y en el orden correcto.

La sincronización es **bidireccional**: no solo cada dispositivo envía sus cambios locales hacia la nube, sino que también recibe los cambios generados en la otra sede (y en otros dispositivos) desde la última sincronización registrada, de modo que ambas sedes eventualmente comparten una vista consistente de toda la información del negocio.

Dado que la base de datos se replica completa en todos los dispositivos (no se particiona por sede), la separación entre sedes se resuelve exclusivamente a nivel de consulta: la capa de Aplicación expone un filtro explícito por `LocationId` que la interfaz de usuario puede activar o desactivar, permitiendo al usuario elegir entre ver únicamente lo cargado en su sede o el total del negocio. En las PCs de administración, la sede activa queda fija según la configuración del dispositivo; en celulares y tablets, en cambio, la sede activa la selecciona el usuario al iniciar sesión, dado que estos dispositivos son inherentemente más móviles y pueden usarse indistintamente en cualquiera de las dos sedes.

Para resolver conflictos entre ediciones concurrentes realizadas en distinta sede sobre el mismo registro mientras ambas estaban offline, se adopta una estrategia de **Last-Write-Wins** basada en `LastModifiedUtc`, decisión razonable dado el volumen y la escala del negocio (dos sedes, no un sistema distribuido de gran escala). Para el caso particular de pacientes creados de forma simultánea en ambas sedes mientras estaban desconectadas, la sincronización central aplica una lógica de deduplicación por clave de negocio (DNI), evitando que un mismo paciente termine duplicado en el sistema.

## 8. Decisiones de diseño confirmadas

- Clean Architecture estricta con las seis capas descriptas en la sección 4.
- Cliente único multiplataforma con .NET 10, MAUI + Blazor Hybrid. .NET 8 llega a end-of-support el 10/11/2026, por lo que se descarta como base para un proyecto nuevo; .NET 10 es la LTS vigente (soporte hasta 11/2028). Si al llegar a la etapa de App.UI persisten problemas de estabilidad reportados en .NET MAUI 10, se evaluarán alternativas (Uno Platform, Avalonia) o fijar una versión de SDK MAUI más estable.
- Persistencia local con SQLite, persistencia central con PostgreSQL o SQL Server, ambas vía Entity Framework Core.
- Claves primarias `Guid` de tipo secuencial (UUID v7) para evitar fragmentación de índices.
- Enums almacenados como string en base de datos.
- Entidad `Payment` separada de `MedicalTest` para soportar cobros múltiples o parciales.
- Snapshot de costo (`AppliedCost`) en `MedicalTestDetail` para preservar el valor histórico de una orden ante cambios futuros de precio en el catálogo de estudios.
- Patrón Outbox para la cola de sincronización, en lugar de un simple flag de estado.
- Replicación completa de la base de datos en todos los dispositivos, con filtro de visualización por sede a nivel de consulta (no de datos).
- Sede activa fija por dispositivo en las PCs de administración; elegida por el usuario al loguearse en celulares y tablets.

## 9. Fuera de alcance del MVP

Para mantener el MVP acotado y enfocado en validar la arquitectura offline-first, quedan explícitamente fuera de esta primera etapa:

- Procesamiento, almacenamiento o visualización de imágenes médicas (estudios radiológicos propiamente dichos).
- Reportes gerenciales o estadísticas avanzadas de facturación.
- Gestión de usuarios y permisos granular más allá de lo mínimo necesario para operar (esto podría incorporarse en una iteración posterior si el negocio lo requiere).
- Notificaciones automáticas a pacientes (turnos, resultados disponibles, etc.).
- Refinamiento visual y de experiencia de usuario más allá de lo funcional, en todas las plataformas salvo la primera validada.

## 10. Plan de implementación

Siguiendo la dirección de dependencias de Clean Architecture, la construcción del MVP avanza de adentro hacia afuera:

1. **Domain**: entidades, enums, generador de GUID secuencial.
2. **Application**: interfaces de repositorios y Unit of Work, DTOs de lectura y escritura, definición del `LocationFilter` y de la entidad/mecanismo de Outbox.
3. **Infrastructure.Local**: `DbContext` de SQLite, configuraciones de mapeo (incluyendo enums como string), migraciones iniciales.
4. **Infrastructure.Cloud**: `DbContext` central, lógica de recepción y fusión de datos sincronizados (incluyendo deduplicación por DNI).
5. **WebAPI**: endpoints de sincronización (push y pull) y de consulta.
6. **Worker Service**: monitoreo de red y ejecución del proceso de sincronización basado en Outbox.
7. **App.UI**: cliente MAUI + Blazor Hybrid, comenzando por el flujo administrativo mínimo (pacientes, órdenes, cobros) en una plataforma de escritorio y validando luego en una plataforma móvil.
