# Intel Wi-Fi 6 / 6E AX210 en macOS Sonoma, Sequoia y Tahoe

<p align="center">
  <img src="IMG/Wi-Fi.png" alt="Intel AX210 Wi-Fi"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/macOS-Sonoma%20%7C%20Sequoia%20%7C%20Tahoe-000?style=flat-square&logo=apple&logoColor=white"/>
  <img src="https://img.shields.io/badge/OpenCore-Compatible-2D3436?style=flat-square"/>
  <img src="https://img.shields.io/badge/SIP-Activo-27ae60?style=flat-square"/>
  <img src="https://img.shields.io/badge/SecureBootModel-Activo-27ae60?style=flat-square"/>
</p>

---

## Contexto

Desde **macOS Sonoma**, Apple eliminó los drivers para tarjetas Broadcom de modelos anteriores a 2017, incluyendo la **Fenvi T-919**, muy usada en sistemas Hackintosh.

OpenCore Legacy Patcher (OCLP) permite restaurar soporte mediante *root patches*, pero exige desactivar `SecureBootModel` y deshabilitar parcialmente `SIP`, reduciendo el nivel de seguridad del sistema.

Esta guía propone una alternativa con la **Intel AX210 Wi-Fi 6E**, compatible vía el proyecto **[OpenIntelWireless](https://github.com/OpenIntelWireless)**, que permite mantener:

| Parámetro | Estado |
|:---|:---|
| `SIP` | ✅ Activo (`csr-active-config = 00000000`) |
| `SecureBootModel` | ✅ Activo |
| Root patches (OCLP) | ✅ No requeridos |

**Sistemas operativos compatibles:** macOS Sonoma · macOS Sequoia · macOS Tahoe

---

## Requisitos previos

- OpenCore actualizado
- Lilu actualizado
- USB correctamente mapeado *(crítico para Bluetooth)*
- Root patches de OCLP revertidos (si aplica)
- Kexts Broadcom desactivados

---

## Hardware compatible

La AX210 se puede adquirir de dos formas:

**Opción A — Tarjeta PCIe ensamblada**
> Intel AX210S PCIe (Ziyituod u otros fabricantes) — lista para instalar directamente en slot PCIe x1.

**Opción B — Módulo + Adaptador**
> Intel AX210 M.2/NGFF (A+E Key) + Adaptador PCIe x1 → M.2/NGFF A+E.

<table>
<tr>
<td align="center"><img src="IMG/Card%20and%20adapter.png" alt="Módulo + Adaptador"/><br/><sub>Módulo + Adaptador</sub></td>
<td align="center"><img src="IMG/AX210%20card.jpg" alt="Tarjeta AX210"/><br/><sub>Tarjeta AX210 M.2</sub></td>
</tr>
</table>

---

## Paso 1 — Revertir Broadcom / OCLP

> ⚠️ Omitir este paso si nunca usaste Fenvi, Broadcom ni OCLP.

### En `config.plist`

**Deshabilitar los siguientes kexts:**
- `IOSkywalk.kext`
- `IO80211FamilyLegacy.kext`
- `AirPortBrcmNIC.kext`

**Deshabilitar los bloqueos de** `IOSkywalk`

**Restaurar valores de seguridad:**
- `csr-active-config` → `00000000`
- `SecureBootModel` → cualquier valor distinto de `Disabled`

### En OCLP

Ir a **Post-Install Root Patch → Revert Root Patches** y reiniciar.

---

## Paso 2 — Instalación Wi-Fi

Descargas oficiales:

| Componente | Repositorio |
|:---|:---|
| `itlwm.kext` / `AirportItlwm.kext` | [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm/releases) |
| HeliPort | [OpenIntelWireless/HeliPort](https://github.com/OpenIntelWireless/HeliPort/releases) |

> ⚠️ **No cargar `itlwm.kext` y `AirportItlwm.kext` simultáneamente.**

---

### Método 1 — `itlwm.kext` + HeliPort *(Recomendado)*

Implementa `IOEthernetController`. La conexión aparece como Ethernet pero opera como Wi-Fi real. HeliPort actúa como cliente de gestión de redes.

| macOS | Versión requerida |
|:---|:---|
| Ventura | itlwm 2.2.0 + HeliPort |
| Sonoma | itlwm 2.3.0 + HeliPort |
| Sequoia | itlwm 2.3.0 + HeliPort 2.0 alpha |
| Tahoe | itlwm 2.3.0 + HeliPort 2.0 alpha |

**✅ Ventajas**
- Estable en Sonoma, Sequoia y Tahoe
- Mayor compatibilidad futura
- No requiere build específico por versión de macOS

**⚠️ Limitaciones**
- Sin menú Wi-Fi nativo
- Sin AWDL / AirDrop / Continuity

---

### Método 2 — `AirportItlwm.kext`

Implementa `IO80211Family`, usando el **menú Wi-Fi nativo** de macOS.

| macOS | Estado |
|:---|:---|
| Ventura | ✅ Estable |
| Sonoma 14.x | ✅ Estable (build específico por versión) |
| Sequoia | ❌ No estable actualmente |
| Tahoe | ❌ No estable actualmente |

**⚠️ Limitaciones**
- Sin AirDrop / AWDL
- Continuity parcial
- No detecta redes ocultas
- Requiere actualizar el kext en cada actualización de macOS

---

### Verificación

<p align="left">
  <img width="740" src="IMG/AX210%20Hackintool.png" alt="Verificación en Hackintool"/>
</p>

---

## Paso 3 — Instalación Bluetooth

| Kext | Función |
|:---|:---|
| `IntelBTPatcher.kext` | Patcher de inicialización BT |
| `IntelBluetoothFirmware.kext` | Firmware del adaptador |
| `BlueToolFixup.kext` | Fix para macOS Monterey+ |

**Descargas:**
- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases)
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) *(BlueToolFixup)*

