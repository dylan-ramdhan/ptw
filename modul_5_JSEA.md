# MODUL 5 — JSEA MODULE LENGKAP

> Create, List, Detail, Review, Approve — Wizard 4 langkah dengan hazard table dinamis.

---

## BAGIAN A — JSEA Service

### FILE: `src/PTW.JSEA.Application/Services/IJSEAService.cs`

```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Application.Services;

public interface IJSEAService
{
    Task<JSEA> CreateAsync(JSEA jsea, List<JSEADetail> details, string requesterId);
    Task<bool> SubmitAsync(int jseaId, string userId);
    Task<bool> StartReviewAsync(int jseaId, string reviewerId);
    Task<bool> RequestRevisionAsync(int jseaId, string reviewerId, string comments);
    Task<bool> ApproveAsync(int jseaId, string approverId);
    Task<bool> RejectAsync(int jseaId, string approverId, string reason);
    Task<JSEA?> CloneAsync(int jseaId, string requesterId);
    Task<bool> ArchiveAsync(int jseaId, string userId);
    RiskLevel CalculateRiskLevel(List<JSEADetail> details);
}
```

---

### FILE: `src/PTW.JSEA.Application/Services/JSEAService.cs`

```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Application.Services;

public class JSEAService : IJSEAService
{
    private readonly IJSEARepository _jseaRepo;
    private readonly IAuditService _auditService;
    private readonly INotificationService _notificationService;

    public JSEAService(
        IJSEARepository jseaRepo,
        IAuditService auditService,
        INotificationService notificationService)
    {
        _jseaRepo = jseaRepo;
        _auditService = auditService;
        _notificationService = notificationService;
    }

    public async Task<JSEA> CreateAsync(JSEA jsea, List<JSEADetail> details, string requesterId)
    {
        jsea.RequesterId = requesterId;
        jsea.Status      = JSEAStatus.Draft;
        jsea.RiskLevel   = CalculateRiskLevel(details);
        jsea.Version     = 1;
        jsea.CreatedAt   = DateTime.UtcNow;
        jsea.CreatedBy   = requesterId;

        int step = 1;
        foreach (var d in details)
        {
            d.StepNumber = step++;
            d.CreatedAt  = DateTime.UtcNow;
        }
        jsea.Details = details;

        var created = await _jseaRepo.AddAsync(jsea);
        await _auditService.LogAsync("JSEA", created.Id, "Created", requesterId);
        return created;
    }

    public async Task<bool> SubmitAsync(int jseaId, string userId)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null || jsea.RequesterId != userId) return false;
        if (jsea.Status != JSEAStatus.Draft && jsea.Status != JSEAStatus.RevisionRequired) return false;

        var old = jsea.Status.ToString();
        jsea.Status    = JSEAStatus.Submitted;
        jsea.UpdatedAt = DateTime.UtcNow;
        jsea.UpdatedBy = userId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "Submitted", userId, old, jsea.Status.ToString());
        await _notificationService.CreateInAppForRoleAsync("HSEOfficer",
            "JSEA Baru Menunggu Review",
            $"JSEA '{jsea.WorkTitle}' di {jsea.Location} menunggu review HSE.",
            NotificationType.JSEAApproved,
            $"/JSEA/Detail/{jseaId}");
        return true;
    }

    public async Task<bool> StartReviewAsync(int jseaId, string reviewerId)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null || jsea.Status != JSEAStatus.Submitted) return false;

        jsea.Status       = JSEAStatus.UnderReview;
        jsea.ReviewedById = reviewerId;
        jsea.ReviewedAt   = DateTime.UtcNow;
        jsea.UpdatedAt    = DateTime.UtcNow;
        jsea.UpdatedBy    = reviewerId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "ReviewStarted", reviewerId);
        return true;
    }

    public async Task<bool> RequestRevisionAsync(int jseaId, string reviewerId, string comments)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null) return false;

        var old = jsea.Status.ToString();
        jsea.Status          = JSEAStatus.RevisionRequired;
        jsea.ReviewComments  = comments;
        jsea.UpdatedAt       = DateTime.UtcNow;
        jsea.UpdatedBy       = reviewerId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "RevisionRequired", reviewerId, old, jsea.Status.ToString(), comments);
        await _notificationService.CreateInAppAsync(jsea.RequesterId,
            "JSEA Perlu Revisi",
            $"JSEA '{jsea.WorkTitle}' memerlukan revisi. Catatan: {comments}",
            NotificationType.JSEARevisionRequired,
            $"/JSEA/Detail/{jseaId}");
        return true;
    }

    public async Task<bool> ApproveAsync(int jseaId, string approverId)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null) return false;

        var old = jsea.Status.ToString();
        jsea.Status       = JSEAStatus.Approved;
        jsea.ApprovedById = approverId;
        jsea.ApprovedAt   = DateTime.UtcNow;
        jsea.UpdatedAt    = DateTime.UtcNow;
        jsea.UpdatedBy    = approverId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "Approved", approverId, old, "Approved");
        await _notificationService.CreateInAppAsync(jsea.RequesterId,
            "JSEA Disetujui",
            $"JSEA '{jsea.WorkTitle}' telah disetujui. Anda sekarang dapat membuat Permit.",
            NotificationType.JSEAApproved,
            $"/JSEA/Detail/{jseaId}");
        return true;
    }

    public async Task<bool> RejectAsync(int jseaId, string approverId, string reason)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null) return false;

        var old = jsea.Status.ToString();
        jsea.Status          = JSEAStatus.Rejected;
        jsea.RejectionReason = reason;
        jsea.UpdatedAt       = DateTime.UtcNow;
        jsea.UpdatedBy       = approverId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "Rejected", approverId, old, "Rejected", reason);
        await _notificationService.CreateInAppAsync(jsea.RequesterId,
            "JSEA Ditolak",
            $"JSEA '{jsea.WorkTitle}' ditolak. Alasan: {reason}",
            NotificationType.JSEARevisionRequired,
            $"/JSEA/Detail/{jseaId}");
        return true;
    }

    public async Task<JSEA?> CloneAsync(int jseaId, string requesterId)
    {
        var source = await _jseaRepo.GetWithDetailsAsync(jseaId);
        if (source == null) return null;

        var clone = new JSEA
        {
            WorkTitle       = $"[CLONE] {source.WorkTitle}",
            Location        = source.Location,
            WorkDescription = source.WorkDescription,
            WorkType        = source.WorkType,
            RiskLevel       = source.RiskLevel,
            Status          = JSEAStatus.Draft,
            RequesterId     = requesterId,
            ParentJSEAId    = source.Id,
            Version         = 1,
            CreatedAt       = DateTime.UtcNow,
            CreatedBy       = requesterId
        };

        var details = source.Details.Select(d => new JSEADetail
        {
            StepNumber      = d.StepNumber,
            StepDescription = d.StepDescription,
            Hazard          = d.Hazard,
            HazardCategory  = d.HazardCategory,
            ControlMeasure  = d.ControlMeasure,
            PPERequired     = d.PPERequired,
            Severity        = d.Severity,
            Likelihood      = d.Likelihood,
            ResidualRisk    = d.ResidualRisk,
            CreatedAt       = DateTime.UtcNow
        }).ToList();

        clone.Details = details;
        var created = await _jseaRepo.AddAsync(clone);
        await _auditService.LogAsync("JSEA", created.Id, "Cloned", requesterId, null, $"ClonedFrom:{jseaId}");
        return created;
    }

    public async Task<bool> ArchiveAsync(int jseaId, string userId)
    {
        var jsea = await _jseaRepo.GetByIdAsync(jseaId);
        if (jsea == null) return false;

        jsea.Status    = JSEAStatus.Archived;
        jsea.UpdatedAt = DateTime.UtcNow;
        jsea.UpdatedBy = userId;
        await _jseaRepo.UpdateAsync(jsea);

        await _auditService.LogAsync("JSEA", jseaId, "Archived", userId);
        return true;
    }

    public RiskLevel CalculateRiskLevel(List<JSEADetail> details)
    {
        if (!details.Any()) return RiskLevel.Low;
        var maxScore = details.Max(d => d.Severity * d.Likelihood);
        return maxScore switch
        {
            >= 15 => RiskLevel.Critical,
            >= 10 => RiskLevel.High,
            >= 5  => RiskLevel.Medium,
            _     => RiskLevel.Low
        };
    }
}
```

