# Referencias

Fuentes consultadas y citadas por la documentación de **Lectura de parámetros CashDro**.
Se distingue entre documentación oficial del fabricante, documentación de las
bibliotecas usadas y material de contexto.

> **Nota de alcance**: la API `Cashdro3WS` **no está documentada públicamente** en este
> repositorio ni se encontró un contrato publicado. Todo lo que la documentación afirma
> sobre `getDiagnosis` y `getPiecesCurrency` proviene de la lectura del código fuente
> (`Form1.cs`, `dataClass.cs`) y de los nombres de campo observados, no de un manual del
> fabricante. Trátalo como contrato observado, no como especificación oficial.

---

## 1. Fabricante y producto

| Referencia | Uso en esta documentación |
|---|---|
| CashDro — sitio oficial <https://www.cashdro.com/cl/> | Contexto del producto: máquinas de gestión y reciclado de efectivo |
| CashDro — soporte y documentación del fabricante <https://www.cashdro.com/> | Vía para solicitar el manual de integración del WS y las credenciales de API |
| Consola web del equipo: `https://<IP_EQUIPO>/Cashdro3Web/` | Interfaz usada por el visor CEF; se abre en el equipo, no en Internet |
| Endpoint del WS: `https://<IP_EQUIPO>/Cashdro3WS/index.php` | Origen de los datos técnicos y financieros |

El README original del proyecto indicaba que el acceso al repositorio requería solicitar
contraseña al autor y enlazaba al sitio del fabricante en su versión chilena. Esa nota se
conserva como antecedente histórico; el código de esta carpeta es el que se documenta.

---

## 2. Bibliotecas y plataformas

### 2.1 CefSharp

| Dato | Valor |
|---|---|
| Versión usada | `CefSharp.Common` 71.0.2 · `CefSharp.WinForms` 71.0.2 |
| Motor nativo | `cef.redist.x86` / `cef.redist.x64` 3.3578.1870 |
| Repositorio | <https://github.com/cefsharp/CefSharp> |
| Documentación | <https://cefsharp.github.io/> |
| Notas de versión | <https://github.com/cefsharp/CefSharp/releases> |
| Guía AnyCPU | <https://github.com/cefsharp/CefSharp/wiki/AnyCPU> |

Referencias para resolver los problemas típicos:

- `CefSharpAnyCpuSupport` y el archivo `CefSharpAnyCpuSupport.targets`: necesarios para
  que el proyecto arranque un proceso con la arquitectura correcta.
- `CefSettings.IgnoreCertificateErrors`: usado en `Form1()`; equivale a desactivar la
  validación TLS del navegador embebido.
- Inicialización única de CEF por proceso: llamar `Cef.Initialize` dos veces lanza
  excepción. En esta app solo se llama en el constructor de `Form1`.

### 2.2 Chromium Embedded Framework (CEF)

| Dato | Valor |
|---|---|
| Build | 3.3578.1870 (rama 3578, Chromium 71) |
| Proyecto | <https://github.com/chromiumembedded/cef> |
| Foro | <https://magpcss.org/ceforum/> |

Chromium 71 es de fines de 2018. No recibe parches de seguridad; si el visor se usa para
navegar algo más que la consola local del equipo, es un riesgo a considerar.

### 2.3 Json.NET (Newtonsoft.Json)

| Dato | Valor |
|---|---|
| Versión usada | 12.0.1 |
| Documentación | <https://www.newtonsoft.com/json> |
| Repositorio | <https://github.com/JamesNK/Newtonsoft.Json> |
| Notas de versión | <https://github.com/JamesNK/Newtonsoft.Json/releases> |

Comportamiento relevante para el contrato: por defecto, las propiedades del JSON que no
existen en la clase se ignoran y las propiedades de la clase que faltan en el JSON quedan
en su valor por defecto (`null`, `0`, `false`). La aplicación no configura
`MissingMemberHandling` ni `NullValueHandling`.

### 2.4 .NET Framework 4.6.1

| Dato | Valor |
|---|---|
| Documentación | <https://learn.microsoft.com/dotnet/framework/> |
| Ciclo de vida | <https://learn.microsoft.com/lifecycle/products/microsoft-net-framework> |
| Descarga | <https://dotnet.microsoft.com/download/dotnet-framework/net461> |

