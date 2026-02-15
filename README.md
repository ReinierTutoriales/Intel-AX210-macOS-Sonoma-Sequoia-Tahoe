# Intel Wi-Fi 6 / 6E AX210 en macOS Sonoma, Sequoia y Tahoe (Hackintosh)

<p align="center">
  <img src="IMG/Wi-Fi.png">
</p>

---

## Contexto

A partir de **macOS Sonoma**, Apple eliminó los drivers para varias tarjetas Broadcom utilizadas en modelos anteriores a 2017.  
Entre ellas, la **Fenvi T-919**, muy común en sistemas Hackintosh.

OpenCore Legacy Patcher (OCLP) permite restaurar soporte mediante *root patches*, pero esto requiere:

- `SecureBootModel` desactivado  
- `SIP` parcialmente deshabilitado  

Esto reduce el nivel de seguridad del sistema.

Esta guía propone una alternativa moderna utilizando la **Intel AX210 Wi-Fi 6E**, compatible gracias al proyecto **OpenIntelWireless**, permitiendo mantener:

- `SIP` activo (`csr-active-config = 00000000`)
- `SecureBootModel` activo
- Modelo de seguridad estándar de macOS

La solución funciona en:

- macOS Sonoma  
- macOS Sequoia  
- macOS Tahoe  

Sin aplicar root patches.

---

## Requisitos

- OpenCore actualizado  
- Lilu actualizado  
- USB correctamente mapeado (importante para Bluetooth)  
- Haber revertido cualquier root patch previo de OCLP  
- No cargar kexts Broadcom simultáneamente  

---

# Hardware

La AX210 puede adquirirse:

### Tarjeta PCIe ensamblada
Intel AX210S PCIe (Ziyituod u otros fabricantes)

### Módulo + Adaptador
- Intel AX210 M.2/NGFF (A+E Key)
- Adaptador PCIe x1 → M.2/NGFF A+E

<table>
<tr>
<td><img src="IMG/Card%20and%20adapter.png"></td>
<td><img src="IMG/AX210%20card.jpg"></td>
</tr>
</table>

---

# Revertir Broadcom / OCLP

Si utilizabas Fenvi o Broadcom con OCLP:

## En config.plist

Deshabilitar:

- `IOSkywalk.kext`
- `IO80211FamilyLegacy.kext`
- `AirPortBrcmNIC.kext`
- Bloqueos de `IOSkywalk`

Restaurar:

- `csr-active-config` → `00000000`
- `SecureBootModel` → valor distinto de `Disabled`

## En OCLP

Post-Install Root Patch → Revert Root Patches

Reiniciar.

---

# Instalación Wi-Fi

Proyecto oficial:

- itlwm: https://github.com/OpenIntelWireless/itlwm/releases  
- HeliPort: https://github.com/OpenIntelWireless/HeliPort/releases  

⚠️ No usar `itlwm.kext` y `AirportItlwm.kext` simultáneamente.

---

## Método 1 — itlwm.kext + HeliPort (Recomendado)

Implementa `IOEthernetController`.

La conexión aparece como Ethernet pero opera como Wi-Fi real.

Funciona en:

- Ventura (2.2.0)
- Sonoma (2.3.0)
- Sequoia (2.3.0 + HeliPort 2.0 alpha)
- Tahoe (2.3.0 + HeliPort 2.0 alpha)

Ventajas:
- Estable en los tres sistemas
- Mayor compatibilidad futura

Limitaciones:
- No menú Wi-Fi nativo
- No AWDL
- No AirDrop
- Sin Continuity

---

## Método 2 — AirportItlwm.kext

Implementa `IO80211Family`.

Usa el menú Wi-Fi nativo.

Compatible:

- Ventura
- Sonoma (incluido 14.4 con build específico)

No estable actualmente en Sequoia/Tahoe.

Limitaciones:
- Sin AirDrop
- Sin AWDL
- Continuity parcial
- No redes ocultas

---

## Verificación

<p align="left">
  <img width="740" src="IMG/AX210%20Hackintool.png">
</p>

---

# Instalación Bluetooth

Requiere:

- `IntelBTPatcher.kext`
- `IntelBluetoothFirmware.kext`
- `BlueToolFixup.kext`

Descargas:

- https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases  
- https://github.com/acidanthera/BrcmPatchRAM  

Bluetooth depende del USB mapping correcto.

---

# Instant Wake después de Sleep (Opcional)

Algunos sistemas pueden experimentar el comportamiento de *instant wake* al utilizar Bluetooth Intel junto con los kexts de OpenIntelWireless: el equipo entra en sleep pero despierta inmediatamente.

Este comportamiento ha sido reportado en el repositorio oficial:
https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/477

El desarrollador no lo considera un problema directo del kext, pero en ciertos sistemas ocurre debido a la gestión ACPI de los métodos de wake.

## Solución

1. Añadir `SSDT-GPRW.aml` a `EFI/OC/ACPI`
2. Activar el patch ACPI:  
   `Change GPRW to XPRW, needs SSDT-GPRW.aml`

Con este ajuste el sistema entra correctamente en sleep.

### Limitación

Después de aplicar el patch:

- El sistema solo podrá despertar mediante el botón de encendido.
- Se pierde wake por teclado o mouse.

Es un compromiso aceptable si se desea conservar la funcionalidad de sleep.

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

# Limitaciones Técnicas

- No soporte AWDL
- No AirDrop
- Continuity limitado
- No es solución oficial Apple
- Depende del desarrollo activo de OpenIntelWireless

---

# Conclusión

La Intel AX210 es una alternativa sólida y estable para macOS Sonoma, Sequoia y Tahoe sin recurrir a OCLP root patches.

Permite mantener el modelo de seguridad completo del sistema y ofrece compatibilidad real en los tres sistemas, especialmente utilizando `itlwm.kext`.

Es una solución moderna, económica y técnicamente coherente para sistemas Hackintosh actuales.