---

## BAGIAN B — JSEA List Page

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Web.Pages.JSEA;

[Authorize]
public class IndexModel : PageModel
{
    private readonly IJSEARepository _jseaRepo;
    private readonly UserManager<ApplicationUser> _userManager;

    public IndexModel(IJSEARepository jseaRepo, UserManager<ApplicationUser> userManager)
    {
        _jseaRepo = jseaRepo;
        _userManager = userManager;
    }

    public List<Domain.Entities.JSEA> JSEAs { get; set; } = new();
    [BindProperty(SupportsGet = true)] public string? StatusFilter { get; set; }
    [BindProperty(SupportsGet = true)] public string? Search { get; set; }

    public async Task OnGetAsync()
    {
        var user   = await _userManager.GetUserAsync(User);
        var roles  = await _userManager.GetRolesAsync(user!);
        var userId = user!.Id;

        IEnumerable<Domain.Entities.JSEA> all;

        // Admin, HSE, Safety Manager, Auditor lihat semua; yang lain hanya milik sendiri
        if (roles.Any(r => r is "SystemAdmin" or "HSEOfficer" or "SafetyManager" or "Auditor" or "AreaOwner" or "Approver"))
            all = await _jseaRepo.GetAllAsync();
        else
            all = await _jseaRepo.GetByUserAsync(userId);

        // Filter status
        if (!string.IsNullOrEmpty(StatusFilter) && Enum.TryParse<JSEAStatus>(StatusFilter, out var status))
            all = all.Where(j => j.Status == status);

        // Search
        if (!string.IsNullOrEmpty(Search))
            all = all.Where(j =>
                j.WorkTitle.Contains(Search, StringComparison.OrdinalIgnoreCase) ||
                j.Location.Contains(Search, StringComparison.OrdinalIgnoreCase));

        JSEAs = all.OrderByDescending(j => j.CreatedAt).ToList();
    }
}
```

---

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.JSEA.IndexModel
@{
    ViewData["Title"] = "Daftar JSEA";
}

<div class="page-header">
    <div>
        <h2 class="page-title">Daftar JSEA</h2>
        <p class="text-sm text-gray-500">Job Safety Environment Analysis</p>
    </div>
    <a asp-page="/JSEA/Create" class="btn-primary">+ Buat JSEA</a>
</div>

<!-- Filter bar -->
<div class="card mb-4">
    <form method="get" class="flex flex-wrap items-end gap-3">
        <div class="flex-1 min-w-48">
            <label class="form-label">Cari</label>
            <input asp-for="Search" class="form-input" placeholder="Judul pekerjaan / lokasi..."/>
        </div>
        <div class="w-48">
            <label class="form-label">Status</label>
            <select asp-for="StatusFilter" class="form-select">
                <option value="">Semua Status</option>
                <option value="Draft">Draft</option>
                <option value="Submitted">Submitted</option>
                <option value="UnderReview">Under Review</option>
                <option value="RevisionRequired">Perlu Revisi</option>
                <option value="Approved">Approved</option>
                <option value="Rejected">Rejected</option>
                <option value="Archived">Archived</option>
            </select>
        </div>
        <button type="submit" class="btn-primary">Filter</button>
        <a asp-page="/JSEA/Index" class="btn-outline">Reset</a>
    </form>
</div>

<!-- Table -->
<div class="card">
    @if (!Model.JSEAs.Any())
    {
        <div class="flex flex-col items-center py-16 text-gray-300">
            <svg class="w-16 h-16 mb-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"/>
            </svg>
            <p class="text-gray-400 font-medium">Belum ada JSEA</p>
            <a asp-page="/JSEA/Create" class="mt-4 btn-primary btn-sm">Buat JSEA Pertama</a>
        </div>
    }
    else
    {
        <div class="table-container">
            <table class="table">
                <thead>
                    <tr>
                        <th>#</th>
                        <th>Judul Pekerjaan</th>
                        <th>Jenis</th>
                        <th>Lokasi</th>
                        <th>Risk</th>
                        <th>Status</th>
                        <th>Dibuat</th>
                        <th>Pemohon</th>
                        <th>Aksi</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var j in Model.JSEAs)
                    {
                        <tr>
                            <td class="text-gray-400 text-xs">@j.Id</td>
                            <td class="font-medium text-gray-900 max-w-xs truncate">@j.WorkTitle</td>
                            <td><span class="badge-blue text-xs">@j.WorkType</span></td>
                            <td class="text-gray-600">@j.Location</td>
                            <td>
                                @{var rc = j.RiskLevel switch { RiskLevel.Low => "risk-low", RiskLevel.Medium => "risk-medium", RiskLevel.High => "risk-high", _ => "risk-critical" };}
                                <span class="@rc">@j.RiskLevel</span>
                            </td>
                            <td>
                                @{var sc = j.Status switch {
                                    JSEAStatus.Draft            => "status-draft",
                                    JSEAStatus.Submitted        => "status-submitted",
                                    JSEAStatus.UnderReview      => "status-review",
                                    JSEAStatus.RevisionRequired => "badge-orange",
                                    JSEAStatus.Approved         => "status-approved",
                                    JSEAStatus.Rejected         => "status-rejected",
                                    JSEAStatus.Archived         => "status-draft",
                                    _ => "badge-gray"
                                };}
                                <span class="@sc">@j.Status</span>
                            </td>
                            <td class="text-gray-500 text-xs">@j.CreatedAt.ToLocalTime().ToString("dd MMM yyyy")</td>
                            <td class="text-gray-600 text-xs">@(j.Requester?.FullName ?? "-")</td>
                            <td>
                                <div class="flex items-center gap-2">
                                    <a asp-page="/JSEA/Detail" asp-route-id="@j.Id"
                                       class="text-blue-600 hover:underline text-xs">Detail</a>
                                    @if (j.Status == JSEAStatus.Draft)
                                    {
                                        <a asp-page="/JSEA/Edit" asp-route-id="@j.Id"
                                           class="text-yellow-600 hover:underline text-xs">Edit</a>
                                    }
                                </div>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
        <p class="text-xs text-gray-400 mt-3">Total: @Model.JSEAs.Count JSEA</p>
    }
</div>
```