.NET Framework 4.6.1 quedó fuera de soporte de Microsoft en abril de 2022. La aplicación
sigue funcionando en Windows actualizado porque el runtime 4.x es compatible hacia
adelante, pero no recibe correcciones. Cualquier trabajo de mantenimiento serio debería
planificar la migración a una versión soportada.

### 2.5 Windows Forms

| Tema | Referencia |
|---|---|
| `DataGridView` | <https://learn.microsoft.com/dotnet/api/system.windows.forms.datagridview> |
| `Timer` (WinForms) | <https://learn.microsoft.com/dotnet/api/system.windows.forms.timer> |
| `Application.Run` / STA | <https://learn.microsoft.com/dotnet/api/system.windows.forms.application.run> |
| `PictureBox` | <https://learn.microsoft.com/dotnet/api/system.windows.forms.picturebox> |

El `System.Windows.Forms.Timer` ejecuta su `Tick` en el hilo de UI y no admite
reentrada: si el manejador tarda más que el intervalo, los ticks se acumulan.

### 2.6 Acceso a datos

| Tema | Referencia |
|---|---|
| `System.Data.SqlClient` | <https://learn.microsoft.com/dotnet/api/system.data.sqlclient> |
| `SqlCommand.CommandTimeout` | <https://learn.microsoft.com/dotnet/api/system.data.sqlclient.sqlcommand.commandtimeout> |
| `Microsoft.Data.SqlClient` (sucesor) | <https://learn.microsoft.com/sql/connect/ado-net/introduction-microsoft-data-sqlclient-namespace> |
| Procedimientos almacenados | <https://learn.microsoft.com/sql/relational-databases/stored-procedures/stored-procedures-database-engine> |

`System.Data.SqlClient` está en modo mantenimiento; Microsoft recomienda
`Microsoft.Data.SqlClient` para proyectos nuevos.

### 2.7 Correo

| Tema | Referencia |
|---|---|
| `SmtpClient` | <https://learn.microsoft.com/dotnet/api/system.net.mail.smtpclient> |
| Aviso de obsolescencia | <https://learn.microsoft.com/dotnet/api/system.net.mail.smtpclient#remarks> |
| Contraseñas de aplicación de Google | <https://support.google.com/accounts/answer/185833> |
| Alternativa recomendada | <https://github.com/jstedfast/MailKit> |

La documentación de `SmtpClient` indica explícitamente que no se recomienda para
implementaciones nuevas; `MailKit` es la alternativa sugerida por el propio equipo de
.NET.

### 2.8 Componente de terceros: DevComponents DotNetBar

| Dato | Valor |
|---|---|
| Referencia en `.csproj` | `DevComponents.DotNetBar2, Version=14.1.0.18, PublicKeyToken=7eb7c3a35b91de04` |
| Uso | Clase base `OfficeForm` del formulario `webBrownser` |
| Sitio | <https://www.devcomponents.com/> |

Es un componente comercial que **no** se distribuye con el repositorio ni está declarado
en `packages.config`. Sin la DLL, el proyecto no compila. Si no se cuenta con licencia, la
salida más simple es cambiar la clase base a `System.Windows.Forms.Form`.

---

## 3. Seguridad: referencias para los hallazgos

| Hallazgo en el código | Referencia |
|---|---|
| Validación de certificado desactivada | CWE-295 *Improper Certificate Validation* — <https://cwe.mitre.org/data/definitions/295.html> |
| Credenciales embebidas en el binario | CWE-798 *Use of Hard-coded Credentials* — <https://cwe.mitre.org/data/definitions/798.html> |
| Clave almacenada en texto plano | CWE-256 *Plaintext Storage of a Password* — <https://cwe.mitre.org/data/definitions/256.html> |
| Credenciales en la URL | CWE-598 *Use of GET Request Method With Sensitive Query Strings* — <https://cwe.mitre.org/data/definitions/598.html> |
| Excepciones descartadas | CWE-390 *Detection of Error Condition Without Action* — <https://cwe.mitre.org/data/definitions/390.html> |
| Falta de registro de eventos | CWE-778 *Insufficient Logging* — <https://cwe.mitre.org/data/definitions/778.html> |
| Categorías generales | OWASP Top 10 2021 — A02 *Cryptographic Failures*, A07 *Identification and Authentication Failures* — <https://owasp.org/Top10/> |

