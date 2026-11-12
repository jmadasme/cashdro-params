# Guía del desarrollador — Lectura de parámetros CashDro

Documento técnico de la aplicación WinForms que consulta los parámetros de un parque de
máquinas CashDro. Está pensado para quien deba **mantenerla, corregirla o extenderla**.

> Todo lo aquí descrito se obtuvo por **lectura estática** del código de
> `versions/Lectura-parametros-maquina-Cashdro/`. No se compiló ni se ejecutó contra
> hardware CashDro en esta revisión.

---

## 1. Inventario

| Archivo | Bytes | Rol |
|---|---:|---|
| `cashdro.sln` | 1 474 | Solución Visual Studio 2017 (formato 12) |
| `cashdro.csproj` | 8 394 | Proyecto clásico MSBuild, `WinExe`, net461 |
| `App.config` | 346 | Runtime v4.0 / net461 + `probing privatePath="x86"` |
| `packages.config` | 466 | CefSharp 71.0.2, cef.redist 3.3578.1870, Json.NET 12.0.1 |
| `Program.cs` | 549 | Punto de entrada STA |
| `Form1.cs` | 30 593 | Toda la lógica: login, consultas, persistencia, correo |
| `Form1.Designer.cs` | 25 463 | Layout del formulario principal |
| `Form1.resx` | 97 216 | Recursos del diseñador (iconos heredados, metadatos) |
| `webBrownser.cs` | 2 250 | Formulario con Chromium embebido |
| `webBrownser.Designer.cs` | 3 607 | Layout del visor |
| `webBrownser.resx` | 5 817 | Recursos del visor |
| `dataClass.cs` | 2 134 | DTOs del contrato JSON |
| `funciones.cs` | 2 414 | Utilidad de acceso a datos (sin uso real) |

**Ausentes en el repositorio y referenciados por el `.csproj`**:
`Properties/AssemblyInfo.cs`, `Properties/Resources.resx`,
`Properties/Resources.Designer.cs`, `Properties/Settings.settings`,
`Properties/Settings.Designer.cs` y la carpeta `packages/`. El proyecto **no compila tal
como está**: hay que recrear esos archivos o eliminarlos del `.csproj`.

---

## 2. Modelo de ejecución

```text
Program.Main  [STAThread]
    └── Application.Run(new Form1())
            ├── ctor: CefSettings.IgnoreCertificateErrors = true; Cef.Initialize()
            ├── Form1_Load
            │      ├── LeerParametros()          → SP GetParametros
            │      ├── timer1.Enabled = true     (ya venía true del diseñador)
            │      ├── timer1.Interval = 1800000
            │      └── timer1.Start()
            └── bucle de mensajes WinForms
                   ├── btnValida.Click → ValidaUsuario → cargaGrid
                   ├── dataGridView.CellClick → cargaGridDevice + cargaGridData
                   ├── dataGridViewDevice.CellClick → webBrownser (CEF)
                   └── timer1.Tick (30 min) → ChequeaCashdroBakgroud
```

### 2.1 Un solo hilo

No hay `async`, `Task`, `BackgroundWorker` ni `Thread`. **Todas** las llamadas HTTP y SQL
se ejecutan en el hilo de interfaz. Con N máquinas, un tick del temporizador hace `2N`
peticiones síncronas y la ventana queda sin responder mientras dura.

No hay `HttpClient`: se usa `WebClient.OpenRead`, cuyo `Timeout` por defecto es de 100 s y
no se modifica. Un equipo colgado puede bloquear la UI hasta 100 s por llamada.

### 2.2 Temporizador

`timer1` se crea en el diseñador con `Enabled = true` y sin `Interval` explícito (100 ms
por defecto). En `Form1_Load` se vuelve a habilitar, se fija `Interval = 1_800_000` y se
llama `Start()`. En la práctica el primer tick llega a los 30 minutos porque el cambio de
intervalo reinicia la cuenta antes de que el bucle de mensajes procese nada, pero el
orden es frágil: si `LeerParametros()` lanzara una excepción no controlada y el
`Load` se abortara, el temporizador quedaría con 100 ms sobre el valor del diseñador.