---

## BAGIAN C — JSEA Create (Wizard 4 Langkah)

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Create.cshtml.cs`

```csharp
using System.Text.Json;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Web.Pages.JSEA;

[Authorize]
public class CreateModel : PageModel
{
    private readonly IJSEAService _jseaService;
    private readonly IFileStorageService _fileStorage;
    private readonly UserManager<ApplicationUser> _userManager;

    public CreateModel(
        IJSEAService jseaService,
        IFileStorageService fileStorage,
        UserManager<ApplicationUser> userManager)
    {
        _jseaService  = jseaService;
        _fileStorage  = fileStorage;
        _userManager  = userManager;
    }

    [BindProperty] public JSEAInputModel Input { get; set; } = new();
    [BindProperty] public string DetailsJson { get; set; } = "[]";

    public class JSEAInputModel
    {
        public string WorkTitle { get; set; } = string.Empty;
        public string Location { get; set; } = string.Empty;
        public string? WorkDescription { get; set; }
        public WorkType WorkType { get; set; }
        public List<IFormFile>? Attachments { get; set; }
    }

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        var user = await _userManager.GetUserAsync(User);

        // Parse hazard details dari JSON
        List<JSEADetail> details = new();
        try
        {
            var raw = JsonSerializer.Deserialize<List<JSEADetailInput>>(DetailsJson,
                new JsonSerializerOptions { PropertyNameCaseInsensitive = true });
            details = raw?.Select(d => new JSEADetail
            {
                StepDescription = d.StepDescription,
                Hazard          = d.Hazard,
                HazardCategory  = d.HazardCategory,
                ControlMeasure  = d.ControlMeasure,
                PPERequired     = d.PPERequired,
                Severity        = d.Severity,
                Likelihood      = d.Likelihood,
                ResidualRisk    = d.ResidualRisk,
                ResponsiblePerson = d.ResponsiblePerson
            }).ToList() ?? new();
        }
        catch { ModelState.AddModelError("", "Format data hazard tidak valid."); return Page(); }

        if (!details.Any())
        {
            ModelState.AddModelError("", "Minimal 1 langkah kerja / hazard harus diisi.");
            return Page();
        }

        var jsea = new JSEA
        {
            WorkTitle       = Input.WorkTitle,
            Location        = Input.Location,
            WorkDescription = Input.WorkDescription,
            WorkType        = Input.WorkType,
        };

        var created = await _jseaService.CreateAsync(jsea, details, user!.Id);

        // Handle file attachments
        if (Input.Attachments != null && Input.Attachments.Any())
        {
            var allowed = new[] { ".pdf", ".jpg", ".jpeg", ".png", ".doc", ".docx", ".xls", ".xlsx" };
            foreach (var file in Input.Attachments)
            {
                if (_fileStorage.IsValidFile(file, allowed))
                {
                    var path = await _fileStorage.SaveFileAsync(file, $"jsea/{created.Id}");
                    // Attachment sudah tersimpan; simpan record ke DB bila perlu
                }
            }
        }