Estas referencias se citan para orientar la corrección, no como auditoría formal. La
aplicación se diseñó para una red administrativa cerrada; los riesgos cambian si el
puesto o la base se exponen fuera de esa red.

---

## 4. Herramientas de construcción

| Herramienta | Referencia |
|---|---|
| Visual Studio 2017 (formato de solución 12) | <https://learn.microsoft.com/visualstudio/> |
| MSBuild | <https://learn.microsoft.com/visualstudio/msbuild/msbuild> |
| NuGet CLI (`nuget restore`) | <https://learn.microsoft.com/nuget/reference/cli-reference/cli-ref-restore> |
| `packages.config` (formato clásico) | <https://learn.microsoft.com/nuget/reference/packages-config> |
| Visual C++ Redistributable | <https://learn.microsoft.com/cpp/windows/latest-supported-vc-redist> |

El proyecto usa el formato clásico de MSBuild (`ToolsVersion 15.0`), no SDK-style. Por eso
los paquetes se resuelven con `packages.config` + `HintPath` y no con
`PackageReference`.

---

## 5. Licencia

| Dato | Valor |
|---|---|
| Licencia del proyecto | MIT |
| Texto de referencia | <https://opensource.org/license/mit> |
| Titular | `Copyright (c) 2026 jmadasme` |

MIT cubre el código de este repositorio. **No** cubre CefSharp (BSD-3), Chromium (BSD-3 y
otras), Newtonsoft.Json (MIT), DevComponents DotNetBar (comercial) ni el firmware o la API
de CashDro, que se rigen por sus propios términos.

---

## 6. Procedencia del código

| Dato | Valor |
|---|---|
| Repositorio original | `jmadasme/Lectura-parametros-maquina-Cashdro` |
| Ruta en el portafolio | `projects/cashdro-params/versions/Lectura-parametros-maquina-Cashdro/` |
| Cambios aplicados al importar | Sustitución de credenciales por `USUARIO_CASHDRO` / `CLAVE_CASHDRO`; exclusión de bases de datos, binarios y artefactos de ejecución |
| Cambios **no** aplicados | Lógica, estructura de archivos, versiones de paquetes |

Las direcciones IP que aparecen literalmente en el código y en comentarios se omitieron de
toda la documentación. Si necesitas reproducir el entorno, usa la IP de tu propio equipo
de laboratorio.

---

## 7. Grado de confianza de las afirmaciones

| Afirmación | Fuente | Confianza |
|---|---|---|
| Framework, versiones de paquetes, arquitectura | `.csproj`, `packages.config`, `App.config` | Alta (lectura directa) |
| Nombres de operaciones y parámetros del WS | `Form1.cs` | Alta (están escritos en el código) |
| Semántica de `DepositLevel`, `LevelRecycler`, `MaxLevel` | Nombres de campo + uso en la regla de alerta | Media |
| Significado de `code` | Inferido por convención; el código lo ignora | Baja |
| Comportamiento del equipo ante errores | No probado | Baja |
| Requisito de VC++ Runtime para CEF | Documentación de CefSharp | Alta |
| Fin de soporte de .NET 4.6.1 | Ciclo de vida de Microsoft | Alta |

Las afirmaciones de confianza baja están marcadas como tales en `datasheet.md` y
`developer.md`. Antes de dar por válido el contrato, captura una respuesta real de tu
equipo y compárala con el anexo JSON del datasheet.

---

## 8. Cómo citar este proyecto

```text
Adasme, J. M. (2026). Lectura de parámetros CashDro [Software].
https://github.com/jmadasme/Lectura-parametros-maquina-Cashdro
Licencia MIT.
```

Para las bibliotecas, cita las versiones exactas de la tabla §2 y no el rango
"latest": el comportamiento de CefSharp 71 y de Json.NET 12 es el que el código asume.