> ⚠️ El Bluetooth depende de un **USB mapping correcto**. Si BT no aparece, verificar el mapa USB antes de continuar.

---

## (Opcional) Fix Instant Wake tras Sleep

En algunos sistemas, los kexts de Bluetooth Intel provocan *instant wake*: el equipo entra en sleep pero despierta inmediatamente por una interrupción ACPI.

Issue de referencia: [#477 — OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/477)

### Solución — SSDT-GPRW

**1.** Añadir `SSDT-GPRW.aml` a `EFI/OC/ACPI`

**2.** Activar el siguiente patch en `config.plist → ACPI → Patch`:

```xml
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
```

**Código fuente del SSDT:**

```c
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

> ⚠️ **Limitación conocida:** Con este patch activo, el sistema solo puede despertar desde sleep mediante el **botón de encendido**. Se pierde wake por teclado y mouse.

---

## Limitaciones técnicas

| Función | Estado |
|:---|:---|
| AWDL | ❌ No soportado |
| AirDrop | ❌ No soportado |
| Continuity (Handoff, Universal Clipboard) | ⚠️ Parcial |
| Sidecar inalámbrico | ❌ No soportado |
| Menú Wi-Fi nativo (Sequoia/Tahoe) | ⚠️ Solo con `itlwm` + HeliPort |

---

## Conclusión

La Intel AX210 es la alternativa más sólida y técnicamente coherente para Wi-Fi en macOS Sonoma, Sequoia y Tahoe sin recurrir a OCLP root patches.

Mantiene el modelo de seguridad completo del sistema, funciona con `SIP` y `SecureBootModel` activos, y con `itlwm.kext` ofrece compatibilidad estable en los tres sistemas operativos actuales.

Para la mayoría de sistemas Hackintosh modernos, esta es la ruta recomendada.

---

<div align="center">
<sub>Guía por <a href="https://www.reiniertutoriales.com/">ReinierTutoriales</a> · Basada en <a href="https://github.com/OpenIntelWireless">OpenIntelWireless</a></sub>
</div>