        TempData["Success"] = $"JSEA berhasil dibuat dengan ID #{created.Id}.";
        return RedirectToPage("/JSEA/Detail", new { id = created.Id });
    }

    public class JSEADetailInput
    {
        public string StepDescription { get; set; } = string.Empty;
        public string Hazard { get; set; } = string.Empty;
        public string HazardCategory { get; set; } = string.Empty;
        public string ControlMeasure { get; set; } = string.Empty;
        public string PPERequired { get; set; } = string.Empty;
        public int Severity { get; set; }
        public int Likelihood { get; set; }
        public int ResidualRisk { get; set; }
        public string? ResponsiblePerson { get; set; }
    }
}
```

---

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Create.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.JSEA.CreateModel
@{
    ViewData["Title"] = "Buat JSEA";
}

<!-- Wizard progress -->
<div class="mb-6">
    <div class="flex items-center gap-0">
        @foreach (var (label, i) in new[] {"Informasi Umum","Hazard Analysis","Lampiran","Review & Submit"}.Select((l,i)=>(l,i+1)))
        {
            <div class="flex items-center @(i < 4 ? "flex-1" : "")">
                <div id="step-@i" class="flex items-center gap-2 px-3 py-2 rounded-lg text-sm font-medium
                     @(i == 1 ? "bg-blue-600 text-white" : "bg-gray-100 text-gray-400")
                     transition-colors duration-200 cursor-pointer" onclick="goToStep(@i)">
                    <span class="w-6 h-6 rounded-full @(i == 1 ? "bg-blue-500" : "bg-gray-200 text-gray-500")
                          flex items-center justify-center text-xs font-bold">@i</span>
                    <span class="hidden sm:block">@label</span>
                </div>
                @if (i < 4)
                {
                    <div id="line-@i" class="flex-1 h-0.5 @(i == 1 ? "bg-blue-300" : "bg-gray-200") mx-1 transition-colors"></div>
                }
            </div>
        }
    </div>
</div>

<form method="post" enctype="multipart/form-data" id="jseaForm">
    @Html.AntiForgeryToken()
    <input type="hidden" asp-for="DetailsJson" id="detailsJsonInput"/>

    <!-- ══ STEP 1: Informasi Umum ══ -->
    <div id="wizard-step-1" class="wizard-step">
        <div class="card">
            <h3 class="section-title">Informasi Umum Pekerjaan</h3>
            <div class="grid grid-cols-1 md:grid-cols-2 gap-5">
                <div class="md:col-span-2">
                    <label class="form-label">Judul Pekerjaan <span class="text-red-500">*</span></label>
                    <input asp-for="Input.WorkTitle" class="form-input"
                           placeholder="Contoh: Penggantian Motor Pompa Area B2"/>
                    <span asp-validation-for="Input.WorkTitle" class="form-error"></span>
                </div>
                <div>
                    <label class="form-label">Jenis Pekerjaan <span class="text-red-500">*</span></label>
                    <select asp-for="Input.WorkType" class="form-select">
                        <option value="">-- Pilih Jenis --</option>
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
                    <input asp-for="Input.Location" class="form-input" placeholder="Contoh: Area Produksi B2, Lantai 2"/>
                </div>
                <div class="md:col-span-2">
                    <label class="form-label">Deskripsi Pekerjaan</label>
                    <textarea asp-for="Input.WorkDescription" class="form-textarea" rows="3"
                              placeholder="Jelaskan pekerjaan secara singkat..."></textarea>
                </div>
            </div>
        </div>
        <div class="flex justify-end mt-4">
            <button type="button" onclick="nextStep(1)" class="btn-primary">
                Lanjut: Hazard Analysis →
            </button>
        </div>
    </div>

    <!-- ══ STEP 2: Hazard Analysis ══ -->
    <div id="wizard-step-2" class="wizard-step hidden">
        <div class="card">
            <div class="flex items-center justify-between mb-4">
                <h3 class="section-title mb-0">Analisa Hazard & Risiko</h3>
                <button type="button" onclick="addHazardRow()" class="btn-primary btn-sm">
                    + Tambah Langkah
                </button>
            </div>

            <!-- Risk matrix legend -->
            <div class="flex flex-wrap gap-3 mb-4 p-3 bg-gray-50 rounded-lg text-xs">
                <span class="font-semibold text-gray-600">Risk Score = Severity × Likelihood:</span>
                <span class="risk-low">1-4: Low</span>
                <span class="risk-medium">5-9: Medium</span>
                <span class="risk-high">10-14: High</span>
                <span class="risk-critical">15-25: Critical</span>
            </div>

            <div class="overflow-x-auto">
                <table class="min-w-full text-xs" id="hazardTable">
                    <thead class="bg-blue-50">
                        <tr>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 w-6">#</th>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 min-w-48">Langkah Kerja</th>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 min-w-36">Bahaya (Hazard)</th>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 min-w-36">Kategori</th>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 min-w-48">Pengendalian (Control)</th>
                            <th class="px-3 py-2 text-left font-semibold text-blue-700 min-w-32">APD</th>
                            <th class="px-3 py-2 text-center font-semibold text-blue-700 w-20">Severity</th>
                            <th class="px-3 py-2 text-center font-semibold text-blue-700 w-24">Likelihood</th>
                            <th class="px-3 py-2 text-center font-semibold text-blue-700 w-20">Score</th>
                            <th class="px-3 py-2 text-center font-semibold text-blue-700 w-20">Residu</th>
                            <th class="px-3 py-2 w-10"></th>
                        </tr>
                    </thead>
                    <tbody id="hazardBody" class="divide-y divide-gray-100 bg-white">
                        <!-- Baris pertama default -->
                    </tbody>
                </table>
            </div>

            <p id="hazardError" class="text-red-500 text-xs mt-2 hidden">Minimal 1 langkah kerja wajib diisi.</p>
        </div>
        <div class="flex justify-between mt-4">
            <button type="button" onclick="prevStep(2)" class="btn-outline">← Kembali</button>
            <button type="button" onclick="nextStep(2)" class="btn-primary">Lanjut: Lampiran →</button>
        </div>
    </div>

    <!-- ══ STEP 3: Lampiran ══ -->
    <div id="wizard-step-3" class="wizard-step hidden">
        <div class="card">
            <h3 class="section-title">Upload Lampiran (Opsional)</h3>
            <p class="text-sm text-gray-500 mb-4">
                Lampirkan dokumen pendukung seperti SOP, sertifikat alat, gambar teknis, atau method statement.
                Format: PDF, DOC, XLS, JPG, PNG. Maksimal 10MB per file.
            </p>
            <div class="border-2 border-dashed border-gray-200 rounded-xl p-8 text-center hover:border-blue-300 transition-colors"
                 id="dropZone">
                <svg class="w-12 h-12 text-gray-300 mx-auto mb-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5"
                          d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"/>
                </svg>
                <p class="text-gray-500 mb-2">Drag & drop file di sini, atau</p>
                <label class="btn-outline btn-sm cursor-pointer">
                    Pilih File
                    <input name="Input.Attachments" type="file" multiple accept=".pdf,.doc,.docx,.xls,.xlsx,.jpg,.jpeg,.png"
                           class="hidden" id="fileInput" onchange="showSelectedFiles(this)"/>
                </label>
                <div id="selectedFiles" class="mt-4 text-left space-y-2"></div>
            </div>
        </div>
        <div class="flex justify-between mt-4">
            <button type="button" onclick="prevStep(3)" class="btn-outline">← Kembali</button>
            <button type="button" onclick="nextStep(3)" class="btn-primary">Lanjut: Review →</button>
        </div>
    </div>

    <!-- ══ STEP 4: Review & Submit ══ -->
    <div id="wizard-step-4" class="wizard-step hidden">
        <div class="card mb-4">
            <h3 class="section-title">Review & Konfirmasi</h3>

            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
                <div class="bg-gray-50 rounded-lg p-4">
                    <p class="text-xs font-semibold text-gray-500 uppercase mb-1">Judul Pekerjaan</p>
                    <p id="review-title" class="font-semibold text-gray-900">-</p>
                </div>
                <div class="bg-gray-50 rounded-lg p-4">
                    <p class="text-xs font-semibold text-gray-500 uppercase mb-1">Jenis Pekerjaan</p>
                    <p id="review-type" class="font-semibold text-gray-900">-</p>
                </div>
                <div class="bg-gray-50 rounded-lg p-4">
                    <p class="text-xs font-semibold text-gray-500 uppercase mb-1">Lokasi</p>
                    <p id="review-location" class="font-semibold text-gray-900">-</p>
                </div>
                <div class="bg-gray-50 rounded-lg p-4">
                    <p class="text-xs font-semibold text-gray-500 uppercase mb-1">Jumlah Langkah Kerja</p>
                    <p id="review-steps" class="font-semibold text-gray-900">-</p>
                </div>
                <div class="bg-gray-50 rounded-lg p-4">
                    <p class="text-xs font-semibold text-gray-500 uppercase mb-1">Risk Level (Calculated)</p>
                    <p id="review-risk" class="font-semibold text-gray-900">-</p>
                </div>
            </div>

            <div class="alert-info mb-4">
                <svg class="w-5 h-5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13 16h-1v-4h-1m1-4h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z"/>
                </svg>
                <p>Setelah disubmit, JSEA akan dikirim ke HSE Officer untuk di-review. Anda tidak dapat mengubah data kecuali diminta revisi.</p>
            </div>
        </div>

        <div class="flex justify-between mt-4">
            <button type="button" onclick="prevStep(4)" class="btn-outline">← Kembali</button>
            <button type="submit" onclick="prepareSubmit()" class="btn-success">
                ✓ Submit JSEA
            </button>
        </div>
    </div>
</form>

@section Scripts {
<script>
// ─── Wizard state ─────────────────────────────────────────────────────────
let currentStep = 1;
let hazardRows  = [];

function goToStep(step) {
    if (step > currentStep) return; // hanya bisa kembali, maju harus lewat validasi
    showStep(step);
}

function showStep(step) {
    document.querySelectorAll('.wizard-step').forEach(el => el.classList.add('hidden'));
    document.getElementById(`wizard-step-${step}`).classList.remove('hidden');

    for (let i = 1; i <= 4; i++) {
        const el = document.getElementById(`step-${i}`);
        const ln = document.getElementById(`line-${i}`);
        el.className = el.className.replace(/bg-\w+-\d+ text-\w+/g, '');
        if (i === step) {
            el.classList.add('bg-blue-600', 'text-white');
        } else if (i < step) {
            el.classList.add('bg-green-100', 'text-green-700');
        } else {
            el.classList.add('bg-gray-100', 'text-gray-400');
        }
        if (ln) ln.className = ln.className.replace(/bg-\w+-\d+/g, i < step ? 'bg-blue-300' : 'bg-gray-200');
    }
    currentStep = step;
}

function nextStep(from) {
    if (from === 1) {
        const title    = document.getElementById('Input_WorkTitle')?.value;
        const workType = document.getElementById('Input_WorkType')?.value;
        const location = document.getElementById('Input_Location')?.value;
        if (!title || !workType || !location) {
            alert('Judul pekerjaan, jenis, dan lokasi wajib diisi.');
            return;
        }
        if (hazardRows.length === 0) addHazardRow(); // auto-add baris pertama
    }
    if (from === 2) {
        if (hazardRows.length === 0) {
            document.getElementById('hazardError').classList.remove('hidden');
            return;
        }
        document.getElementById('hazardError').classList.add('hidden');
    }
    if (from === 3) {
        updateReviewStep();
    }
    showStep(from + 1);
}

function prevStep(from) { showStep(from - 1); }

// ─── Hazard table ─────────────────────────────────────────────────────────
let rowCounter = 0;

function addHazardRow() {
    rowCounter++;
    const id = rowCounter;
    hazardRows.push(id);
    const tbody = document.getElementById('hazardBody');

    const hazardCategories = ['Hot Work','Electrical','Mechanical','Chemical','Ergonomics','Height','Confined Space','Excavation','Manual Handling','Slip/Trip/Fall'];
    const catOptions = hazardCategories.map(c => `<option>${c}</option>`).join('');

    const tr = document.createElement('tr');
    tr.id = `row-${id}`;
    tr.className = 'hover:bg-gray-50';
    tr.innerHTML = `
        <td class="px-3 py-2 text-gray-400 font-medium">${hazardRows.length}</td>
        <td class="px-3 py-1">
            <textarea class="form-textarea text-xs py-1" rows="2" placeholder="Langkah kerja..." 
                      onchange="updateRow(${id},'StepDescription',this.value)" style="min-width:180px"></textarea>
        </td>
        <td class="px-3 py-1">
            <textarea class="form-textarea text-xs py-1" rows="2" placeholder="Identifikasi bahaya..."
                      onchange="updateRow(${id},'Hazard',this.value)" style="min-width:140px"></textarea>
        </td>
        <td class="px-3 py-1">
            <select class="form-select text-xs" onchange="updateRow(${id},'HazardCategory',this.value)">
                ${catOptions}
            </select>
        </td>
        <td class="px-3 py-1">
            <textarea class="form-textarea text-xs py-1" rows="2" placeholder="Tindakan pengendalian..."
                      onchange="updateRow(${id},'ControlMeasure',this.value)" style="min-width:180px"></textarea>
        </td>
        <td class="px-3 py-1">
            <input type="text" class="form-input text-xs" placeholder="Helm, sarung tangan..."
                   onchange="updateRow(${id},'PPERequired',this.value)" style="min-width:120px"/>
        </td>
        <td class="px-3 py-1 text-center">
            <select class="form-select text-xs w-16" id="sev-${id}" onchange="calcScore(${id})">
                ${[1,2,3,4,5].map(n=>`<option value="${n}">${n}</option>`).join('')}
            </select>
        </td>
        <td class="px-3 py-1 text-center">
            <select class="form-select text-xs w-16" id="lik-${id}" onchange="calcScore(${id})">
                ${[1,2,3,4,5].map(n=>`<option value="${n}">${n}</option>`).join('')}
            </select>
        </td>
        <td class="px-3 py-1 text-center">
            <span id="score-${id}" class="badge-gray font-bold text-sm">1</span>
        </td>
        <td class="px-3 py-1 text-center">
            <select class="form-select text-xs w-16" onchange="updateRow(${id},'ResidualRisk',parseInt(this.value))">
                ${[1,2,3,4,5].map(n=>`<option value="${n}">${n}</option>`).join('')}
            </select>
        </td>
        <td class="px-3 py-1 text-center">
            <button type="button" onclick="removeRow(${id})" class="text-red-400 hover:text-red-600 p-1">
                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/>
                </svg>
            </button>
        </td>
    `;
    tbody.appendChild(tr);
}

const rowData = {};
function updateRow(id, field, value) {
    if (!rowData[id]) rowData[id] = { Severity:1, Likelihood:1, ResidualRisk:1 };
    rowData[id][field] = value;
}

function calcScore(id) {
    const s = parseInt(document.getElementById(`sev-${id}`).value);
    const l = parseInt(document.getElementById(`lik-${id}`).value);
    const score = s * l;
    if (!rowData[id]) rowData[id] = {};
    rowData[id].Severity    = s;
    rowData[id].Likelihood  = l;
    const el = document.getElementById(`score-${id}`);
    el.textContent = score;
    el.className = score >= 15 ? 'risk-critical' : score >= 10 ? 'risk-high' : score >= 5 ? 'risk-medium' : 'risk-low';
}

function removeRow(id) {
    const idx = hazardRows.indexOf(id);
    if (idx > -1) hazardRows.splice(idx, 1);
    delete rowData[id];
    document.getElementById(`row-${id}`)?.remove();
    // Renumber
    const rows = document.querySelectorAll('#hazardBody tr');
    rows.forEach((r, i) => { const fc = r.querySelector('td:first-child'); if (fc) fc.textContent = i+1; });
}

function getHazardDetailsAsJson() {
    return hazardRows.map(id => {
        const data = rowData[id] || {};
        const row  = document.getElementById(`row-${id}`);
        if (!row) return null;
        const textareas = row.querySelectorAll('textarea');
        const inputs    = row.querySelectorAll('input[type=text]');
        const selects   = row.querySelectorAll('select');
        return {
            StepDescription   : textareas[0]?.value || '',
            Hazard            : textareas[1]?.value || '',
            HazardCategory    : selects[0]?.value   || '',
            ControlMeasure    : textareas[2]?.value || '',
            PPERequired       : inputs[0]?.value    || '',
            Severity          : parseInt(selects[1]?.value || '1'),
            Likelihood        : parseInt(selects[2]?.value || '1'),
            ResidualRisk      : parseInt(selects[3]?.value || '1'),
            ResponsiblePerson : data.ResponsiblePerson || ''
        };
    }).filter(Boolean);
}

// ─── File selection display ───────────────────────────────────────────────
function showSelectedFiles(input) {
    const container = document.getElementById('selectedFiles');
    container.innerHTML = '';
    Array.from(input.files).forEach(f => {
        const div = document.createElement('div');
        div.className = 'flex items-center gap-2 text-sm text-gray-600 bg-gray-50 rounded p-2';
        div.innerHTML = `<svg class="w-4 h-4 text-blue-500" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15.172 7l-6.586 6.586a2 2 0 102.828 2.828l6.414-6.586a4 4 0 00-5.656-5.656l-6.415 6.585a6 6 0 108.486 8.486L20.5 13"/></svg>
            <span class="truncate">${f.name}</span>
            <span class="text-gray-400 text-xs ml-auto">${(f.size/1024/1024).toFixed(1)} MB</span>`;
        container.appendChild(div);
    });
}

// ─── Review step ──────────────────────────────────────────────────────────
function updateReviewStep() {
    document.getElementById('review-title').textContent    = document.getElementById('Input_WorkTitle')?.value || '-';
    document.getElementById('review-type').textContent     = document.getElementById('Input_WorkType')?.value || '-';
    document.getElementById('review-location').textContent = document.getElementById('Input_Location')?.value || '-';

    const details = getHazardDetailsAsJson();
    document.getElementById('review-steps').textContent   = details.length + ' langkah';

    const maxScore = Math.max(...details.map(d => d.Severity * d.Likelihood), 0);
    const risk = maxScore >= 15 ? 'Critical' : maxScore >= 10 ? 'High' : maxScore >= 5 ? 'Medium' : 'Low';
    document.getElementById('review-risk').textContent    = risk;
}

function prepareSubmit() {
    const details = getHazardDetailsAsJson();
    document.getElementById('detailsJsonInput').value = JSON.stringify(details);
}

// Init: tambah 1 baris default
addHazardRow();
</script>
}
```

