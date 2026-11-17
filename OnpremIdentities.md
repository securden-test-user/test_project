# Windows, Active Directory and Linux: what can be discovered and managed

Two paths exist in the product, and they do not feed the same list.

| Path | Where it runs | What it writes |
| --- | --- | --- |
| No agent on the target | Securden server talks to the target: ADSI for Active Directory, WinRM for Windows, SSH for Linux | Non-human identities in **Non-Human Identities** |
| Securden endpoint agent installed | The agent on that Windows computer runs the job and reports back | Accounts, local groups and dependencies in **Accounts**. It does not create NHI identities |

Installing the agent on a server does not change the NHI discovery. NHI still uses WinRM, ADSI or SSH. The agent continues to serve Accounts and privilege management.

Linux has no equivalent endpoint agent for this. A Linux landing server (Unix connector) is a jump host, not an agent installed on the target.

## Without an agent (NHI)

Nothing is installed on the domain controller, the Windows server or the Linux host. The account configured on the data source is the account used later to rotate, reconcile, disable and enable.

### Active Directory

Connection: the Securden server (Windows) queries the domain controller with the same ADSI connection used when a domain is added in Securden. **Use SSL** is on by default.

A user object is kept only when it looks like a service account. The score needs to reach **Minimum confidence** (default 50). Points come from a service principal name (50), a service-account OU (40), a name pattern such as `svc_` (30), "password never expires" (20), a service-like description (15), and no given name or surname (10). A normal person account below that score is not imported. Guest (RID 501) is skipped.

| Identity | When it is imported | Rotate password | Reconcile | Disable / enable |
| --- | --- | --- | --- | --- |
| AD service account (`CORP\svc_sql`) | User object at or above the confidence score | Yes | After a rotation, or when the last run needs reconcile | Yes |
| Group managed service account (gMSA) | `msDS-GroupManagedServiceAccount` | No. Active Directory rotates it | No | Yes |
| Standalone managed service account (sMSA) | `msDS-ManagedServiceAccount` | No. Active Directory rotates it | No | Yes |
| Computer account | Only when **Include computer accounts** is on | No. The member computer rotates it | No | Yes |
| KRBTGT | Always, when the search returns it | No. Needs the forest recovery procedure (two resets) | No | No |
| Trust account | Inter-domain trust objects | No. The trust relationship owns the password | No | No |

What else is recorded, but is not itself an identity you rotate:

* Risk on the identity: Kerberoastable, AS-REP roastable, privileged group membership, password not required, reversible encryption, unconstrained or constrained delegation, DES or RC4 only, password age, password never expires.
* Dependencies, when **Map dependencies on member servers** is on. Securden connects to those servers over WinRM with the AD account and reads services, scheduled tasks and IIS application pools that run as the domain account. Rotation then updates the ones that store the password. A task set to "run only when the user is logged on" is listed and is not updated.

### Windows servers

Connection: WinRM from the Securden server. The same **Use WinRM over HTTPS** setting as the rest of the product applies. The account must be a local administrator. No local accounts are read on a domain controller.

A local account is kept only when at least one of these is true: a service, task or IIS pool runs as it, it is in the local Administrators group, it is the built-in Administrator (RID 500), or its name matches a service-account pattern. Guest, DefaultAccount and WDAGUtilityAccount are skipped unless a workload runs as them. `LocalSystem`, `LocalService`, `NetworkService` and virtual accounts (`NT SERVICE\...`, `IIS APPPOOL\...`) are not identities.

| Identity | When it is imported | Rotate password | Reconcile | Disable / enable |
| --- | --- | --- | --- | --- |
| Windows local account (`WEB01\svc_local`) | Passes the rule above | Yes, by SID | After a rotation, or when the last run needs reconcile | Yes |
| Host | One per reachable server | No. It holds the certificate inventory | No | No |
| Domain account reference (`CORP\svc_sql` seen on a server, not yet in an AD data source) | A service, task or app pool runs as a domain account or a SID that is not a local account | No. The next AD sync moves the dependencies onto the real AD identity and retires the reference | No | No |

