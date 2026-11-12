# Datasheet — Lectura de parámetros CashDro

| Campo | Valor |
|---|---|
| Producto | Lectura de parámetros máquina CashDro |
| Tipo | Aplicación de escritorio Windows (WinForms) |
| Versión documentada | `versions/Lectura-parametros-maquina-Cashdro` |
| Plataforma | Windows 7 SP1 o superior · .NET Framework 4.6.1 |
| Autor | José Manuel Adasme (jmadasme) |
| Licencia | MIT (ver `LICENSE`) |
| Estado de esta revisión | Documentación estática. **No se ejecutó contra hardware CashDro real.** |

---

## 1. Propósito

La aplicación concentra en una sola pantalla la lectura de dos familias de datos de un
parque de máquinas CashDro:

1. **Parámetros técnicos**: qué dispositivos internos reportan falla (`getDiagnosis`).
2. **Parámetros financieros**: cuántas piezas hay por denominación, cuál es el mínimo
   tolerado y cuál es la capacidad máxima de cada recirculador (`getPiecesCurrency`).

Además, un temporizador interno recorre el parque cada 30 minutos y persiste el
resultado en una base SQL Server (`GestionCashdro`), de modo que el estado quede
registrado aunque nadie esté mirando la pantalla.

### 1.1 Qué resuelve

| Problema operativo | Cómo lo resuelve |
|---|---|
| Saber si una máquina tiene un dispositivo en falla sin ir al sitio | Semáforo por equipo en la grilla del parque |
| Detectar que un recirculador quedó bajo el mínimo | Comparación `LevelRecycler` vs `DepositLevel` por denominación |
| Revisar el detalle que muestra la propia máquina | Navegador Chromium embebido sobre `Cashdro3Web` |
| Guardar histórico para análisis posterior | Inserción periódica vía procedimientos almacenados |
| Evitar revisar equipo por equipo a mano | Recorrido automático cada 30 minutos |

### 1.2 Qué **no** resuelve

- No configura ni modifica parámetros de la máquina: es **solo lectura**.
- No reemplaza la consola de administración del fabricante.
- No opera el ciclo de recaudación, carga ni retiro de efectivo.
- No autentica contra CashDro de forma persistente: las credenciales se envían en
  cada petición, tomadas de un campo de texto en memoria.
- No es multi-moneda: la consulta de niveles está fijada a `CLP`.

---

## 2. Requisitos

### 2.1 Hardware

| Componente | Mínimo | Recomendado |
|---|---|---|
| CPU | x86/x64 de dos núcleos | Cuatro núcleos |
| RAM | 2 GB | 4 GB o más (CefSharp es exigente) |
| Disco | 300 MB libres | 1 GB (CEF + caché de perfiles) |
| Pantalla | 1024 × 768 | 1366 × 768 o superior (el formulario mide 1088 × 580) |
| Red | Acceso TCP/443 a cada equipo CashDro y TCP/1433 al SQL Server | Enlace estable dentro de la LAN |

### 2.2 Software

| Requisito | Detalle |
|---|---|
| Sistema operativo | Windows (la aplicación es **Windows-only**; usa WinForms, CEF y `System.Data.SqlClient`) |
| .NET Framework | 4.6.1 (declarado en `App.config` y en el `.csproj`) |
| Visual C++ Runtime | Requerido por las bibliotecas nativas de CEF (`libcef.dll`) |
| SQL Server | Instancia accesible con la base `GestionCashdro` y los objetos listados en §7 |
| Certificado TLS | El equipo CashDro usa certificado autofirmado; la aplicación lo acepta sin validar |

### 2.3 Cuentas y permisos

| Recurso | Necesidad |
|---|---|
| SQL Server | Usuario con permiso de lectura sobre `[user]` y ejecución de `GetParametros`, `getCasdro`, `pInsertDevices`, `pInsertdata` |
| CashDro | Usuario y clave válidos para `Cashdro3WS` y `Cashdro3Web` |
| Correo (opcional) | Cuenta SMTP con contraseña de aplicación si se habilita el envío de alertas |

> Las credenciales reales **no** se distribuyen con este repositorio. En el código
> publicado aparecen como `USUARIO_CASHDRO` y `CLAVE_CASHDRO`; debes reemplazarlas por
> las de tu instalación antes de compilar.