### 2.3 Ciclo de vida de CEF

`Cef.Initialize` se llama en el constructor de `Form1` y **nunca** se llama `Cef.Shutdown`
(las dos invocaciones están comentadas, en `webBrownser.InitializeChromium` y en
`webBrownser_FormClosed`). Consecuencia: los procesos hijos de Chromium pueden quedar
vivos después de cerrar la ventana del visor. En una app de un solo formulario y una sola
ventana modal el efecto es tolerable, pero es la primera cosa a corregir si se agregan
más vistas.

---

## 3. Mapa de dependencias

```text
Program.cs ──► Form1 ──┬──► dataClass.cs      (DTOs JSON)
                       ├──► webBrownser       (CEF)
                       ├──► System.Data.SqlClient
                       ├──► Newtonsoft.Json
                       └──► System.Net.Mail

funciones.cs  (aislado, nadie lo referencia)
```

`Form1` concentra UI, acceso a datos, HTTP, parseo, reglas de negocio y notificación.
`funciones.cs` define `cnLocal` y `GetDataSet`, pero **nadie los usa**: `Form1` tiene su
propio `_cnLocal`. Es código muerto que conviene borrar o, mejor, convertir en la capa de
datos real.

---

## 4. Recorrido por archivo

### 4.1 `Program.cs`

```csharp
[STAThread]
static void Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    Application.Run(new Form1());
}
```

Sin manejo global de excepciones (`Application.ThreadException`,
`AppDomain.UnhandledException`). Cualquier fallo no capturado cierra la aplicación con el
cuadro genérico de .NET.

### 4.2 `Form1.cs` — estado

| Campo | Tipo | Uso |
|---|---|---|
| `n` | `int` | Índice de fila seleccionada; se reutiliza para varias grillas |
| `pathImg` | `string` | `D:\imagenes\` — ruta fija de los iconos |
| `IPActual` | `string` | IP del equipo seleccionado, usada por el visor CEF |
| `_cnLocal` | `string` | Cadena de conexión SQL embebida |
| `_cliente` | `string` | Nombre de cliente, de `GetParametros` |
| `_correoFrom` | `string` | Remitente SMTP |
| `_correoPass` | `string` | Clave SMTP |
| `_correoAlerta` | `string` | Destinatario, leído de `getCasdro` |
| `_alias` | `string` | Alias del equipo en curso |
| `_IP` | `string` | IP del equipo en curso durante el barrido |

Todos son estado mutable de formulario. `_alias`, `_IP` y `_correoAlerta` se
**sobrescriben** en cada iteración del barrido, por lo que una alerta generada en el
`catch` de `cargaTblDevice` usa el valor del equipo que se estaba procesando; no hay
riesgo de mezcla porque el flujo es secuencial, pero sí acoplamiento implícito.

### 4.3 `Form1.cs` — métodos, en orden de aparición

| Líneas | Método | Qué hace | Observación |
|---:|---|---|---|
| 24–31 | `Form1()` | Inicializa CEF con certificados ignorados | No verifica el retorno de `Cef.Initialize` |
| 45–55 | `Form1_Load` | Lee parámetros y arma el timer | `LeerParametros` puede fallar en silencio |
| 59–141 | `cargaGrid()` | Llama `getCasdro` y pinta el parque | Iconos se cargan de disco **por fila** |
| 165–187 | `dataGridView_CellClick` | Selecciona equipo y abre detalle | Si la fila está `ReadOnly`, retorna antes |
| 192–248 | `cargaGridDevice(IP)` | `getDiagnosis` → grilla de dispositivos | **Usa una URL fija, ignora `IP`** |
| 250–294 | `cargaTblDevice(IP)` | `getDiagnosis` → `pInsertDevices` | `catch` dispara correo |
| 296–308 | `InsertaDispositivo(...)` | Abre conexión y ejecuta SP | Una conexión por fila |
| 309–325 | `InsertaData(...)` | Abre conexión y ejecuta SP | Ídem |
| 328–392 | `cargaGridData(IP)` | `getPiecesCurrency` → niveles | Divide `Value` por 100 |
| 394–426 | `cargaTblData(IP)` | `getPiecesCurrency` → `pInsertdata` | `catch` vacío: error invisible |
| 430–459 | `enviaCorreo(IP, device)` | SMTP Gmail 587 | Sin `using` ni timeout |
| 496–542 | `ChequeaCashdroBakgroud()` | Barrido completo del parque | **Limpia la grilla visible** |
| 544–582 | `getError(IP)` | `getDiagnosis` → bool | Devuelve `true` ante excepción |
| 583–606 | `dataGridViewDevice_CellClick` | Abre el visor CEF | URL con credenciales |
| 607–618 | `btnValida_Click` | Login → `cargaGrid` | No hay feedback si falla |
| 627–657 | `ValidaUsuario(u, p)` | `SELECT count(*) FROM [user]` | Clave en texto plano |
| 659–695 | `LeerParametros()` | `GetParametros` | 3 columnas |
| 697–703 | `timer1_Tick` | Llama al barrido | — |
| 720–775 | `cargaGridDevice2()` | Ruta de depuración | Host fijo con puerto; sin `catch` |

### 4.4 `Form1.Designer.cs`

Tres `DataGridView` con columnas de diseñador (`id`, `IP`, `ALIAS`, `ESTADO`;
`IPdevice`, `deviceName`, `DeviceWithError`, `STATUS`; `CURRENCYID`, `VALUE`,
`DEPOSITLEVEL`, `LEVELRECYCLER`, `MAXLEVEL`, `ALERT`), un `GroupBox` "LOGIN", un
`PictureBox` de semáforo global y ocho `PictureBox` decorativos. El formulario mide
1088 × 580 y se centra en pantalla. El `timer1` se declara con `Enabled = true`.

### 4.5 `dataClass.cs`

DTOs planos, sin atributos, sin `JsonProperty`, sin validación:

```csharp
RootObjectDevice { int code; Data data; }
Data             { bool WithError; List<Device> Devices; }
Device           { string deviceName; object deviceType; object deviceState;
                   string portName; int portProtocol; object deviceId;
                   List<object> portMessagesInfo; List<object> deviceMessagesInfo;
                   bool deviceWithError; string deviceModel; string protocolName;
                   string deviceImage; int? portState; string ip; bool? tooManyChanges; }

