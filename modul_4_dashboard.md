# MODUL 4 — DASHBOARD LENGKAP

> Fase 2: UI Shell — Dashboard real dengan KPI cards, active permits table, chart, dan SignalR live update.

---

## BAGIAN A — Dashboard Service (lengkap)

### FILE: `src/PTW.JSEA.Application/Services/DashboardService.cs`
*(Replace file dari Modul 3 dengan versi lengkap ini)*

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Application.DTOs;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Application.Services;

public class DashboardService : IDashboardService
{
    private readonly ApplicationDbContext _context;
    private readonly IWorkPermitRepository _permitRepo;
    private readonly INotificationService _notificationService;
    private readonly UserManager<ApplicationUser> _userManager;

    public DashboardService(
        ApplicationDbContext context,
        IWorkPermitRepository permitRepo,
        INotificationService notificationService,
        UserManager<ApplicationUser> userManager)
    {
        _context = context;
        _permitRepo = permitRepo;
        _notificationService = notificationService;
        _userManager = userManager;
    }

    public async Task<DashboardStatsDto> GetDashboardStatsAsync(ClaimsPrincipal userPrincipal)
    {
        var userId = _userManager.GetUserId(userPrincipal) ?? string.Empty;

        var activePermits = await _context.WorkPermits
            .Include(p => p.Requester)
            .Include(p => p.Workers)
            .Include(p => p.AdherenceLogs)
            .Where(p => p.Status == PermitStatus.Active && !p.IsDeleted)
            .OrderBy(p => p.EndTime)
            .ToListAsync();

        // Overdue adherence check (>4 jam tidak ada log)
        var adherenceThreshold = DateTime.UtcNow.AddHours(-4);
        var overdueCount = activePermits.Count(p =>
        {
            var lastLog = p.AdherenceLogs.OrderByDescending(a => a.CheckTime).FirstOrDefault();
            return lastLog == null || lastLog.CheckTime < adherenceThreshold;
        });

        // Pending approvals
        var pendingApprovals = await _context.PermitApprovals
            .CountAsync(a => a.Status == ApprovalStatus.Pending);

        // Unread notifications
        var unreadCount = await _notificationService.GetUnreadCountAsync(userId);

        // Permit trend 7 hari terakhir
        var trend = await GetPermitTrendAsync(7);

        // Recent audit activities
        var recentActivities = await _context.AuditLogs
            .OrderByDescending(x => x.ActionDate)
            .Take(8)
            .Select(x => new RecentActivityDto
            {
                Timestamp  = x.ActionDate,
                Description = $"{x.Action} pada {x.Module} #{x.EntityId}",
                Module     = x.Module,
                ActionBy   = x.ActionBy
            })
            .ToListAsync();

        // Permit by work type (untuk pie chart)
        var byWorkType = await _context.WorkPermits
            .Where(p => p.Status == PermitStatus.Active && !p.IsDeleted)
            .GroupBy(p => p.WorkType)
            .Select(g => new WorkTypeCountDto { WorkType = g.Key.ToString(), Count = g.Count() })
            .ToListAsync();

        return new DashboardStatsDto
        {
            ActivePermitCount      = activePermits.Count,
            HighRiskPermitCount    = activePermits.Count(p => p.RiskLevel == RiskLevel.High || p.RiskLevel == RiskLevel.Critical),
            ActiveWorkerCount      = activePermits.Sum(p => p.Workers.Count(w => w.CheckInTime.HasValue && !w.CheckOutTime.HasValue)),
            OverdueAdherenceCount  = overdueCount,
            PendingApprovalCount   = pendingApprovals,
            UnreadNotificationCount = unreadCount,
            PermitTrend            = trend,
            PermitByWorkType       = byWorkType,
            ActivePermits          = activePermits.Select(p => new ActivePermitDto
            {
                Id            = p.Id,
                PermitNumber  = p.PermitNumber,
                WorkType      = p.WorkType.ToString(),
                Location      = p.Location,
                RequesterName = p.Requester?.FullName ?? "-",
                WorkerCount   = p.Workers.Count,
                EndTime       = p.EndTime,
                RiskLevel     = p.RiskLevel.ToString(),
                IsExpiringSoon = p.IsExpiringSoon
            }).ToList(),
            RecentActivities = recentActivities
        };
    }

