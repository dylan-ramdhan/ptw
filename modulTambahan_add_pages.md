# MODUL TAMBAHAN — HALAMAN YANG MASIH STUB

> Dua halaman ini masih berupa placeholder di modul sebelumnya.
> Lengkapi dengan kode di bawah ini.

---

## BAGIAN A — JSEA Edit Page

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Edit.cshtml.cs`

```csharp
using System.Text.Json;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Web.Pages.JSEA;

[Authorize]
public class EditModel : PageModel
{
    private readonly IJSEARepository _jseaRepo;
    private readonly UserManager<ApplicationUser> _userManager;

    public EditModel(IJSEARepository jseaRepo, UserManager<ApplicationUser> userManager)
    {
        _jseaRepo    = jseaRepo;
        _userManager = userManager;
    }

    [BindProperty] public JSEAEditInput Input   { get; set; } = new();
    [BindProperty] public string DetailsJson    { get; set; } = "[]";
    public Domain.Entities.JSEA JSEA { get; set; } = null!;

    public class JSEAEditInput
    {
        public string WorkTitle       { get; set; } = string.Empty;
        public string Location        { get; set; } = string.Empty;
        public string? WorkDescription{ get; set; }
        public WorkType WorkType      { get; set; }
    }

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var jsea = await _jseaRepo.GetWithDetailsAsync(id);
        if (jsea == null) return NotFound();

        var user = await _userManager.GetUserAsync(User);
        if (jsea.RequesterId != user!.Id &&
            !User.IsInRole("SystemAdmin") && !User.IsInRole("SafetyManager"))
            return Forbid();

        if (jsea.Status != JSEAStatus.Draft && jsea.Status != JSEAStatus.RevisionRequired)
        {
            TempData["Error"] = "JSEA hanya bisa diedit dalam status Draft atau Revision Required.";
            return RedirectToPage("/JSEA/Detail", new { id });
        }

        JSEA  = jsea;
        Input = new JSEAEditInput
        {
            WorkTitle       = jsea.WorkTitle,
            Location        = jsea.Location,
            WorkDescription = jsea.WorkDescription,
            WorkType        = jsea.WorkType
        };

        // Serialize existing details untuk prefill tabel
        DetailsJson = JsonSerializer.Serialize(jsea.Details.OrderBy(d => d.StepNumber).Select(d => new
        {
            d.StepDescription, d.Hazard, d.HazardCategory,
            d.ControlMeasure, d.PPERequired,
            d.Severity, d.Likelihood, d.ResidualRisk, d.ResponsiblePerson
        }));

        return Page();
    }

    public async Task<IActionResult> OnPostAsync(int id)
    {
        var jsea = await _jseaRepo.GetWithDetailsAsync(id);
        if (jsea == null) return NotFound();
        JSEA = jsea;

        var user = await _userManager.GetUserAsync(User);
        if (jsea.RequesterId != user!.Id && !User.IsInRole("SystemAdmin"))
            return Forbid();

        // Parse details baru
        List<JSEADetail> newDetails = new();
        try
        {
            var raw = JsonSerializer.Deserialize<List<DetailInput>>(DetailsJson,
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            int step = 1;
            newDetails = raw?.Select(d => new JSEADetail
            {
                JSEAId          = id,
                StepNumber      = step++,
                StepDescription = d.StepDescription,
                Hazard          = d.Hazard,
                HazardCategory  = d.HazardCategory,
                ControlMeasure  = d.ControlMeasure,
                PPERequired     = d.PPERequired,
                Severity        = d.Severity,
                Likelihood      = d.Likelihood,
                ResidualRisk    = d.ResidualRisk,
                ResponsiblePerson = d.ResponsiblePerson,
                CreatedAt       = DateTime.UtcNow
            }).ToList() ?? new();
        }
        catch { ModelState.AddModelError("", "Format data hazard tidak valid."); return Page(); }

        // Update header
        jsea.WorkTitle        = Input.WorkTitle;
        jsea.Location         = Input.Location;
        jsea.WorkDescription  = Input.WorkDescription;
        jsea.WorkType         = Input.WorkType;
        jsea.UpdatedAt        = DateTime.UtcNow;
        jsea.UpdatedBy        = user.Id;

        // Replace details — hapus lama, masukkan baru
        jsea.Details.Clear();
        foreach (var d in newDetails) jsea.Details.Add(d);

        // Recalculate risk
        var maxScore = newDetails.Any() ? newDetails.Max(d => d.Severity * d.Likelihood) : 0;
        jsea.RiskLevel = maxScore switch
        {
            >= 15 => RiskLevel.Critical,
            >= 10 => RiskLevel.High,
            >= 5  => RiskLevel.Medium,
            _     => RiskLevel.Low
        };

        await _jseaRepo.UpdateAsync(jsea);
        TempData["Success"] = "JSEA berhasil diperbarui.";
        return RedirectToPage("/JSEA/Detail", new { id });
    }

    public class DetailInput
    {
        public string StepDescription   { get; set; } = string.Empty;
        public string Hazard            { get; set; } = string.Empty;
        public string HazardCategory    { get; set; } = string.Empty;
        public string ControlMeasure    { get; set; } = string.Empty;
        public string PPERequired       { get; set; } = string.Empty;
        public int Severity             { get; set; } = 1;
        public int Likelihood           { get; set; } = 1;
        public int ResidualRisk         { get; set; } = 1;
        public string? ResponsiblePerson{ get; set; }
    }
}
```

---

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Edit.cshtml`

