# objetivos-apv

Captura mensual de objetivos por APV (asesor de ventas), agrupados por Gerente y Sucursal,
para alimentar el modelo de Power BI **`PROYECTO WINSTON`**. Reemplaza el Excel
`MAESTRO - OBJETIVO POR APV.xlsx` que hoy vive en SharePoint.

Sigue el patrón estándar del equipo (skill `despliegue-interno-vercel`), el mismo de
`tablaMarcos`, `calculadora-gtc` y `tact-rcn`:

```
Navegador ──HTTPS──> Vercel ──HTTPS+token──> Tailscale Funnel ──> Express ──> SQL Server
           (cookie)   /api/*  (BACKEND_TOKEN)   *.ts.net          :3006      144.217.90.168
```

Los gerentes capturan desde el celular, sin VPN. El token del Funnel es lo único que separa
la base de datos de internet.

**Todo vive en el VPS `144.217.90.168`**, junto a `raven-c4` y `bitacora-servicio`: ahí está
tanto la base de negocio `objetivos_apv` como el espejo de `INTELIGENCIA_NEGOCIOS`. Se eligió
sobre la `10.10.10.12` por uptime — la otra es una máquina local.

> Que las dos bases estén en la **misma instancia** es lo que permite que `vw_Sucursales` y
> `vw_NombresFuente` lean el catálogo con nombres de tres partes
> (`INTELIGENCIA_NEGOCIOS.dbo.SUCURSALES`). Si algún día se separaran, esos nombres dejan de
> resolver y habría que pasar a un segundo pool en el backend, como hace `raven-c4`.

## Dónde vive cada cosa

| Pieza | Dónde | Estado al 2026-09-09 |
|---|---|---|
| Frontend + BFF | Vercel, `objetivos-apv.vercel.app` | En línea |
| Backend | VPS `144.217.90.168:3006`, tras `https://mithril-api.tail3225eb.ts.net/objetivos_apv` | En línea |
| Base de negocio | `objetivos_apv` en el VPS | `001`–`006` aplicados |
| Identidad | central `APPS` + `auth-service`, en `10.10.10.12` | Compartida con las otras tres apps |

> El **proyecto** de Vercel se llama `objetvios-apv`, con el dedazo, y así quedó en
> `.vercel/project.json`; el **dominio** que sirve es `objetivos-apv.vercel.app`, bien escrito.
> Pegarle al del dedazo devuelve `DEPLOYMENT_NOT_FOUND`, que parece que se cayó el despliegue.

> ⚠️ El nodo de tailnet del VPS es **`mithril-api`**, el mismo de Mithril SkyLine. El andamiaje
> de la skill imprime `desktop-5si83fk` por omisión, que es **otra máquina**: dejar ese valor en
> `BACKEND_URL` da un 404 de Express que parece un fallo de la app.
>
> Y pegarle a la URL pelada (`.../objetivos_apv`) **también da 404, y es normal**: el Funnel
> recorta el prefijo y el backend no tiene ruta en `/`. Lo que se prueba es `/health`.

---

## Lo que este proyecto resuelve y el anterior no

Tres cosas del negocio no estaban modeladas, y las tres rompen por el mismo lado:

**1. Un APV vive en varias agencias a la vez.** Está resuelto con `dbo.asignaciones`:
un renglón por cada par (APV, sucursal), varios abiertos al mismo tiempo.

**2. Varios gerentes conviven en una agencia.** Por eso `dbo.gerentes` **no tiene columna
de sucursal**: la agencia es del par (APV, gerente), no del gerente.

**3. Los gerentes rotan mucho a los APV, y esa historia importa.** Ésta es la que decide
el diseño, y es donde el proyecto anterior se equivocaba:

> **Una asignación nunca se reactiva.** Se cierra poniéndole `vigente_hasta`, y volver a
> asignar **inserta un renglón nuevo**.

El diseño anterior reactivaba el renglón previo (su índice único estaba filtrado por
`[Activo] = 1`, así que un insert habría dejado dos). Con eso, un APV que estuvo en NAVA con
el gerente A, salió, y volvió con el gerente B terminaba con **un solo renglón** que decía
"NAVA, gerente B, desde siempre". El paso por A desaparecía, y con él la respuesta a "¿de
quién era este APV en marzo?".

Aquí quedan dos renglones fechados, y `vw_AsignacionesAPV` los expone. La prueba 5 de
`db/pruebas/pruebas_esquema.sql` es exactamente ese caso.

**Lo que sí puede impedirse:** un mismo APV con **dos gerentes vivos en la misma agencia**
(`UX_asig_abierta`). No es un capricho: el objetivo tiene grano (APV, sucursal, mes), así que
dos gerentes abiertos en la misma agencia dejarían sin respuesta "¿de quién es este objetivo?".

---

## Quién puede qué

Tres roles. El de cada persona vive en la central `APPS` (`app_members.role`); lo que **alcanza**
vive aquí.

| | Admin | RH | Gerente |
|---|---|---|---|
| Ver y capturar objetivos | todos | **de sus agencias** | los suyos |
| Alta de APV, vincularlos y darlos de baja | todos | **de sus agencias** | **no** |
| Ver su equipo y su historial de asignaciones | todos | de sus agencias | los suyos |
| Registrar gerentes y generar su acceso | sí | **de sus agencias** (nunca como admin) | no |
| Dar de alta fuera del padrón de nómina | sí | no | no |
| Ver la lista de gerentes | todos | **de sus agencias** | solo a sí mismo |
| Restablecer contraseñas | cualquiera | **de sus agencias** | no |
| Reabrir un mes cerrado | sí | no | no |
| Asignar agencias al personal de RH | sí | no | no |

**RH es un administrador acotado a sus agencias.** Ese modelo mental es lo que evita un tercer
juego de permisos: es el mismo con un filtro. Lo único que no hereda es reabrir un mes — esa es
la autorización explícita para escribir fuera de la ventana.

### Armar el equipo es de RH, no del gerente

Hasta el 2026-09-11 el gerente daba de alta a sus APV, se los vinculaba y los daba de baja. Ya
no: lee a los suyos, consulta su historial y les captura objetivos, y nada más.

El motivo es de negocio, no de seguridad. La plantilla se mueve en nómina, y lo de aquí no es
más que su reflejo; cuando la baja la daba el gerente, el reflejo **se adelantaba al hecho** —
capturada el día que el APV avisa, semanas antes de que nómina la registre — y la trazabilidad
terminaba contando una historia distinta a la de RH. Ahora el movimiento pasa por quien lo
registra en los dos lados.

Se aplica en `mueveEquipo()` (`backend/server.js`), en los tres endpoints que mueven el equipo:
`POST /apv`, `POST /asignaciones` y `POST /asignaciones/:id/cerrar`. La pantalla esconde la
tarjeta de alta y el botón de baja, pero eso es cortesía: los tres endpoints siguen siendo
alcanzables con una sesión de gerente, y responden **403**.

La rama de gerente de `filtroAlcance` sigue ahí y sigue haciendo falta: es la misma función que
acota las **lecturas** — el listado de su equipo, el historial, la reja del mes — donde el
gerente sí entra.

### El alcance vive en un solo lugar

`backend/alcance.js`, con pruebas propias. Antes se decidía con un `esAdmin` binario repetido en
doce sitios, y equivocarse ahí **no da error**: da permiso de más o de menos, en silencio.

Tres distinciones que las pruebas cubren y que es fácil romper al tocar esto:

- **`null` no es lo mismo que `''`.** Cadena vacía significa "sin filtro" (el admin ve todo);
  `null` significa "no alcanza nada" (un RH sin agencias, un gerente sin alta). Confundirlos
  abriría la app entera justo a quien no alcanza nada.
- **Un rol desconocido no hereda nada.** El rol de la central es texto libre y las otras apps
  usan valores propios (`ventas`, `user`), así que lo que no se reconoce cae a `gerente`, el más
  acotado. `superadmin` o `ADMIN` en mayúsculas no ascienden.
- **La comparación de agencia es exacta**, no por prefijo: `NAVA` no alcanza `NAVA CENTRO`.

Los nombres de agencia **nunca se concatenan** en el SQL: van parametrizados uno por uno.