    private async Task<List<PermitTrendDto>> GetPermitTrendAsync(int days)
    {
        var result = new List<PermitTrendDto>();
        for (int i = days - 1; i >= 0; i--)
        {
            var date = DateTime.UtcNow.Date.AddDays(-i);
            var count = await _context.WorkPermits
                .CountAsync(p => p.CreatedAt.Date == date && !p.IsDeleted);
            result.Add(new PermitTrendDto
            {
                Date  = date.ToString("dd MMM"),
                Count = count
            });
        }
        return result;
    }
}
```

---

## BAGIAN B — DTOs lengkap

### FILE: `src/PTW.JSEA.Application/DTOs/DashboardDto.cs`
*(Replace dengan versi lengkap)*

```csharp
namespace PTW.JSEA.Application.DTOs;

public class DashboardStatsDto
{
    public int ActivePermitCount { get; set; }
    public int HighRiskPermitCount { get; set; }
    public int ActiveWorkerCount { get; set; }
    public int OverdueAdherenceCount { get; set; }
    public int PendingApprovalCount { get; set; }
    public int UnreadNotificationCount { get; set; }
    public List<ActivePermitDto> ActivePermits { get; set; } = new();
    public List<RecentActivityDto> RecentActivities { get; set; } = new();
    public List<PermitTrendDto> PermitTrend { get; set; } = new();
    public List<WorkTypeCountDto> PermitByWorkType { get; set; } = new();
}

public class ActivePermitDto
{
    public int Id { get; set; }
    public string PermitNumber { get; set; } = string.Empty;
    public string WorkType { get; set; } = string.Empty;
    public string Location { get; set; } = string.Empty;
    public string RequesterName { get; set; } = string.Empty;
    public int WorkerCount { get; set; }
    public DateTime EndTime { get; set; }
    public string RiskLevel { get; set; } = string.Empty;
    public bool IsExpiringSoon { get; set; }
}

public class RecentActivityDto
{
    public DateTime Timestamp { get; set; }
    public string Description { get; set; } = string.Empty;
    public string Module { get; set; } = string.Empty;
    public string ActionBy { get; set; } = string.Empty;
}

public class PermitTrendDto
{
    public string Date { get; set; } = string.Empty;
    public int Count { get; set; }
}

public class WorkTypeCountDto
{
    public string WorkType { get; set; } = string.Empty;
    public int Count { get; set; }
}
```

---

## BAGIAN C — Dashboard Page Model

### FILE: `src/PTW.JSEA.Web/Pages/Dashboard/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Application.DTOs;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Entities;

namespace PTW.JSEA.Web.Pages.Dashboard;

[Authorize]
public class IndexModel : PageModel
{
    private readonly IDashboardService _dashboardService;
    private readonly UserManager<ApplicationUser> _userManager;

    public IndexModel(IDashboardService dashboardService, UserManager<ApplicationUser> userManager)
    {
        _dashboardService = dashboardService;
        _userManager = userManager;
    }

    public DashboardStatsDto Stats { get; set; } = new();
    public string UserFullName { get; set; } = string.Empty;
    public string UserRole { get; set; } = string.Empty;

    public async Task OnGetAsync()
    {
        Stats = await _dashboardService.GetDashboardStatsAsync(User);

        var user = await _userManager.GetUserAsync(User);
        UserFullName = user?.FullName ?? User.Identity?.Name ?? "User";

        var roles = await _userManager.GetRolesAsync(user!);
        UserRole = roles.FirstOrDefault() ?? "User";
    }
}
```

---

## BAGIAN D — Dashboard Page HTML

### FILE: `src/PTW.JSEA.Web/Pages/Dashboard/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Dashboard.IndexModel
@{
    ViewData["Title"] = "Dashboard";
    var trendJson  = System.Text.Json.JsonSerializer.Serialize(Model.Stats.PermitTrend);
    var wtJson     = System.Text.Json.JsonSerializer.Serialize(Model.Stats.PermitByWorkType);
}