```html
@page "{id:int}"
@model PTW.JSEA.Web.Pages.JSEA.EditModel
@{
    ViewData["Title"] = $"Edit JSEA #{Model.JSEA?.Id}";
    // Prefill data dari server ke JS
    var prefillJson = Model.DetailsJson;
}

<div class="page-header">
    <div>
        <h2 class="page-title">Edit JSEA</h2>
        <p class="text-sm text-gray-500">@(Model.JSEA?.WorkTitle)</p>
    </div>
    <a asp-page="/JSEA/Detail" asp-route-id="@Model.JSEA?.Id" class="btn-outline">← Kembali</a>
</div>

@if (Model.JSEA?.Status == PTW.JSEA.Domain.Enums.JSEAStatus.RevisionRequired &&
     !string.IsNullOrEmpty(Model.JSEA.ReviewComments))
{
    <div class="alert-warning mb-4">
        <svg class="w-5 h-5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
        </svg>
        <div>
            <p class="font-semibold">Catatan Revisi dari HSE:</p>
            <p>@Model.JSEA.ReviewComments</p>
        </div>
    </div>
}

<form method="post" id="editForm">
    @Html.AntiForgeryToken()
    <input type="hidden" asp-for="DetailsJson" id="detailsJsonInput"/>

    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div class="lg:col-span-2 space-y-6">

            <!-- Info Umum -->
            <div class="card">
                <h3 class="section-title">Informasi Umum</h3>
                <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <div class="md:col-span-2">
                        <label class="form-label">Judul Pekerjaan <span class="text-red-500">*</span></label>
                        <input asp-for="Input.WorkTitle" class="form-input" required/>
                    </div>
                    <div>
                        <label class="form-label">Jenis Pekerjaan <span class="text-red-500">*</span></label>
                        <select asp-for="Input.WorkType" class="form-select">
                            <option value="HotWork">Hot Work</option>
                            <option value="ColdWork">Cold Work</option>
                            <option value="ElectricalIsolation">Electrical Isolation</option>
                            <option value="ConfinedSpaceEntry">Confined Space Entry</option>
                            <option value="Excavation">Excavation</option>
                            <option value="Lifting">Lifting</option>
                            <option value="WorkingAtHeight">Working at Height</option>
                        </select>
                    </div>
                    <div>
                        <label class="form-label">Lokasi <span class="text-red-500">*</span></label>
                        <input asp-for="Input.Location" class="form-input" required/>
                    </div>
                    <div class="md:col-span-2">
                        <label class="form-label">Deskripsi</label>
                        <textarea asp-for="Input.WorkDescription" class="form-textarea" rows="3"></textarea>
                    </div>
                </div>
            </div>

            <!-- Hazard Table -->
            <div class="card">
                <div class="flex items-center justify-between mb-4">
                    <h3 class="section-title mb-0">Analisa Hazard</h3>
                    <button type="button" onclick="addRow()" class="btn-primary btn-sm">+ Tambah Baris</button>
                </div>
                <div class="overflow-x-auto">
                    <table class="min-w-full text-xs" id="hazardTable">
                        <thead class="bg-blue-50">
                            <tr>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold w-6">#</th>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold min-w-44">Langkah Kerja</th>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold min-w-36">Bahaya</th>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold min-w-32">Kategori</th>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold min-w-44">Pengendalian</th>
                                <th class="px-3 py-2 text-left text-blue-700 font-semibold min-w-28">APD</th>
                                <th class="px-3 py-2 text-center text-blue-700 font-semibold w-16">S</th>
                                <th class="px-3 py-2 text-center text-blue-700 font-semibold w-16">L</th>
                                <th class="px-3 py-2 text-center text-blue-700 font-semibold w-16">Score</th>
                                <th class="px-3 py-2 text-center text-blue-700 font-semibold w-16">Residu</th>
                                <th class="px-3 py-2 w-8"></th>
                            </tr>
                        </thead>
                        <tbody id="hazardBody" class="divide-y divide-gray-100 bg-white"></tbody>
                    </table>
                </div>
            </div>
        </div>

        <div class="space-y-4">
            <div class="card border-orange-200 bg-orange-50">
                <h4 class="font-semibold text-orange-800 mb-2 text-sm">⚠️ Perhatian</h4>
                <p class="text-xs text-orange-700">
                    Setelah disimpan, JSEA kembali ke status <strong>Draft</strong>.
                    Anda perlu submit ulang untuk direview HSE Officer.
                </p>
            </div>
            <button type="submit" onclick="prepareSubmit()" class="btn-primary w-full btn-lg">
                💾 Simpan Perubahan
            </button>
            <a asp-page="/JSEA/Detail" asp-route-id="@Model.JSEA?.Id"
               class="btn-outline w-full text-center block">Batal</a>
        </div>
    </div>
</form>

@section Scripts {
<script>
// ─── Prefill dari server ──────────────────────────────────────────────────
const prefillData = @Html.Raw(prefillJson ?? "[]");
const hazardCats  = ['Hot Work','Electrical','Mechanical','Chemical','Ergonomics','Height','Confined Space','Excavation','Manual Handling','Slip/Trip/Fall'];
let rowCounter    = 0;
const rows        = [];

function addRow(data = null) {
    rowCounter++;
    const id = rowCounter;
    rows.push(id);
    const tbody = document.getElementById('hazardBody');
    const catOpts = hazardCats.map(c => `<option ${data?.hazardCategory === c ? 'selected' : ''}>${c}</option>`).join('');
    const mkSel = (val, max=5) => Array.from({length:max},(_,i)=>i+1).map(n=>`<option value="${n}" ${n==val?'selected':''}>${n}</option>`).join('');

    const tr = document.createElement('tr');
    tr.id = `row-${id}`;
    tr.className = 'hover:bg-gray-50';
    tr.innerHTML = `
        <td class="px-3 py-1 text-gray-400">${rows.length}</td>
        <td class="px-3 py-1"><textarea class="form-textarea text-xs py-1" rows="2" style="min-width:160px"
            onchange="updateScore(${id})">${data?.stepDescription??''}</textarea></td>
        <td class="px-3 py-1"><textarea class="form-textarea text-xs py-1" rows="2" style="min-width:130px">${data?.hazard??''}</textarea></td>
        <td class="px-3 py-1"><select class="form-select text-xs">${catOpts}</select></td>
        <td class="px-3 py-1"><textarea class="form-textarea text-xs py-1" rows="2" style="min-width:160px">${data?.controlMeasure??''}</textarea></td>
        <td class="px-3 py-1"><input type="text" class="form-input text-xs" style="min-width:100px" value="${data?.ppeRequired??''}"/></td>
        <td class="px-3 py-1"><select class="form-select text-xs w-14" id="sev-${id}" onchange="calcScore(${id})">${mkSel(data?.severity??1)}</select></td>
        <td class="px-3 py-1"><select class="form-select text-xs w-14" id="lik-${id}" onchange="calcScore(${id})">${mkSel(data?.likelihood??1)}</select></td>
        <td class="px-3 py-1 text-center"><span id="sc-${id}" class="badge font-bold">${(data?.severity??1)*(data?.likelihood??1)}</span></td>
        <td class="px-3 py-1"><select class="form-select text-xs w-14" id="res-${id}">${mkSel(data?.residualRisk??1)}</select></td>
        <td class="px-3 py-1"><button type="button" onclick="removeRow(${id})" class="text-red-400 hover:text-red-600">✕</button></td>
    `;
    tbody.appendChild(tr);
    calcScore(id);
}

function calcScore(id) {
    const s = parseInt(document.getElementById(`sev-${id}`)?.value ?? 1);
    const l = parseInt(document.getElementById(`lik-${id}`)?.value ?? 1);
    const sc = s * l;
    const el = document.getElementById(`sc-${id}`);
    if (el) {
        el.textContent = sc;
        el.className = sc>=15?'risk-critical':sc>=10?'risk-high':sc>=5?'risk-medium':'risk-low';
    }
}

function removeRow(id) {
    const idx = rows.indexOf(id);
    if (idx > -1) rows.splice(idx, 1);
    document.getElementById(`row-${id}`)?.remove();
    document.querySelectorAll('#hazardBody tr').forEach((r,i) => {
        const fc = r.querySelector('td:first-child'); if(fc) fc.textContent = i+1;
    });
}

function prepareSubmit() {
    const details = [];
    document.querySelectorAll('#hazardBody tr').forEach(row => {
        const id = row.id?.split('-')[1];
        if (!id) return;
        const textareas = row.querySelectorAll('textarea');
        const selects   = row.querySelectorAll('select');
        const inputs    = row.querySelectorAll('input[type=text]');
        details.push({
            StepDescription : textareas[0]?.value || '',
            Hazard          : textareas[1]?.value || '',
            HazardCategory  : selects[0]?.value   || '',
            ControlMeasure  : textareas[2]?.value || '',
            PPERequired     : inputs[0]?.value     || '',
            Severity        : parseInt(selects[1]?.value || '1'),
            Likelihood      : parseInt(selects[2]?.value || '1'),
            ResidualRisk    : parseInt(selects[3]?.value || '1')
        });
    });
    document.getElementById('detailsJsonInput').value = JSON.stringify(details);
}

// Prefill existing data
if (prefillData && prefillData.length > 0)
    prefillData.forEach(d => addRow(d));
else
    addRow();
</script>
}
```