> **Ocultar un botón no protege nada.** La pantalla esconde lo que no aplica por cortesía; el
> permiso lo aplica el backend renglón por renglón, en cada petición.

### El alta está cerrada al padrón de nómina

Ni el APV ni el gerente se teclean: se **eligen** de un combo que sale de `CONTPAQ_2`. Quien no
esté cargado en nómina no se puede dar de alta aquí.

Es una **palanca a propósito**, no una validación de datos: obliga a que el personal nuevo quede
cargado en nómina el mismo día, en vez de vivir meses capturado a mano solo en esta app. Va en
contra de lo que db/003 y db/004 decidieron en su momento (*"es una AYUDA, no una restricción"*),
y se cambió a petición del negocio.

Lo que eso implica, dicho completo:

- **Si el padrón no se puede leer, el alta se detiene** — no vuelve a abrirse a mano. Saltarse la
  comprobación justo cuando no está disponible es lo que la volvería inútil. La pantalla apaga el
  botón y dice el error de SQL.
- **El admin queda fuera de la restricción.** Si el padrón está mal y hay que capturar a alguien
  hoy, tiene que existir una salida; que sea el admin es lo que la mantiene siendo una excepción
  y no la puerta de siempre.
- La marca **EN_CRM** sigue viva y sigue siendo un aviso, no un candado: un candidato que nómina
  reporta pero que el CRM escribe distinto se puede dar de alta, avisando que va a salir en ceros.

#### Los puestos autorizados, y por qué van acotados por departamento

`dbo.nomina_puestos_apv` dice qué puestos de nómina cuentan como APV. Cada renglón es
`(puesto, depa)`: `depa` en NULL significa *cualquier departamento*, y con texto significa **solo
ese**. El acote existe porque un puesto puede nombrar a la **outsourcera y no a la función** —
`EXTERNO OLEKSEI` sin acotar arrastraba también a su gente de servicio, motorrad y flotillas.

El **2026-09-10** se autorizaron seis puestos más, todos acotados. El padrón pasó de **186 a 232
personas**:

| Puesto | Depa | Entran |
|---|---|---|
| `EMBAJADOR DE SEMINUEVOS` | SEMINUEVOS | 11 |
| `EMBAJADOR SEMINUEVOS` (sin "DE") | SEMINUEVOS | 4 |
| `EXTERNO OLEKSEI` | SEMINUEVOS | 6 |
| `EXTERNO OLEKSEY` | SEMINUEVOS | 11 |
| `VENDEDOR EXTERNO` | COMERCIAL | 12 |
| `PROMOTORES COMERCIALES` | COMERCIAL | 1 |
| `EJECUTIVO COMERCIAL TRADICIONAL` | COMERCIAL | 1 |

Dos cosas que se ven en esa tabla y conviene no olvidar: **nómina escribe el mismo puesto de
varias formas** (`OLEKSEI`/`OLEKSEY`, `EMBAJADOR DE SEMINUEVOS`/`EMBAJADOR SEMINUEVOS`), así que
autorizar uno rara vez basta; y `VENDEDOR EXTERNO` acotado a COMERCIAL **deja fuera a propósito**
a los 3 de FLOTILLAS.

Eso obligó a cambiar la llave de la tabla: `EXTERNO OLEKSEI` necesita existir dos veces, una por
COMERCIAL y otra por SEMINUEVOS. Pasó de `PRIMARY KEY (puesto)` a un **índice único
`(puesto, depa)`** — no una PK, porque una PK exige columnas `NOT NULL` y aquí el `depa` en NULL
es un valor con significado. La nota larga está en db/004.

### La agencia se acota sola

En el alta de APV, el combo de **Agencia** sale del gerente elegido, no del catálogo completo. Si
ese gerente lleva una sola plaza, sale esa y el campo queda bloqueado.

En *Objetivos del mes*, junto al control de periodo hay un filtro de agencia. Un gerente de dos
plazas capturaba las dos revueltas en la misma reja, con el renglón de al lado de la otra ciudad.
Los renglones filtrados se **ocultan, no se quitan**: el guardado recorre todo el DOM y manda los
que traen cambios, así que cambiar de agencia a media captura no pierde lo tecleado — y por eso la
marca de «modificado» tiene que mirar todos los renglones y no solo los visibles.

### Qué hace que un gerente sea "de" una agencia

Lo dice **nómina**, no una columna que alguien mantenga a mano: `dbo.vw_GerenteSucursal`
(db/007) cruza `dbo.gerentes` contra `CONTPAQ_2` con el mismo mapeo de agencia de db/004. Un
gerente puede llevar tres agencias a la vez, y basta que **una** sea del DH para que lo alcance.

Antes esto se decidía con *"tiene una asignación viva en tus agencias"*, y esa regla **se podía
fabricar**: `POST /apv` dejaba a un DH asignar un APV de su agencia a un gerente de otra, y esa
asignación recién creada volvía al gerente ajeno administrable por quien la creó — o sea, el DH
se abría solo la puerta al restablecimiento de contraseña. Con la pertenencia en nómina, crear
una asignación ya no reparte permisos: **nómina no se mueve por lo que la app haga**.

La regla vive en `decideAlcanceGerente` (`backend/alcance.js`), con pruebas, y la usan los
cuatro sitios que la necesitan: listar gerentes, restablecer contraseñas, dar de alta un APV y
vincular una asignación. Una sola definición, no cuatro que se desalinean.

#### Qué puestos cuentan como gerente

`dbo.nomina_puestos_gerente` (db/005). Nómina tiene 34 puestos que empiezan con GERENTE y casi
ninguno es de esta app — servicio, refacciones, TI, capital humano, tesorería. Solo entran los
que llevan APV a cargo.

El **2026-09-10**, al autorizar en db/004 a los APV de seminuevos, hubo que autorizar también a
sus jefes: si no, esos APV entraban al padrón y **no tenían gerente que los capturara**. Se
agregaron `GERENTE DE SEMINUEVOS` (8 vigentes en 9 agencias), `GERENTE DE FLOTILLAS` (1) y
`COORDNADOR DE SEMINUEVOS` (1). Los candidatos a gerente pasaron de **4 a 14**.

⚠️ `COORDNADOR` va **sin la I a propósito**: así está escrito en nómina. Si allá lo corrigen, esa
persona desaparece de la lista sin que nadie se entere — el mismo fallo callado que obligó a
declarar `OLEKSEI` y `OLEKSEY` por separado.

Quedaron **fuera a propósito** y son decisión pendiente: `GERENTE FLOTILLAS GOBIERNO` (1) y
`GERENTE FLOTILLAS PRIVADAS` (1) —no son variantes de escritura, son otro alcance.

El **2026-10-30** se autorizó `GERENTE GENERAL` (5), que también estaba pendiente. Entra como
un gerente más: sale en candidatos y RH de sus agencias lo alcanza.

**El costo, dicho en voz alta:**

- Un gerente que nómina no ubica — nombre escrito distinto, puesto fuera de
  `dbo.nomina_puestos_gerente`, agencia sin mapear — **no es de nadie**: solo el admin lo
  alcanza. Sale listado en `dbo.vw_GerenteSinNomina` con el motivo, para que la falta se vea en
  vez de que simplemente falten permisos. Revísala después de correr db/007.
- Si `CONTPAQ_2` no se puede leer, **ningún DH alcanza a ningún gerente**. Es el lado correcto de
  fallar, y la pantalla lo dice con el error de SQL en vez de mostrar una lista vacía.

### El personal de RH no sale de ningún padrón

Los APV salen de `CONTPAQ_2` y los gerentes también. RH no: no es plantilla comercial, así que no
aparece en `vw_PadronNomina` ni en las fuentes del CRM. Se dan de alta a mano desde
*Administración › Personal de RH*, que crea su acceso en la central con perfil `rh` y le asigna
sus agencias. Son pocos y cambian poco.

El campo de agencia es **multi-selección**: un DH que cubre tres plazas se da de alta en un solo
envío. Antes era una por envío, y repetir el formulario tres veces es la forma de que a alguien le
quede el alcance a medias — con acceso creado y una sola agencia, que no se nota. El backend
reporta agencia por agencia qué pasó (agregadas, las que ya cubría, las que no están en el
catálogo) en vez de un «listo» que esconde a la que no entró.

