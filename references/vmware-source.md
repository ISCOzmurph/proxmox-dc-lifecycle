# VMware source — rebuild a VMware DC as a new Proxmox VM

Use this when the DC being rebuilt lives on VMware (ESXi / vCenter) and its replacement will be a **new** Proxmox VM with the **same computer name and IP**. Nothing is copied from VMware: the old DC is demoted in place, retired, and a fresh OS is installed on Proxmox and promoted as a replica. The Active Directory side (metadata cleanup, promotion, verification) is identical to the Proxmox-source flow; only the source-side steps and the VM creation differ.

There is no QEMU guest agent on the VMware side. Source-side steps run **at the VMware console (vSphere web console / VMRC), logged in interactively as the breakglass account** — see `authlite.md` for why that account and what it must be able to do. Proxmox-side steps use QGA exactly as in `qga-execution.md`.

## V0. Inventory the source (read-only)

Record these before touching anything. They become the `dcs[]` entry and the Proxmox VM shape.

From vSphere (UI, or `govc vm.info -json -e <vm>` / PowerCLI `Get-VM` from an operator machine):
- vCPU count, memory, disk count and sizes (the new VM needs a boot disk only, but size it comparably)
- firmware: EFI or BIOS (the new VM is OVMF/UEFI regardless; this is just for the record)
- port group and its **VLAN ID** for each NIC — this decides the Proxmox `bridge` + `tag`
- MAC address (only matters if something downstream binds to it: DHCP reservation, NAC, firewall object)

From inside the guest (console, as breakglass):
```powershell
hostname
Get-NetIPConfiguration | ? IPv4Address | % {
  "$($_.InterfaceAlias) $($_.IPv4Address.IPAddress)/$($_.IPv4Address.PrefixLength) gw=$($_.IPv4DefaultGateway.NextHop) dns=$(($_.DNSServer|?{$_.AddressFamily-eq2}).ServerAddresses-join',')"
}
(Get-TimeZone).Id
(Get-CimInstance Win32_OperatingSystem).Caption
Get-WindowsFeature | ? Installed | Select Name                 # roles beyond AD DS / DNS?
Get-ADDomainController -Identity $env:COMPUTERNAME | Select Site,IsGlobalCatalog,IsReadOnly,OperationMasterRoles
Get-SmbShare | ? Special -eq $false | Select Name,Path          # non-default shares
w32tm /query /configuration | Select-String 'Type|NtpServer'
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' |
  ? DisplayName | Select DisplayName,DisplayVersion | Sort DisplayName   # AuthLite version, agents, backup clients
```

**Stop and reassess** if the inventory shows anything a fresh replica won't recreate: FSMO roles (transfer them first), DHCP, AD CS, NPS, file shares, or other line-of-business roles. Those need their own migration plan before the box is demoted. Also note `IsGlobalCatalog` and `Site` so the promotion matches.

## V1. Plan the outage window

Between powering off the VMware DC (V4) and the Proxmox DC finishing promotion (Phase 5), **nothing answers on that IP**. Anything that points at it for DNS, LDAP, NTP or Kerberos by address will fail over to the other DCs or break. Before starting, confirm clients and servers have a second DNS server configured, and list any hard-coded references to the IP (firewall rules, appliances, LDAP binds, relays). A typical window is 1–2 hours.

## V2. Snapshot (optional) — and when you may never use it

A VMware snapshot taken **before demotion** is a rollback only for "demotion itself blew up the guest OS before AD changed." Once the demotion has replicated, or once metadata cleanup has run, **never revert it**: you would bring back a DC that AD has already forgotten, with a stale database. Delete the snapshot as soon as demotion is verified (also required before the VM can be cleanly retired from backup chains). If in doubt, skip the snapshot — the safety net in this flow is the healthy surviving DCs, not the VM image.

## V3. Demote on VMware (interactive, as breakglass)

Same cmdlet and parameters as `demotion.md`, but run interactively from an elevated PowerShell at the VMware console while logged in as the breakglass account. An interactive session is a real logon, so the SYSTEM/LDAP-error-58 problem and the scheduled-task workaround in `demotion.md` do not apply here.

```powershell
$cred = Get-Credential '<DOMAIN>\<breakglass>'
$la   = Read-Host -AsSecureString 'New local Administrator password for the demoted server'
Import-Module ADDSDeployment
Uninstall-ADDSDomainController -Credential $cred -LocalAdministratorPassword $la `
  -RemoveDnsDelegation:$false -SkipPreChecks -Force -NoRebootOnCompletion -WhatIf
# if the WhatIf resolves cleanly, run it for real:
Uninstall-ADDSDomainController -Credential $cred -LocalAdministratorPassword $la `
  -RemoveDnsDelegation:$false -SkipPreChecks -Force -NoRebootOnCompletion
Restart-Computer
```
The local Administrator password is throwaway — this server is about to be retired.

