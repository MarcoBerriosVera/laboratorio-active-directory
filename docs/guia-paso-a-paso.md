# Guía paso a paso

Guía para armar el laboratorio. Mientras la sigo, anoto en el [README](../README.md) los problemas reales que aparezcan y guardo las capturas en `capturas/`.

Convención de nombres de capturas: `01-vm-servidor.png`, `02-instalar-ad-ds.png`, etc.

## 1. Máquinas virtuales y red

1. Descargar las ISO de evaluación desde el [Evaluation Center de Microsoft](https://www.microsoft.com/evalcenter): Windows Server y un Windows cliente.
2. En VirtualBox crear la VM del servidor: 2 CPU, 2 GB de RAM, disco de 50 GB (dinámico).
3. Crear la VM del cliente (Windows 10): 1 CPU, 2 GB de RAM, disco de 50 GB (dinámico). Windows 11 pide 4 GB, por eso uso Windows 10.
4. En **ambas**, adaptador de red 1 → **Red interna**, con el mismo nombre (por ejemplo `labnet`).
5. Opcional: en el servidor, un segundo adaptador en **NAT** para actualizaciones. Después se puede desactivar.

> Mi equipo tiene 8 GB de RAM. Apago las otras máquinas virtuales y cierro el navegador mientras trabajo. Los pasos 2, 3 y 6 solo necesitan el servidor encendido; el cliente se enciende para los pasos 4, 5 y 7.

## 2. Servidor y dominio

1. Instalar Windows Server (con experiencia de escritorio).
2. Cambiar el nombre del equipo a `SRV-DC01`.
3. Asignar IP fija: `10.0.0.10`, máscara `255.255.255.0`. Como DNS, el mismo servidor (`10.0.0.10`).
4. En el Administrador del servidor: **Agregar roles y características** → **Servicios de dominio de Active Directory** (incluye DNS).
5. **Promover este servidor a controlador de dominio** → *Agregar un nuevo bosque* → dominio raíz `lab.local`.
6. Definir la contraseña de restauración (DSRM) y guardarla en un lugar seguro, fuera del repositorio.
7. Reiniciar e iniciar sesión como `LAB\Administrador`.

> Nota: `.local` es común en laboratorios, pero en un entorno real conviene usar un subdominio de un dominio propio.

## 3. Unidades organizativas, usuarios y grupos

Herramienta: **Usuarios y equipos de Active Directory** (`dsa.msc`).

1. Crear la unidad organizativa (OU) `Empleados` y dentro `Ventas` y `TI`. Crear también `Equipos` y `Grupos`.
2. Crear 2–3 usuarios ficticios por área (por ejemplo `ana.ventas`, `luis.ti`).
3. Crear grupos de seguridad: `GG_Ventas` y `GG_TI`, y agregar a cada usuario en su grupo.

Opcional, versión en PowerShell (lo hago a mano primero para entender qué hace cada cosa):

```powershell
New-ADOrganizationalUnit -Name "Empleados" -Path "DC=lab,DC=local"
New-ADUser -Name "Ana Ventas" -SamAccountName "ana.ventas" `
  -Path "OU=Ventas,OU=Empleados,DC=lab,DC=local" `
  -AccountPassword (Read-Host -AsSecureString "Contraseña") -Enabled $true
Add-ADGroupMember -Identity "GG_Ventas" -Members "ana.ventas"
```

## 4. Unir el cliente al dominio

1. Instalar Windows en la VM cliente y nombrarla `PC-CLIENTE01`.
2. IP fija `10.0.0.20`, máscara `255.255.255.0`, **DNS `10.0.0.10`** (si el DNS no apunta al servidor, no encuentra el dominio).
3. Comprobar con `ping 10.0.0.10` y `nslookup lab.local`.
4. Configuración → Sistema → Acerca de → **Unirse a un dominio** → `lab.local` con una cuenta con permisos.
5. Reiniciar e iniciar sesión con un usuario del dominio (`LAB\ana.ventas`).
6. Verificar con `whoami` y `echo %USERDOMAIN%`.

## 5. Tareas de soporte

Practicar en `dsa.msc` (clic derecho sobre el usuario):

- **Restablecer contraseña** y marcar "El usuario debe cambiar la contraseña en el siguiente inicio de sesión".
- **Desbloquear cuenta**: provocar el bloqueo escribiendo mal la contraseña varias veces desde el cliente (ver paso 6).
- **Deshabilitar cuenta** (baja de un empleado) y comprobar que ya no puede iniciar sesión.
- **Mover** un usuario a otra OU y **cambiar** su grupo.

## 6. Políticas de grupo (GPO)

Herramienta: **Administración de directivas de grupo** (`gpmc.msc`).

1. Política de contraseñas, en *Default Domain Policy* → Configuración del equipo → Directivas → Configuración de Windows → Configuración de seguridad → Directivas de cuenta:
   - longitud mínima de contraseña, vigencia máxima y umbral de bloqueo de cuenta (por ejemplo, 5 intentos fallidos).
2. Una GPO simple vinculada a la OU `Ventas`, por ejemplo cambiar el fondo de pantalla o restringir el Panel de control.
3. En el cliente: `gpupdate /force` y luego `gpresult /r` para confirmar que la política se aplicó.

## 7. Revisar eventos de seguridad

En el servidor y en el cliente: **Visor de eventos** → Registros de Windows → **Seguridad**. Eventos útiles:

| ID | Significado |
|---|---|
| 4624 | Inicio de sesión correcto |
| 4625 | Inicio de sesión fallido |
| 4740 | Cuenta bloqueada |
| 4720 | Cuenta de usuario creada |

Dejo una captura de cada uno. Estos eventos son la base del proyecto de análisis de logs.

## Si algo falla

- El cliente no encuentra el dominio: revisar que el DNS del cliente sea la IP del servidor.
- No se puede unir al dominio: revisar que ambas VMs usen la misma red interna y que el ping responda.
- Error de autenticación aunque la clave es correcta: Kerberos exige que los relojes estén sincronizados (diferencia menor a 5 minutos).