La tabla lleva **un renglón por persona**, no uno por agencia, aunque en la base sean renglones
sueltos: lo que se revisa aquí es a quién le toca qué, y con tres renglones separados el alcance
completo había que armarlo con la vista. Cada renglón trae *Editar agencias* y *Restablecer
contraseña*.

*Editar agencias* manda el conjunto **completo** (`PUT /usuarios-rh/:usuario/sucursales`) y el
backend reconcilia en una transacción: da de baja lo que sobra, de alta lo que falta y **no toca
lo que ya estaba** — reinsertarlo movería su `created_at` y borraría desde cuándo cubre esa plaza.
Editarlo a pedazos, como antes, dejaba a la persona entre un clic y el siguiente con un alcance
que nadie quiso.

Dos casos que la pantalla no esconde: quien se queda **sin ninguna** agencia sigue en la tabla
(su acceso vive en la central y no se toca aquí; si desapareciera, no habría desde dónde
devolvérsela ni restablecerle la contraseña), y una agencia asignada que **salió del catálogo**
se ofrece marcada y señalada, porque borrarla en silencio al guardar sería quitarle alcance sin
que nadie lo haya decidido.

**Un RH no queda en `dbo.gerentes`**, y es a propósito: ese nombre es el que Power BI agrupa en
la columna `GERENTE`, y meter ahí a quien no lleva APV ensuciaría el reporte.

---

## El Dashboard: objetivo contra real

La pestaña **Dashboard** es el port de la página **APV** de `INDICADORES TACTICOS`: por asesor,
lo capturado aquí contra lo que de verdad hizo. Lo real lo calculan dos funciones de
`db/010_dashboard_reales.sql` con **las mismas cuentas de las medidas `APV *` de IT** — la tabla
de equivalencias está en la cabecera del script. Si una medida cambia allá, se cambia ahí.

| Fuente | Rubros | Función |
|---|---|---|
| `Hubspot_BI_Deals` | prospectos, demos, solicitudes | `dbo.fn_RealesHubspot(@desde, @hasta)` |
| `QVN` | facturas, crednissa, entregas | `dbo.fn_RealesQVN(@desde, @hasta)` |
| `inca_cvn` (BMW / MINI / Motorrad) | facturas, entregas | `dbo.fn_RealesIncadea(@desde, @hasta)` — db/014 |

Ventas y entregas **suman** QVN e incadea (cada asesor vive en una o en otra). BMW no tiene
medida en IT: las reglas son de esta app y están en la cabecera de db/014. Si incadea tiene a
un vendedor con otro nombre, su `SalespersonCode` va en `dbo.alias_incadea` y ese código manda
sobre el nombre — solo para incadea; HubSpot sigue empatando por nombre. El desplegable de autos
y el Forecast también suman incadea (`fn_FacturasAsesorIncadea`, `fn_FacturasAsesorPorDiaIncadea`);
la curva de cierre del Forecast se queda solo con QVN.

Lo que no es obvio:

- **El universo no es la reja de captura.** La reja es "a quién le capturo hoy"; el Dashboard es
  "de quién era este mes": los objetivos del mes con su gerente y agencia de la foto, más las
  asignaciones vigentes en algún día del mes que no tienen objetivo (salen como *Sin objetivo*).
- **Lo real es de la persona, el objetivo de la agencia.** Un APV multi-agencia trae sus
  facturas en todos sus renglones (marcado *Multi-agencia*), y los totales lo cuentan una vez.
- **Una fuente caída es `null`, no cero**: se pinta como raya, el total también, y arriba se dice
  cuál fuente falló y por qué. *Sin empate* solo se afirma si las dos fuentes se leyeron.
- **Las cuentas son las de IT; la forma no.** Se lee de arriba abajo: las ventas del mes en una
  cifra, el embudo (cada etapa contra su objetivo y la conversión contra su meta), y una sola
  tabla que se ve por sucursal, gerente o asesor. Un clic en un renglón baja al siguiente nivel ya
  filtrado; las columnas se ordenan con clic. Tres tonos: 100 % o más, 60–99 %, menos de 60 %.
- Sin objetivo, el cumplimiento es raya y no un `0.0 %` en rojo como en IT.
- **Autos facturados por asesor** (`db/011`, `GET /dashboard/facturas`): en la vista por asesor,
  un clic en el nombre despliega sus autos del mes con VIN, año, modelo y fecha de venta (la de
  factura). Son los mismos renglones que cuenta su celda de Ventas (comprobado: 558 = 558 en
  septiembre). El permiso es el de la tabla: un asesor fuera del alcance da 404. Se pidieron
  también los días en inventario y se cambiaron por la fecha de venta: no hay fuente, porque
  `QINVN` es la foto del inventario de hoy y lo vendido sale de ahí.
  Desde el 2026-10-19 cada auto trae también **precio de venta sin IVA** (con ISAN, la base de la
  utilidad; en incadea `PrecioVenta + ISAN`) y **utilidad bruta y total**, con las fórmulas de las
  columnas de `QVN FACTURAS` en INDICADORES TACTICOS (están en la cabecera de db/011). En BMW
  (incadea) la bruta es `PrecioVenta − CostoUnidad` y la total va en raya: incadea no trae
  rebates ni gastos. Hay que volver a correr **db/011 y db/014**; mientras no, el desplegable
  dice que falta la migración en vez de listar.
- **Top 3 agencias**: gana la que tiene más de sus seis indicadores en 95 % o más; empate, por %
  de ventas. Una agencia sin objetivo en el mes no compite. Solo lo ve el admin y quien lleva 3
  agencias o más (`ctx.mis_sucursales`), y se oculta con una sucursal elegida en el filtro. Es
  cortesía de la pantalla: son los mismos números que ya ve en la tabla.
- Empata **un poco más** que Power BI: la app colapsa los dobles espacios y IT no.

### Forecast

Pestaña aparte, junto al Dashboard (`backend/forecast.js`, `db/012`, `GET /forecast`). Estima con
cuántas **ventas** cierra el mes cada asesor:

> estimado = lo que lleva + su promedio de los 3 meses cerrados × lo que falta del mes

- **"Lo que falta" no es el calendario**: es la curva de cierre de la empresa en los 12 meses
  cerrados más recientes, alineada por días que faltan al cierre. La venta se carga al final: al
  día 20 va el 52 % del mes, no el 67 %. La pantalla grafica esa curva contra la del calendario
  para que se entienda por qué el estimado no es regla de tres.
- Se eligió con un backtest contra QVN (abr-2025 a ago-2026, asesores de la app). A mitad de mes
  el total queda a ±10 % del cierre; la regla de tres fallaba un 30 %, siempre hacia abajo. La
  tabla está en el encabezado de `forecast.js`.
- **El día es el corte de QVN, no el de hoy**: si el espejo se atrasa, se estima desde el último
  día que trae y la pantalla dice "QVN al 29 sep".
- El % y la diferencia comparan **solo a los asesores con objetivo**; si no son todos, se dice
  cuántos. Universo, alcance, multi-agencia y `null` por fuente caída: los del Dashboard.
- Asesor sin ventas en los 3 meses: su estimado es lo que lleva, y se marca.
- **Mes cerrado: estimado contra real.** El backend reconstruye lo que el Forecast habría dicho
  cada día (`fn_FacturasAsesorPorDia`, en el mismo `db/012`), solo con lo que existía ese día.
  Se compara al día 5, 10, 15, 20, 25 o **Cierre**: el último día del mes, con QVN hasta el
  anterior, que es como se vive en la realidad. La gráfica pone lo que estimaba contra lo que
  llevaban, día por día.

### Cache de las fuentes

Lo que viene de **HubSpot, QVN e incadea** se guarda en la memoria del backend (`backend/cache.js`)
y se refresca **al minuto 10 de cada hora** (QVN se actualiza al 57; el resto es margen). Lo usan
el Dashboard, el desplegable de autos y el Forecast. Medido el 2026-10-20: el Dashboard bajó de
2.6 s a 0.3 s y el Forecast de un mes cerrado de 1.3 s a 0.3 s.

