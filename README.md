# Intel Wi-Fi 6 / 6E AX210 en macOS Sonoma, Sequoia y Tahoe (Hackintosh)

<p align="center">
  <img src="main/IMG/Wi-Fi.png">
</p>

---

## Introducción

A partir de **macOS Sonoma**, Apple eliminó los drivers para tarjetas Broadcom utilizadas en modelos anteriores a 2017.  
Una de las más afectadas en el entorno Hackintosh es la **Fenvi T-919**.

El equipo de **OpenCore Legacy Patcher (OCLP)** permite restaurar compatibilidad mediante *root patches*, pero esto requiere:

- `SecureBootModel` desactivado  
- `SIP` parcialmente deshabilitado  

Esto implica una reducción del nivel de seguridad del sistema.

Esta guía propone una alternativa moderna utilizando la **Intel AX210 WiFi 6E PCIe**, compatible gracias al proyecto **OpenIntelWireless**, sin necesidad de aplicar root patches y manteniendo el nivel de seguridad estándar de macOS.

---

## Ventajas de la Intel AX210

- No requiere OCLP root patches  
- Permite mantener `SIP` activo (`csr-active-config = 00000000`)  
- Permite mantener `SecureBootModel` activo  
- Compatible con macOS Sonoma, Sequoia y Tahoe  
- Solución moderna y económica  

---

# Hardware

La tarjeta puede adquirirse de dos formas:

### Opción 1 — Tarjeta PCIe ensamblada
Ejemplo: Intel AX210S PCIe (Ziyituod u otros fabricantes)

### Opción 2 — Módulo + Adaptador
- Módulo: Intel AX210 M.2/NGFF (A+E Key)  
- Adaptador: PCIe x1 → M.2/NGFF A+E  

<table>
<tr>
<td><img src="main/IMG/Card and adapter.png"></td>
<td><img src="main/IMG/AX210 card.jpg"></td>
</tr>
</table>

---

# Revertir configuración Broadcom / OCLP

Si utilizabas Fenvi o Broadcom con root patches, debes revertir todo antes de continuar.

## Cambios en config.plist

Deshabilitar:

- `IOSkywalk.kext`
- `IO80211FamilyLegacy.kext`
- `AirPortBrcmNIC.kext`
- Cualquier bloqueo de `IOSkywalk`

Restaurar valores de seguridad:

- `csr-active-config` → `00000000`
- `SecureBootModel` → cualquier valor distinto de `Disabled`

## En OCLP

OpenCore Legacy Patcher →  
Post-Install Root Patch →  
Revert Root Patches

Reiniciar el sistema antes de continuar.

---

# Instalación del módulo Wi-Fi

Los kexts están disponibles en el proyecto **OpenIntelWireless**.

Existen dos métodos.  
⚠️ Solo debe utilizarse uno.

No usar `itlwm.kext` y `AirportItlwm.kext` simultáneamente.

---

## Método 1 — itlwm.kext + HeliPort

`itlwm.kext` utiliza `IOEthernetController` en lugar de `IO80211Family`.  
La conexión se presenta como Ethernet, aunque funciona como Wi-Fi real.

Requiere la aplicación **HeliPort** para gestionar redes.

### Versiones recomendadas

- Ventura → itlwm v2.2.0  
- Sonoma → itlwm v2.3.0  
- Sequoia / Tahoe → itlwm v2.3.0 + HeliPort 2.0.0 alpha  

### Ventajas

- Mayor estabilidad en versiones nuevas  
- Compatibilidad más consistente  

### Limitaciones

- No utiliza el menú Wi-Fi nativo de macOS  
- Sin AirDrop  
- Sin Continuity  

---

## Método 2 — AirportItlwm.kext

Utiliza `IO80211Family`, funcionando como Wi-Fi nativo del sistema.

No requiere HeliPort.

### Versiones recomendadas

- Ventura → AirportItlwm v2.2.0  
- Sonoma < 14.4 → AirportItlwm v2.3.0  
- Sonoma 14.4 → AirportItlwm v2.3.0 específico para 14.4  
- Sequoia / Tahoe → pendiente actualización  

### Ventajas

- Integración con el menú Wi-Fi nativo  
- Soporte parcial de Continuity (Handoff, Universal Clipboard)  

### Limitaciones

- Sin AirDrop  
- No conecta a redes ocultas  
- Continuity incompleto  

---

## Verificación en Hackintool

La tarjeta es correctamente detectada:

<p align="left">
<img width="740" src="main/IMG/AX210 Hackintool.png">
</p>

---

# Instalación del módulo Bluetooth

En **macOS Monterey o superior** se requieren tres extensiones:

- `IntelBTPatcher.kext` (requiere Lilu 1.6.2+)  
- `IntelBluetoothFirmware.kext`  
- `BlueToolFixup.kext` (incluido en BrcmPatchRAM de Acidanthera)  

Estos kexts están disponibles en el proyecto **IntelBluetoothFirmware**.

---

# Instant Wake después de Sleep

Algunos sistemas presentan wake inmediato tras entrar en sleep al usar Bluetooth Intel.

Solución:

- Añadir `SSDT-GPRW.aml` a `EFI/OC/ACPI`  
- Añadir el patch ACPI `Change GPRW to XPRW`  

Desventaja: el sistema solo podrá despertar mediante el botón de encendido.

---

## SSDT-GPRW

```c++
DefinitionBlock ("", "SSDT", 2, "DRTNIA", "GPRW", 0x00000000)
{
    External (XPRW, MethodObj)

    Method (GPRW, 2, NotSerialized)
    {
        If (_OSI ("Darwin"))
        {
            If ((0x6D == Arg0))
            {
                Return (Package (0x02) { 0x6D, Zero })
            }

            If ((0x0D == Arg0))
            {
                Return (Package (0x02) { 0x0D, Zero })
            }
        }

        Return (XPRW (Arg0, Arg1))
    }
}
```

---

## Patch ACPI

```xml
<key>ACPI</key>
<dict>
  <key>Patch</key>
  <array>
    <dict>
      <key>Comment</key>
      <string>Change GPRW to XPRW, needs SSDT-GPRW.aml</string>
      <key>Enabled</key>
      <true/>
      <key>Find</key>
      <data>R1BSVwI=</data>
      <key>Replace</key>
      <data>WFBSVwI=</data>
    </dict>
  </array>
</dict>
```

---

# Nota sobre Hackintool

Hackintool muestra `itlwm` tanto cuando se usa `itlwm.kext` como `AirportItlwm.kext`, ya que ambos comparten nombre interno.

Puede modificarse editando:

```
/Applications/Hackintool.app/Contents/Resources/Kexts/kexts.plist
```

Cambiando `Name=itlwm` por `Name=AirportItlwm`.

### Antes

<p align="left">
<img width="660" src="main/IMG/itlwm name.png">
</p>

<p align="left">
<img width="440" src="main/IMG/Hackintool itlwm.png">
</p>

### Después

<p align="left">
<img width="660" src="main/IMG/AirportItlwm name.png">
</p>

<p align="left">
<img width="440" src="main/IMG/Hackintool AirportItlwm.png">
</p>

---

# Conclusión

La Intel AX210 es una alternativa sólida para:

- Usuarios que han perdido soporte Broadcom en Sonoma+  
- Usuarios que desean mantener seguridad completa sin root patches  
- Sistemas Hackintosh modernos  

Limitación principal: ausencia total de AirDrop y Continuity completo.
