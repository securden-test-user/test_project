# Securden PAM — Oracle Smart View (Excel) Custom Application Launcher

Share this guide with anyone testing Oracle EPM / Hyperion Planning login from **Securden PAM** on a Windows PC.

This is the PAM equivalent of vault autofill into the **Oracle Cloud Sign in** window that appears from **Excel → Smart View**.

---

## 1. What this test proves

| In scope | Out of scope |
|---|---|
| PAM stores the Oracle user and password | Creating an Oracle EPM Cloud tenant |
| PAM launches Excel (optional) | Smart View inside a **macro-enabled** `.xlsm` workbook (Oracle does not support this) |
| PAM types username / password into the **Login** window | Retrieving Planning cubes if the Oracle user has no access |

**Do not** use a `.xlsm` / `.xlam` workbook as the Smart View file. Open a **blank** `.xlsx` workbook or Excel with no file. Smart View is a COM add-in on the ribbon, not an Office Store add-in.

---

## 2. Prerequisites (customer PC)

1. Microsoft Excel (desktop), typical path:  
   `C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE`
2. **Oracle Smart View for Office** installed; Excel shows a **Smart View** ribbon tab.  
   Confirm: **File → Options → Add-ins → Manage: COM Add-ins → Go** → **Oracle Smart View for Office** is ticked.