---

## 3. Instalación

### 3.1 Desde código fuente

```text
1. Abrir cashdro.sln con Visual Studio 2017 (o superior con soporte .NET Framework).
2. Restaurar paquetes NuGet (clic derecho en la solución → Restore NuGet Packages).
3. Verificar que existe ..\packages\ (CefSharp, cef.redist, Newtonsoft.Json).
4. Verificar la referencia a DevComponents.DotNetBar2 (v14.1.0.18) — ver §3.3.
5. Compilar en Release (Any CPU o x86).
6. Copiar la carpeta bin\Release\ completa al puesto de operación.
```

### 3.2 Archivos que el binario necesita en tiempo de ejecución

| Recurso | Ubicación esperada | Si falta |
|---|---|---|
| `warning.png` | `D:\imagenes\warning.png` | Excepción al pintar el semáforo de error |
| `check-icon.png` | `D:\imagenes\check-icon.png` | Excepción al pintar el semáforo correcto |
| `libcef.dll` + recursos CEF | junto al `.exe` (los copia NuGet) | El formulario del navegador no abre |
| `Newtonsoft.Json.dll` | junto al `.exe` | Falla el parseo de todas las respuestas |

> La ruta `D:\imagenes\` está fija en el código (`pathImg`). No es configurable sin
> recompilar. Es la causa más frecuente de "la grilla sale vacía" en una instalación nueva.

### 3.3 Dependencia no NuGet: DevComponents DotNetBar

El formulario `webBrownser` hereda de `DevComponents.DotNetBar.OfficeForm`. El
`.csproj` referencia `DevComponents.DotNetBar2, Version=14.1.0.18` **sin `HintPath`**,
por lo que la DLL debe estar instalada en la máquina de compilación (GAC o copia local
+ referencia manual). Sin ella el proyecto no compila, aunque el resto del código esté
correcto. Si no cuentas con licencia del componente, la alternativa es cambiar la clase
base de `webBrownser` a `System.Windows.Forms.Form` y eliminar el `using`
correspondiente.

### 3.4 Primer arranque

Al iniciar, la aplicación:

1. Inicializa el subsistema CEF (Chromium) con `IgnoreCertificateErrors = true`.
2. Ejecuta el procedimiento `GetParametros` para cargar correo remitente, clave de
   correo y nombre de cliente.
3. Habilita el temporizador de 30 minutos.

Si la base de datos no responde, la aplicación **no se cierra**: el error se traga en
silencio y la pantalla queda operativa pero sin parámetros. El login tampoco funcionará,
porque valida contra la misma base.

---

## 4. La pantalla principal

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│  Sistema integrado                                                           │
│                                                                              │
│  ┌─ LOGIN ───────────┐   Parque de maquinas        Estado tecnico            │
│  │ Usuario [_______] │   ┌───────────────────────┐  ┌─────────────────────┐   │
│  │ Clave   [*******] │   │Item│IP Equipo│Nombre │  │Item│Nombre│Error?│ST│   │
│  │        [validar]  │   │ 1  │ ...     │ ...   │  │ 1  │ ...  │False │✓ │   │
│  └───────────────────┘   │ 2  │ ...     │ ...   │  │ 2  │ ...  │True  │! │   │
│                          │ 3  │ ...     │ ...   │  └─────────────────────┘   │
│                          └───────────────────────┘                          │
│                                                        Consulta Niveles      │
│                          [semáforo global]             On Line                │
│                                                        ┌───────────────────┐ │
│                                                        │Moneda│Denom│Fianza│ │
│                                                        │ CLP  │ 100 │  20  │ │
│                                                        │ CLP  │ 500 │  10  │ │
│                                                        └───────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.1 Controles

| Control | Tipo | Función |
|---|---|---|
| `txtUser` | TextBox | Usuario local de la aplicación |
| `txtpass` | TextBox (`PasswordChar='*'`) | Clave local |
| `btnValida` | Button "validar" | Valida contra `[user]` y carga el parque |
| `dataGridView` | DataGridView | Parque de máquinas (Item, IP Equipo, Nombre, ESTADO) |
| `dataGridViewDevice` | DataGridView | Dispositivos de la máquina elegida |
| `dataGridViewdata` | DataGridView | Niveles por denominación |
| `picOk` | PictureBox | Semáforo agregado del parque |
| `lblMensaje` | Label | Mensaje "Conectando con …" |
| `timer1` | Timer | Ciclo automático de 30 minutos |
| `button1` | Button (sin texto útil) | Ruta de depuración con host fijo; **no usar en producción** |

---

## 5. Operación paso a paso

### Paso 1 — Iniciar sesión

Escribe el usuario y la clave **locales** (los de la tabla `[user]`, no los de CashDro)
y pulsa **validar**.

- Si la combinación existe: se carga el parque de máquinas.
- Si no: aparece `YOU ARE NOT GRANTED WITH ACCESS` y ambos campos se limpian.

> Este login es una compuerta de interfaz, no un control de seguridad robusto: la
> conexión a SQL viaja con credenciales embebidas en el binario y la comparación de
> clave es en texto plano.

### Paso 2 — Leer el parque

Cada fila muestra `Item`, `IP Equipo`, `Nombre` (alias) y un icono de estado:

| Icono | Significado |
|---|---|
| ✓ (`check-icon.png`) | Todos los dispositivos reportan `deviceWithError = false` |
| ⚠ (`warning.png`) | Al menos un dispositivo reporta error, **o** la máquina no respondió |

Cuando una máquina no responde, `getError()` devuelve `true` (criterio conservador): se
marca en rojo en lugar de asumir que está sana.

### Paso 3 — Abrir el detalle

Haz **clic sobre la celda de estado** (columna ESTADO) de la fila deseada. Se cargan dos
grillas:

- **Estado técnico** — un renglón por dispositivo.
- **Consulta Niveles On Line** — un renglón por denominación.

> Limitación real: si la máquina está marcada con ⚠, la celda queda en modo solo
> lectura y el clic **no** abre el detalle. Para inspeccionar una máquina en falla hay
> que usar la consola web del fabricante.

### Paso 4 — Interpretar las grillas

**Estado técnico**

| Columna | Origen | Lectura |
|---|---|---|
| Item | `deviceId` | Identificador del dispositivo |
| Nombre | `deviceName` | Nombre lógico (lector, recirculador, impresora…) |
| Error ? | `deviceWithError` | `True` / `False` |
| STATUS | calculado | Semáforo |

**Niveles**

| Columna | Origen | Lectura |
|---|---|---|
| Moneda | `CurrencyId` | Siempre `CLP` en esta versión |
| Denominación | `Value / 100` | Valor facial en pesos |
| Fianza | `DepositLevel` | Mínimo tolerado antes de alertar |
| Total | `LevelRecycler` | Piezas presentes ahora |
| Recuento | `MaxLevel` | Capacidad máxima declarada |
| ALERT | calculado | ⚠ si `Total < Fianza` |

### Paso 5 — Abrir la consola del fabricante

Doble clic (en realidad, clic en la columna STATUS de la grilla de dispositivos) abre
una ventana con Chromium embebido apuntando a:

```text
https://<IP_EQUIPO>/Cashdro3Web/#/diagnosis/false?username=USUARIO_CASHDRO&password=CLAVE_CASHDRO
```

Esa ventana es la interfaz web propia de CashDro; permite ver el detalle que la API no
expone en `getDiagnosis`.

### Paso 6 — Dejar correr el monitoreo

No hay que hacer nada: cada 30 minutos el temporizador recorre **todas** las máquinas
del parque y escribe en SQL Server. Durante ese recorrido la interfaz puede quedarse
congelada unos segundos por máquina (las llamadas son síncronas en el hilo de UI).

---

## 6. Diccionario de parámetros

### 6.1 Diagnóstico (`getDiagnosis`)

| Campo JSON | Tipo | Uso en la aplicación |
|---|---|---|
| `code` | int | **Ignorado** (no se valida) |
| `data.WithError` | bool | No se usa |
| `data.Devices[].deviceId` | object | Columna Item |
| `data.Devices[].deviceName` | string | Columna Nombre |
| `data.Devices[].deviceWithError` | bool | Columna Error? y semáforo |
| `data.Devices[].deviceType` | object | No se usa |
| `data.Devices[].deviceState` | object | No se usa |
| `data.Devices[].portName` | string | No se usa |
| `data.Devices[].portProtocol` | int | No se usa |
| `data.Devices[].deviceModel` | string | No se usa |
| `data.Devices[].portState` | int? | No se usa |
| `data.Devices[].ip` | string | No se usa |
| `data.Devices[].tooManyChanges` | bool? | No se usa |
| `data.Devices[].deviceMessagesInfo` | array | No se usa |
| `data.Devices[].portMessagesInfo` | array | No se usa |

### 6.2 Niveles (`getPiecesCurrency`)

| Campo JSON | Tipo | Uso en la aplicación |
|---|---|---|
| `code` | int | **Ignorado** |
| `data[].CurrencyId` | string | Columna Moneda |
| `data[].Value` | string | Columna Denominación (÷100) |
| `data[].DepositLevel` | string | Columna Fianza |
| `data[].LevelRecycler` | object | Columna Total |
| `data[].MaxLevel` | string | Columna Recuento |
| `data[].MinLevel` | string | No se usa |
| `data[].MaxLevelTemp` | string | No se usa |
| `data[].DepositLevelTemp` | string | No se usa |
| `data[].Type` | string | No se usa |
| `data[].Destination` | string | No se usa |
| `data[].MaxPiecesExchange` | string | No se usa |
| `data[].MaxPiecesChange` | string | No se usa |
| `data[].MaxPiecesCancel` | string | No se usa |
| `data[].IsChargeable` | string | No se usa |
| `data[].State` | string | No se usa |
| `data[].Image` | string | No se usa (`includeImages=0`) |
| `data[].LevelCasete` | object | No se usa |

---

## 7. Objetos de base de datos esperados

| Objeto | Tipo | Entradas | Salidas / efectos |
|---|---|---|---|
| `GetParametros` | SP | — | `CorreoFrom`, `CorreoPass`, `Cliente` |
| `getCasdro` | SP | — | `id`, `ip`, `alias`, `correoAlerta` |
| `[user]` | Tabla | `usuario`, `pass` | Conteo para validar login |
| `pInsertDevices` | SP | `@cliente`, `@deviceName`, `@deviceWithError`, `@IP` | Inserta estado de dispositivo |
| `pInsertdata` | SP | `@cliente`, `@currencyId`, `@Value`, `@depositLevel`, `@LevelRecycler`, `@IP`, `@MaxLevel` | Inserta nivel por denominación |

> `[user]` va entre corchetes porque `USER` es palabra reservada en T-SQL.

La base `GestionCashdro` **no** se incluye en el repositorio. Debes crearla o restaurarla
desde tu respaldo corporativo antes de usar la aplicación.

---

## 8. Mensajes y códigos de error

La aplicación no define códigos propios. Los escenarios observables son:

| # | Situación | Síntoma visible | Causa probable | Acción |
|---|---|---|---|---|
| E01 | Sin conexión a SQL | Nada se carga; el login siempre falla | Instancia apagada, nombre mal escrito, firewall | Verificar `Data Source` y TCP/1433 |
| E02 | Login inválido | `YOU ARE NOT GRANTED WITH ACCESS` | Usuario/clave fuera de `[user]` | Revisar tabla; la comparación es exacta |
| E03 | Máquina inaccesible | Fila con ⚠ aunque la máquina esté sana | `getError()` devuelve `true` ante cualquier excepción | Comprobar ping y TCP/443 |
| E04 | Certificado TLS no confiable | Conecta igual | La app acepta cualquier certificado | Ninguna (por diseño) |
| E05 | Iconos ausentes | Excepción al pintar filas | Falta `D:\imagenes\*.png` | Crear carpeta y copiar los PNG |
| E06 | JSON inesperado | Grilla vacía o excepción no controlada | El endpoint devolvió HTML de error o cambió el contrato | Capturar la respuesta cruda (`page`) |
| E07 | `Value` no numérico | Excepción al dividir por 100 | Cambio de contrato o moneda distinta | Revisar `CurrencyId` |
| E08 | `LevelRecycler` nulo | Excepción al comparar | Denominación sin recirculador | Filtrar nulos antes de convertir |
| E09 | CEF no inicializa | El detalle web no abre y puede cerrar la app | Falta `libcef.dll`, arquitectura cruzada, VC++ ausente | Reinstalar runtime, alinear x86/x64 |
| E10 | SMTP rechaza | `MessageBox` con el texto de la excepción | Contraseña normal en vez de "de aplicación", 2FA activo | Generar contraseña de aplicación |
| E11 | Temporizador silencioso | No hay histórico nuevo | Excepción tragada en `ChequeaCashdroBakgroud` | Revisar permisos de los SP |

### 8.1 Árbol de decisión rápido

```text
¿La app abre? ──no──► Revisar .NET 4.6.1 + VC++ Runtime + integridad de bin\
     │sí
