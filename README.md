<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/banner-dark.svg">
  <img src="assets/banner-light.svg" alt="tochtlani: BI, data engineering y software interno" width="100%">
</picture>

Trabajo en el área de **Business Intelligence** de una red de concesionarios automotrices en
México. Mi trabajo empieza donde termina el dashboard: cuando un proceso vive en un Excel
compartido, en correos o en la memoria de alguien, lo convierto en una app con permisos,
auditoría y datos que cuadran con Power BI.

- Construyo **apps de escritorio (.NET/WPF)** y **apps web** con su API, base de datos e identidad centralizada.
- Mantengo los **ETL y robots** que alimentan el almacén de datos todos los días.
- Me importa que la regla de negocio viva **en un solo lugar, probada**, y que el cliente no pueda saltársela.

## Proyectos destacados

<table>
  <tr>
    <td width="50%" valign="top">
      <a href="https://github.com/tochtlani/robin-dms"><img src="https://raw.githubusercontent.com/tochtlani/robin-dms/main/assets/hero.png" alt="Robin"></a>
      <h3><a href="https://github.com/tochtlani/robin-dms">Robin</a></h3>
      <p>DMS de escritorio para expedientes por VIN y prorrateo de vales. Cliente WPF, API con JWT, permisos por campo validados en el servidor y auditoría a nivel de campo.</p>
      <sub><code>C#</code> <code>.NET 10</code> <code>WPF/MVVM</code> <code>ASP.NET Core</code> <code>SQL Server</code></sub>
    </td>
    <td width="50%" valign="top">
      <a href="https://github.com/tochtlani/mithril-skyline"><img src="https://raw.githubusercontent.com/tochtlani/mithril-skyline/main/assets/hero.png" alt="Mithril SkyLine"></a>
      <h3><a href="https://github.com/tochtlani/mithril-skyline">Mithril SkyLine</a></h3>
      <p>Control diario del flujo de dinero por agencia, con bloqueo por día calendario y desbloqueo administrado. Las reglas de dominio son código puro compartido entre cliente y servidor, cubierto por 262 pruebas.</p>
      <sub><code>C#</code> <code>.NET 10</code> <code>WPF</code> <code>xUnit</code> <code>Inno Setup</code></sub>
    </td>
  </tr>
  <tr>
    <td width="50%" valign="top">
      <a href="https://github.com/tochtlani/nomina-quincenal"><img src="https://raw.githubusercontent.com/tochtlani/nomina-quincenal/main/assets/hero.png" alt="Nómina Quincenal"></a>
      <h3><a href="https://github.com/tochtlani/nomina-quincenal">Nómina Quincenal</a></h3>
      <p>Cálculo y autorización de nómina con segregación de funciones: quien envía una quincena no puede autorizarla. Préstamos, faltas, prima vacacional y pronóstico de efectivo; 550 pruebas.</p>
      <sub><code>C#</code> <code>.NET 10</code> <code>WPF</code> <code>SQL Server</code></sub>
    </td>
    <td width="50%" valign="top">
      <a href="https://github.com/tochtlani/objetivos-apv"><img src="https://raw.githubusercontent.com/tochtlani/objetivos-apv/main/assets/hero.png" alt="Objetivos APV"></a>
      <h3><a href="https://github.com/tochtlani/objetivos-apv">Objetivos APV</a> · <a href="https://tochtlani.github.io/objetivos-apv/">demo ▶</a></h3>
      <p>Captura mensual de objetivos por asesor desde el celular, con historia de asignaciones, cierre de mes y una vista compatible con el modelo de Power BI existente.</p>
      <sub><code>JavaScript</code> <code>Node/Express</code> <code>Vercel</code> <code>SQL Server</code> <code>Power BI</code></sub>
    </td>
  </tr>
  <tr>
    <td colspan="2" valign="top">
      <h3><a href="https://github.com/tochtlani/auth-service">auth-service</a> · <a href="https://tochtlani.github.io/auth-service/">demo ▶</a></h3>
      <p>Identidad central para toda la flota de apps: una persona, una contraseña, un rol por app. Lista blanca de IP por persona, panel de administración aislado de internet y bitácora de auditoría.</p>
      <sub><code>Node.js</code> <code>Express</code> <code>scrypt</code> <code>Tailscale Funnel</code> <code>SQL Server</code></sub>
    </td>
  </tr>
</table>

## Cómo encajan

No son proyectos sueltos: comparten identidad, patrones de despliegue y un "chasis" de
escritorio que se reutiliza de una app a la siguiente.

```mermaid
flowchart LR
    subgraph Escritorio["Chasis de escritorio · WPF + API + Shared"]
        R[Robin] --> M[Mithril SkyLine] --> N[Nómina Quincenal]
    end
    subgraph Web["Apps web · Vercel BFF + backend interno"]
        O[Objetivos APV]
        X[Otras apps internas]
    end
    A{{auth-service<br/>identidad central}}
    O --> A
    X --> A
    DW[(Almacén de datos<br/>SQL Server)]
    ETL[ETL y robots diarios] --> DW
    DW -. solo lectura .-> R
    DW -. solo lectura .-> M
    O -->|vista compatible| PBI[Power BI]
    DW --> PBI
```

## Con qué trabajo

<p>
  <img src="https://skillicons.dev/icons?i=cs,dotnet,js,ts,react,nodejs,express,py,vite,tailwind,vercel,powershell,git&perline=13" alt="Stack">
</p>

**Datos:** SQL Server (T-SQL, vistas, procedimientos, migraciones idempotentes), Power BI (DAX, modelos semánticos `.pbip`), MySQL, ETL en Python.
**Infra:** Tailscale (Funnel / serve), Windows Server, Proxmox, servicios y tareas programadas de Windows.

## Cómo trabajo

- **El servidor es la única autoridad.** Permisos, cálculos y bloqueos se validan en la API; el cliente solo los refleja.
- **Las reglas que importan son código puro y tienen pruebas.** Sin base de datos ni UI de por medio.
- **Las credenciales nunca llegan al cliente.** Ni la cadena de conexión ni el password de SQL.
- **Documento el porqué, no solo el qué.** Cada repo tiene su sección de decisiones, incluidas las que se descartaron.
- **Migro sin romper lo que ya existe.** Si Power BI consume una tabla, la nueva fuente respeta sus columnas.

---

<sub>Los proyectos de arriba están en producción dentro de la empresa. Su código de negocio es
privado; cada repo contiene documentación, capturas y una demo con datos ficticios.</sub>