<!-- ═══ WELCOME BANNER ══════════════════════════════════════════════════════ -->
<div class="bg-gradient-to-r from-blue-700 to-blue-900 rounded-2xl p-6 mb-6 text-white">
    <div class="flex items-center justify-between">
        <div>
            <p class="text-blue-200 text-sm font-medium mb-1">Selamat datang kembali 👋</p>
            <h2 class="text-2xl font-bold">@Model.UserFullName</h2>
            <p class="text-blue-200 text-sm mt-1">@Model.UserRole — @DateTime.Now.ToString("dddd, dd MMMM yyyy", new System.Globalization.CultureInfo("id-ID"))</p>
        </div>
        <div class="hidden md:flex items-center gap-3">
            <a asp-page="/Permits/Create" class="bg-white text-blue-700 font-semibold px-4 py-2 rounded-lg text-sm hover:bg-blue-50 transition-colors">
                + Buat Permit
            </a>
            <a asp-page="/JSEA/Create" class="bg-blue-600 border border-blue-400 text-white font-semibold px-4 py-2 rounded-lg text-sm hover:bg-blue-500 transition-colors">
                + Buat JSEA
            </a>
        </div>
    </div>
</div>

<!-- ═══ KPI CARDS ════════════════════════════════════════════════════════════ -->
<div class="grid grid-cols-2 lg:grid-cols-5 gap-4 mb-6">

    <!-- Active Permits -->
    <div class="card flex items-center gap-4 col-span-1">
        <div class="stat-icon bg-blue-100">
            <svg class="w-6 h-6 text-blue-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
            </svg>
        </div>
        <div>
            <p id="active-permits-count" class="text-2xl font-bold text-gray-900">@Model.Stats.ActivePermitCount</p>
            <p class="text-xs text-gray-500">Permit Aktif</p>
        </div>
    </div>

    <!-- High Risk -->
    <div class="card flex items-center gap-4 col-span-1">
        <div class="stat-icon bg-red-100">
            <svg class="w-6 h-6 text-red-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/>
            </svg>
        </div>
        <div>
            <p class="text-2xl font-bold text-red-600">@Model.Stats.HighRiskPermitCount</p>
            <p class="text-xs text-gray-500">High Risk</p>
        </div>
    </div>

    <!-- Active Workers -->
    <div class="card flex items-center gap-4 col-span-1">
        <div class="stat-icon bg-green-100">
            <svg class="w-6 h-6 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0z"/>
            </svg>
        </div>
        <div>
            <p id="active-workers-count" class="text-2xl font-bold text-green-600">@Model.Stats.ActiveWorkerCount</p>
            <p class="text-xs text-gray-500">Pekerja Aktif</p>
        </div>
    </div>

    <!-- Overdue Adherence -->
    <div class="card flex items-center gap-4 col-span-1">
        <div class="stat-icon bg-orange-100">
            <svg class="w-6 h-6 text-orange-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
            </svg>
        </div>
        <div>
            <p class="text-2xl font-bold text-orange-600">@Model.Stats.OverdueAdherenceCount</p>
            <p class="text-xs text-gray-500">Adherence Overdue</p>
        </div>
    </div>

    <!-- Pending Approvals -->
    <div class="card flex items-center gap-4 col-span-1">
        <div class="stat-icon bg-yellow-100">
            <svg class="w-6 h-6 text-yellow-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                      d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-3 7h3m-3 4h3m-6-4h.01M9 16h.01"/>
            </svg>
        </div>
        <div>
            <p class="text-2xl font-bold text-yellow-600">@Model.Stats.PendingApprovalCount</p>
            <p class="text-xs text-gray-500">Pending Approval</p>
        </div>
    </div>
</div>

<!-- ═══ CHARTS + ACTIVE TABLE ════════════════════════════════════════════════ -->
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6 mb-6">

    <!-- Permit Trend (line chart, 2/3 width) -->
    <div class="card lg:col-span-2">
        <div class="flex items-center justify-between mb-4">
            <h3 class="section-title mb-0">Tren Permit 7 Hari Terakhir</h3>
            <span class="text-xs text-gray-400">Diperbarui otomatis</span>
        </div>
        <canvas id="trendChart" height="120"></canvas>
    </div>

    <!-- Permit by work type (donut) -->
    <div class="card">
        <h3 class="section-title">Distribusi Jenis Pekerjaan</h3>
        @if (Model.Stats.ActivePermitCount == 0)
        {
            <div class="flex flex-col items-center justify-center h-40 text-gray-300">
                <svg class="w-12 h-12 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M9 19v-6a2 2 0 00-2-2H5a2 2 0 00-2 2v6a2 2 0 002 2h2a2 2 0 002-2zm0 0V9a2 2 0 012-2h2a2 2 0 012 2v10m-6 0a2 2 0 002 2h2a2 2 0 002-2m0 0V5a2 2 0 012-2h2a2 2 0 012 2v14a2 2 0 01-2 2h-2a2 2 0 01-2-2z"/>
                </svg>
                <p class="text-sm">Tidak ada permit aktif</p>
            </div>
        }
        else
        {
            <canvas id="workTypeChart" height="180"></canvas>
        }
    </div>