- **Las tablas propias no se guardan nunca** (objetivos, asignaciones, gerentes, aperturas): son
  milisegundos y de ahí salen el alcance y la ventana de captura. Por eso un objetivo guardado se ve
  al instante sin invalidar nada: lo que se guarda en cache es lo que esta app no escribe.
- **Una fuente caída no se guarda**: la pantalla lo sigue diciendo y la siguiente carga reintenta.
  Si el refresco del minuto 10 falla, la entrada se borra en vez de servirse vieja.
- Todo vence en el siguiente minuto 10 aunque el refresco no corra; lo que nadie pide en 2 horas
  se suelta. Reiniciar el backend vacía el cache: la primera carga de cada mes paga la consulta.
- Entre el minuto 57 y el 10 la pantalla puede ir hasta 13 minutos detrás de QVN. El desplegable
  usa el mismo cache que la celda, así que los dos siempre cuadran entre sí.

---

## La entrega a Power BI

Es la parte delicada, y **no es solo cambiar el origen de datos**.

`AUXILIAR_MAESTRO_APV` no es una tabla de hechos: es la **dimensión de asesores**. Su columna
`ASESOR` está del lado *uno* de cuatro relaciones muchos-a-uno:

| Tabla | Columna | Origen real |
|---|---|---|
| `PIPEFY_COMERCIAL` | `Ejecutivo Comercial` | `dbo.PIPEFYFULL` |
| `PIPEFY_FINANCIAMIENTO` | `Ejecutivo Comercial` | `dbo.VW_PIPEFY_FINANCIAMIENTO` |
| `HUBSPOT_AFLUENCIA` | `ejecutivo_comercial_nombre` | `dbo.Hubspot_BI_Deals` |
| `BIQ_VN` | `NOMBRE_VENDEDOR` | `dwh.comercial_vn` |

Eso obliga a que **`ASESOR` sea único**. Los objetivos, en cambio, tienen grano mensual y
multi-sucursal: el mismo asesor aparece varias veces. Apuntar la partición actual directo a
los objetivos falla el refresh con *«la columna contiene valores duplicados»*.

Por eso se entregan **tres vistas**, cada una con un trabajo distinto:

| Vista | Grano | Para qué |
|---|---|---|
| `vw_ObjetivosAPV_Asesores` | 1 renglón por asesor | La **dimensión**. Conserva las 4 relaciones |
| `vw_ObjetivosAPV_Hechos` | asesor × sucursal × año × mes | Los **hechos**. Trae `FECHA_MES` para relacionar con el calendario |
| `vw_MAESTRO_APV_COMPAT` | 1 renglón por asesor, 9 columnas | El **parche**: reemplazo directo del Excel |

Más `vw_AsignacionesAPV` (la trazabilidad) y `vw_Sucursales` (el combo de la app).

### Opción A — el parche, hoy mismo, sin tocar el modelo

Cambiar la partición de `AUXILIAR_MAESTRO_APV` de SharePoint a SQL:

```m
let
    Origen = Sql.Database("144.217.90.168", "objetivos_apv"),
    vista  = Origen{[Schema="dbo", Item="vw_MAESTRO_APV_COMPAT"]}[Data]
in
    vista
```

Las 9 columnas son las mismas, `ASESOR` sigue siendo único y las cuatro relaciones no se
tocan. **Y ya no hace falta la limpieza de Power Query**: la vista entrega el nombre ya
normalizado igual que hoy (`Text.Trim(Text.Replace(Text.Upper(_), "Ñ", "N"))`), pero calculado
por la base, en `dbo.apv.nombre_normalizado`.

⚠️ **Esta vista miente por diseño, en dos cosas**, y hay que saberlo antes de usarla:

1. `SUCURSAL` y `GERENTE` son los de la **asignación viva más reciente**. Un APV que trabaja
   en tres agencias aparece con una sola. Es exactamente la limitación que ya tiene el Excel
   —no se empeora nada— pero tampoco se arregla.
2. Las cifras son las del **mes en curso**, sumadas sobre todas sus agencias. El año no se
   puede reconstruir desde ahí.

Sirve porque **hoy ninguna medida DAX ni ningún visual consume esas seis columnas de
objetivo**: del Excel solo se usan `ASESOR` (dimensión y filtro de página) y `SUCURSAL` (vía
`TREATAS`). El parche no rompe nada que esté vivo. Pero tampoco entrega valor nuevo.

### Opción B — lo correcto: dimensión + hechos

Es la que hace que los objetivos sirvan de algo, porque hoy **no hay ni una medida de
cumplimiento en el modelo**.

1. **Repuntar `AUXILIAR_MAESTRO_APV`** a `vw_ObjetivosAPV_Asesores` (mismo nombre de tabla,
   para no tocar las 4 relaciones ni los visuales que ya la usan).
2. **Agregar `HECHOS_OBJETIVOS_APV`** desde `vw_ObjetivosAPV_Hechos`, con tres relaciones:

   | De | A | Cardinalidad | Dirección |
   |---|---|---|---|
   | `HECHOS_OBJETIVOS_APV[ASESOR]` | `AUXILIAR_MAESTRO_APV[ASESOR]` | muchos a uno | sencilla |
   | `HECHOS_OBJETIVOS_APV[FECHA_MES]` | `AUXILIAR_CALENDARIO[Date]` | muchos a uno | sencilla |
   | `AUXLIAR_SUCURSALES[S_Homologada]` | `HECHOS_OBJETIVOS_APV[SUCURSAL]` | **muchos a muchos** | **sencilla**, de `AUXLIAR_SUCURSALES` hacia los hechos |

   La tercera tiene que ser muchos-a-muchos porque `S_Homologada` **no es única** en
   `AUXLIAR_SUCURSALES` (82 renglones, 35 valores distintos). Y la dirección importa: es lo
   que hace que **el RLS de los 16 roles filtre los objetivos solo**, sin tocar nada más.
   No la pongas bidireccional: RLS más filtrado cruzado bidireccional es la receta conocida
   de resultados que nadie puede explicar.

3. **Escribir las medidas**, que hoy no existen. Con el calendario ya relacionado:

   ```dax
   OBJ Prospectos = SUM ( HECHOS_OBJETIVOS_APV[PROSPECTOS] )

   Cumplimiento Prospectos =
   DIVIDE ( [PROSPECTOS_90D], [OBJ Prospectos] )
   ```

   (`[PROSPECTOS_90D]` ya existe en `MEDIDAS HTML`; ajusta a la medida real de cada rubro.)

### ⚠️ Cosas que ya están mal en Winston y conviene revisar al hacer esto

Encontradas al leer el `.pbip`. No las causa este proyecto, pero le pegan:

- **La relación `AUXILIAR_MAESTRO_APV[SUCURSAL]` → `AUXLIAR_SUCURSALES[Sucursal]` está
  INACTIVA**, y apunta a la columna **cruda** (82 variantes), no a `S_Homologada`.
- **Dos medidas de `MEDIDAS HTML` hacen `TREATAS` sobre la misma columna con columnas
  distintas**: una usa `VALUES(AUXLIAR_SUCURSALES[Sucursal])` y otra
  `VALUES(AUXLIAR_SUCURSALES[S_Homologada])`, ambas contra `AUXILIAR_MAESTRO_APV[SUCURSAL]`.
  **Las dos no pueden estar bien.** Como el RLS filtra por `[S_Homologada]` y los valores de
  los roles son de esa columna (`CARRANZA`, `LOMAS`, `NAVA`...), la del `Sucursal` crudo es la
  sospechosa: filtra por valores que esa columna no contiene, así que devuelve vacío en
  silencio.
- **`AUXILIAR_MAESTRO_APV` no tiene RLS**: ningún rol le pone `tablePermission`. Por eso las
  medidas empujan la sucursal con `TREATAS` en vez de dejar que el modelo la propague. Con la
  relación muchos-a-muchos de la Opción B, los **hechos** sí quedan cubiertos por RLS; la
  dimensión sigue sin estarlo, igual que hoy.
- **La normalización de `GERENTE` en Power Query solo reemplaza `Í`, `Á` y `Ó`** — no `É`, `Ú`
  ni `Ñ`. Un gerente "PÉREZ" agrupa aparte de "PEREZ". Esta app guarda el nombre ya
  normalizado completo, así que de aquí en adelante deja de pasar, pero los datos históricos
  del Excel pueden traerlo.