RootObjectDatum  { int code; List<Datum> data; }
Datum            { string CurrencyId, Value, Type, Destination, MinLevel, MaxLevel,
                   MaxLevelTemp, DepositLevel, DepositLevelTemp, MaxPiecesExchange,
                   MaxPiecesChange, MaxPiecesCancel, IsChargeable, State, Image;
                   object LevelRecycler, LevelCasete; }
```

Notas de diseño:

- `deviceId` y `LevelRecycler` son `object` porque el emisor alterna entre número y
  cadena. Eso obliga a `.ToString()` en todos los usos y traslada la conversión a la UI.
- `Datum` mezcla campos usados y no usados; no hay separación entre contrato y modelo.
- Si el emisor agrega campos, Json.NET los ignora sin error (comportamiento por
  defecto). Si **quita** o cambia el tipo de uno usado, la app falla en runtime.
- `code` se deserializa y **nunca se consulta**.

### 4.6 `funciones.cs`

```csharp
static class funciones
{
    public static string cnLocal  = @"...";   // no usado
    public static string cnRemota = @"";      // no usado
    static public DataSet GetDataSet(string ConnectionString, string SQL) { ... }
}
```

`GetDataSet` no cierra la conexión si `da.Fill` lanza excepción (no hay `try/finally` ni
`using`). No lo llama nadie, así que el defecto es latente.

### 4.7 `webBrownser.cs` / `.Designer.cs`

Formulario que hereda de `DevComponents.DotNetBar.OfficeForm`, título "Diagnosis".
Contiene tres contenedores (`pContainer`, `flowLayoutPanel1`, `panel1`) que **no se usan**:
el `ChromiumWebBrowser` se agrega directamente a `this.Controls` con `Dock = Fill`, por lo
que cubre los paneles. Tiene dos constructores: sin argumentos (URL por defecto
`http://www.google.cl`) y con URL. El evento `AddressChanged` solo ejecuta un
`this.Invoke` con el cuerpo comentado: es un residuo de una barra de direcciones que ya no
existe.