**Gate:** from a surviving DC, the target is absent from `Get-ADDomainController` and has no `nTDSDSA` object; `repadmin /replsummary` is `0 fails`. Then run `metadata-cleanup.md`. **Keep the computer object** (graceful demotion moves it to `CN=Computers`); delete only the empty server-object shell under Sites. Keep the DNS A record — the new DC reuses the IP.

## V4. Retire the VMware VM

Only after the V3 gate and metadata cleanup are green:
1. Shut the guest down (`Stop-Computer` at the console, or Shut Down Guest OS in vSphere).
2. In vSphere **Edit Settings**: on every network adapter, clear **Connect At Power On** (and Connected). The VM must not be able to reappear on the network with this name/IP.
3. Disable VM autostart on the host, and remove the VM from any backup/replication job (Veeam, etc.).
4. Rename it (e.g. `<name>-RETIRED-<yyyymmdd>`) and add an annotation pointing at the Proxmox VMID.
5. Do **not** delete it yet. Keep it powered off until the Proxmox DC has been healthy for an agreed period, then delete from disk.

**The retired VM must never boot with a connected NIC.** It would collide with the new DC's name and IP and present a stale machine account.

## V5. Create the Proxmox VM shell

A new, empty VM — not an import. Fill from the inventory and the environment config:

```bash
VMID=<new vmid>; NAME=<DC name>; STORE=<vm storage, e.g. local-zfs>
sudo qm create $VMID --name $NAME \
  --machine q35 --bios ovmf --ostype win11 --cpu host \
  --sockets 1 --cores <vcpu> --memory <MiB> \
  --scsihw virtio-scsi-single \
  --efidisk0 $STORE:1,efitype=4m,pre-enrolled-keys=1 \
  --tpmstate0 $STORE:1,version=v2.0 \
  --scsi0 $STORE:<GiB>,iothread=1,discard=on,ssd=1 \
  --net0 virtio,bridge=<bridge>,tag=<vlan> \
  --agent enabled=1 --onboot 1
sudo qm config $VMID | grep -E 'vmgenid|net0|scsi0|bios|efidisk|tpmstate'
```
- `ostype win11` is Proxmox's type for Server 2022/2025 too.
- Confirm a `vmgenid` is present (Proxmox generates one by default). AD's virtualization safeguards rely on it.
- Omit `tag=` if the VMware port group was untagged on the same physical network as `<bridge>`. The VMware VLAN ID and the Proxmox tag can differ if the uplinks' native VLANs differ — confirm with the network owner rather than copying the number blindly.
- MAC: let Proxmox generate one. The DC has a static IP, so the MAC is not identity. Only pass `virtio=<vmware-mac>,bridge=...` if a DHCP reservation, NAC policy or firewall object is bound to it — VMware's `00:50:56` prefix works fine on Proxmox, and the retired VM is off with its NIC disconnected.
- Add the new `{ name, vmid, ip }` to the environment config's `dcs[]`.

The disk is `scsi0` on VirtIO SCSI, so Setup needs **vioscsi** (the template injects both vioscsi and viostor). When following `os-reinstall.md`, substitute `scsi0` for `virtio0` in the boot order (`--boot order='ide2;scsi0'`, later `order='scsi0'`). There is nothing to wipe on a new VM, but keep the operator confirmation habit for the VMID.

## V6. Install, join, promote

Continue with the shared phases:
- `os-reinstall.md` from step 2 (image index, virtio paths, unattend ISO, attach, boot). Same name and IP in the unattend as the inventory.
- `promotion.md`. Because the computer object was kept but the new install has never had its secret, do an explicit **domain join + reboot before promoting** (this is the prevention path in `gotchas.md`):
  ```powershell
  # via QGA as SYSTEM, credential over stdin (qga-execution.md)
  Add-Computer -DomainName '<domain>' -Credential $cred -Force   # reuses the existing CN=Computers object
  ```
  Reboot, then confirm `Test-ComputerSecureChannel` = `True` before `Install-ADDSDomainController`. Reusing an existing computer account is restricted by Windows' account-reuse hardening (KB5020276); a Domain Admins member (the breakglass account) is allowed. If the join is refused, delete the computer object and rejoin, or prestage per `promotion.md` §0.
- Match the source's `Site` and Global Catalog setting in the promotion parameters.
- AuthLite on the new DC — `authlite.md`, before the DC is considered in service.

## V7. Finalize and decommission

Everything in Phase 6 of `SKILL.md`, plus:
- Update anything that referenced the old VM (monitoring, backup jobs now against Proxmox/PBS, CMDB).
- After the agreed healthy period: delete the retired VMware VM from disk and remove any leftover VMware snapshot.