### El nombre es la única llave, y es de vidrio

No hay clave: `ASESOR` se une a las cuatro fuentes **por el texto**. Si el gerente teclea
`JOSE PENA` y en `BIQ_VN` dice `JOSÉ PENA`, Power BI no los une y ese asesor cae en el balde
`(SIN ASESOR / SIN MATCH)` que ya aparece en las medidas. El objetivo queda bien capturado, se
ve bien en la app, y en el reporte sale en ceros. **Nadie se entera hasta fin de mes.**

Por eso `db/003_vista_nombres_fuente.sql` (opcional) junta los nombres tal como existen hoy en
las cuatro fuentes, normalizados igual, y la pantalla de alta los **sugiere** y avisa cuando el
nombre tecleado no empata con ninguno. Avisa, no impide: un APV de nuevo ingreso todavía no
existe en ninguna fuente.

`dbo.vw_AsesoresSinMatch` lista los que hoy no empatan. Vale la pena mirarla después de cada
alta masiva.

#### El prefijo `(E) ` de BIQ_VN

`db/003` recorta un `(E) ` al principio del nombre antes de agrupar. No es parte del nombre: es
basura que dejó una prueba en BIQ para marcar externos. Medido el **2026-09-10** lo traen 5
nombres, todos solo en `BIQ_VN`, y de esos **2 son APV vigentes de nómina** — `JULIO CESAR
SANCHEZ OVALLE` y `ANDREYNA DE LOS ANGELES CRESPO MANZANILLA`, los dos facturando en INFINITI
QUERETARO. Sin el recorte la app los marcaba como ausentes del CRM cuando sí están vendiendo.

⚠️ **Esto arregla el aviso, no el reporte.** Winston une `AUXILIAR_MAESTRO_APV[ASESOR]` contra el
`BIQ_VN[NOMBRE_VENDEDOR]` **crudo**, que sigue trayendo el prefijo: sus facturas siguen cayendo en
`(SIN ASESOR / SIN MATCH)`, y ahora sin que la app avise. Queda pendiente limpiarlo **allá** — en
el origen (`dwh.comercial_vn`) o en el Power Query de esa tabla. Hasta entonces el recorte de
db/003 es media solución, y la mitad que falta es la que se ve en el tablero.

---

## Puesta en marcha

### 1. Base de datos (una vez, como `sa`)

```bash
sqlcmd -S 144.217.90.168 -U sa -b -i db\001_objetivos_apv_schema.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\002_vistas_powerbi.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\003_vista_nombres_fuente.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\004_padron_nomina.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\005_padron_gerentes.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\006_usuarios_rh.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\007_alcance_gerente.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\008_normalizacion_espacios.sql
```

```bash
sqlcmd -S 144.217.90.168 -U sa -b -d objetivos_apv -i db\009_objetivos_calculados.sql
```

| Script | Qué deja | ¿Obligatorio? |
|---|---|---|
| `001` | Tablas, restricciones y el trigger de historial | Sí |
| `002` | Las tres vistas de Power BI + `vw_Sucursales` | Sí |
| `003` | Cotejo de nombres contra las 4 fuentes del CRM | No, pero es lo que evita el `SIN MATCH` |
| `004` | El padrón de nómina como lista viva de APV | No; sin él se teclean los nombres a mano |
| `005` | Lo mismo para los gerentes comerciales | No |
| `006` | El rol RH y su alcance por agencia | Solo si vas a usar RH |
| `007` | En qué agencia ubica nómina a cada gerente | Sí, si usas RH: sin él ningún DH alcanza a nadie |
| `008` | Colapsa los espacios internos del nombre en una base que ya traía datos | Sólo en una base viva anterior al 2026-09-10 |
| `009` | Ajusta a la fórmula los objetivos que ya estaban cargados | Sólo si ya se capturó con los seis rubros abiertos. Enseña lo que va a mover ANTES de moverlo |

El `004` necesita además, en el mismo servidor:

```bash
sqlcmd -S 144.217.90.168 -U sa -b -Q "USE INTELIGENCIA_NEGOCIOS; GRANT SELECT ON SCHEMA::dbo TO objetivos_apv_rw;"
```

Se da por **esquema** y no por objeto a propósito. `CONTPAQ_2` se recrea con `DROP`+`CREATE`
varias veces al día, y un `GRANT` sobre la tabla se va con la tabla: la app amanece degradada
con un error 229 y hay que volver a darlo a mano. El permiso por esquema lo hereda la tabla que
nazca mañana. Si prefieres acotarlo a una sola tabla, tendrás que reponerlo en cada carga.

Antes de correr el `001`, reemplaza `<CAMBIAR_PASSWORD>`. Omitir `-P` hace que `sqlcmd` pida la
contraseña de forma interactiva, y así no queda en el historial de PowerShell, que se guarda en
disco.

⚠️ **`-b` no es opcional.** Sin ella `sqlcmd` **sigue adelante después de un error**: si fallara
el `CREATE DATABASE` o el `USE`, las tablas se crearían en la base por omisión del login —que en
este servidor puede ser `INTELIGENCIA_NEGOCIOS`— y el síntoma sería un montón de *"Invalid column
name"* que no dicen nada. Los tres scripts traen además una **guarda de contexto** que corta la
conexión si no están en `objetivos_apv`, pero `-b` es la primera línea de defensa.

Los scripts son **reparables**: cada tabla, restricción e índice se comprueba por separado, así
que si una corrida se corta a la mitad, volver a correrla completa lo que falte. Probado tirando
a mano dos restricciones y un índice y re-ejecutando.

Si algo quedó a medias **en otra base** y todavía no hay datos, lo más limpio es empezar de cero:

```bash
sqlcmd -S 144.217.90.168 -U sa -b -Q "DROP TABLE IF EXISTS dbo.objetivos_historial, dbo.aperturas_mes, dbo.objetivos, dbo.asignaciones, dbo.apv, dbo.gerentes, dbo.logs" -d <la_base_equivocada>
```

El `002` incluye un `GRANT SELECT` sobre `INTELIGENCIA_NEGOCIOS.dbo.SUCURSALES`. **No es
opcional**: el encadenamiento de propiedad entre bases está apagado por omisión, así que sin él
`vw_Sucursales` devuelve "permiso denegado" y la app se queda sin combo de agencias.

El `003` es opcional (ver arriba) y pide `SELECT` sobre cuatro tablas más.

El `011` es el del desplegable de autos facturados del Dashboard; no pide permisos nuevos (usa QVN,
que ya concede el `010`). Sin él, el desplegable dice "falta correr la migración".

El `012` es el del Forecast (las facturas de la empresa por día); tampoco pide permisos nuevos.
Sin él, el Forecast enseña objetivo y lo que llevan, con el estimado en raya y el motivo arriba.

El `010` es el del **Dashboard** y pide `SELECT` sobre `INTELIGENCIA_NEGOCIOS.dbo.QVN` (HubSpot ya
lo tenía desde el `003`). Sin él la app funciona igual: el Dashboard enseña los objetivos y dice
qué fuente no pudo leer, en vez de pintar ceros.

### 2. Alta en la central de identidad (`APPS`)

Esta app **no tiene tabla de usuarios**. La identidad vive en la central, igual que las demás.

```sql
USE APPS;
IF NOT EXISTS (SELECT 1 FROM dbo.apps WHERE clave = 'OBJETIVOS_APV')
    INSERT INTO dbo.apps (clave, nombre) VALUES ('OBJETIVOS_APV', N'objetivos-apv');
```

Y **un primer administrador**, que es el único que se da de alta a mano:

```bash
node backend/tools/hash-password.js "LaContrasena" tu.usuario admin
```

Imprime el `INSERT` completo contra `APPS`. Los roles que esta app entiende son **`admin`**,
**`rh`** y cualquier otra cosa, que entra como **gerente**.