---

## BAGIAN D — JSEA Detail Page

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Detail.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Web.Pages.JSEA;

[Authorize]
public class DetailModel : PageModel
{
    private readonly IJSEARepository _jseaRepo;
    private readonly IJSEAService _jseaService;
    private readonly UserManager<ApplicationUser> _userManager;

    public DetailModel(
        IJSEARepository jseaRepo,
        IJSEAService jseaService,
        UserManager<ApplicationUser> userManager)
    {
        _jseaRepo    = jseaRepo;
        _jseaService = jseaService;
        _userManager = userManager;
    }

    public Domain.Entities.JSEA JSEA { get; set; } = null!;
    public bool CanSubmit  { get; set; }
    public bool CanReview  { get; set; }
    public bool CanApprove { get; set; }
    public bool CanReject  { get; set; }
    public bool CanClone   { get; set; }

    [BindProperty] public string? ReviewComments { get; set; }
    [BindProperty] public string? RejectionReason { get; set; }

    public async Task<IActionResult> OnGetAsync(int id)
    {
        var jsea = await _jseaRepo.GetWithDetailsAsync(id);
        if (jsea == null) return NotFound();
        JSEA = jsea;

        var user  = await _userManager.GetUserAsync(User);
        var roles = await _userManager.GetRolesAsync(user!);

        CanSubmit  = jsea.RequesterId == user!.Id &&
                     (jsea.Status == Domain.Enums.JSEAStatus.Draft || jsea.Status == Domain.Enums.JSEAStatus.RevisionRequired);
        CanReview  = roles.Any(r => r is "HSEOfficer" or "SafetyManager" or "SystemAdmin") &&
                     jsea.Status == Domain.Enums.JSEAStatus.Submitted;
        CanApprove = roles.Any(r => r is "HSEOfficer" or "SafetyManager" or "SystemAdmin") &&
                     jsea.Status == Domain.Enums.JSEAStatus.UnderReview;
        CanReject  = CanApprove;
        CanClone   = jsea.Status == Domain.Enums.JSEAStatus.Approved;

        if (TempData["Success"] != null) ViewData["Success"] = TempData["Success"];
        return Page();
    }

