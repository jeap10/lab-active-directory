# 🏢 Laboratorio Active Directory: Windows Server 2022 + Windows 11

Laboratorio práctico donde se monta una **mini red de empresa** en VirtualBox: un **controlador de dominio** con Windows Server 2022 (Active Directory + DNS) y un cliente **Windows 11** unido al dominio.

> Autor: **Jerson Palacios** — Estudiante de Ingeniería de Ciberseguridad (UNI)

---

## 🗺️ Topología final

```
Red NAT "LabAD" — 192.168.100.0/24
Gateway (salida a internet): 192.168.100.1

practica.local
├── SRV-DC01 ........ 192.168.100.10 (IP fija)  → AD DS + DNS
├── OU: TI
│   └── jpalacios (usuario del dominio)
└── PC-PRACTICA01 ... 192.168.100.x (DHCP)       → Windows 11 Pro unido al dominio
```

---

## 📋 Contenido

1. [Crear la VM del servidor](#1-crear-la-vm-del-servidor)
2. [Instalar Windows Server 2022](#2-instalar-windows-server-2022)
3. [Primer inicio y Guest Additions](#3-primer-inicio-y-guest-additions)
4. [Zona horaria y nombre del servidor](#4-zona-horaria-y-nombre-del-servidor)
5. [Crear la red del laboratorio (Red NAT)](#5-crear-la-red-del-laboratorio-red-nat)
6. [IP fija del servidor](#6-ip-fija-del-servidor)
7. [Instalar Active Directory (AD DS)](#7-instalar-active-directory-ad-ds)
8. [Promover a controlador de dominio](#8-promover-a-controlador-de-dominio)
9. [Reenviador DNS (internet para el servidor)](#9-reenviador-dns-internet-para-el-servidor)
10. [Crear OU y usuario](#10-crear-ou-y-usuario)
11. [Unir Windows 11 al dominio](#11-unir-windows-11-al-dominio)
12. [Iniciar sesión con el usuario del dominio](#12-iniciar-sesión-con-el-usuario-del-dominio)
13. [Resumen de comandos](#13-resumen-de-comandos)
14. [Problemas encontrados y solución](#14-problemas-encontrados-y-solución)

---

## 1. Crear la VM del servidor

**VirtualBox → Nueva**

| Parámetro | Valor | Por qué |
|---|---|---|
| Nombre | `SRV-DC01` | Estándar de nombres: **SRV**idor – **D**omain **C**ontroller – **01** |
| ISO | `SERVER_EVAL_x64FRE_es-es.iso` | Windows Server 2022 Evaluation (gratis 180 días) |
| Instalación desatendida | ❌ Omitida | Para hacer la instalación paso a paso |
| RAM / CPU | 4096 MB / 2 | Suficiente para un DC de laboratorio |
| Disco | 50 GB VDI, **dinámico** | Solo ocupa en el equipo real lo que realmente usa |

> ⚠️ No marcar **"Reservar completamente"**: reservaría los 50 GB de golpe.

---

## 2. Instalar Windows Server 2022

1. Iniciar la VM y **presionar una tecla** en `Press any key to boot from CD or DVD...`
2. Idioma: Español · Teclado: **Latinoamericano** (para que los símbolos de las contraseñas coincidan con el teclado real).
3. Edición: **Windows Server 2022 Standard Evaluation (experiencia de escritorio)**.
4. Tipo: **Personalizada** (instalación limpia) → disco sin asignar.
5. Contraseña del **Administrador** (mín. 8 caracteres, 3 de 4 tipos: mayúsculas, minúsculas, números, símbolos).

| Edición | Descripción |
|---|---|
| Standard | **Server Core**: sin interfaz gráfica, solo comandos/PowerShell. Más ligero y seguro |
| Standard (experiencia de escritorio) | Con interfaz gráfica ← **la usada** |
| Datacenter | Para mucha virtualización (VMs ilimitadas por licencia) |

> 📝 **Personalizada** = instalación limpia. **Actualización** solo sirve para subir de versión un servidor existente conservando datos.

---

## 3. Primer inicio y Guest Additions

- **Desbloquear:** el servidor pide `Ctrl + Alt + Supr`, pero esa combinación la captura el equipo real. En VirtualBox:
  - **Entrada → Teclado → Insertar Ctrl-Alt-Supr**, o
  - **Ctrl derecho + Supr**
- **Guest Additions** (drivers de VirtualBox para resolución y mouse):
  1. **Dispositivos → Insertar imagen de CD de los complementos del invitado**
  2. Explorador → unidad de CD → `VBoxWindowsAdditions-amd64.exe` → Install → **Reboot now**
  3. **Ver → Autoredimensionar pantalla del invitado** para que el escritorio se ajuste a la ventana.
- Cerrar el aviso de **Windows Admin Center** (otra herramienta de administración web, opcional).

---

## 4. Zona horaria y nombre del servidor

**Administrador del servidor → Servidor local**

- **Zona horaria:** `(UTC-05:00) Bogotá, Lima, Quito, Río Branco`
  > 🔐 La hora es crítica en AD: **Kerberos** solo tolera **~5 minutos** de diferencia entre equipos; si no, falla el inicio de sesión. También es clave para correlacionar **logs**.
- **Nombre del equipo:** enlace del nombre → **Cambiar...** → `SRV-DC01` → reiniciar.
  > ⚠️ **"Descripción del equipo"** es solo un comentario; el nombre se cambia con el botón **Cambiar...**
  >
  > 📝 Se cambia **antes** de crear el dominio, porque el nombre del DC queda registrado en AD y en DNS.

---

## 5. Crear la red del laboratorio (Red NAT)

**Problema:** en modo **NAT** (por defecto) cada VM tiene su propia red aislada `10.0.2.0/24` y siempre recibe `10.0.2.15`. Tienen internet, pero **no se ven entre sí** → el cliente no podría encontrar al servidor.

**Solución:** una **Red NAT** compartida.

**VirtualBox → Herramientas → Red → Redes NAT → Crear**

| Campo | Valor |
|---|---|
| Nombre | `LabAD` |
| Prefijo IPv4 | `192.168.100.0/24` |
| DHCP | ✅ Habilitado |

Conectar ambas VMs: **Configuración → Red → Adaptador 1 → Conectado a: Red NAT → LabAD**.

| Modo de red | ¿Se ven entre VMs? | ¿Internet? |
|---|---|---|
| NAT | ❌ | ✅ |
| **Red NAT** ← usada | ✅ | ✅ |
| Adaptador puente | ✅ (y con la red de la casa) | ✅ |
| Solo anfitrión | ✅ (y con el host) | ❌ |
| Red interna | ✅ (solo entre VMs) | ❌ |

En el servidor, para tomar IP de la nueva red sin reiniciar:

```cmd
ipconfig /release   :: devuelve la IP actual al DHCP (queda sin IP)
ipconfig /renew     :: pide una IP nueva al DHCP (proceso DORA)
ipconfig            :: verifica: 192.168.100.x, gateway 192.168.100.1
```

> 📝 Dentro de una **misma red**, dos equipos **no pueden** tener la misma IP. En redes distintas sí pueden repetirse.

---

## 6. IP fija del servidor

Un controlador de dominio **nunca usa DHCP**: si su IP cambia, los equipos del dominio dejan de encontrarlo.

`Win + R` → **`ncpa.cpl`** (Conexiones de red) → Ethernet → Propiedades → **TCP/IPv4** → Propiedades:

| Campo | Valor | Significado |
|---|---|---|
| Dirección IP | `192.168.100.10` | IP fija del servidor |
| Máscara | `255.255.255.0` | Red /24 (254 hosts) |
| Puerta de enlace | `192.168.100.1` | Router de la Red NAT (salida a internet) |
| DNS preferido | `127.0.0.1` | **"Yo mismo"**: el servidor será el DNS del dominio |

Verificación:

```cmd
ipconfig /all
```

| Dato esperado | Valor |
|---|---|
| Nombre de host | SRV-DC01 |
| DHCP habilitado | **no** |
| Dirección IPv4 | 192.168.100.10 |
| Servidores DNS | 127.0.0.1 |

---

## 7. Instalar Active Directory (AD DS)

**Administrador del servidor → Administrar → Agregar roles y características**

1. Tipo: **Instalación basada en características o en roles**
2. Servidor: `SRV-DC01 (192.168.100.10)`
3. Rol: ✅ **Servicios de dominio de Active Directory** → *Agregar características*
4. **Instalar**

> 📝 Esto solo **instala** el rol. Todavía no existe ningún dominio.

---

## 8. Promover a controlador de dominio

**Banderita ⚠️ → Promover este servidor a controlador de dominio**

| Paso | Valor |
|---|---|
| Configuración de implementación | **Agregar un nuevo bosque** |
| Nombre de dominio raíz | `practica.local` |
| Opciones del DC | DNS ✅ · Catálogo global ✅ · Contraseña **DSRM** |
| NetBIOS | `PRACTICA` (nombre corto → `PRACTICA\usuario`) |
| Rutas (NTDS, SYSVOL) | Por defecto |
| Comprobación de requisitos | Advertencias amarillas → normales en laboratorio |

→ **Instalar** → reinicio automático. Al volver, el inicio de sesión es **`PRACTICA\Administrador`**.

### Las 3 contraseñas del laboratorio

| Contraseña | Dónde se creó | Para qué sirve |
|---|---|---|
| **Administrador** | Instalación de Windows Server | Iniciar sesión. Tras la promoción pasa a ser `PRACTICA\Administrador` (misma contraseña) |
| **DSRM** | Asistente de promoción | **Solo emergencias**: reparar AD en modo de restauración |
| **jpalacios** | Al crear el usuario | Iniciar sesión como ese usuario |

---

## 9. Reenviador DNS (internet para el servidor)

Como el DNS del servidor apunta a sí mismo (`127.0.0.1`), no sabe resolver nombres de internet. El **reenviador** le dice: *"lo que no conozcas, pregúntaselo a 8.8.8.8"*.

**Herramientas → DNS → SRV-DC01 → Reenviadores → Editar → `8.8.8.8`**

O por PowerShell:

```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8 -PassThru   # agrega el reenviador
Get-DnsServerForwarder                                # lista los reenviadores
```

Verificación:

```cmd
nslookup google.com
```

| Línea | Significado |
|---|---|
| `Servidor: UnKnown / Address: ::1` | Se preguntó a sí mismo (`::1` = localhost en IPv6). "UnKnown" = sin registro PTR; normal |
| `Respuesta no autoritativa` | El servidor **no es dueño** de google.com; la respuesta vino del reenviador |
| `Addresses` | IPs de Google (IPv6 e IPv4) |

### Partes del Administrador de DNS

| Carpeta | Contenido |
|---|---|
| Zonas de búsqueda directa | Nombre → IP (ej. `practica.local`, `SRV-DC01 → 192.168.100.10`) |
| Zonas de búsqueda inversa | IP → nombre (registros **PTR**) |
| Reenviadores | A quién preguntar lo que no se conoce (internet) |
| Sugerencias de raíz | Servidores raíz de internet (último recurso) |

---

## 10. Crear OU y usuario

**Herramientas → Usuarios y equipos de Active Directory**

1. Clic derecho en `practica.local` → **Nuevo → Unidad organizativa** → `TI`
2. Clic derecho en `TI` → **Nuevo → Usuario**
   - Nombre: Jerson · Apellidos: Palacios · Inicio de sesión: `jpalacios`
   - Contraseña (en laboratorio se desmarcó *"debe cambiar la contraseña en el siguiente inicio"*)

> 📝 **OU (Unidad Organizativa):** carpeta para ordenar usuarios y equipos (por área: TI, Contabilidad, Nóminas). A cada OU se le pueden aplicar **GPO** distintas.

| Contenedor | Contenido |
|---|---|
| Builtin | Grupos integrados (Administradores, Usuarios…) |
| Computers | Equipos que se unen al dominio (por defecto) |
| Domain Controllers | Los controladores de dominio (SRV-DC01) |
| Users | Usuarios y grupos por defecto (Administrador, Admins. del dominio…) |
| **TI** | OU creada en el laboratorio |

---

## 11. Unir Windows 11 al dominio

### 11.1 DNS del cliente apuntando al servidor

Sin esto, Windows 11 **no encuentra** el dominio (AD depende del DNS).

`Win + R` → **`ncpa.cpl`** → Ethernet → TCP/IPv4:
- IP: **automática** (DHCP de la Red NAT)
- **DNS preferido:** `192.168.100.10`

### 11.2 Verificar conectividad

```cmd
ping 192.168.100.10        :: ¿llego al servidor?
nslookup practica.local    :: ¿el DNS resuelve el dominio? → 192.168.100.10
```

### 11.3 Unirse

`Win + R` → **`sysdm.cpl`** (Propiedades del sistema) → Nombre de equipo → **Cambiar...**

- Nombre del equipo: `PC-PRACTICA01`
- Miembro de: **Dominio** → `practica.local`
- Credenciales: `PRACTICA\Administrador` o `Administrador@practica.local`
- Mensaje: **"Se unió correctamente al dominio practica.local"** → reiniciar

> 📝 Cualquier usuario del dominio puede unir hasta **10 equipos** por defecto (por eso también funcionaría `PRACTICA\jpalacios`).

---

## 12. Iniciar sesión con el usuario del dominio

Pantalla de inicio → **Otro usuario** → `PRACTICA\jpalacios`

```cmd
whoami
```

| Resultado | Significado |
|---|---|
| `pc-practica01\jerson` | Cuenta **local** (vive en la PC) |
| `practica\jpalacios` | Cuenta **del dominio** (vive en AD, en el servidor) ✅ |
| `practica\administrador` | CMD elevado con credenciales de administrador del dominio |

### UAC y mínimo privilegio

`jpalacios` es un **usuario estándar**. Al abrir CMD *como administrador*, el **Control de cuentas de usuario (UAC)** pide credenciales de un administrador. Al ingresar `PRACTICA\Administrador`, **solo esa ventana** corre como administrador; la sesión sigue siendo de `jpalacios`. Así trabaja TI en una empresa.

---

## 13. Resumen de comandos

| Comando | Dónde | Para qué |
|---|---|---|
| `ipconfig` | CMD | Ver IP, máscara y gateway |
| `ipconfig /all` | CMD | + MAC, DHCP habilitado, servidores DNS, nombre de host |
| `ipconfig /release` | CMD | Liberar la IP obtenida por DHCP |
| `ipconfig /renew` | CMD | Pedir una IP nueva al DHCP |
| `ncpa.cpl` | Win + R | Abrir Conexiones de red (configurar IP/DNS) |
| `sysdm.cpl` | Win + R | Propiedades del sistema (nombre, dominio) |
| `nslookup google.com` | CMD | Probar resolución DNS de internet |
| `nslookup practica.local` | CMD | Probar resolución del dominio |
| `ping 192.168.100.10` | CMD | Probar conectividad con el servidor |
| `whoami` | CMD | Ver con qué cuenta se está trabajando |
| `Add-DnsServerForwarder -IPAddress 8.8.8.8` | PowerShell | Agregar reenviador DNS |
| `Get-DnsServerForwarder` | PowerShell | Listar reenviadores DNS |
| `Ctrl derecho + Supr` | VirtualBox | Enviar Ctrl+Alt+Supr a la VM |

---

## 14. Problemas encontrados y solución

| Problema | Causa | Solución |
|---|---|---|
| `No bootable medium found` | No se presionó una tecla a tiempo | Montar la ISO y reintentar; presionar una tecla en *Press any key* |
| Pantalla pequeña | Sin Guest Additions | Instalarlas + **Ver → Autoredimensionar pantalla del invitado** |
| Hora del servidor incorrecta | Zona horaria de España por el idioma | Cambiar a UTC-05:00 Lima |
| Nombre no cambiaba | Se escribió en "Descripción del equipo" | Usar el botón **Cambiar...** |
| Las VMs no se veían | Modo NAT aislado | Red NAT compartida `LabAD` |
| Menú de DNS en gris | La consola no terminó de cargar tras crear el dominio | Esperar / F5 / reabrir la consola |
| "El nombre de usuario no es válido" | Se usó `/` en vez de `\` | `PRACTICA\Administrador` o `Administrador@practica.local` |
| UAC pide credenciales | `jpalacios` es usuario estándar | Usar credenciales de administrador solo para esa tarea |

---

## ✅ Próximos pasos

- [ ] Crear y aplicar una **GPO** (ej. fondo de pantalla obligatorio, bloqueo de USB) en la OU TI
- [ ] Crear **grupos** de seguridad y carpetas compartidas con permisos
- [ ] Revisar eventos de inicio de sesión (**4624 / 4625**) en el Visor de eventos del DC
- [ ] Configurar **DHCP** en el servidor en lugar del DHCP de VirtualBox