> **De ahí en adelante no hace falta SQL.** Desde *Administración* se dan de alta gerentes y
> personal de RH: la app crea su usuario en la central con una contraseña generada de 9
> caracteres y la muestra **una sola vez**, en un diálogo que hay que cerrar a mano. Si se
> pierde, el botón *Restablecer contraseña* genera otra.
>
> Esa contraseña es de la **persona**, no de la app: restablecerla aquí también cambia la que usa
> en `tablaMarcos` y las demás herramientas internas. La pantalla lo advierte antes de confirmar.
>
> Si alguien **ya existía** en la central por otra app, no se le toca la contraseña: solo se le
> suma la membresía, y la app dice "entra con la que ya usa".

**Después hay que registrar al gerente en esta app**, desde *Administración → Gerentes*: la
central sabe *quién es*, pero el **nombre que Power BI agrupa en la columna `GERENTE`** vive
aquí, en `dbo.gerentes`. Un usuario sin ese renglón entra y ve el aviso, pero no puede capturar.

### 3. Backend, en la máquina interna

```bash
cd backend && npm install && copy .env.example .env
```

Llena `DB_PASSWORD` y genera el token:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

```bash
npm start
```

```bash
curl http://localhost:3006/health
```

### 4. Túnel

```bash
tailscale funnel --bg --set-path=/objetivos_apv 3006
```

Puerto **3006** y ruta `/objetivos-apv`, ya anotados en el registro de la skill. `--set-path`
**sí recorta el prefijo**: la petición pública `/objetivos-apv/health` llega al backend como
`/health`, así que `BASE_PATH` va **vacío**.

### 5. Vercel

| Variable | Valor |
|---|---|
| `BACKEND_URL` | `https://mithril-api.tail3225eb.ts.net/objetivos_apv` (sin barra final) |
| `BACKEND_TOKEN` | el mismo `API_TOKEN` del `.env` |
| `SESSION_SECRET` | cadena larga y aleatoria, **propia de esta app** |
| `AUTH_URL` | `https://desktop-5si83fk.tail3225eb.ts.net/central-auth` (sin barra final) |
| `AUTH_TOKEN` | el `AUTH_TOKEN` del `.env` del `auth-service` en la `10.10.10.12` |
| `APP_CLAVE` | `OBJETIVOS_APV` |

El `SESSION_SECRET` se genera y **no se comparte con nadie ni con otra app**: es lo único que
separa las sesiones de ésta de las de las demás, y rotarlo cierra todas de golpe.

```bash
node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"
```

⚠️ **Hay que redesplegar después de agregarlas.** Las variables se leen al construir el
despliegue: agregarlas no toca el que ya está en línea, así que el error sigue exactamente
igual y parece que Vercel las ignoró.

Los errores salen **en orden**, uno por petición, así que conviene ponerlas todas de una vez en
lugar de ir descubriéndolas: primero `Login sin configurar (falta SESSION_SECRET)`, luego
`(faltan AUTH_URL/AUTH_TOKEN/APP_CLAVE)`, y ya con la sesión abierta,
`BACKEND_URL no está configurado en Vercel` al cargar la primera pantalla. Los tres son 500
explícitos a propósito: un 401 ahí parecería "contraseña incorrecta" y mandaría a buscar el
problema al lado equivocado.

### 6. Watchdog

```bash
powershell -ExecutionPolicy Bypass -File backend\install-watchdog.ps1
```

Sin él, el servicio se cae en el primer corte de luz o actualización de Windows y nadie se
entera hasta que un gerente reclama.

---

### El historial de un APV va acotado

Hasta el 2026-09-09 `GET /apv/:id/historial` devolvía la carrera completa a **cualquiera con
sesión**: agencias y gerentes que quien miraba no alcanza. Ahora se acota con `filtroAlcance`, la
misma regla del resto y no una propia:

| Quién mira | Qué periodos ve |
|---|---|
| admin | todos |
| rh | los de sus agencias |
| gerente | **los suyos** |

Para el gerente eso es más estrecho que «su agencia», y es a propósito: si el APV estuvo con otro
gerente en la misma plaza, ese tramo es del otro equipo. Errar del lado corto no cuesta nada.

Cuando no hay nada que mostrar, el modal dice *«no hay periodos que puedas ver»* y **no** «no
tiene historial»: puede tenerlo y no alcanzarlo quien mira, y afirmar de más manda a buscar un
problema que no existe.

**Debajo de cada mes queda su estela.** `dbo.objetivos_historial` lo llena un trigger en cada
INSERT y en cada UPDATE, y la pantalla de captura guarda **todos** los renglones del mes en cada
clic de «Guardar» — no solo los que se tocaron. Un gerente que guarda cinco veces deja cinco
renglones idénticos por asesor. Eso está bien para una bitácora y mal para leerla, así que
`soloLosQueCambiaron()` (`backend/bitacora.js`, con pruebas) conserva el alta y después solo los
renglones en que alguno de los seis rubros cambió de verdad. La fecha que queda es la de la
**primera** vez que tomó ese valor: volver a guardar lo mismo no es un cambio, y cambiar de
gerente o de hora tampoco.

**El mes resaltado es el que cuenta.** Los objetivos se cuelgan de cada periodo por agencia y por
traslape de fechas, no por `objetivos.asignacion_id` — esa columna guarda quién lo *guardó al
final*, así que después de una rotación apunta al gerente nuevo aunque el mes sea del anterior.
`vigente_al_cierre` marca el periodo que seguía vivo el último día del mes, que es el que Power BI
reporta. En una rotación a media captura el mismo mes aparece bajo los dos periodos y **solo uno
lleva la marca** — es la otra cara de «cargado antes de la rotación» en la reja de captura.

---

### Los rewrites de `vercel.json` no son opcionales

Vercel estaba tratando `api/datos/[...ruta].js` como si fuera `[ruta].js`: casaba **un** segmento
y nada más. Todo lo de dos o más — `apv/:id/historial`, `objetivos/:id/historial`,
`nomina/bajas`, `gerentes/:usuario/administrable`, `asignaciones/:id/cerrar`,
`aperturas/:id/cerrar`, `usuarios-rh/:id/quitar` — moría en el 404 del enrutador **sin llegar
nunca a la función**. Comprobado en producción el 2026-09-09: `/api/datos/apv` daba 401 y
`/api/datos/apv/1/historial` daba `NOT_FOUND`.

Las dos reglas de `rewrites` dejan de depender de ese comportamiento: mandan la ruta completa en
`?ruta=` y aterrizan en el primer segmento, que es lo único que Vercel casa de forma confiable.
Por eso `rutaPedida()` **prefiere el parámetro** sobre el pathname — al revés, `apv/1/historial`
se leería como `apv`, una ruta válida y equivocada, que es peor que el 404.

A propósito **no** hay regla para un solo segmento: esas rutas ya funcionan y no vale la pena
arriesgarlas. Si una regla estuviera mal, lo roto sigue roto y lo bueno sigue bueno.

⚠️ `vercel.json` se valida contra un esquema estricto: **una clave desconocida tumba el
despliegue entero**. No se pueden dejar comentarios ahí; por eso esto vive aquí.

```bash
node api/pruebas/ruta.test.mjs
```

Cubre los dos caminos (parámetro y pathname) y no necesita backend. Lo que **no** puede cubrir:
el enrutador de Vercel no existe en local, así que que un rewrite esté bien escrito solo lo dice
un despliegue de preview.

---

## Reglas que no se deben romper

1. **El alcance sale de la identidad del token, nunca del cuerpo de la petición.** Un gerente
   solo ve y captura sus APV; un RH, los de sus agencias. Que el JSON traiga un `gerente_id` o
   una sucursal no significa nada por sí solo. Vive en `backend/alcance.js` y en
   `gerenteObjetivo()`, un solo lugar cada uno. Ver *Quién puede qué*.
   **Y armar el equipo es de RH y del admin** (`mueveEquipo()`): alta, vinculación y baja de
   APV. El gerente lee a los suyos y les captura. Ver *Armar el equipo es de RH, no del
   gerente*.
2. **La ventana de captura se re-evalúa en el servidor, renglón por renglón, en cada guardado.**
   Que la pantalla haya pintado un campo como editable no autoriza nada.
3. **La fecha sale del reloj del servidor**, nunca del navegador. Si viniera del cliente,
   cambiarle la fecha al equipo abriría un mes cerrado. Todo servidor que corra esto va en
   `Central Standard Time (Mexico)`.