    public async Task<IActionResult> OnPostSubmitAsync(int id)
    {
        var user = await _userManager.GetUserAsync(User);
        await _jseaService.SubmitAsync(id, user!.Id);
        TempData["Success"] = "JSEA berhasil disubmit untuk review.";
        return RedirectToPage(new { id });
    }

    public async Task<IActionResult> OnPostStartReviewAsync(int id)
    {
        var user = await _userManager.GetUserAsync(User);
        await _jseaService.StartReviewAsync(id, user!.Id);
        TempData["Success"] = "Review dimulai.";
        return RedirectToPage(new { id });
    }

    public async Task<IActionResult> OnPostRequestRevisionAsync(int id)
    {
        var user = await _userManager.GetUserAsync(User);
        await _jseaService.RequestRevisionAsync(id, user!.Id, ReviewComments ?? "-");
        TempData["Success"] = "Permintaan revisi telah dikirim ke pemohon.";
        return RedirectToPage(new { id });
    }

    public async Task<IActionResult> OnPostApproveAsync(int id)
    {
        var user = await _userManager.GetUserAsync(User);
        await _jseaService.ApproveAsync(id, user!.Id);
        TempData["Success"] = "JSEA berhasil disetujui.";
        return RedirectToPage(new { id });
    }

    public async Task<IActionResult> OnPostRejectAsync(int id)
    {
        var user = await _userManager.GetUserAsync(User);
        await _jseaService.RejectAsync(id, user!.Id, RejectionReason ?? "-");
        TempData["Success"] = "JSEA ditolak.";
        return RedirectToPage(new { id });
    }

