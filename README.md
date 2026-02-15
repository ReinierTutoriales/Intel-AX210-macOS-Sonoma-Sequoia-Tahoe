# Intel Wi-Fi 6 / 6E AX210 en macOS Sonoma, Sequoia y Tahoe (Hackintosh)

<p align="center">
  <img src="IMG/Wi-Fi.png">
</p>

---

## Objetivo

Esta guía explica cómo utilizar la **Intel AX210 (Wi-Fi 6E)** en **macOS Sonoma, Sequoia y Tahoe** sin aplicar **OpenCore Legacy Patcher (OCLP) root patches**, manteniendo:

- `SIP` activo (`csr-active-config = 00000000`)
- `SecureBootModel` activo
- Configuración de seguridad estándar de macOS

Está dirigida a usuarios que han perdido soporte Broadcom (ej. Fenvi T-919) en Sonoma+ o que desean mantener la seguridad completa del sistema sin depender de root patches.

---

## Qué cubre esta guía

- Instalación de Wi-Fi con `itlwm.kext` o `AirportItlwm.kext`
- Instalación de Bluetooth para AX210
- Solución al problema de *instant wake* (opcional)
- Compatibilidad real en Sonoma, Sequoia y Tahoe

No cubre:

- Instalación completa de OpenCore
- USB mapping
- Configuración de BIOS
- Instalación limpia de macOS

---

## Requisitos

- OpenCore actualizado
- Lilu actualizado
- USB correctamente mapeado (importante para Bluetooth)
- Haber revertido cualquier root patch previo de OCLP
- No utilizar Broadcom kexts simultáneamente

---

# Hardware

La AX210 puede adquirirse de dos formas:

### Opción 1 — Tarjeta PCIe ensamblada
Ejemplo: Intel AX210S PCIe (Ziyituod u otros)

### Opción 2 — Módulo + Adaptador
- Módulo: Intel AX210 M.2/NGFF (A+E Key)
- Adaptador: PCIe x1 → M.2/NGFF A+E

<table>
<tr>
<td><img src="IMG/Card%20and%20adapter.png"></td>
<td><img src="IMG/AX210%20card.jpg"></td>
</tr>
</table>

---

# Revertir configuración Broadcom / OCLP

Si anteriormente utilizabas Fenvi/Broadcom con OCLP root patches, debes revertir todo.

## En config.plist

Deshabilitar:

- `IOSkywalk.kext`
- `IO80211FamilyLegacy.kext`
- `AirPortBrcmNIC.kext`
- Cualquier bloqueo de `IOSkywalk`

Restaurar:

- `csr-active-config` → `00000000`
- `SecureBootModel` → valor distinto de `Disabled`

## En OCLP

OpenCore Legacy Patcher →  
Post-Install Root Patch →  
Revert Root Patches

Reiniciar antes de continuar.

---

# Instalación Wi-Fi

Los drivers pertenecen al proyecto **OpenIntelWireless**:

- itlwm: https://github.com/OpenIntelWireless/itlwm/releases  
- HeliPort: https://github.com/OpenIntelWireless/HeliPort/releases  

⚠️ No usar `itlwm.kext` y `AirportItlwm.kext` al mismo tiempo.

---

## Método 1 — itlwm.kext + HeliPort (Recomendado para máxima compatibilidad)

`itlwm.kext` implementa `IOEthernetController`.  
La conexión aparece como Ethernet pero funciona como Wi-Fi real.

Requiere la app **HeliPort** para gestionar redes.

### Compatibilidad comprobada

- Ventura → itlwm 2.2.0
- Sonoma → itlwm 2.3.0
- Sequoia → itlwm 2.3.0 + HeliPort 2.0.0 alpha
- Tahoe → itlwm 2.3.0 + HeliPort 2.0.0 alpha

### Ventajas

- Mayor estabilidad en sistemas nuevos
- Funciona en los tres sistemas mencionados

### Limitaciones

- No usa el menú Wi-Fi nativo
- Sin AirDrop
- Sin soporte AWDL
- Continuity no disponible

---

## Método 2 — AirportItlwm.kext (Integración nativa)

`AirportItlwm.kext` implementa `IO80211Family`.

No requiere HeliPort y utiliza el menú Wi-Fi nativo.

### Compatibilidad comprobada

- Ventura → 2.2.0
- Sonoma < 14.4 → 2.3.0
- Sonoma 14.4 → build específico 14.4
- Sequoia / Tahoe → actualmente no estable

### Ventajas

- Integración visual nativa
- Soporte parcial de Handoff y Universal Clipboard

### Limitaciones

- Sin AirDrop
- No redes ocultas
- Continuity parcial
- No estable en Sequoia/Tahoe actualmente

---

## Verificación

La AX210 debe aparecer correctamente en Hackintool:

<p align="left">
  <img width="740" src="IMG/AX210%20Hackintool.png">
</p>

---

# Instalación Bluetooth

Para **Monterey o superior** se requieren:

- `IntelBTPatcher.kext` (requiere Lilu 1.6.2+)
- `IntelBluetoothFirmware.kext`
- `BlueToolFixup.kext` (incluido en BrcmPatchRAM)

Descargas:

- IntelBluetoothFirmware: https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases  
- BrcmPatchRAM: https://github.com/acidanthera/BrcmPatchRAM  

⚠️ Bluetooth depende del mapeo correcto de USB.

---

# Instant Wake después de Sleep (Opcional)

Algunos sistemas despiertan inmediatamente al usar Bluetooth Intel.

Solución:

- Añadir `SSDT-GPRW.aml` a `EFI/OC/ACPI`
- Añadir patch `Change GPRW to XPRW`

Desventaja: solo se podrá despertar mediante botón de encendido.

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

# Limitaciones técnicas

- No soporte AWDL
- No AirDrop
- Continuity parcial o inexistente
- No es solución oficial Apple
- Depende del desarrollo activo de OpenIntelWireless

---

# Conclusión

La Intel AX210 es una alternativa estable y funcional para macOS Sonoma, Sequoia y Tahoe sin necesidad de OCLP root patches.

Permite mantener el modelo de seguridad completo del sistema y ofrece compatibilidad sólida en los tres sistemas, especialmente utilizando `itlwm.kext`.

Es una solución moderna, económica y adecuada para sistemas Hackintosh actuales.