---

## BAGIAN B — User Edit Page

### FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Edit.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.AspNetCore.Mvc.Rendering;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Admin.Users;

[Authorize(Roles = "SystemAdmin,SafetyManager")]
public class EditModel : PageModel
{
    private readonly UserManager<ApplicationUser> _userManager;

    public EditModel(UserManager<ApplicationUser> userManager)
        => _userManager = userManager;

    [BindProperty] public UserEditInput Input { get; set; } = new();
    public string UserId { get; set; } = string.Empty;

    public List<SelectListItem> RoleOptions { get; set; } = DatabaseSeeder.SystemRoles
        .Select(r => new SelectListItem { Value = r, Text = r }).ToList();

    public class UserEditInput
    {
        public string FullName       { get; set; } = string.Empty;
        public string? EmployeeId   { get; set; }
        public string? Department   { get; set; }
        public string? Position     { get; set; }
        public string? PhoneWhatsApp{ get; set; }
        public string Role          { get; set; } = "Requester";
        public string? NewPassword  { get; set; }
    }

    public async Task<IActionResult> OnGetAsync(string id)
    {
        var user = await _userManager.FindByIdAsync(id);
        if (user == null) return NotFound();

        UserId = id;
        var roles = await _userManager.GetRolesAsync(user);
        Input = new UserEditInput
        {
            FullName      = user.FullName,
            EmployeeId    = user.EmployeeId,
            Department    = user.Department,
            Position      = user.Position,
            PhoneWhatsApp = user.PhoneWhatsApp,
            Role          = roles.FirstOrDefault() ?? "Requester"
        };
        return Page();
    }

