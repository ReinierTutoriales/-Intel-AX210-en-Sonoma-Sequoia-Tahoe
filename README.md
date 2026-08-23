# Intel Wi-Fi 6 / 6E AX210 en macOS Sonoma, Sequoia y Tahoe

<p align="center">
  <img src="IMG/Wi-Fi.png" alt="Intel AX210 Wi-Fi" width="80"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/macOS-Sonoma%20%7C%20Sequoia%20%7C%20Tahoe-000000?style=for-the-badge&logo=apple&logoColor=white"/>
  <img src="https://img.shields.io/badge/OpenCore-Compatible-161b22?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/SIP-Activo-27ae60?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/SecureBootModel-Activo-27ae60?style=for-the-badge"/>
</p>

---

## Contexto

Desde **macOS Sonoma**, Apple eliminó el soporte nativo para tarjetas Broadcom anteriores a 2017 (incluida la Fenvi T-919).

OpenCore Legacy Patcher (OCLP) restaura ese soporte mediante *root patches*, pero obliga a:

- Desactivar `SecureBootModel`
- Relajar `SIP` (`csr-active-config` ≠ `00000000`)

Esto reduce la seguridad del sistema.

**Esta guía** usa la **Intel AX210 (Wi-Fi 6E)** con el proyecto [OpenIntelWireless](https://github.com/OpenIntelWireless). Permite mantener el modelo de seguridad completo:

| Parámetro              | Estado                          |
|------------------------|---------------------------------|
| `SIP`                  | Activo (`csr-active-config = 00000000`) |
| `SecureBootModel`      | Activo                          |
| Root patches (OCLP)    | No requeridos                   |

**Sistemas soportados:** macOS Sonoma · Sequoia · Tahoe

---

## Requisitos previos

Antes de empezar, verifica lo siguiente:

- [ ] OpenCore actualizado (versión reciente)
- [ ] Lilu actualizado
- [ ] Mapa USB correcto y funcional (**crítico para Bluetooth**)
- [ ] Root patches de OCLP revertidos (si los usabas)
- [ ] Kexts de Broadcom deshabilitados

---

## Hardware compatible

La AX210 se puede montar de dos formas:

| Opción | Descripción |
|--------|-------------|
| **A — Tarjeta PCIe** | Intel AX210S PCIe (Ziyituod u otros). Lista para slot PCIe x1. |
| **B — Módulo + Adaptador** | Intel AX210 M.2/NGFF (A+E Key) + adaptador PCIe x1 → M.2 A+E. |

<table>
  <tr>
    <td align="center">
      <img src="IMG/Card%20and%20adapter.png" alt="Módulo + Adaptador" width="380"/><br/>
      <sub>Módulo + Adaptador</sub>
    </td>
    <td align="center">
      <img src="IMG/AX210%20card.jpg" alt="Tarjeta AX210" width="380"/><br/>
      <sub>Tarjeta AX210 M.2</sub>
    </td>
  </tr>
</table>

---

## Paso 1 — Revertir Broadcom / OCLP

> **Omitir este paso** si nunca usaste Fenvi, Broadcom ni OCLP.

### 1.1 En `config.plist`

**Deshabilitar estos kexts** (si existen):

- `IOSkywalk.kext`
- `IO80211FamilyLegacy.kext`
- `AirPortBrcmNIC.kext`

**Deshabilitar** cualquier bloqueo (`Kernel → Block`) relacionado con `IOSkywalk`.

**Restaurar seguridad:**

```xml
<!-- NVRAM → Add → 7C436110-AB2A-4BBB-A880-FE41995C9F82 -->
<key>csr-active-config</key>
<data>AAAAAA==</data>   <!-- 00000000 -->
```

```xml
<!-- Misc → Security -->
<key>SecureBootModel</key>
<string>Default</string>   <!-- o cualquier valor distinto de Disabled -->
```

### 1.2 En OpenCore Legacy Patcher

1. Abrir OCLP
2. **Post-Install Root Patch → Revert Root Patches**
3. Reiniciar

---

## Paso 2 — Instalación Wi-Fi

### Descargas oficiales

| Componente                    | Enlace |
|-------------------------------|--------|
| `itlwm.kext` / `AirportItlwm.kext` | [OpenIntelWireless/itlwm](https://github.com/OpenIntelWireless/itlwm/releases) |
| HeliPort                      | [OpenIntelWireless/HeliPort](https://github.com/OpenIntelWireless/HeliPort/releases) |

> **Importante:** No cargar `itlwm.kext` y `AirportItlwm.kext` al mismo tiempo.

---

### Método 1 — `itlwm.kext` + HeliPort (recomendado)

Implementa `IOEthernetController`. La interfaz aparece como Ethernet, pero es Wi-Fi real. HeliPort gestiona las redes.

| macOS     | Versión requerida              |
|-----------|--------------------------------|
| Ventura   | itlwm 2.2.0 + HeliPort         |
| Sonoma    | itlwm 2.3.0 + HeliPort         |
| Sequoia   | itlwm 2.3.0 + HeliPort 2.0 alpha |
| Tahoe     | itlwm 2.3.0 + HeliPort 2.0 alpha |

**Ventajas**
- Estable en Sonoma, Sequoia y Tahoe
- Mejor compatibilidad futura
- No necesita build específico por versión menor de macOS

**Limitaciones**
- No hay menú Wi-Fi nativo de macOS
- Sin AWDL → sin AirDrop ni Continuity completo

**Orden de carga recomendado (Kernel → Add):**

```text
1. Lilu.kext
2. itlwm.kext
```

Después de reiniciar, abre **HeliPort** para conectar a redes.

---

### Método 2 — `AirportItlwm.kext`

Implementa `IO80211Family` y usa el **menú Wi-Fi nativo** de macOS.

| macOS        | Estado                                      |
|--------------|---------------------------------------------|
| Ventura      | Estable                                     |
| Sonoma 14.x  | Estable (build específico por versión)      |
| Sequoia      | No estable actualmente                      |
| Tahoe        | No estable actualmente                      |

**Limitaciones**
- Sin AirDrop / AWDL
- Continuity parcial
- No detecta redes ocultas
- Hay que actualizar el kext en cada actualización de macOS

**Orden de carga recomendado:**

```text
1. Lilu.kext
2. AirportItlwm.kext
```

---

### Verificación Wi-Fi

Usa **Hackintool → PCIe** o **System Information → Wi-Fi / Ethernet** para confirmar que el dispositivo se detecta.

<p align="center">
  <img src="IMG/AX210%20Hackintool.png" alt="Verificación en Hackintool" width="700"/>
</p>

| Kext              | Cómo se ve en Hackintool / Sistema      |
|-------------------|-----------------------------------------|
| `itlwm.kext`      | Interfaz tipo Ethernet                  |
| `AirportItlwm.kext` | Dispositivo Wi-Fi nativo              |

---

## Paso 3 — Instalación Bluetooth

| Kext                          | Función                              |
|-------------------------------|--------------------------------------|
| `IntelBTPatcher.kext`         | Patch de inicialización Bluetooth    |
| `IntelBluetoothFirmware.kext` | Firmware del adaptador               |
| `BlueToolFixup.kext`          | Fix necesario desde Monterey+        |

**Descargas:**

- [OpenIntelWireless/IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/releases)
- [acidanthera/BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) → solo `BlueToolFixup.kext`

> **Crítico:** Bluetooth depende de un **mapa USB correcto**. Si el dispositivo no aparece, revisa primero el USB mapping.

**Orden de carga recomendado (Kernel → Add):**

```text
1. Lilu.kext
2. IntelBTPatcher.kext
3. IntelBluetoothFirmware.kext
4. BlueToolFixup.kext
```

---

## (Opcional) Fix Instant Wake tras Sleep

En algunos equipos, los kexts de Bluetooth Intel provocan *instant wake*: el Mac entra en sleep y se despierta de inmediato por una interrupción ACPI.

Referencia: [Issue #477 — IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware/issues/477)

### Solución — SSDT-GPRW

**1.** Compila y coloca `SSDT-GPRW.aml` en `EFI/OC/ACPI`.

**2.** Añade este patch en `config.plist → ACPI → Patch`:

```xml
<dict>
    <key>Comment</key>
    <string>Change GPRW to XPRW (SSDT-GPRW.aml)</string>
    <key>Enabled</key>
    <true/>
    <key>Find</key>
    <data>R1BSVwI=</data>
    <key>Replace</key>
    <data>WFBSVwI=</data>
</dict>
```

**Código fuente del SSDT (SSDT-GPRW.dsl):**

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
                Return (Package (0x02)
                {
                    0x6D,
                    Zero
                })
            }

            If ((0x0D == Arg0))
            {
                Return (Package (0x02)
                {
                    0x0D,
                    Zero
                })
            }
        }

        Return (XPRW (Arg0, Arg1))
    }
}
```

> **Limitación:** Con este patch activo, el wake desde sleep solo funciona con el **botón de encendido**. Se pierde wake por teclado y ratón.

---

## Limitaciones técnicas

| Función                              | Estado                          |
|--------------------------------------|---------------------------------|
| AWDL                                 | No soportado                    |
| AirDrop                              | No soportado                    |
| Continuity (Handoff, Clipboard)      | Parcial                         |
| Sidecar inalámbrico                  | No soportado                    |
| Menú Wi-Fi nativo (Sequoia / Tahoe)  | Solo con `itlwm` + HeliPort     |

---

## Resumen rápido

| Objetivo                    | Acción recomendada                          |
|-----------------------------|---------------------------------------------|
| Máxima estabilidad          | `itlwm.kext` + HeliPort                     |
| Menú Wi-Fi nativo (Sonoma)  | `AirportItlwm.kext` (build correcto)        |
| Mantener SIP + SecureBoot   | No usar OCLP root patches                   |
| Bluetooth                   | IntelBTPatcher + IntelBluetoothFirmware + BlueToolFixup |
| Instant wake                | SSDT-GPRW + patch ACPI                      |

---

## Conclusión

La **Intel AX210** es actualmente la opción más sólida para Wi-Fi en macOS Sonoma, Sequoia y Tahoe sin sacrificar seguridad.

- Mantiene `SIP` y `SecureBootModel` activos
- No requiere root patches de OCLP
- Con `itlwm.kext` + HeliPort ofrece compatibilidad estable en los tres sistemas

Para la mayoría de Hackintosh modernos, esta es la ruta recomendada.

---

<div align="center">
  <sub>
    Guía por <a href="https://www.reiniertutoriales.com/">ReinierTutoriales</a> ·
    Basada en <a href="https://github.com/OpenIntelWireless">OpenIntelWireless</a>
  </sub>
</div>