---

## 5. Contrato con Cashdro3WS

### 5.1 Construcción de la petición

```csharp
string url = "https://" + IP + "/Cashdro3WS/index.php?operation=getDiagnosis";
Uri address = new Uri(url);
ServicePointManager.ServerCertificateValidationCallback = delegate { return true; };
using (WebClient webClient = new WebClient())
{
    var stream = webClient.OpenRead(address);
    using (var sr = new StreamReader(stream))
    {
        var page = sr.ReadToEnd();
        RootObjectDevice newTarget = JsonConvert.DeserializeObject<RootObjectDevice>(page);
    }
}
```

Detalles relevantes:

- **GET** con todos los parámetros en la query string, incluidas las credenciales.
- El callback de certificado se reasigna en **cada** método (8 veces). Es global y
  afecta a todo el proceso, incluido el tráfico SMTP y cualquier otra conexión TLS.
- `getPiecesCurrency` se invoca con `currencyId=CLP`, `includeImages=0` y
  `includeLevels=1`. El parámetro llega como `&%20currencyId=CLP`: un espacio codificado
  de más, inofensivo pero sintomático de una URL armada a mano.
- No se envía `User-Agent` ni `Accept`; `WebClient` usa los valores por defecto.

### 5.2 Operaciones usadas

| Operación | Método | Parámetros | Respuesta |
|---|---|---|---|
| `getDiagnosis` | GET | ninguno | `RootObjectDevice` |
| `getPiecesCurrency` | GET | `currencyId`, `includeImages`, `includeLevels`, `name`, `password` | `RootObjectDatum` |

`getDiagnosis` no recibe credenciales en el código. `getPiecesCurrency` sí. Si el
servidor exigiera autenticación en ambos, el primero fallaría; conviene verificarlo en
sitio.

### 5.3 Mapeo a la interfaz

| Celda | Expresión | Conversión |
|---|---|---|
| Dispositivos · Item | `d.deviceId.ToString()` | directa |
| Dispositivos · Nombre | `d.deviceName.ToString()` | directa |
| Dispositivos · Error ? | `d.deviceWithError.ToString()` | comparación con `"False"` |
| Niveles · Moneda | `d.CurrencyId.ToString()` | directa |
| Niveles · Denominación | `Convert.ToUInt32(d.Value) / 100` | centavos → pesos |
| Niveles · Fianza | `d.DepositLevel.ToString()` | directa |
| Niveles · Total | `d.LevelRecycler.ToString()` | directa |
| Niveles · Recuento | `d.MaxLevel.ToString()` | directa |
| Niveles · ALERT | `Convert.ToInt32(LevelRecycler) < Convert.ToInt32(DepositLevel)` | entero |

`Convert.ToUInt32` / `Convert.ToInt32` usan la cultura actual. Con separadores de miles
en la respuesta (`"1.000"`) el parseo fallaría; el contrato observado entrega enteros
simples.

### 5.4 Estado derivado del equipo

```csharp
bool getError(string IP) {
    try { ... return tieneError; }   // true si algún deviceWithError != "False"
    catch { return true; }           // falla cerrado
}
```

El "semáforo" del parque es, entonces, una conjunción: *hay error* si el endpoint falla
**o** si algún dispositivo lo reporta. Es una decisión correcta para monitoreo, pero
impide distinguir "máquina caída" de "máquina con falla" en la interfaz.

---

## 6. Contrato con SQL Server

### 6.1 Objetos

| Objeto | Llamada | Parámetros | Columnas leídas |
|---|---|---|---|
| `GetParametros` | `CommandType.StoredProcedure` | — | `CorreoFrom`, `CorreoPass`, `Cliente` |
| `getCasdro` | ídem | — | `id`, `ip`, `alias`, `correoAlerta` |
| `[user]` | SQL en línea | `@usr`, `@pwd` | `count(*)` |
| `pInsertDevices` | SQL con parámetros | `@cliente`, `@deviceName`, `@deviceWithError`, `@IP` | — |
| `pInsertdata` | ídem | `@cliente`, `@currencyId`, `@Value`, `@depositLevel`, `@LevelRecycler`, `@IP`, `@MaxLevel` | — |