    public async Task<IActionResult> OnPostAsync(string id)
    {
        var user = await _userManager.FindByIdAsync(id);
        if (user == null) return NotFound();

        UserId = id;
        if (!ModelState.IsValid) return Page();

        // Update profil
        user.FullName      = Input.FullName;
        user.EmployeeId    = Input.EmployeeId;
        user.Department    = Input.Department;
        user.Position      = Input.Position;
        user.PhoneWhatsApp = Input.PhoneWhatsApp;

        var updateResult = await _userManager.UpdateAsync(user);
        if (!updateResult.Succeeded)
        {
            foreach (var e in updateResult.Errors)
                ModelState.AddModelError("", e.Description);
            return Page();
        }

        // Update role
        var currentRoles = await _userManager.GetRolesAsync(user);
        await _userManager.RemoveFromRolesAsync(user, currentRoles);
        await _userManager.AddToRoleAsync(user, Input.Role);

        // Update password jika diisi
        if (!string.IsNullOrWhiteSpace(Input.NewPassword))
        {
            var token  = await _userManager.GeneratePasswordResetTokenAsync(user);
            var result = await _userManager.ResetPasswordAsync(user, token, Input.NewPassword);
            if (!result.Succeeded)
            {
                foreach (var e in result.Errors)
                    ModelState.AddModelError("", e.Description);
                return Page();
            }
        }

        TempData["Success"] = $"User {user.FullName} berhasil diperbarui.";
        return RedirectToPage("/Admin/Users/Index");
    }
}
```

---

### FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Edit.cshtml`

