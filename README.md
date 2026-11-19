# ChefKiss Inferno ![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/ChefKissInc/Inferno/build.yaml?style=for-the-badge) ![Written by humans, not AI](https://img.shields.io/badge/written_by_humans-not_ai-blue?style=for-the-badge)

ChefKiss Inferno is derivative of the QEMU project which provides Apple ARM device guest support.

## About this fork

This is an unofficial fork of ChefKiss Inferno (upstream archived on 2026-11-01), not affiliated with or endorsed by
the ChefKiss project. It is used to run iOS on the emulated iPhone 11 (`t8030`) for development and testing. The
upstream badge above describes upstream's code; the changes listed below were written with AI assistance (Claude),
as their `Co-Authored-By` lines show.

### Changes in this fork

| Change | Why |
|---|---|
| `target/arm`: EL1 accesses to TTBR0 (user) addresses use `SPRR_EL1BR0_EL1` | Every EL1 access used `SPRR_EL1BR1_EL1`. On iOS 26 this broke kernel copies into dyld's TPRO-protected memory, so the restore engine died at boot and never waited for a host. |
| `a13_gxf`: flush EL1 and guarded-mode (GXF) TLB entries when `SPRR_EL1BR0_EL1` is written | XNU rewrites the register on every context switch; cached translations kept the old permissions. |
| A13: the rest of the PMU registers (PMC2 and up, PMESR, OPMAT, OPMSK) as plain storage | iOS 26 saves and restores them; the missing registers stopped its kernel at boot. |
| Display pipe: raise `OUTPUT_READY` from a 60 Hz vblank timer | It was only raised when a display client redrew, so with VNC and no viewer connected iOS waited for a frame forever (a restore stalled after the upload). |
| Build without `ENABLE_DATA_ENCRYPTION` | Lets the machine start with a simulated SEP, without a dumped SEP ROM. This turns iOS data protection off in every configuration, also with real SEP emulation. |
| Touch controller (`mt-spi`): 30-byte path record (contact density, pitch, force, flags), edge flag, velocity in units per second clamped to INT16_MAX, 60 Hz reports, reports sent in order | The old 20-byte record and zero velocity were not enough for iOS to treat a drag as a gesture; queued reports could reach iOS in reverse order. Layout follows [MakrSas/Inferno](https://github.com/MakrSas/Inferno). Taps reach iOS 14; swipe to unlock is still being worked on. |

| Accept iOS 27 build versions | Lets the machine load iOS 27 images (they still do not boot, see below). |
| Kernel patch: allow an unencrypted Data volume | Without `ENABLE_DATA_ENCRYPTION`, restore creates the Data volume unencrypted, and iOS 26 APFS panics "unencrypted data volume is not allowed" when mounting it. |
| Simulated SEP: answer XNU's PRNG entropy request | iOS 26 asks the SEP for entropy at boot and waited forever without a reply. |
| Simulated SEP keystore: Get Device State (`0x19`) and Get Configuration (`0x23`) reply with a DER dictionary, padded to 4 bytes | Without a real reply the first boot looped on "Get Device State" forever, and an unpadded blob made the kernel driver reject the call (`0xe00002bc`). |
| Simulated SEP keystore: unhandled requests reply status 0 + an empty DER set | The old default echoed the request, which the driver read as an output-blob length, so several getters failed. |
| Simulated SEP credentials (ACM, `scrd`): command requests get a well-formed success reply; the init request gets an error | Without replies every ACM command waited a 5 s timeout. |

Tested with iOS 14.0 beta 5 (`18A5351d`): restore with real SEP emulation, first boot and USB pairing work. iOS 26.6.2
(`23G90`): restore completes, first boot runs data migration, and the second boot reaches the end of the progress bar;
Setup Assistant is not reached yet. iOS 27 does not boot: it needs SPTM and TXM, which Inferno does not emulate.

Known keystore failures left as they are (each happens once per boot):
- `aks_assert_consume` (selector 101) returns `0xe00002e2`: the kernel driver refuses it itself because no global
  assertion is held, which is normal outside a restore or update. The request never reaches the SEP.
- `aks_stash_load` (selector 34) returns `0xe00002bc`: no update stash exists because no software update was run. A real
  device should fail this call on a normal boot too, but that is not confirmed.

> [!CAUTION]
> Please consider donating/tipping to help continue the project's development,
> keeping it free and open-source.
>
> PayPal or credit/debit card: https://ko-fi.com/chefkiss
>
> BTC: `bc1qgu56kptepex2csuzl5nhzc4vxuj8c6ggjzhcem`
>
> SOL: `4PJU3iB5rimN9BNcpzduvmTnptJyedZiAd23so33SAdi`
>
> ETH: `0x038A25849c23Bc5A736484351a8B6Ad71bC46676`

## More info and usage

Please see [here](https://chefkiss.dev/applehax/inferno/) for more information about the project and how to use it.

## Legal Disclaimers

“ChefKiss Inferno” is provided "as is", without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose and noninfringement.

By using this software, you acknowledge that you are solely responsible for how you use it. The “ChefKiss Inferno” project team is not responsible for any damage, legal issues, data loss, or other consequences arising from its use.

This project is intended for lawful purposes only. Users are responsible for complying with all applicable laws, licenses, and agreements, including copyright, trademark, and End User License Agreements (EULAs).

The “ChefKiss Inferno” project team does not condone or support piracy, copyright infringement, or any illegal activity.

This derivative project (“ChefKiss Inferno”) is licensed under the GNU General Public License, version 3, with our own code licensed under the GNU Affero General Public License, version 3. See `LICENSE` for details.

This was done due to other entities shadowing the project with proprietary, paid solutions based mostly verbatim on this project.

The parts of QEMU which are licensed under the "version 2 or later" clause retain their original license restrictions and copyright holders.
The parts that are only GNU General Public License, version 2, with no "or later" clause, will be removed in subsequent commits.
You may explicitly contact us for early/faster removal of your code if desired.

Restrictions apply for the branding of the “ChefKiss Inferno” project, see [here](./ui/icons/CKBrandingNotice.md).

QEMU is a copyright and trademark of Fabrice Bellard. QEMU is unaffiliated with this project.

iOS is a copyright and trademark of Apple, Inc. Apple, Inc. is unaffiliated with this project. 