3. Securden **Windows Remote Launcher** installed and running on that PC (**Admin → General → Windows Remote Launcher**).
4. A valid Oracle EPM URL and user (from the customer). Private connection URL form:  
   `https://<epm-host>/HyperionPlanning/SmartView`  
   Shared connection form (optional):  
   `https://<epm-host>/workspace/SmartViewProviders`  
   Official URL syntax: [URL Syntax for Smart View Connections](https://docs.oracle.com/en/cloud/saas/enterprise-performance-management-common/cgsad/2_working_with_epm_clients_connect_url_syntax.html)
5. First-time only on that PC: create the Smart View **Private Connection** once (section 4). After that, Connect shows **Login**.

Confirm Excel KeyTips on **that** PC (they are not always `S` / `P`):

1. Open Excel.
2. Press **Alt**. Letters appear on the ribbon.
3. Note the letter under **Smart View** (often **S**).
4. Press that letter. Note the letter under **Panel** (often **P**).
5. Write those two letters down. Use them in Option B below.

Keyboard reference used for this guide: Alt (Excel KeyTips) → Smart View → Panel → Tab between fields → Enter to submit. See also [Configuring a Private Connection](https://docs.oracle.com/en/cloud/saas/enterprise-performance-management-common/cgsus/2_working_with_epm_clients_connect_pvt_conn_sv_configure.html).

---

## 3. Create the PAM account (once)

**Accounts → Add**

| Field | Value |
|---|---|
| Account name | Oracle Cloud **user name** (often an email) |
| Password | Oracle Cloud password |
| Account type | **Web Account** (or a dedicated type you will attach the launcher to) |
| URL (optional) | `https://<epm-host>/HyperionPlanning/SmartView` |

Save. Do **not** use **Open Connection** on the URL for this test. That opens a browser. Use the **Custom Application Launcher** from the account’s connection list.

---

## 4. One-time Smart View connection on the PC

Do this once so **Login** can appear later.

1. Open Excel (blank workbook).
2. **Smart View** ribbon → **Panel**.
3. Top of the panel: arrow → **Private Connections**.
4. **Create new connection** → **Smart View HTTP Provider**.
5. URL: `https://<customer-epm-host>/HyperionPlanning/SmartView`
6. **Next**. **Oracle Cloud Sign in** / **Login** appears. Sign in once by hand to confirm the URL and user work.
7. Finish naming the connection (for example `Oracle EPM`).

If **Login** never appears, the URL, Smart View add-in, or user access is wrong. Fix that before PAM autofill.

---

## 5. Custom Application Launchers (pick by scenario)

Create these under **Admin → Remote Sessions and Recordings → Custom Application Launchers → Add**.

Authentication for all options below:

- **Authenticate using Account Credentials**
- **Associate Account Type** = same type as the account (Web Account)
- Users: **All users** or the testers’ groups
- **Take connections through Remote Gateway** = off (unless the customer requires SSM)

Use these placeholders in fill rows:

- `{%ACCOUNT_NAME%}` — account name  
- `{%ACCOUNT_PASSWORD%}` — password  

---

### Option A — Recommended: Excel already open, Login already visible

Same idea as vault “fill the next window”. Most reliable in a customer demo.

| Field | Value |
|---|---|
| Application Profile Name | `Oracle Smart View — Fill Login` |
| Application Launch Type | **Autofill on next window** |
| Application file path | leave empty (hidden) |
| Input Form Name | leave empty |
| Total Time Delay | `3000` |

**Input Data Order**

| # | Operation | Value / Action | Delay (ms) |
|---|---|---|---|
| 1 | Clear and Fill Detail | `{%ACCOUNT_NAME%}` | `800` |
| 2 | Action to Perform | **TAB** | `800` |
| 3 | Clear and Fill Detail | `{%ACCOUNT_PASSWORD%}` | `800` |
| 4 | Action to Perform | **ENTER** | `300` |

Do **not** put a TAB row *before* username. The Login box is already focused on User name.

**How to run**

1. Excel is already open.
2. Smart View → Panel → Private Connections → the saved connection → **Connect**.
3. When **Login** is visible, click **User name**.
4. Alt+Tab so **Login** is the next window after PAM.
5. In PAM, open the Oracle account → launch **Oracle Smart View — Fill Login**.

---

### Option B — PAM opens Excel, then keyboard-opens Smart View Panel

Use when the tester should start from PAM only. PAM cannot “click” the ribbon; it sends the same keys as [Excel KeyTips](https://support.microsoft.com/office/use-the-keyboard-to-work-with-the-ribbon-e6d04fc8-e4d8-4b5b-9dcb-0e8a1a0a2f2e): **Alt**, then the Smart View letter, then Panel.

Confirm KeyTips first (section 2). If Smart View is not **S** or Panel is not **P**, change the **Value** letters in rows 2 and 3.

| Field | Value |
|---|---|
| Application Profile Name | `Oracle Smart View — Open Excel` |
| Application Launch Type | **Custom App** |
| Application file path | `C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE` |
| Arguments | `/x` |
| Input Form Name | `Login` |
| Total Time Delay | `12000` |

`/x` starts a **new** Excel instance so fill does not go into an old window.

**Input Data Order** (open Smart View, then fill Login when it appears)

| # | Operation | Value / Action | Delay (ms) |
|---|---|---|---|
| 1 | Action to Perform | **ALT**, Value = `S` | `800` |
| 2 | Fill Detail | `P` | `800` |
| 3 | Action to Perform | **ENTER** | `1000` |
| 4 | Clear and Fill Detail | `{%ACCOUNT_NAME%}` | `800` |
| 5 | Action to Perform | **TAB** | `800` |
| 6 | Clear and Fill Detail | `{%ACCOUNT_PASSWORD%}` | `800` |
| 7 | Action to Perform | **ENTER** | `300` |

**How to run**

1. Close extra Excel windows.
2. PAM account → launch **Oracle Smart View — Open Excel**.
3. Excel opens. After the delay, keys try **Alt, S, P**.
4. If a saved connection auto-prompts **Login**, rows 4–7 fill it.
5. If Panel opens but **Login** does not, the user must click **Connect** on the saved private connection, then use **Option A** for the fill (or raise Total Time Delay to `18000` and click Connect during the wait).

This option is **sensitive to ribbon KeyTips and window focus**. Prefer Option A for a customer demo unless you have verified Alt / S / P on that PC.

---

### Option C — PAM only opens Excel; user opens Login; then fill

Two profiles. Simplest split of “open app” vs “type password”.

**Profile C1 — open Excel only**

| Field | Value |
|---|---|
| Application Profile Name | `Open Excel` |
| Application Launch Type | **Open App** |
| Application file path | `C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE` |
| Arguments | `/x` |
| Input Form Name | *(not used)* |

No Input Data Order (Open App does not fill).

**Profile C2** = same as **Option A** (`Oracle Smart View — Fill Login`).

**How to run:** launch **Open Excel** → user clicks Smart View → Connect → Login → launch **Fill Login**.

---

### Option D — Native App (usually will not work)

| Field | Value |
|---|---|
| Application Launch Type | **Native App** |
| Application file path | `C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE` |
| Input Form Name | `Login` |

Native App looks up **Windows control names**. Oracle Cloud Sign in inside Smart View is a **web form** (WebView2), not Win32 text boxes. Use this only if a customer login is a real Windows dialog. For Oracle Cloud Sign in, use Option A or B.

---

### Option E — Browser only (not Excel Smart View)

Only if you need to prove fill against Oracle’s web login, not the add-in.

| Field | Value |
|---|---|
| Application Launch Type | **Microsoft Edge** or **Google Chrome** |
| URL | customer EPM URL, e.g. `https://<host>/epmcloud` |
| Total Time Delay | `5000` |

Same four fill rows as Option A (username, TAB, password, ENTER). This does **not** exercise Excel Smart View.

---

## 6. Which option to use when

| Situation | Use |
|---|---|
| Customer demo, Excel already running, Login on screen | **A** |
| Start from PAM, open Excel, then keys for Smart View | **B** |
| Tester will click Smart View by hand | **C** |
| Win32 login dialog (not Oracle Cloud web) | **D** |
| Browser Oracle login only | **E** |

Keep **A** and **C1** on the same account type so both appear on the account’s launch list.

---

## 7. Command line (optional check before PAM)

Blank Excel (not a macro workbook):

```text
"C:\Program Files\Microsoft Office\root\Office16\EXCEL.EXE" /x
```

Do not pass a `.xlsm` file. Smart View reports that it is not supported in macro-enabled workbooks.

---

## 8. Troubleshooting

| Symptom | Fix |
|---|---|
| No **Smart View** tab | Enable COM add-in **Oracle Smart View for Office**; restart Excel |
| `manifest.xml is not a valid add-in` | Wrong dialog. Developer → Excel Add-ins is for `.xlam` only. Ignore Office.js manifests |
| Smart View error on `.xlsm` | Use a blank workbook / `/x`, not a macro file |
| Login never appears | Create Private Connection (section 4); URL must end with `/HyperionPlanning/SmartView` |
| Fill goes into the sheet or Notepad | Alt+Tab so **Login** is the next window (Option A) |
| Username empty, password in User name | Extra TAB at the start; remove it |
| Password in User name, password box empty | TAB did not move; increase TAB delay to `1200` |
| Keys hit the Microsoft “Edit Office files on the web” dialog | Excel is unlicensed; close that dialog, then launch again |
| Alt, S, P does nothing | Press **Alt** in Excel and read the real KeyTips; update Option B values |
| **Open Connection** opens a browser | Use the launcher name on the account, not the URL button |
| Nothing fills | Windows Remote Launcher not running; approve the launcher if dual-control is on |

---

## 9. Customer checklist (copy)

- [ ] Excel path confirmed  
- [ ] Smart View ribbon visible  
- [ ] Remote Launcher running  
- [ ] EPM host URL received  
- [ ] Private Connection created once  
- [ ] PAM account created (Oracle user / password / Web Account)  
- [ ] Launcher **Oracle Smart View — Fill Login** (Option A) saved and approved  
- [ ] Optional: **Oracle Smart View — Open Excel** (Option B) after KeyTips verified  
- [ ] Test: Connect → Login visible → launch from PAM → both fields filled  

When the customer provides only a host name, the Private Connection URL is:

`https://<host>/HyperionPlanning/SmartView`