```html
@page "{id}"
@model PTW.JSEA.Web.Pages.Admin.Users.EditModel
@{
    ViewData["Title"] = "Edit User";
}

<div class="page-header">
    <h2 class="page-title">Edit User</h2>
    <a asp-page="/Admin/Users/Index" class="btn-outline">← Kembali</a>
</div>

<div class="card max-w-2xl">
    <form method="post">
        @Html.AntiForgeryToken()
        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div class="md:col-span-2">
                <label class="form-label">Nama Lengkap <span class="text-red-500">*</span></label>
                <input asp-for="Input.FullName" class="form-input" required/>
            </div>
            <div>
                <label class="form-label">Employee ID</label>
                <input asp-for="Input.EmployeeId" class="form-input" placeholder="MAT-XXX-001"/>
            </div>
            <div>
                <label class="form-label">Departemen</label>
                <input asp-for="Input.Department" class="form-input"/>
            </div>
            <div>
                <label class="form-label">Jabatan</label>
                <input asp-for="Input.Position" class="form-input"/>
            </div>
            <div>
                <label class="form-label">No. WhatsApp</label>
                <input asp-for="Input.PhoneWhatsApp" class="form-input" placeholder="08xx..."/>
            </div>
            <div>
                <label class="form-label">Role</label>
                <select asp-for="Input.Role" asp-items="Model.RoleOptions" class="form-select"></select>
            </div>
            <div>
                <label class="form-label">Password Baru
                    <span class="text-gray-400 font-normal text-xs">(kosongkan jika tidak diubah)</span>
                </label>
                <input asp-for="Input.NewPassword" type="password" class="form-input"
                       placeholder="Min 8 karakter..."/>
            </div>
        </div>

        <div class="alert-info mt-5">
            <svg class="w-4 h-4 flex-shrink-0 mt-0.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
            </svg>
            <p class="text-xs">Email tidak bisa diubah. Untuk mengubah email, buat user baru dan nonaktifkan yang lama.</p>
        </div>

        <div class="mt-6 flex gap-3">
            <button type="submit" class="btn-primary">💾 Simpan Perubahan</button>
            <a asp-page="/Admin/Users/Index" class="btn-outline">Batal</a>
        </div>
    </form>
</div>
```

---

> ✅ Semua stub page sudah dilengkapi.
> Tidak ada lagi "Coming soon" atau placeholder yang belum diisi.