¿El login pasa? ──no──► SQL Server / tabla [user] / credenciales
     │sí
¿Aparecen filas? ──no──► getCasdro sin datos o error silencioso
     │sí
¿Todas con ⚠? ──sí──► Red hacia los equipos o certificado/TLS
     │no
¿El detalle carga? ──no──► Credenciales CashDro en la URL o endpoint cambiado
     │sí
¿Hay histórico? ──no──► Permisos EXECUTE en pInsertDevices / pInsertdata
```

---

## 9. Especificaciones técnicas

| Parámetro | Valor |
|---|---|
| Framework | .NET Framework 4.6.1 |
| Tipo de salida | `WinExe` |
| Namespace raíz | `cashdro` |
| Tamaño del formulario | 1088 × 580 px, centrado en pantalla |
| Título de ventana | ` Sistema integrado ` |
| Intervalo de monitoreo | 1 800 000 ms (30 min) |
| Timeout de comandos SQL | 5 s (solo en `getCasdro` y `GetParametros`) |
| Timeout HTTP | No configurado (valor por defecto de `WebClient`: 100 s) |
| Moneda consultada | `CLP` (fija) |
| Imágenes de piezas | Excluidas (`includeImages=0`) |
| Niveles | Incluidos (`includeLevels=1`) |
| Protocolo CashDro | HTTPS con certificado no validado |
| Persistencia | SQL Server, autenticación SQL |
| Alertas | SMTP implícito sobre 587 con SSL |
| Idioma de la interfaz | Español, con dos mensajes en inglés |

### 9.1 Dependencias declaradas en `packages.config`

| Paquete | Versión | Para qué |
|---|---|---|
| `CefSharp.Common` | 71.0.2 | Envoltorio .NET de Chromium |
| `CefSharp.WinForms` | 71.0.2 | Control `ChromiumWebBrowser` |
| `cef.redist.x86` | 3.3578.1870 | Binarios nativos 32 bits |
| `cef.redist.x64` | 3.3578.1870 | Binarios nativos 64 bits |
| `Newtonsoft.Json` | 12.0.1 | Deserialización de las respuestas |

Referencias de framework: `System`, `System.Core`, `System.Data`,
`System.Data.DataSetExtensions`, `System.Drawing`, `System.Net.Http`,
`System.Windows.Forms`, `System.Xml`, `System.Xml.Linq`, `Microsoft.CSharp`,
`System.Deployment`.

Referencia de terceros fuera de NuGet: `DevComponents.DotNetBar2` 14.1.0.18.

**Instalación de dependencias**: no hay `requirements.txt` (el proyecto es C#). El
equivalente es restaurar NuGet:

```powershell
nuget restore cashdro.sln
# o desde Visual Studio: clic derecho en la solución → Restore NuGet Packages
```

Las DLL quedan en `..\packages\` respecto de la carpeta del proyecto, porque el
`.csproj` usa esa ruta relativa en sus `HintPath`.

---

## 10. Seguridad y manejo de datos

| Tema | Situación actual | Recomendación |
|---|---|---|
| Credenciales SQL | Embebidas en el código fuente | Mover a `App.config` cifrado o a un secreto del sistema |
| Credenciales CashDro | En la query string de cada URL | Usar POST o encabezado; rotar credenciales si se comparte el equipo |
| Validación TLS | Deshabilitada globalmente | Instalar el certificado del equipo en el almacén y validar |
| Login de la app | Clave en texto plano en SQL | Almacenar hash con sal |
| Correo | Clave SMTP en la base | Contraseña de aplicación + rotación periódica |
| Datos financieros | Niveles por denominación en SQL | Restringir lectura y auditar accesos |
| Registro de actividad | Inexistente | Agregar log a archivo con nivel y rotación |

> Los datos que maneja la aplicación son sensibles desde el punto de vista operativo
> (cuánto efectivo hay en cada equipo). Trata la base `GestionCashdro` como información
> restringida y no la expongas fuera de la red administrativa.

---

## 11. Mantenimiento

| Tarea | Frecuencia | Procedimiento |
|---|---|---|
| Verificar histórico | Semanal | Consultar las tablas pobladas por `pInsertDevices` y `pInsertdata` |
| Respaldar la base | Diaria | Respaldo completo de `GestionCashdro` |
| Rotar credenciales | Trimestral | Cambiar clave en CashDro, SQL y SMTP; recompilar si están embebidas |
| Revisar iconos | Al desplegar | Confirmar `D:\imagenes\check-icon.png` y `warning.png` |
| Actualizar .NET | Según ciclo de vida | 4.6.1 está fuera de soporte extendido; planificar migración |

---

## 12. Limitaciones conocidas de esta versión

1. **Windows-only**, sin versión multiplataforma.
2. **No probada contra hardware** en esta revisión documental: el comportamiento
   descrito proviene del análisis estático del código.
3. La consulta de detalle por equipo usa una **dirección fija** en
   `cargaGridDevice`, por lo que el clic puede mostrar siempre el mismo equipo.
4. El botón `button1` ejecuta una ruta de depuración con otro host fijo.
5. Moneda fija en `CLP`.
6. Sin manejo de timeouts ni reintentos.
7. Sin registro de errores; las excepciones se descartan.
8. La alerta por correo existe pero está desactivada en las rutas principales.
9. Los umbrales (`DepositLevel`) se configuran en la máquina, no en la aplicación.
10. El proyecto no incluye `Properties\AssemblyInfo.cs` ni la carpeta `packages\`;
    hay que recrearlos para compilar.

---

## 13. Glosario

| Término | Significado |
|---|---|
| **CashDro** | Máquina de gestión de efectivo (reciclador de billetes y monedas) |
| **Recirculador** | Módulo que acepta y vuelve a entregar piezas |
| **Denominación** | Valor facial de una pieza (10, 50, 100, 500, 1000, 2000 CLP) |
| **Fianza / `DepositLevel`** | Mínimo que debe quedar disponible para operar |
| **Total / `LevelRecycler`** | Cantidad presente en el recirculador en el momento de la lectura |
| **Recuento / `MaxLevel`** | Capacidad máxima de la denominación en ese módulo |
| **`getDiagnosis`** | Operación del WS que devuelve el estado de los dispositivos |
| **`getPiecesCurrency`** | Operación del WS que devuelve piezas y niveles por moneda |
| **CEF** | Chromium Embedded Framework, motor del navegador embebido |
| **SP** | Procedimiento almacenado de SQL Server |

---

## 14. Anexo — Respuestas JSON de referencia

Estructura esperada de `getDiagnosis` (valores ficticios):

```json
{
  "code": 0,
  "data": {
    "WithError": true,
    "Devices": [
      {
        "deviceId": 1,
        "deviceName": "LECTOR_BILLETES",
        "deviceWithError": false,
        "deviceType": 3,
        "portName": "COM1",
        "portProtocol": 1,
        "deviceModel": "MFL",
        "portState": 1,
        "ip": null,
        "tooManyChanges": false
      },
      {
        "deviceId": 4,
        "deviceName": "RECIRCULADOR_1",
        "deviceWithError": true,
        "deviceModel": "RCD",
        "portState": 0
      }
    ]
  }
}
```

Estructura esperada de `getPiecesCurrency` (valores ficticios):

```json
{
  "code": 0,
  "data": [
    {
      "CurrencyId": "CLP",
      "Value": "10000",
      "Type": "COIN",
      "DepositLevel": "20",
      "LevelRecycler": 148,
      "MaxLevel": "300",
      "MinLevel": "0",
      "State": "OK"
    }
  ]
}
```

`Value = "10000"` corresponde a una pieza de $100 (valor en centavos). La aplicación
muestra `10000 / 100 = 100`.

---

*Documento generado por revisión estática del código. No sustituye las pruebas de
aceptación en sitio.*