### 6.2 Esquema mínimo para reconstruir la base

No se incluye DDL en el repositorio. Para levantar un entorno de pruebas sirve algo así
(ajusta tipos y nombres a tu instalación):

```sql
CREATE TABLE [user] (
    usuario VARCHAR(50) NOT NULL PRIMARY KEY,
    pass    VARCHAR(50) NOT NULL
);

CREATE TABLE Parametros (
    CorreoFrom VARCHAR(120),
    CorreoPass VARCHAR(120),
    Cliente    VARCHAR(80)
);

CREATE TABLE Maquinas (
    id           INT IDENTITY PRIMARY KEY,
    ip           VARCHAR(45),
    alias        VARCHAR(80),
    correoAlerta VARCHAR(120)
);

CREATE TABLE Devices (
    id              BIGINT IDENTITY PRIMARY KEY,
    cliente         VARCHAR(80),
    deviceName      VARCHAR(80),
    deviceWithError VARCHAR(10),
    ip              VARCHAR(45),
    leido           DATETIME DEFAULT GETDATE()
);

CREATE TABLE Niveles (
    id            BIGINT IDENTITY PRIMARY KEY,
    cliente       VARCHAR(80),
    currencyId    VARCHAR(10),
    Value         VARCHAR(20),
    depositLevel  VARCHAR(20),
    levelRecycler VARCHAR(20),
    maxLevel      VARCHAR(20),
    ip            VARCHAR(45),
    leido         DATETIME DEFAULT GETDATE()
);
```

### 6.3 Comportamiento transaccional

- `getCasdro` y `GetParametros` fijan `CommandTimeout = 5`; las inserciones no.
- Cada inserción abre y cierra su propia conexión: en un parque de 20 máquinas con 8
  denominaciones y 6 dispositivos, un tick genera 280 aperturas de conexión.
- No hay transacción que agrupe el tick: si falla a mitad, el histórico queda parcial.
- No hay `try/finally` alrededor de las conexiones de inserción: una excepción deja la
  conexión abierta hasta que el GC la finalice.

---

## 7. API de funciones

### 7.1 `Form1`

| Firma | Visibilidad | Descripción | Fallas |
|---|---|---|---|
| `Form1()` | public | Inicializa controles y CEF | Excepción si CEF no puede inicializar |
| `void LeerParametros()` | public | Carga correo y cliente desde SQL | Silenciosa: `catch` vacío |
| `bool getError(string IP)` | private | `true` si hay falla o no responde | Nunca lanza (captura todo) |
| `bool ValidaUsuario(string u, string p)` | private | Valida contra `[user]` | Lanza si SQL no responde |
| `void cargaGrid()` | private | Pinta el parque | Silenciosa |
| `void cargaGridDevice(string IP)` | private | Pinta dispositivos | `IP` ignorado |
| `void cargaGridData(string IP)` | private | Pinta niveles | Lanza si hay valores no numéricos |
| `void cargaTblDevice(string IP)` | private | Persiste dispositivos | Dispara correo en `catch` |
| `void cargaTblData(string IP)` | private | Persiste niveles | `catch` vacío |
| `void ChequeaCashdroBakgroud()` | private | Barrido completo | Silenciosa; limpia la grilla |
| `void enviaCorreo(string IP, string dev)` | private | Envía alerta SMTP | Muestra `MessageBox` con el error |
| `void InsertaDispositivo(...)` | private | Inserta un dispositivo | Lanza |
| `void InsertaData(...)` | private | Inserta un nivel | Lanza |

### 7.2 `webBrownser`

| Firma | Descripción |
|---|---|
| `webBrownser()` | Usa `http://www.google.cl` como URL |
| `webBrownser(string url)` | Carga la URL indicada |
| `void InitializeChromium()` | Crea el `ChromiumWebBrowser` y lo acopla al formulario |

### 7.3 `funciones`

| Firma | Estado |
|---|---|
| `static DataSet GetDataSet(string ConnectionString, string SQL)` | Sin uso; sin cierre garantizado |