</div>

<!-- ═══ ACTIVE PERMITS TABLE ═════════════════════════════════════════════════ -->
<div class="card mb-6">
    <div class="page-header mb-4">
        <h3 class="section-title mb-0">Permit Aktif</h3>
        <a asp-page="/Permits/Index" class="text-blue-600 hover:underline text-sm font-medium">
            Lihat semua →
        </a>
    </div>

    @if (!Model.Stats.ActivePermits.Any())
    {
        <div class="flex flex-col items-center py-12 text-gray-300">
            <svg class="w-16 h-16 mb-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
            </svg>
            <p class="text-gray-400 font-medium">Tidak ada permit aktif saat ini</p>
        </div>
    }
    else
    {
        <div class="table-container">
            <table class="table">
                <thead>
                    <tr>
                        <th>No. Permit</th>
                        <th>Jenis Pekerjaan</th>
                        <th>Lokasi</th>
                        <th>Pemohon</th>
                        <th>Pekerja</th>
                        <th>Risk</th>
                        <th>Berakhir</th>
                        <th>Sisa Waktu</th>
                        <th>Aksi</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var p in Model.Stats.ActivePermits)
                    {
                        <tr>
                            <td>
                                <span class="font-mono text-blue-700 font-semibold text-xs">@p.PermitNumber</span>
                            </td>
                            <td>
                                <span class="badge-blue">@p.WorkType.Replace("WorkingAtHeight","At Height").Replace("ConfinedSpaceEntry","Confined Space").Replace("ElectricalIsolation","Electrical")</span>
                            </td>
                            <td class="text-gray-600">@p.Location</td>
                            <td class="text-gray-600">@p.RequesterName</td>
                            <td class="text-center">
                                <span class="badge-gray">@p.WorkerCount org</span>
                            </td>
                            <td>
                                @{
                                    var riskClass = p.RiskLevel switch {
                                        "Low"      => "risk-low",
                                        "Medium"   => "risk-medium",
                                        "High"     => "risk-high",
                                        "Critical" => "risk-critical",
                                        _          => "badge-gray"
                                    };
                                }
                                <span class="@riskClass">@p.RiskLevel</span>
                            </td>
                            <td class="text-gray-600 text-xs">
                                @p.EndTime.ToLocalTime().ToString("dd MMM HH:mm")
                            </td>
                            <td>
                                <span id="countdown-@p.Id"
                                      class="font-mono text-sm @(p.IsExpiringSoon ? "text-red-600 font-bold" : "text-gray-700")">
                                </span>
                            </td>
                            <td>
                                <a asp-page="/Permits/Detail" asp-route-id="@p.Id"
                                   class="text-blue-600 hover:underline text-xs font-medium">
                                    Detail
                                </a>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    }
</div>