Machine certificates in `LocalMachine\My` are stored as secrets on the host identity. The private key is not exported, and renewal is not done from here.

On rotation, Securden updates Windows services, scheduled tasks that store a password, and IIS application pools that run as the account. Services keep the new password for the next start unless the API is called with `restart_services=true`.

### Linux devices

Connection: SSH from the Securden server. This is two steps.

1. The existing Linux discovery imports accounts it can see over SSH (and their private SSH keys, when key discovery is on) as Linux service-account identities. This step can go through a landing server.
2. **Assess service accounts and map workloads** then reads `/etc/passwd`, `/etc/shadow`, sudoers, systemd `User=`, cron, authorized-key fingerprints, the host keytab and certificates. This second step needs direct SSH. A data source that uses a landing server is discovered in step 1 and is not assessed or managed.

Until step 2 has run, Rotate, Reconcile, Disable and Enable stay hidden.

| Identity | When it is imported | Rotate password | Reconcile | Disable / enable |
| --- | --- | --- | --- | --- |
| Linux account with a password hash, including a locked hash | Found by Linux discovery, then assessed | Yes, through `chpasswd` as root or sudo | After a rotation, or when the last run needs reconcile | Yes, except `root` |
| Linux account with no password (key-only, `!` or `*`) | Found and assessed | No. There is no password to change | No | Yes, except `root` |
| Host | One per assessed device | No. It holds the certificate inventory | No | No |

What is mapped onto the account and is not updated during a password rotation: systemd units, cron jobs, authorized keys and sudo rules. They are there so you can see what breaks if the account is disabled. Authorized keys are fingerprinted only. The private key material comes from the existing Linux discovery, not from this assessment.

Certificates under the configured paths are stored on the host identity without the private key. Renewal is not done from here.

## With the Securden endpoint agent installed

The agent is the Windows privilege-management agent. When it is listening on a computer, **Accounts** uses it instead of a remote call for that computer. Those rows stay in Accounts. They are not added to Non-Human Identities, and the NHI actions (the on-prem rotate, reconcile, disable and enable) do not run through the agent.

| What the agent collects or does | Where it lands | What you can do with it |
| --- | --- | --- |
| Local user accounts | Accounts on that computer | Import, and then verify or reset the password through the agent |
| Local administrators | The computer's local admin list | Used by privilege management |
| Dependencies: Windows services, scheduled tasks, IIS application pools that use the account | Account dependencies | A password reset through the agent updates those services, tasks and app pools, and can restart services when that configuration is on |
| Password verify and password reset | The account in Accounts | Queued as an agent job. The agent runs it on the computer |

The agent does not discover Active Directory service accounts, gMSA, sMSA, KRBTGT, trust accounts or computer accounts. Those come only from the agentless AD data source.

The agent does not discover Linux accounts, SSH keys, systemd units, cron, sudo rules or Linux certificates. Linux stays on SSH.

If both are configured for the same Windows server:

* NHI still inventories the local service account, its workloads and its certificates over WinRM, and rotation from the Identities page uses WinRM.
* Accounts still resets that computer's vault account through the agent.

They do not share a password or a dependency list. Resetting in one place does not update the other.

## What is never managed from NHI

* A person account in Active Directory that scores under the confidence threshold.
* gMSA, sMSA, computer, KRBTGT and trust passwords. They are discovered so the risk is visible. Windows or the forest procedure changes the password.
* Certificate renewal, on Windows and on Linux.
* Workloads that do not store the password: "interactive only" scheduled tasks, systemd units, cron, authorized keys and sudo rules. Disabling the account still affects them.
* Linux hosts reached only through a landing server. Accounts can be imported. Rotate, reconcile, disable and enable are not offered.
* The domain-account reference and the host row. They exist to hold dependencies or certificates until the real account is known.