    public async Task<IActionResult> OnPostCloneAsync(int id)
    {
        var user   = await _userManager.GetUserAsync(User);
        var cloned = await _jseaService.CloneAsync(id, user!.Id);
        if (cloned == null) return NotFound();
        TempData["Success"] = $"JSEA berhasil di-clone sebagai #{cloned.Id}.";
        return RedirectToPage(new { id = cloned.Id });
    }
}
```

---

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Detail.cshtml`

```html
@page "{id:int}"
@model PTW.JSEA.Web.Pages.JSEA.DetailModel
@{
    ViewData["Title"] = $"JSEA — {Model.JSEA.WorkTitle}";
    var j = Model.JSEA;
}

@if (ViewData["Success"] != null)
{
    <div class="alert-success mb-4">
        <svg class="w-5 h-5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
        </svg>
        <span>@ViewData["Success"]</span>
    </div>
}

<!-- Header -->
<div class="card mb-4">
    <div class="flex items-start justify-between flex-wrap gap-4">
        <div>
            <div class="flex items-center gap-3 mb-2 flex-wrap">
                @{
                    var sc = j.Status switch {
                        JSEAStatus.Draft            => "status-draft",
                        JSEAStatus.Submitted        => "status-submitted",
                        JSEAStatus.UnderReview      => "status-review",
                        JSEAStatus.RevisionRequired => "badge-orange",
                        JSEAStatus.Approved         => "status-approved",
                        JSEAStatus.Rejected         => "status-rejected",
                        _                            => "badge-gray"
                    };
                }
                <span class="@sc text-sm">@j.Status</span>
                @{
                    var rc = j.RiskLevel switch { RiskLevel.Low => "risk-low", RiskLevel.Medium => "risk-medium", RiskLevel.High => "risk-high", _ => "risk-critical" };
                }
                <span class="@rc text-sm">@j.RiskLevel Risk</span>
                <span class="badge-blue">@j.WorkType</span>
                <span class="badge-gray">v@j.Version</span>
            </div>
            <h2 class="text-xl font-bold text-gray-900">@j.WorkTitle</h2>
            <p class="text-gray-500 mt-1">📍 @j.Location</p>
        </div>
        <!-- Action buttons -->
        <div class="flex flex-wrap gap-2">
            @if (Model.CanSubmit)
            {
                <form method="post" asp-page-handler="Submit" asp-route-id="@j.Id">
                    @Html.AntiForgeryToken()
                    <button type="submit" class="btn-primary">▶ Submit untuk Review</button>
                </form>
            }
            @if (Model.CanReview)
            {
                <form method="post" asp-page-handler="StartReview" asp-route-id="@j.Id">
                    @Html.AntiForgeryToken()
                    <button type="submit" class="btn-warning">🔍 Mulai Review</button>
                </form>
            }
            @if (Model.CanApprove)
            {
                <form method="post" asp-page-handler="Approve" asp-route-id="@j.Id">
                    @Html.AntiForgeryToken()
                    <button type="submit" class="btn-success">✓ Setujui</button>
                </form>
                <button type="button" onclick="document.getElementById('rejectModal').classList.remove('hidden')"
                        class="btn-danger">✗ Tolak</button>
                <button type="button" onclick="document.getElementById('revisionModal').classList.remove('hidden')"
                        class="btn-warning">↩ Minta Revisi</button>
            }
            @if (Model.CanClone)
            {
                <form method="post" asp-page-handler="Clone" asp-route-id="@j.Id">
                    @Html.AntiForgeryToken()
                    <button type="submit" class="btn-outline">⎘ Clone JSEA</button>
                </form>
            }
            <a asp-page="/JSEA/Index" class="btn-outline">← Kembali</a>
        </div>
    </div>
</div>

<!-- Info grid -->
<div class="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
    <div class="card-sm">
        <p class="text-xs text-gray-500 uppercase font-semibold mb-1">Pemohon</p>
        <p class="font-semibold text-gray-900">@(j.Requester?.FullName ?? "-")</p>
        <p class="text-xs text-gray-400">@(j.Requester?.Department ?? "")</p>
    </div>
    <div class="card-sm">
        <p class="text-xs text-gray-500 uppercase font-semibold mb-1">Dibuat</p>
        <p class="font-semibold text-gray-900">@j.CreatedAt.ToLocalTime().ToString("dd MMM yyyy HH:mm")</p>
    </div>
    @if (j.ApprovedAt.HasValue)
    {
        <div class="card-sm">
            <p class="text-xs text-gray-500 uppercase font-semibold mb-1">Disetujui</p>
            <p class="font-semibold text-gray-900">@j.ApprovedAt.Value.ToLocalTime().ToString("dd MMM yyyy HH:mm")</p>
            <p class="text-xs text-gray-400">oleh @(j.ApprovedBy?.FullName ?? "-")</p>
        </div>
    }
    @if (!string.IsNullOrEmpty(j.ReviewComments))
    {
        <div class="card-sm md:col-span-3 border-orange-200 bg-orange-50">
            <p class="text-xs text-orange-600 uppercase font-semibold mb-1">Catatan Review</p>
            <p class="text-orange-800">@j.ReviewComments</p>
        </div>
    }
    @if (!string.IsNullOrEmpty(j.RejectionReason))
    {
        <div class="card-sm md:col-span-3 border-red-200 bg-red-50">
            <p class="text-xs text-red-600 uppercase font-semibold mb-1">Alasan Penolakan</p>
            <p class="text-red-800">@j.RejectionReason</p>
        </div>
    }
</div>

<!-- Hazard Details Table -->
<div class="card mb-4">
    <h3 class="section-title">Analisa Hazard (@j.Details.Count langkah)</h3>
    @if (!j.Details.Any())
    {
        <p class="text-gray-400 text-sm">Belum ada data hazard.</p>
    }
    else
    {
        <div class="table-container">
            <table class="table text-xs">
                <thead>
                    <tr>
                        <th class="w-8">#</th>
                        <th>Langkah Kerja</th>
                        <th>Bahaya</th>
                        <th>Kategori</th>
                        <th>Pengendalian</th>
                        <th>APD</th>
                        <th class="text-center">S</th>
                        <th class="text-center">L</th>
                        <th class="text-center">Score</th>
                        <th class="text-center">Residu</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var d in j.Details.OrderBy(x => x.StepNumber))
                    {
                        var score = d.Severity * d.Likelihood;
                        var scoreClass = score >= 15 ? "risk-critical" : score >= 10 ? "risk-high" : score >= 5 ? "risk-medium" : "risk-low";
                        <tr>
                            <td class="text-gray-400">@d.StepNumber</td>
                            <td class="font-medium">@d.StepDescription</td>
                            <td>@d.Hazard</td>
                            <td><span class="badge-gray">@d.HazardCategory</span></td>
                            <td>@d.ControlMeasure</td>
                            <td>@d.PPERequired</td>
                            <td class="text-center font-bold">@d.Severity</td>
                            <td class="text-center font-bold">@d.Likelihood</td>
                            <td class="text-center"><span class="@scoreClass font-bold">@score</span></td>
                            <td class="text-center font-bold text-gray-600">@d.ResidualRisk</td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    }
</div>

<!-- Modals -->
<!-- Revision Modal -->
<div id="revisionModal" class="hidden fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
    <div class="bg-white rounded-2xl shadow-2xl p-6 w-full max-w-md">
        <h3 class="font-bold text-lg mb-4">Minta Revisi</h3>
        <form method="post" asp-page-handler="RequestRevision" asp-route-id="@j.Id">
            @Html.AntiForgeryToken()
            <label class="form-label">Catatan untuk pemohon <span class="text-red-500">*</span></label>
            <textarea asp-for="ReviewComments" class="form-textarea mb-4" rows="4"
                      placeholder="Jelaskan apa yang perlu direvisi..." required></textarea>
            <div class="flex gap-3 justify-end">
                <button type="button" onclick="document.getElementById('revisionModal').classList.add('hidden')"
                        class="btn-outline">Batal</button>
                <button type="submit" class="btn-warning">Kirim Permintaan Revisi</button>
            </div>
        </form>
    </div>
</div>

<!-- Reject Modal -->
<div id="rejectModal" class="hidden fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
    <div class="bg-white rounded-2xl shadow-2xl p-6 w-full max-w-md">
        <h3 class="font-bold text-lg mb-4 text-red-700">Tolak JSEA</h3>
        <form method="post" asp-page-handler="Reject" asp-route-id="@j.Id">
            @Html.AntiForgeryToken()
            <label class="form-label">Alasan penolakan <span class="text-red-500">*</span></label>
            <textarea asp-for="RejectionReason" class="form-textarea mb-4" rows="4"
                      placeholder="Jelaskan alasan penolakan..." required></textarea>
            <div class="flex gap-3 justify-end">
                <button type="button" onclick="document.getElementById('rejectModal').classList.add('hidden')"
                        class="btn-outline">Batal</button>
                <button type="submit" class="btn-danger">Tolak JSEA</button>
            </div>
        </form>
    </div>
</div>
```

---

### Tambahkan placeholder `Detail.cshtml` dan `Edit.cshtml`:

```bash
# Detail sudah dibuat di atas
# Edit page (placeholder, akan diisi nanti)
```

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Edit.cshtml`
```html
@page "{id:int}"
@model PTW.JSEA.Web.Pages.JSEA.EditModel
@{ ViewData["Title"] = "Edit JSEA"; }
<p class="text-gray-500">Edit page — mirip Create, implementasi serupa.</p>
```

### FILE: `src/PTW.JSEA.Web/Pages/JSEA/Edit.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
namespace PTW.JSEA.Web.Pages.JSEA;
[Authorize] public class EditModel : PageModel { public void OnGet(int id) { } }
```

---

### Daftarkan IJSEAService di DependencyInjection.cs Application:

```csharp
// Tambahkan ke src/PTW.JSEA.Application/DependencyInjection.cs
services.AddScoped<IJSEAService, JSEAService>();
```

---

> ✅ Modul 5 selesai. JSEA: List, Create (Wizard), Detail, Review, Approve.