---

## 8. Deuda técnica priorizada

| Prioridad | Hallazgo | Evidencia | Impacto |
|---|---|---|---|
| **P0** | `cargaGridDevice` ignora el parámetro `IP` | URL literal en el método | El detalle siempre muestra el mismo equipo |
| **P0** | No se valida `code` de la respuesta | `newTarget.code` nunca se lee | Se pinta una grilla vacía como si fuera éxito |
| **P0** | Credenciales SQL embebidas | `_cnLocal`, `funciones.cnLocal` | Cualquiera con el binario accede a la base |
| **P1** | Barrido limpia la grilla visible | `dataGridView.Rows.Clear()` en `ChequeaCashdroBakgroud` | A los 30 min el operador ve el parque vacío |
| **P1** | Bloqueo del hilo de UI | `WebClient.OpenRead` sin `async` | Ventana congelada durante el barrido |
| **P1** | TLS sin validar | `ServerCertificateValidationCallback` + CEF | Vulnerable a intercepción en la LAN |
| **P1** | Credenciales en URL | `getPiecesCurrency`, visor CEF | Quedan en logs de proxy e historial |
| **P2** | `catch` vacíos o que solo cierran conexión | 6 bloques | Fallas invisibles, difícil diagnóstico |
| **P2** | Sin timeouts ni reintentos | `WebClient` por defecto | Cortes de red cuelgan 100 s por equipo |
| **P2** | Iconos desde `D:\imagenes\` | `pathImg` constante | Falla en cualquier equipo con otra estructura |
| **P2** | Login con clave en texto plano | `pass=@pwd` | Compromiso trivial si se filtra la base |
| **P3** | Código muerto | `funciones.cs`, `button1`, `pictureBox1..8` | Ruido de mantenimiento |
| **P3** | `Bitmap` sin `Dispose` | `new Bitmap(...)` por fila | Presión de memoria en parques grandes |
| **P3** | CEF sin `Shutdown` | llamadas comentadas | Procesos huérfanos |
| **P3** | Moneda fija `CLP` | literal en la URL | No reutilizable en otras geografías |

---

## 9. Guía de extensión

### 9.1 Externalizar la configuración (empezar por aquí)

Mueve cadena de conexión, ruta de imágenes, moneda, intervalo y credenciales a
`App.config`:

```xml
<appSettings>
  <add key="Sql.ConnectionString" value="Data Source=.\SQLEXPRESS;Initial Catalog=GestionCashdro;User Id=USUARIO_CASHDRO;Password=CLAVE_CASHDRO"/>
  <add key="Imagenes.Path"        value="C:\CashDro\img\"/>
  <add key="CashDro.Currency"     value="CLP"/>
  <add key="Monitoreo.Minutos"    value="30"/>
  <add key="Smtp.Host"            value="smtp.example.com"/>
  <add key="Smtp.Port"            value="587"/>
</app.config>
```

Y una clase de configuración que falle al arrancar si falta un valor, en vez de fallar
silenciosamente a mitad de la operación.

### 9.2 Corregir la consulta de detalle

```csharp
void cargaGridDevice(string IP)
{
    string url = "https://" + IP + "/Cashdro3WS/index.php?operation=getDiagnosis";
    // resto igual
}
```

Y añadir `Uri.EscapeDataString` para cualquier parámetro dinámico.

### 9.3 Validar el código de respuesta

```csharp
var resp = JsonConvert.DeserializeObject<RootObjectDevice>(page);
if (resp == null || resp.code != 0 || resp.data?.Devices == null)
    throw new InvalidOperationException($"CashDro devolvió code={resp?.code}");
```

Conviene encapsularlo en un helper `CashDroClient` que devuelva un resultado tipado
(`Ok`, `CodigoError`, `Mensaje`) en lugar de lanzar.

### 9.4 Volver asíncrono el acceso

```csharp
private static readonly HttpClient Http = new HttpClient(
    new HttpClientHandler { /* validación de certificado explícita */ })
{ Timeout = TimeSpan.FromSeconds(10) };