4. **Una asignación nunca se reactiva.** Ahí vive la trazabilidad de las rotaciones.
5. **Rotar a un APV no reescribe ni un objetivo ya cargado.** El `gerente_id` del objetivo es
   una foto, no un apuntador. Solo cambia si alguien vuelve a editar ese renglón, y la versión
   anterior queda en `objetivos_historial`.
6. **El backend nunca sin token.** El Funnel lo expone a internet entero; el server se niega a
   arrancar sin `API_TOKEN`. Déjalo así.
7. **La allowlist de `api/datos/[...ruta].js` es el acto de publicar una ruta.** Sin ella,
   cualquier endpoint que alguien agregue mañana al backend queda en internet el mismo día.
8. **De los seis rubros del objetivo se captura uno: `facturas`.** Los otros cinco los calcula
   `backend/calculados.js` al guardar, y lo que el cuerpo del PUT traiga en ellos **se ignora**
   —la reja los pinta de solo lectura, pero eso no es la garantía—. Las proporciones son las de
   la hoja de Excel que se venía usando:

   | Rubro | Fórmula |
   |---|---|
   | `prospectos` | `REDONDEAR(facturas / 0.2 ; 0)` — facturas × 5 |
   | `demos` | `REDONDEAR(prospectos × 0.5 ; 0)` |
   | `solicitudes` | `REDONDEAR(prospectos × 0.8 ; 0)` |
   | `crednissa` | `REDONDEAR(prospectos × 0.7 ; 0)` |
   | `entregas` | igual a `facturas` |

   Las cuentas van **en enteros**, no multiplicando por `0.8` y `0.7`: en IEEE 754 `45 * 0.7`
   da `31.499999999999996`, y redondear eso regresa `31` donde la hoja dice `32` — un rubro
   menos, sin un solo error a la vista. `app.js` repite la fórmula para pintar el renglón
   mientras se teclea; si una proporción cambia, **se cambian las dos**, y la de `backend/` es
   la que manda. Por lo mismo el tope de captura es `MAX_OBJETIVO / 5`: es el que deja al mayor
   de los calculados dentro de `INT`.

   **Lo ya cargado no se ajusta solo.** El cálculo corre al guardar, así que un renglón viejo
   se queda como está hasta que su gerente vuelva a guardarlo —y un mes cerrado no se puede
   volver a guardar sin reapertura—. Para emparejar lo que ya está en la base existe
   `db/009_objetivos_calculados.sql`, que por omisión sólo toca los meses que todavía se
   pueden capturar.

### Lo que la base garantiza por sí sola

Las pantallas validan para dar mensajes decentes, pero la garantía está en la base: dos
pestañas abiertas, o dos capturistas a la vez, se saltan cualquier validación que viva solo en
el código.

| Restricción | Qué impide |
|---|---|
| `UX_apv_nombre_normalizado` | Dos asesores que normalicen igual. Es la llave de negocio: sería un refresh roto en Winston |
| `UX_asig_abierta` (filtrada) | Dos asignaciones vivas del mismo APV en la misma agencia. Filtrada a propósito: las cerradas sí se repiten, porque volver a una agencia es normal |
| `UX_obj_grano` | Dos objetivos del mismo mes y agencia. Sin ella, un cambio de gerente haría que Power BI sumara doble |
| `CK_obj_no_negativos` | Objetivos negativos, que ensuciarían las medidas de cumplimiento en silencio |
| `CK_obj_mes` / `CK_obj_anio` | Un mes 13 o un año imposible |
| `UX_ap_viva` (filtrada) | Dos aperturas vivas del mismo mes para el mismo gerente. Que esté en la base es lo que impide que dos admins simultáneos abran el mismo mes dos veces |
| `TR_objetivos_historial` | Que un objetivo cambie sin dejar rastro — incluso desde SSMS |

---

## Pruebas

**Esquema y vistas**, contra una LocalDB (motor SQL Server de verdad: ejercita columnas
calculadas persistidas, índices filtrados, triggers y `CREATE OR ALTER VIEW`). No toca ningún
servidor real:

```bash
powershell -ExecutionPolicy Bypass -File db\pruebas\correr.ps1
```

Corre `001`–`009` y las comprobaciones del esquema: entre ellas la rotación que conserva los
dos periodos, el ASESOR duplicado que la base rechaza, y el cotejo de nombres contra las cuatro
fuentes.

Para `009` no basta con que no truene: en una base recién creada no tendría nada que ajustar.
Por eso `db/pruebas/009_siembra.sql` le pone delante tres objetivos torcidos a propósito —uno
desfasado del mes en curso, uno que ya cuadra y uno de un mes cerrado— y
`009_comprobar.sql` verifica que mueva el primero, no toque los otros dos, y deje su rastro en
el historial. Después limpia lo suyo, porque las pruebas del esquema cuentan renglones.

> El nombre del named pipe de LocalDB **cambia en cada reinicio**, y LocalDB se auto-apaga por
> inactividad. Por eso el script lo consulta cada vez en vez de anotarlo. El síntoma de tenerlo
> viejo es `El sistema no puede encontrar el archivo especificado`, que suena a que falta un
> `.sql` y no tiene nada que ver.

**Revisión completa sin base** — ESLint, tipos y las pruebas que no necesitan servidor, de un
golpe. Es lo que hay que correr antes de desplegar:

```bash
cd tools && npm install && npm run revisar
```

- `npm run lint`: ESLint con solo las reglas que atrapan errores (variables sin usar, nombres sin
  declarar); nada de estilo. Al encenderlo encontró un `catch` de `/sucursales` que usaba `req`
  sin tenerlo: un fallo de SQL ahí tiraba el proceso entero en vez de responder 500.
- `npm run tipos`: TypeScript **solo revisa**, no compila. Lee los tipos del JSDoc de los archivos
  que empiezan con `// @ts-check` (backend, BFF y frontend); el editor marca lo mismo.
- `npm run pruebas`: las de `backend/pruebas/*.test.js` y `api/pruebas/ruta.test.mjs`.

Todo eso vive en `tools/`, con su propio `package.json`, y no en la raíz: un `package.json` en
la raíz cambiaría cómo Vercel detecta el build, y `tools/` está en `.vercelignore`.

Lo que **no** entra en `revisar` porque necesita un backend vivo: `api/pruebas/proxy.test.mjs` y
las pruebas de humo de `backend/pruebas/*.mjs` (estas además **dejan datos**; ver su cabecera).

**Lógica pura** — la ventana de captura y el alcance por rol, sin base ni servidor:

```bash
cd backend && npm test
```

Las primeras 25 pruebas fueron 12 de `ventana.js` y 13 de `alcance.js`. Las dos viven aparte de `server.js` porque
son las reglas que, si se rompen, **no dan error**: un mes que se abre cuando no debe, o un
permiso de más. Las de la ventana están sometidas a prueba de mutación — cambiar el `>= 0` de
`dentroDeVentana` por `> 0` hace fallar 4 de ellas, así que sí muerden.

Y la central tiene las suyas:

```bash
cd ../../auth-service && npm test
```

8 pruebas sobre `password.js`. La que más importa: **los hashes de 64 bytes que ya existen (BMW)
siguen verificando**. Si eso se rompe, los usuarios viejos de otra app dejan de entrar.

**La API de punta a punta**, contra SQL Server de verdad. Habla con el backend como lo hace el
BFF (bearer + `X-User`/`X-Role`), así que un fallo aquí señala al backend y no al proxy:

```bash
cd backend && BASE=http://localhost:3006 TOKEN=<el API_TOKEN> node pruebas/humo-api.mjs
```

27 comprobaciones: la puerta del token, el alcance por gerente, el alta duplicada que ofrece
vincular, la ventana de captura rechazada en el servidor, la reapertura por admin, el historial
con sus dos versiones, y la rotación que conserva los dos periodos.

**La regla de un solo gerente por agencia**, que es la que hace que el objetivo mensual tenga
dueño, con el caso del negocio tal cual:

```bash
cd backend && BASE=http://localhost:3006 TOKEN=<el API_TOKEN> node pruebas/regla-un-gerente-por-agencia.mjs
```