<!-- ═══ BOTTOM: RECENT ACTIVITY ══════════════════════════════════════════════ -->
<div class="grid grid-cols-1 lg:grid-cols-2 gap-6">

    <!-- Recent Activity -->
    <div class="card">
        <h3 class="section-title">Aktivitas Terbaru</h3>
        @if (!Model.Stats.RecentActivities.Any())
        {
            <p class="text-gray-400 text-sm text-center py-6">Belum ada aktivitas</p>
        }
        else
        {
            <div class="space-y-3">
                @foreach (var act in Model.Stats.RecentActivities)
                {
                    <div class="flex items-start gap-3">
                        <div class="w-1.5 h-1.5 rounded-full bg-blue-400 mt-2 flex-shrink-0"></div>
                        <div class="flex-1 min-w-0">
                            <p class="text-sm text-gray-700 truncate">@act.Description</p>
                            <p class="text-xs text-gray-400 mt-0.5">
                                @act.Timestamp.ToLocalTime().ToString("dd MMM HH:mm") — @act.ActionBy
                            </p>
                        </div>
                    </div>
                }
            </div>
        }
    </div>

    <!-- Quick Links / Shortcuts -->
    <div class="card">
        <h3 class="section-title">Akses Cepat</h3>
        <div class="grid grid-cols-2 gap-3">
            <a asp-page="/Permits/Create"
               class="flex flex-col items-center justify-center p-4 bg-blue-50 hover:bg-blue-100 rounded-xl transition-colors group">
                <svg class="w-8 h-8 text-blue-600 mb-2 group-hover:scale-110 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4"/>
                </svg>
                <p class="text-sm font-medium text-blue-700">Buat Permit</p>
            </a>
            <a asp-page="/JSEA/Create"
               class="flex flex-col items-center justify-center p-4 bg-green-50 hover:bg-green-100 rounded-xl transition-colors group">
                <svg class="w-8 h-8 text-green-600 mb-2 group-hover:scale-110 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2"/>
                </svg>
                <p class="text-sm font-medium text-green-700">Buat JSEA</p>
            </a>
            <a asp-page="/Permits/MyApprovals"
               class="flex flex-col items-center justify-center p-4 bg-yellow-50 hover:bg-yellow-100 rounded-xl transition-colors group">
                <svg class="w-8 h-8 text-yellow-600 mb-2 group-hover:scale-110 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
                </svg>
                <p class="text-sm font-medium text-yellow-700">Approval Queue</p>
                @if (Model.Stats.PendingApprovalCount > 0)
                {
                    <span class="mt-1 bg-yellow-500 text-white text-xs px-2 py-0.5 rounded-full">
                        @Model.Stats.PendingApprovalCount
                    </span>
                }
            </a>
            <a asp-page="/Monitoring/Active"
               class="flex flex-col items-center justify-center p-4 bg-purple-50 hover:bg-purple-100 rounded-xl transition-colors group">
                <svg class="w-8 h-8 text-purple-600 mb-2 group-hover:scale-110 transition-transform" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"/>
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M2.458 12C3.732 7.943 7.523 5 12 5c4.478 0 8.268 2.943 9.542 7-1.274 4.057-5.064 7-9.542 7-4.477 0-8.268-2.943-9.542-7z"/>
                </svg>
                <p class="text-sm font-medium text-purple-700">Active Work</p>
            </a>
        </div>
    </div>
</div>

@section Scripts {
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script>
// ─── Data dari server ─────────────────────────────────────────────────────
const trendData   = @Html.Raw(trendJson);
const wtData      = @Html.Raw(wtJson);

// ─── Permit Trend Chart (Line) ────────────────────────────────────────────
const trendCtx = document.getElementById('trendChart');
if (trendCtx) {
    new Chart(trendCtx, {
        type: 'line',
        data: {
            labels: trendData.map(d => d.date),
            datasets: [{
                label: 'Permit Dibuat',
                data: trendData.map(d => d.count),
                borderColor: '#3b82f6',
                backgroundColor: 'rgba(59,130,246,0.08)',
                borderWidth: 2.5,
                pointBackgroundColor: '#3b82f6',
                pointRadius: 4,
                tension: 0.4,
                fill: true
            }]
        },
        options: {
            responsive: true,
            plugins: { legend: { display: false } },
            scales: {
                y: { beginAtZero: true, ticks: { stepSize: 1, font: { size: 11 } }, grid: { color: '#f1f5f9' } },
                x: { ticks: { font: { size: 11 } }, grid: { display: false } }
            }
        }
    });
}

// ─── Work Type Donut Chart ────────────────────────────────────────────────
const wtCtx = document.getElementById('workTypeChart');
if (wtCtx && wtData.length > 0) {
    const palette = ['#3b82f6','#ef4444','#f59e0b','#10b981','#8b5cf6','#06b6d4','#f97316'];
    new Chart(wtCtx, {
        type: 'doughnut',
        data: {
            labels: wtData.map(d => d.workType.replace('WorkingAtHeight','At Height').replace('ConfinedSpaceEntry','Confined Space')),
            datasets: [{
                data: wtData.map(d => d.count),
                backgroundColor: palette.slice(0, wtData.length),
                borderWidth: 2,
                borderColor: '#fff'
            }]
        },
        options: {
            responsive: true,
            cutout: '65%',
            plugins: {
                legend: { position: 'bottom', labels: { font: { size: 11 }, padding: 12 } }
            }
        }
    });
}

// ─── Countdown timers ─────────────────────────────────────────────────────
@foreach (var p in Model.Stats.ActivePermits)
{
    <text>startCountdown('@p.EndTime.ToString("o")', 'countdown-@p.Id');</text>
}
</script>
}
```

---

> ✅ Modul 4 selesai.