private async Task<RootObjectDatum> GetPiecesAsync(string ip, string currency)
{
    string url = $"https://{ip}/Cashdro3WS/index.php?operation=getPiecesCurrency" +
                 $"&currencyId={Uri.EscapeDataString(currency)}" +
                 "&includeImages=0&includeLevels=1" +
                 $"&name={Uri.EscapeDataString(_usuario)}" +
                 $"&password={Uri.EscapeDataString(_clave)}";
    string json = await Http.GetStringAsync(url).ConfigureAwait(false);
    return JsonConvert.DeserializeObject<RootObjectDatum>(json);
}
```

Al pasar a `async`, el tick del temporizador debe volverse `async void` con guarda de
reentrada (`if (_barridoEnCurso) return;`) y el barrido debe usar
`Task.WhenAll` con un límite de concurrencia (`SemaphoreSlim`) para no saturar la LAN.

### 9.5 Registrar en lugar de tragar

```csharp
catch (Exception ex)
{
    Log.Error($"getDiagnosis {ip}", ex);
    return true;   // mantiene el criterio de falla cerrada
}
```

Con `System.Diagnostics.TraceSource` o Serilog. Regla práctica: **ningún `catch` vacío**.

### 9.6 Agregar una moneda

1. Reemplazar el literal `CLP` por configuración.
2. Iterar las monedas soportadas y concatenar resultados, o consultar sin `currencyId`
   si el endpoint lo permite (verificar en sitio).
3. Añadir la moneda a la clave de la tabla de niveles si se quiere histórico separado.

### 9.7 Añadir una operación nueva del WS

1. Crear el DTO en `dataClass.cs` (o en una carpeta `Contracts/`).
2. Añadir el método al cliente HTTP con su `operation=`.
3. Mapear a la grilla o a un nuevo panel.
4. Si se persiste, crear el SP y documentar sus parámetros en este archivo.

### 9.8 Pruebas

No hay proyecto de tests. El mínimo viable:

| Nivel | Qué probar | Cómo |
|---|---|---|
| Unitario | DTOs con JSON de ejemplo (válido, vacío, campo faltante, tipo cambiado) | xUnit + archivos `.json` de fixture |
| Unitario | Regla `LevelRecycler < DepositLevel` | Extraer a función pura |
| Integración | `getDiagnosis` contra un equipo de laboratorio | Marcar como `[Trait("Category","Hardware")]` |
| Integración | SP de inserción con base vacía | Script de creación + rollback |
| Humo | Arranque sin SQL y sin red | Verifica que no haya cierre inesperado |

Para poder testear hay que **extraer la lógica del formulario**: hoy todo está en
`Form1` y depende de controles concretos.

### 9.9 Migración recomendada

| Etapa | Acción | Riesgo |
|---|---|---|
| 1 | Externalizar configuración y agregar logging | Bajo |
| 2 | Extraer `CashDroClient` (HTTP + DTO) y `CashDroRepository` (SQL) | Medio |
| 3 | Pasar a `async` con cancelación | Medio |
| 4 | Migrar a .NET 8 (`net8.0-windows`) y CefSharp reciente | Alto: CEF, DotNetBar, `System.Data.SqlClient` |
| 5 | Reemplazar `System.Data.SqlClient` por `Microsoft.Data.SqlClient` | Bajo tras la etapa 2 |

CefSharp 71 corresponde a Chromium 71 (2018). En .NET moderno usa la rama actual y
verifica `CefSharpAnyCpuSupport` o fija `x86`/`x64`.

---

## 10. Compilación y depuración

### 10.1 Preparar el árbol

```powershell
# Estructura esperada por los HintPath del .csproj
<raiz>\
  packages\                 <- restaurar con NuGet
  cashdro\                  <- carpeta del proyecto (esta versión)
  cashdro.sln