14 comprobaciones. El gerente A de NAVA tiene a Juanito; no se le puede crear otro, ni
vinculárselo a B en esa misma agencia, ni B capturarle objetivos, y ni dos peticiones a la vez
logran dos asignaciones vivas. En **otra** agencia sí. Cuando RH cierra el periodo con A, B ya
puede — y el historial conserva los dos.

Los movimientos van como admin, porque armar el equipo dejó de ser del gerente; la regla que se
prueba no cambia con eso, vive en el índice único filtrado de la base y no en quién la pide. De
paso quedan comprobados los **403** del gerente: ni se agrega a nadie ni cierra su propio
periodo.

**El BFF de Vercel** (`/api/datos/*`), contra el backend corriendo:

```bash
BACKEND=http://localhost:3006 TOKEN=<el API_TOKEN> node api/pruebas/proxy.test.mjs
```

10 comprobaciones: la sesión, la firma de la cookie, la allowlist, el reenvío de la query
string, el `Cache-Control: private`, y los dos fallos de despliegue que ya pasaron —el
`req.query.ruta` vacío y el `BACKEND_URL` apuntando a otra máquina.

**Casos borde**, buscando romperla en vez de confirmarla:

```bash
cd backend && BASE=http://localhost:3006 TOKEN=<el API_TOKEN> node pruebas/humo-bordes.mjs
```

21 comprobaciones: agencias fuera del catálogo, números fuera del rango de `INT`, textos más
largos que su columna, referencias a cosas que no existen, periodos imposibles, cuerpos basura,
y **concurrencia** — cinco guardados simultáneos del mismo renglón y tres altas simultáneas del
mismo nombre. Encontró cuatro bugs reales la primera vez que se corrió; ver *Lo que encontraron
las pruebas de borde*.

⚠️ **Deja datos.** Todo lo que crea lleva el prefijo `ZZ PRUEBA` / `zz.prueba`. Se borra con:

```bash
sqlcmd -S 144.217.90.168 -U objetivos_apv_rw -d objetivos_apv -b -i backend\pruebas\limpiar-humo.sql
```

Córrelo contra una base recién instalada, no contra una con capturas reales.

### Lo que encontraron las pruebas de borde

Seis defectos reales, todos corregidos. Se dejan escritos porque las correcciones parecen
arbitrarias sin el caso que las motivó:

| Qué pasaba | Por qué importaba | Cómo quedó |
|---|---|---|
| **Cinco guardados simultáneos morían con `operation timed out`** | Cada `PUT` abría una transacción (una conexión) y **dentro** pedía otra conexión del mismo pool para leer las aperturas. Con `pool.max = 5`, cinco peticiones concurrentes se bloqueaban entre sí: un interbloqueo del pool contra sí mismo. Dos gerentes guardando a la vez bastaba para empezar a verlo | `tieneApertura` recibe una **fábrica de requests**, y dentro de una transacción usa la conexión de esa transacción |
| **Una agencia inventada se guardaba** | La pantalla usa un combo, pero la pantalla no es la autoridad. Un `POST` a mano metía cualquier texto, llegaba a `vw_ObjetivosAPV_Hechos`, y en Power BI no empataba con ningún `S_Homologada`: el objetivo desaparecía del reporte **sin error y sin aviso** | `resolverSucursal()` valida contra el catálogo y guarda el nombre canónico (además arregla la caja: `carranza` → `CARRANZA`) |
| **Un número mayor que `INT` tumbaba el lote completo** | Todo el mes se guarda en **una** transacción. Un solo valor absurdo reventaba al mandar el parámetro y el gerente perdía los otros treinta renglones | Se valida contra `MAX_OBJETIVO` antes de tocar SQL; el renglón malo se rechaza solo |
| **Un nombre de 300 caracteres daba un 500 de protocolo TDS** | El mensaje (`Data type 0xE7 has an invalid data length`) suena a servidor roto, no a dato inválido | Largos validados contra el ancho real de cada columna, con 400 y motivo |
| **La reja mostraba el gerente equivocado tras una rotación** | Mostraba el de la **asignación viva**, no el del objetivo. La pantalla decía "es tuyo" y Power BI lo reportaba del gerente anterior: dos respuestas a la misma pregunta | Se muestra el del objetivo, marcado *(cargado antes de la rotación)* cuando difiere |
| **"Último cambio" podía mostrar el día siguiente** | `created_at` viaja en UTC y se cortaba el ISO a diez caracteres. Una captura a las 7 de la noche se veía con la fecha de mañana | `fecha()` formatea en `America/Mexico_City`; las columnas `DATE`, que ya son fecha civil, se siguen cortando |

Además, el backend ahora **avisa al arrancar si el reloj del servidor no está en hora de
México**. La ventana de captura sale de ese reloj, y un VPS recién instalado suele quedar en
UTC: con eso el último día del mes cerraría seis horas antes, sin dar un solo error.
(Comprobado el 2026-09-08: el VPS está en `-06:00`, correcto.)

### Bordes conocidos que NO se cambiaron

Son decisiones, no descuidos:

- **Un APV cuya asignación se cerró desaparece de la reja del mes**, aunque ya tuviera objetivos
  capturados. La reja se arma de asignaciones vivas. Los objetivos siguen en la base y en las
  vistas de Power BI; lo que se pierde es la vista del gerente sobre lo que ya cargó.
- **No hay forma de desactivar un gerente desde la app.** La columna `activo` existe y se
  respeta, pero el corte de acceso real se hace en la central `APPS`, que es donde vive la
  identidad.
- **Dos gerentes no pueden compartir nombre normalizado** (`UX_gerentes_nombre`). Es necesario
  —Power BI agrupa por ese texto— pero significa que dos "JUAN PEREZ" no pueden coexistir.
- **El mismo renglón repetido dentro de un `PUT`** se guarda dos veces: gana el último y el
  historial registra las dos versiones. Es ruido, no un error.

---

## Lo que NO está verificado

Para no dar por probado lo que no lo está:

- **El rol RH no se ejercitó con una persona real.** El alcance está probado como lógica pura
  (13 casos) y los límites entre roles se comprobaron contra el backend en vivo —un RH no ve
  `/usuarios-rh`, no reabre meses, y sin agencias no alcanza nada—, pero nadie ha entrado como
  RH a mover un equipo de punta a punta.
- **Nada de lo que toca la central se probó contra la central real.** No hay `AUTH_TOKEN` a la
  mano ni forma de correr el `auth-service` contra `APPS` desde aquí. Eso deja sin ejercitar:
  `api/login.js`, el alta de usuarios, el restablecimiento de contraseña y el diálogo que la
  muestra. Lo verificado es la lógica de contraseñas, que el BFF exige el rol correcto, y que el
  diálogo renderiza bien.
- **La sustitución del rol al ascender a alguien no se probó.** Si una persona ya era miembro
  con perfil `user` y se le vuelve a dar de alta como `rh`, la central ahora **actualiza** la
  membresía. Está escrito y razonado, no ejecutado.
- **⚠️ El espejo del VPS no coincide con lo que documentaba el proyecto viejo.**
  `INTELIGENCIA_NEGOCIOS.dbo.SUCURSALES` trae en el VPS **66 renglones / 29 `S_Homologada`**,
  y el README anterior anotaba **82 / 35** medidos en la `10.10.10.12` ese mismo día. O el
  espejo va atrasado, o esa medición ya no era válida. Si va atrasado, **faltan agencias en el
  combo**. Se compara así, con un login que exista en la `.12`:

  ```bash
  sqlcmd -S 10.10.10.12 -U sa -W -Q "SELECT COUNT(*) AS renglones, COUNT(DISTINCT S_Homologada) AS homologadas FROM INTELIGENCIA_NEGOCIOS.dbo.SUCURSALES"
  ```
- **La contraseña del login técnico sigue siendo la de la instalación.** Cámbiala y actualiza
  el `.env` del backend.
- **El modelo de Winston no se tocó.** Las relaciones y medidas de la Opción B están descritas,
  no aplicadas. La relación muchos-a-muchos con `AUXLIAR_SUCURSALES` conviene confirmarla con
  un rol de RLS puesto antes de darla por buena.
