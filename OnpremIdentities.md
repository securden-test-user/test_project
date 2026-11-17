On-premises Active Directory discovery finds six kinds of directory objects. Only a service account password is rotated from Securden. The Securden server has to be Windows, because the connection is the same ADSI connection used when a domain is added.

| Identity | When it is imported | Rotate | Reconcile | Disable / enable |
| --- | --- | --- | --- | --- |
| AD service account, such as `CORP\svc_sql` | User object whose score reaches Minimum confidence (default 50) | Yes | Yes, after a rotation or a failed verify | Yes |
| Group managed service account (gMSA) | Object class `msDS-GroupManagedServiceAccount` | No. Active Directory rotates it | No | Yes |
| Standalone managed service account (sMSA) | Object class `msDS-ManagedServiceAccount` | No. Active Directory rotates it | No | Yes |
| Computer account | Only when **Include computer accounts** is selected | No. The member computer rotates it | No | Yes |
| KRBTGT | The built-in `krbtgt` account, when the search returns it | No | No | No |
| Trust account | An inter-domain trust object | No | No | No |

A normal person account below the score is not imported. Guest (RID 501), DefaultAccount (503), and WDAGUtilityAccount (504) are skipped.

A user object is scored like this. It is kept only when the total reaches 50.

| Signal | Points |
| --- | --- |
| Has a service principal name | 50 |
| Sits in a service-account OU (`OU=Service Accounts`, `OU=Service`, `OU=svc`, or an OU you type on the form) | 40 |
| Name matches a service pattern (`svc_`, `sql-`, `app_`, and the other built-in patterns) | 30 |
| Password never expires | 20 |
| Description contains a word such as service, application, batch, or scheduled | 15 |
| No given name and no surname | 10 |

An SPN alone is enough. So is a service OU plus a `svc_` name (40 + 30).

The password Securden manages is a secret on that identity, type `ONPREM_MANAGED_CREDENTIAL`. Rotation writes the secret first, then sets the password in Active Directory and checks it by signing in as that account. If Active Directory rejects the reset, the previous secret value is put back.

## Create them in a lab domain

Use a test OU and a test account. Do not do this against production `krbtgt` or a live trust.

**Service account that will be imported and rotated.** An SPN makes the score 50 by itself.

```powershell
New-ADUser -Name "svc_sql" -SamAccountName "svc_sql" -UserPrincipalName "svc_sql@corp.example.com" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -Enabled $true -PasswordNeverExpires $true -Description "SQL service account"
Set-ADUser svc_sql -ServicePrincipalNames @{Add="MSSQLSvc/sql01.corp.example.com:1433"}
```

**User that should not appear.** Give it a first and last name, no SPN, and put it in a normal Users OU. Its score stays under 50.

**gMSA.** The domain needs a KDS root key first. In a lab you can backdate it so it is usable immediately.

```powershell
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
New-ADServiceAccount -Name "gmsa_sql" -DNSHostName "gmsa_sql.corp.example.com" `
  -PrincipalsAllowedToRetrieveManagedPassword "SQLServers"
```

**sMSA.** This is tied to one computer.

```powershell
New-ADServiceAccount -Name "smsa_app" -RestrictToSingleComputer
```

**Computer.** Turn on **Include computer accounts**, or create one with `New-ADComputer -Name "TEST01"`.

**KRBTGT and trust.** Every domain already has `krbtgt`. A trust account appears only when a domain trust exists. Discover them so you can see them. Do not rotate or disable them from this test.

The account you type on the data source needs permission to query the domain, reset the test service account password, and change the disabled flag.

## Test it in Securden

1. Open **Non-Human Identities > Data Sources > Active Directory**.
2. Set a profile name and the domain controller address. Add a second controller only if you want the failover path.
3. Leave **SSL** off unless that controller is listening on port 636 with a certificate this server trusts.
4. Under **Supply Administrator Credentials**, type the query account or pick a Windows domain account already stored in Securden.
5. Put the test OU in **Service Account OUs** if the account does not already have an SPN. Select **Include computer accounts** only when you want computers.
6. Click **Next**. The application log should show `NHI_ONPREM_AD sync started` and then `sync completed`, with a count in `by_kind`.

Then check **Identities**:

* `CORP\svc_sql` is type **AD Service Account**. **Rotate password** stores a new secret and changes the password in Active Directory. In Active Directory, `pwdLastSet` moves to now. **Reconcile password** pushes that vault password again. **Disable account** sets only the disabled bit, so "password never expires" stays set. **Enable account** clears that bit.
* `gmsa_sql$` and `smsa_app$` show disable and enable, and no rotate action.
* `TEST01$` appears only when computer accounts were included. It can be disabled, not rotated.
* `krbtgt` and a trust account are listed with no rotate, reconcile, disable, or enable action.

A user such as `john` with a given name and no SPN should be absent. If `svc_sql` is missing, open its identity after lowering **Minimum Classification Confidence** under advanced settings, or add the SPN. A search that hits **Maximum Directory Objects** (default 20000) logs `truncated: true` and does not mark the accounts it did not read as removed.
