# AuthLite (2FA on domain controllers)

If the domain uses AuthLite to require a second factor for DC logons, the rebuild has two concerns: **which account can do the work without a token**, and **the new DC must not serve logons before AuthLite is on it**. AuthLite behavior varies by version and configuration; confirm each point below against your AuthLite version's documentation and your own policy before relying on it.

## Which identity each step uses

| Step | Where it runs | Logon the guest sees | Account |
|---|---|---|---|
| Inventory, demotion (VMware source) | VMware console, interactive | Interactive | breakglass |
| Demotion (Proxmox source) | QGA → scheduled task | Batch logon as the task principal | breakglass as `/ru` |
| Credential passed to `Uninstall-`/`Install-ADDSDomainController`, `Add-Computer`, `Reset-ComputerMachinePassword` | Inside the guest | Network logon to a surviving DC | breakglass |
| Health checks, role install, other QGA commands | QGA | none — runs as `NT AUTHORITY\SYSTEM` | none |

QGA itself never logs a user on, so AuthLite does not prompt there. Every step that carries a domain credential does, which is why the reference scripts' `<DOMAIN>\Administrator` should be read as **the breakglass account** in an AuthLite domain.

## Breakglass account requirements
- Member of **Domain Admins** (enough to demote/promote an additional DC in its own domain and to reuse the existing computer account on join).
- Allowed to log on with one factor everywhere this flow needs it: interactive on the source DC, batch (scheduled task), and network logons to the surviving DCs. Verify this with a test logon **before** demoting — finding out mid-demotion is the worst time.
- Password lives in the operator's vault, is fed over stdin (`qga-execution.md`) or typed at the console, and is **rotated after the rebuild**. Treat the rebuild as an exposure of it.

## Before demotion
- Record the AuthLite version on the source DC and on the surviving DCs (V0 inventory lists installed software). The new DC should match the surviving DCs.
- Make sure the installer, license and any configuration needed for a new DC are on hand, not on the box being retired.
- Confirm the surviving DCs have AuthLite installed and enforcing. They carry all logons during the outage window.

## On the new DC — before it is in service
AuthLite enforcement depends on it being present on the DCs that handle the logon. A promoted DC without AuthLite advertises itself in DNS immediately and may authenticate users without the second factor. Close that window:
1. **Preferred, if your AuthLite version supports it:** install AuthLite on the freshly joined member server **before** `Install-ADDSDomainController`, so it is present from the first logon the DC serves.
2. **Otherwise:** install it immediately after the promotion reboot, before declaring the rebuild done. Keep that gap as short as possible.
3. Reboot if the installer requires it (`msiexec` exit `3010` = success, reboot required — see `qga-execution.md`).

**Gate (in addition to Phase 5's):**
- AuthLite installed at the expected version on the new DC; its services running.
- A test user in an AuthLite-enforced group is **refused** a one-factor logon against the new DC and **allowed** with the token (target the DC explicitly, e.g. log on to a test machine after `nltest /sc_reset:<domain>\<newDC>`).
- The breakglass account still works.

## After
Rotate the breakglass password from the operator's vault machine (`credential-handling.md`), and record the new DC in whatever AuthLite inventory or monitoring you keep.