```

El `.sln` referencia `cashdro\cashdro.csproj`; el proyecto referencia
`..\packages\...`. Si abres la carpeta tal como viene en `versions/`, la solución no
encontrará el proyecto: hay que reproducir esa disposición o ajustar las rutas.

### 10.2 Compilar

```powershell
nuget restore cashdro.sln
msbuild cashdro.sln /p:Configuration=Release /p:Platform="Any CPU"
```

En Visual Studio: **Build → Configuration Manager** y elige `Any CPU` (con
`CefSharpAnyCpuSupport=true`) o `x86` si el puesto es de 32 bits. Mezclar arquitecturas
entre `libcef.dll` y el proceso es la causa clásica de `BadImageFormatException`.

### 10.3 Errores de compilación frecuentes

| Error | Causa | Solución |
|---|---|---|
| `The type or namespace name 'CefSharp' could not be found` | Paquetes no restaurados | `nuget restore` |
| `The type or namespace name 'DevComponents' could not be found` | Falta DotNetBar | Instalar componente o cambiar clase base |
| `Source file 'Properties\AssemblyInfo.cs' could not be found` | Archivo ausente | Recrearlo o quitarlo del `.csproj` |
| `The missing file is ..\packages\cef.redist...props` | Restauración incompleta | Borrar `obj\`, restaurar de nuevo |
| `CS0246 StreamReader` | Falta `using System.IO` | Ya está en el código; verificar `Form1.cs` íntegro |

### 10.4 Errores en ejecución frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| Cierre inmediato sin mensaje | `Cef.Initialize` falló | Copiar `libcef.dll`, `*.pak`, `locales\`; instalar VC++ |
| `Could not load file or assembly 'CefSharp.Core'` | Arquitectura cruzada | Igualar `Platform` con el redist |
| Ventana del visor en blanco | Certificado o DNS | Probar la URL en un navegador normal |
| Grillas vacías | SQL o JSON inválido | Registrar `page` crudo |
| Excepción en `new Bitmap` | Falta `D:\imagenes\` | Crear carpeta y copiar iconos |

### 10.5 Depuración del contrato

Para ver qué devuelve realmente el equipo, sin recompilar:

```powershell
# Guarda la respuesta cruda para inspección
Invoke-WebRequest -Uri "https://<IP>/Cashdro3WS/index.php?operation=getDiagnosis" `
                  -SkipCertificateCheck -OutFile diag.json
```

`-SkipCertificateCheck` requiere PowerShell 6+. En Windows PowerShell 5.1 hay que
desactivar la validación con `[ServerCertificateValidationCallback]` o usar `curl -k`.
**Nunca** ejecutes esto contra producción sin autorización: la URL de niveles lleva
credenciales.

---

## 11. Convenciones del código existente

- Nombres de métodos en español (`cargaGrid`, `enviaCorreo`), campos con prefijo `_`.
- Mezcla de PascalCase y camelCase en miembros públicos (`CurrencyId` vs `deviceName`).
- Ausencia de comentarios de contrato; los existentes son notas de trabajo.
- Todo en un solo archivo de 800 líneas: al modificar, **no agregues más lógica ahí**.
  Crea clases nuevas y deja `Form1` como orquestador delgado.
- Los literales de URL y credenciales están repetidos; cualquier cambio debe hacerse en
  todos los puntos (busca `Cashdro3WS` y `Cashdro3Web`).

---

## 12. Checklist antes de un cambio

- [ ] ¿La ruta de imágenes sigue siendo `D:\imagenes\` o la externalizaste?
- [ ] ¿La nueva llamada HTTP valida `code` y maneja timeout?
- [ ] ¿El `catch` registra algo?
- [ ] ¿Cerraste conexiones y `Dispose` de bitmaps?
- [ ] ¿Probaste con una máquina apagada (falla cerrada) y con una sin dispositivos?
- [ ] ¿Probaste con respuesta JSON vacía, con `LevelRecycler` nulo y con `Value` no numérico?
- [ ] ¿Actualizaste `docs/datasheet.md` si cambió la interfaz o el contrato?
- [ ] ¿Sigue compilando en `Any CPU` **y** en `x86`?

---

## 13. Referencias internas

- Contrato y campos: [`datasheet.md`](datasheet.md)
- Fuentes externas y versiones: [`REFERENCES.md`](REFERENCES.md)
- Diagramas: `assets/arquitectura.svg`, `assets/flujo-sesion.svg`,
  `assets/modelo-niveles.svg`
