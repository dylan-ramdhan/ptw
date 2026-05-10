# MODUL 9 — ACTIVE MONITORING & SIGNALR DASHBOARD
# MODUL 10 — BACKGROUND SERVICES
# MODUL 11 — NOTIFICATION ENGINE
# MODUL 12 — AUDIT TRAIL & REPORTS
# MODUL 13 — USER MANAGEMENT & CERTIFICATE MANAGEMENT
# MODUL 14 — DEPLOYMENT IIS

---

# ══════════════════════════════════════════════════════
# MODUL 9 — ACTIVE MONITORING & SIGNALR DASHBOARD
# ══════════════════════════════════════════════════════

## FILE: `src/PTW.JSEA.Web/Pages/Monitoring/Active.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Monitoring;

[Authorize]
public class ActiveModel : PageModel
{
    private readonly ApplicationDbContext _context;

    public ActiveModel(ApplicationDbContext context) => _context = context;

    public List<WorkPermit> ActivePermits { get; set; } = new();
    public int TotalWorkers { get; set; }
    public int HighRiskCount { get; set; }
    public int OverdueAdherenceCount { get; set; }

    public async Task OnGetAsync()
    {
        ActivePermits = await _context.WorkPermits
            .Include(p => p.Requester)
            .Include(p => p.Workers)
            .Include(p => p.AdherenceLogs)
            .Include(p => p.PAIChecklists)
            .Where(p => p.Status == PermitStatus.Active && !p.IsDeleted)
            .OrderBy(p => p.EndTime)
            .ToListAsync();

        TotalWorkers = ActivePermits.Sum(p =>
            p.Workers.Count(w => w.CheckInTime.HasValue && !w.CheckOutTime.HasValue));

        HighRiskCount = ActivePermits.Count(p =>
            p.RiskLevel == RiskLevel.High || p.RiskLevel == RiskLevel.Critical);

        var threshold = DateTime.UtcNow.AddHours(-4);
        OverdueAdherenceCount = ActivePermits.Count(p =>
        {
            var last = p.AdherenceLogs.OrderByDescending(a => a.CheckTime).FirstOrDefault();
            return last == null || last.CheckTime < threshold;
        });
    }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Monitoring/Active.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Monitoring.ActiveModel
@{
    ViewData["Title"] = "Active Work Monitoring";
}

<!-- Live indicator -->
<div class="flex items-center justify-between mb-6">
    <div>
        <h2 class="page-title">Active Work Monitoring</h2>
        <p class="text-sm text-gray-500">Real-time · diperbarui otomatis via SignalR</p>
    </div>
    <div class="flex items-center gap-2 bg-green-50 border border-green-200 px-4 py-2 rounded-full">
        <span class="w-2.5 h-2.5 bg-green-500 rounded-full animate-pulse"></span>
        <span class="text-green-700 text-sm font-medium">LIVE</span>
    </div>
</div>

<!-- Stats row -->
<div class="grid grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
    <div class="card text-center">
        <p id="monitor-active-count" class="text-3xl font-bold text-blue-600">@Model.ActivePermits.Count</p>
        <p class="text-sm text-gray-500 mt-1">Permit Aktif</p>
    </div>
    <div class="card text-center">
        <p id="monitor-worker-count" class="text-3xl font-bold text-green-600">@Model.TotalWorkers</p>
        <p class="text-sm text-gray-500 mt-1">Pekerja di Lapangan</p>
    </div>
    <div class="card text-center">
        <p class="text-3xl font-bold text-red-600">@Model.HighRiskCount</p>
        <p class="text-sm text-gray-500 mt-1">High / Critical Risk</p>
    </div>
    <div class="card text-center">
        <p class="text-3xl font-bold @(Model.OverdueAdherenceCount > 0 ? "text-orange-600" : "text-gray-400")">
            @Model.OverdueAdherenceCount
        </p>
        <p class="text-sm text-gray-500 mt-1">Adherence Overdue</p>
    </div>
</div>

<!-- Active permits grid -->
@if (!Model.ActivePermits.Any())
{
    <div class="card flex flex-col items-center py-20 text-gray-300">
        <svg class="w-20 h-20 mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1"
                  d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
        </svg>
        <p class="text-gray-400 text-lg font-medium">Tidak ada pekerjaan aktif saat ini</p>
        <p class="text-gray-300 text-sm mt-1">Area kerja dalam kondisi aman</p>
    </div>
}
else
{
    <div class="grid grid-cols-1 md:grid-cols-2 xl:grid-cols-3 gap-4">
        @foreach (var p in Model.ActivePermits)
        {
            var adherenceThreshold = DateTime.UtcNow.AddHours(-4);
            var lastAdherence = p.AdherenceLogs.OrderByDescending(a => a.CheckTime).FirstOrDefault();
            var isAdherenceOverdue = lastAdherence == null || lastAdherence.CheckTime < adherenceThreshold;
            var isExpiringSoon = p.EndTime < DateTime.UtcNow.AddHours(2);

            var borderColor = isExpiringSoon || p.RiskLevel == RiskLevel.Critical ? "border-red-300" :
                              p.RiskLevel == RiskLevel.High || isAdherenceOverdue ? "border-orange-300" :
                              "border-gray-100";

            <div class="card @borderColor border-2 hover:shadow-lg transition-shadow" id="permit-card-@p.Id">
                <!-- Card header -->
                <div class="flex items-start justify-between mb-3">
                    <div>
                        <span class="font-mono text-blue-700 font-bold text-sm">@p.PermitNumber</span>
                        @{var rc = p.RiskLevel switch { RiskLevel.Low => "risk-low", RiskLevel.Medium => "risk-medium", RiskLevel.High => "risk-high", _ => "risk-critical" };}
                        <span class="@rc ml-2 text-xs">@p.RiskLevel</span>
                    </div>
                    @if (isAdherenceOverdue)
                    {
                        <span class="badge-orange text-xs animate-pulse">⚠ Adherence Overdue</span>
                    }
                    @if (isExpiringSoon)
                    {
                        <span class="badge-red text-xs animate-pulse">⏰ Expiring Soon</span>
                    }
                </div>

                <!-- Work type & location -->
                <div class="mb-3">
                    <span class="badge-blue text-xs mb-1">@p.WorkType</span>
                    <p class="font-semibold text-gray-900 mt-1">📍 @p.Location</p>
                    @if (!string.IsNullOrEmpty(p.SpecificArea))
                    {
                        <p class="text-xs text-gray-400">@p.SpecificArea</p>
                    }
                    <p class="text-xs text-gray-500 mt-1">Pemohon: @(p.Requester?.FullName ?? "-")</p>
                </div>

                <!-- Stats row -->
                <div class="grid grid-cols-3 gap-2 mb-3">
                    <div class="bg-gray-50 rounded-lg p-2 text-center">
                        <p class="text-lg font-bold text-gray-800">
                            @p.Workers.Count(w => w.CheckInTime.HasValue && !w.CheckOutTime.HasValue)
                        </p>
                        <p class="text-xs text-gray-400">Aktif</p>
                    </div>
                    <div class="bg-gray-50 rounded-lg p-2 text-center">
                        <p class="text-lg font-bold text-gray-800">@p.Workers.Count</p>
                        <p class="text-xs text-gray-400">Total</p>
                    </div>
                    <div class="bg-gray-50 rounded-lg p-2 text-center">
                        <p class="text-lg font-bold @(p.PAICompleted ? "text-green-600" : "text-red-500")">
                            @(p.PAICompleted ? "✓" : "✗")
                        </p>
                        <p class="text-xs text-gray-400">PAI</p>
                    </div>
                </div>

                <!-- Countdown -->
                <div class="bg-gray-50 rounded-lg p-2 mb-3 flex items-center justify-between">
                    <span class="text-xs text-gray-500">Sisa waktu:</span>
                    <span id="countdown-@p.Id"
                          class="font-mono font-bold text-sm @(isExpiringSoon ? "text-red-600" : "text-gray-700")">
                    </span>
                </div>

                <!-- Last adherence -->
                <div class="text-xs text-gray-400 mb-3">
                    Adherence terakhir:
                    @(lastAdherence != null
                        ? lastAdherence.CheckTime.ToLocalTime().ToString("dd MMM HH:mm")
                        : "Belum ada")
                </div>

                <!-- Actions -->
                <div class="flex gap-2">
                    <a asp-page="/Permits/Detail" asp-route-id="@p.Id"
                       class="btn-outline btn-sm flex-1 text-center">Detail</a>
                    <a asp-page="/Permits/Adherence" asp-route-id="@p.Id"
                       class="btn-primary btn-sm flex-1 text-center">Adherence</a>
                </div>
            </div>
        }
    </div>
}

@section Scripts {
<script>
// Start all countdowns
@foreach (var p in Model.ActivePermits)
{
    <text>startCountdown('@p.EndTime.ToString("o")', 'countdown-@p.Id');</text>
}

// SignalR: handle real-time updates pada monitoring page
if (typeof connection !== 'undefined') {
    connection.on("PermitStatusChanged", (data) => {
        const card = document.getElementById(`permit-card-${data.permitId}`);
        if (card && (data.newStatus === 'Closed' || data.newStatus === 'Suspended')) {
            card.style.opacity = '0.4';
            card.style.transition = 'opacity 1s';
            setTimeout(() => card.remove(), 1500);

            const countEl = document.getElementById('monitor-active-count');
            if (countEl) countEl.textContent = Math.max(0, parseInt(countEl.textContent) - 1);
        }
    });
}

// Auto-refresh page setiap 5 menit sebagai fallback
setTimeout(() => location.reload(), 300000);
</script>
}
```

---

# ══════════════════════════════════════════════════════
# MODUL 10 — BACKGROUND SERVICES
# ══════════════════════════════════════════════════════

## FILE: `src/PTW.JSEA.Infrastructure/BackgroundServices/PermitExpiryService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.BackgroundServices;

public class PermitExpiryService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<PermitExpiryService> _logger;

    public PermitExpiryService(IServiceProvider services, ILogger<PermitExpiryService> logger)
    {
        _services = services;
        _logger   = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("PermitExpiryService started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope    = _services.CreateScope();
                var context        = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
                var notifier       = scope.ServiceProvider.GetRequiredService<IDashboardNotifier>();
                var notifService   = scope.ServiceProvider.GetRequiredService<INotificationService>();

                // 1. Auto-expire permit yang EndTime sudah lewat
                var expired = await context.WorkPermits
                    .Where(p => p.Status == PermitStatus.Active
                             && p.EndTime < DateTime.UtcNow
                             && !p.IsDeleted)
                    .ToListAsync(stoppingToken);

                foreach (var permit in expired)
                {
                    permit.Status    = PermitStatus.Closed;
                    permit.ClosedAt  = DateTime.UtcNow;
                    permit.CloseReason = "Auto-expired: melebihi waktu selesai.";
                    permit.UpdatedAt = DateTime.UtcNow;

                    await notifService.CreateInAppAsync(
                        permit.RequesterId,
                        "Permit Expired",
                        $"Permit {permit.PermitNumber} telah expired dan otomatis ditutup.",
                        NotificationType.PermitExpired,
                        $"/Permits/Detail/{permit.Id}");

                    await notifier.NotifyPermitStatusChanged(permit.Id, "Expired");
                    _logger.LogInformation("Permit {No} auto-expired.", permit.PermitNumber);
                }

                // 2. Warn permit yang hampir expired (< 2 jam)
                var expiringSoon = await context.WorkPermits
                    .Where(p => p.Status == PermitStatus.Active
                             && p.EndTime < DateTime.UtcNow.AddHours(2)
                             && p.EndTime > DateTime.UtcNow
                             && !p.IsDeleted)
                    .ToListAsync(stoppingToken);

                foreach (var permit in expiringSoon)
                {
                    // Hanya kirim sekali (cek apakah sudah pernah dikirim dalam 1 jam terakhir)
                    var alreadyNotified = await context.Notifications
                        .AnyAsync(n => n.UserId == permit.RequesterId
                                    && n.Type == NotificationType.PermitExpiring
                                    && n.CreatedAt > DateTime.UtcNow.AddHours(-1)
                                    && n.Message.Contains(permit.PermitNumber), stoppingToken);

                    if (!alreadyNotified)
                    {
                        var remaining = (permit.EndTime - DateTime.UtcNow).TotalMinutes;
                        await notifService.CreateInAppAsync(
                            permit.RequesterId,
                            "Permit Hampir Expired",
                            $"Permit {permit.PermitNumber} akan expired dalam {remaining:F0} menit.",
                            NotificationType.PermitExpiring,
                            $"/Permits/Detail/{permit.Id}");
                    }
                }

                if (expired.Any() || expiringSoon.Any())
                    await context.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in PermitExpiryService.");
            }

            // Jalankan setiap 5 menit
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

---

## FILE: `src/PTW.JSEA.Infrastructure/BackgroundServices/AdherenceReminderService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.BackgroundServices;

public class AdherenceReminderService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<AdherenceReminderService> _logger;

    public AdherenceReminderService(IServiceProvider services, ILogger<AdherenceReminderService> logger)
    {
        _services = services;
        _logger   = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("AdherenceReminderService started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope  = _services.CreateScope();
                var context      = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
                var notifier     = scope.ServiceProvider.GetRequiredService<IDashboardNotifier>();
                var notifService = scope.ServiceProvider.GetRequiredService<INotificationService>();

                var threshold = DateTime.UtcNow.AddHours(-4);

                var activePermits = await context.WorkPermits
                    .Include(p => p.AdherenceLogs)
                    .Where(p => p.Status == PermitStatus.Active && !p.IsDeleted)
                    .ToListAsync(stoppingToken);

                foreach (var permit in activePermits)
                {
                    var lastLog = permit.AdherenceLogs
                        .OrderByDescending(l => l.CheckTime)
                        .FirstOrDefault();

                    bool isOverdue = lastLog == null || lastLog.CheckTime < threshold;
                    if (!isOverdue) continue;

                    // Cek apakah sudah kirim notif overdue dalam 30 menit terakhir
                    var recentlySent = await context.Notifications
                        .AnyAsync(n => n.Type == NotificationType.AdherenceOverdue
                                    && n.Message.Contains(permit.PermitNumber)
                                    && n.CreatedAt > DateTime.UtcNow.AddMinutes(-30), stoppingToken);

                    if (recentlySent) continue;

                    // Kirim ke HSE Officer dan Area Owner
                    foreach (var role in new[] { "HSEOfficer", "SafetyManager" })
                    {
                        await notifService.CreateInAppForRoleAsync(
                            role,
                            "Adherence Overdue",
                            $"Permit {permit.PermitNumber} di {permit.Location} belum ada adherence check > 4 jam.",
                            NotificationType.AdherenceOverdue,
                            $"/Permits/Detail/{permit.Id}");
                    }

                    await notifier.NotifyAdherenceOverdue(permit.Id, permit.PermitNumber);
                    _logger.LogWarning("Adherence overdue: {No}", permit.PermitNumber);
                }

                await context.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in AdherenceReminderService.");
            }

            // Cek setiap 30 menit
            await Task.Delay(TimeSpan.FromMinutes(30), stoppingToken);
        }
    }
}
```

---

## FILE: `src/PTW.JSEA.Infrastructure/BackgroundServices/CertificateExpiryService.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.BackgroundServices;

public class CertificateExpiryService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<CertificateExpiryService> _logger;

    public CertificateExpiryService(IServiceProvider services, ILogger<CertificateExpiryService> logger)
    {
        _services = services;
        _logger   = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope  = _services.CreateScope();
                var context      = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
                var notifService = scope.ServiceProvider.GetRequiredService<INotificationService>();

                var warningDate = DateTime.UtcNow.AddDays(30);

                // Sertifikat yang expired — update status
                var justExpired = await context.UserCertificates
                    .Where(c => c.Status == CertificateStatus.Active
                             && c.ExpiryDate < DateTime.UtcNow
                             && !c.IsDeleted)
                    .ToListAsync(stoppingToken);

                foreach (var cert in justExpired)
                {
                    cert.Status    = CertificateStatus.Expired;
                    cert.UpdatedAt = DateTime.UtcNow;

                    await notifService.CreateInAppAsync(
                        cert.UserId,
                        "Sertifikat Expired",
                        $"Sertifikat {cert.CertificateName} ({cert.WorkType}) Anda telah expired pada {cert.ExpiryDate:dd MMM yyyy}.",
                        NotificationType.CertificateExpired,
                        "/Admin/Certificates/Index");

                    // Notif ke HSE Officer
                    await notifService.CreateInAppForRoleAsync(
                        "HSEOfficer",
                        "Sertifikat User Expired",
                        $"Sertifikat {cert.CertificateName} milik user ID {cert.UserId} telah expired.",
                        NotificationType.CertificateExpired);
                }

                // Warning 30 hari sebelum expired
                var expiringSoon = await context.UserCertificates
                    .Include(c => c.User)
                    .Where(c => c.Status == CertificateStatus.Active
                             && c.ExpiryDate <= warningDate
                             && c.ExpiryDate > DateTime.UtcNow
                             && !c.IsDeleted)
                    .ToListAsync(stoppingToken);

                foreach (var cert in expiringSoon)
                {
                    var alreadyWarned = await context.Notifications
                        .AnyAsync(n => n.UserId == cert.UserId
                                    && n.Type == NotificationType.CertificateExpiring
                                    && n.CreatedAt > DateTime.UtcNow.AddDays(-1)
                                    && n.Message.Contains(cert.CertificateName), stoppingToken);

                    if (alreadyWarned) continue;

                    var daysLeft = (cert.ExpiryDate - DateTime.UtcNow).Days;
                    await notifService.CreateInAppAsync(
                        cert.UserId,
                        "Sertifikat Hampir Expired",
                        $"Sertifikat {cert.CertificateName} ({cert.WorkType}) akan expired dalam {daysLeft} hari ({cert.ExpiryDate:dd MMM yyyy}). Segera perbarui.",
                        NotificationType.CertificateExpiring,
                        "/Admin/Certificates/Index");
                }

                if (justExpired.Any())
                    await context.SaveChangesAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in CertificateExpiryService.");
            }

            // Jalankan sekali sehari (jam 7 pagi)
            var now  = DateTime.Now;
            var next = now.Date.AddDays(1).AddHours(7);
            await Task.Delay(next - now, stoppingToken);
        }
    }
}
```

---

## Daftarkan di `DependencyInjection.cs` Infrastructure:

```csharp
// Tambahkan ke AddInfrastructureServices
services.AddHostedService<PermitExpiryService>();
services.AddHostedService<AdherenceReminderService>();
services.AddHostedService<CertificateExpiryService>();
```

---

# ══════════════════════════════════════════════════════
# MODUL 11 — NOTIFICATION ENGINE (EMAIL)
# ══════════════════════════════════════════════════════

## FILE: `src/PTW.JSEA.Infrastructure/Services/EmailService.cs`

```csharp
using MailKit.Net.Smtp;
using MailKit.Security;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using MimeKit;

namespace PTW.JSEA.Infrastructure.Services;

public interface IEmailService
{
    Task SendAsync(string toEmail, string toName, string subject, string htmlBody);
}

public class EmailService : IEmailService
{
    private readonly IConfiguration _config;
    private readonly ILogger<EmailService> _logger;

    public EmailService(IConfiguration config, ILogger<EmailService> logger)
    {
        _config = config;
        _logger = logger;
    }

    public async Task SendAsync(string toEmail, string toName, string subject, string htmlBody)
    {
        try
        {
            var settings = _config.GetSection("EmailSettings");
            var message  = new MimeMessage();

            message.From.Add(new MailboxAddress(
                settings["SenderName"] ?? "PTW System",
                settings["SenderEmail"] ?? "noreply@ptw.com"));
            message.To.Add(new MailboxAddress(toName, toEmail));
            message.Subject = subject;

            var bodyBuilder = new BodyBuilder
            {
                HtmlBody = WrapInTemplate(subject, htmlBody)
            };
            message.Body = bodyBuilder.ToMessageBody();

            using var smtp = new SmtpClient();
            await smtp.ConnectAsync(
                settings["SmtpHost"] ?? "smtp.gmail.com",
                int.Parse(settings["SmtpPort"] ?? "587"),
                SecureSocketOptions.StartTls);
            await smtp.AuthenticateAsync(
                settings["Username"] ?? "",
                settings["Password"] ?? "");
            await smtp.SendAsync(message);
            await smtp.DisconnectAsync(true);

            _logger.LogInformation("Email sent to {Email}: {Subject}", toEmail, subject);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send email to {Email}", toEmail);
        }
    }

    private static string WrapInTemplate(string title, string content) => $"""
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8"/>
            <style>
                body {{ font-family: Arial, sans-serif; background: #f4f6f9; margin: 0; padding: 0; }}
                .container {{ max-width: 560px; margin: 32px auto; background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.08); }}
                .header {{ background: linear-gradient(135deg, #1e3a8a, #2563eb); padding: 28px 32px; }}
                .header h1 {{ color: white; margin: 0; font-size: 18px; font-weight: 600; }}
                .header p {{ color: #93c5fd; margin: 4px 0 0; font-size: 13px; }}
                .body {{ padding: 28px 32px; }}
                .body p {{ color: #374151; line-height: 1.6; margin: 0 0 12px; }}
                .btn {{ display: inline-block; background: #2563eb; color: white; padding: 12px 24px; border-radius: 8px; text-decoration: none; font-weight: 600; margin-top: 8px; }}
                .footer {{ background: #f9fafb; padding: 16px 32px; text-align: center; color: #9ca3af; font-size: 12px; border-top: 1px solid #e5e7eb; }}
            </style>
        </head>
        <body>
            <div class="container">
                <div class="header">
                    <h1>🛡 PTW & JSEA System</h1>
                    <p>PT. Mattel Indonesia — Safety Management</p>
                </div>
                <div class="body">
                    <p style="font-size:16px; font-weight:600; color:#111827;">{title}</p>
                    {content}
                </div>
                <div class="footer">
                    Email otomatis dari PTW & JSEA System · PT. Mattel Indonesia<br/>
                    Jangan balas email ini.
                </div>
            </div>
        </body>
        </html>
        """;
}
```

---

## FILE: `src/PTW.JSEA.Infrastructure/Services/NotificationDispatcher.cs`

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.SignalR;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.Services;

/// <summary>
/// Mengirim notifikasi real-time ke browser via SignalR ketika
/// notifikasi in-app baru dibuat.
/// </summary>
public class NotificationDispatcher
{
    private readonly IHubContext<PTW.JSEA.Web.Hubs.PermitHub> _hubContext;
    private readonly UserManager<ApplicationUser> _userManager;

    public NotificationDispatcher(
        IHubContext<PTW.JSEA.Web.Hubs.PermitHub> hubContext,
        UserManager<ApplicationUser> userManager)
    {
        _hubContext  = hubContext;
        _userManager = userManager;
    }

    public async Task PushToUserAsync(string userId, string title, string message)
    {
        await _hubContext.Clients.User(userId)
            .SendAsync("NewNotification", new { title, message, timestamp = DateTime.UtcNow });
    }
}
```

---

## Tambahkan API endpoint notifikasi di `Program.cs`:

```csharp
// Tambahkan setelah endpoint /api/notifications/count yang sudah ada

app.MapGet("/api/notifications/unread", async (
    UserManager<ApplicationUser> um,
    INotificationService ns,
    ClaimsPrincipal user) =>
{
    var userId = um.GetUserId(user);
    if (userId == null) return Results.Unauthorized();
    var notifs = await ns.GetUnreadAsync(userId);
    return Results.Ok(notifs.Select(n => new {
        n.Id, n.Title, n.Message, n.IsRead,
        n.Link, n.CreatedAt, Type = n.Type.ToString()
    }));
}).RequireAuthorization();

app.MapPost("/api/notifications/read-all", async (
    UserManager<ApplicationUser> um,
    INotificationService ns,
    ClaimsPrincipal user) =>
{
    var userId = um.GetUserId(user);
    if (userId == null) return Results.Unauthorized();
    await ns.MarkAllAsReadAsync(userId);
    return Results.Ok();
}).RequireAuthorization();
```

---

# ══════════════════════════════════════════════════════
# MODUL 12 — AUDIT TRAIL & REPORTS
# ══════════════════════════════════════════════════════

## FILE: `src/PTW.JSEA.Web/Pages/Admin/AuditTrail/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Admin.AuditTrail;

[Authorize(Roles = "SystemAdmin,SafetyManager,Auditor")]
public class IndexModel : PageModel
{
    private readonly ApplicationDbContext _context;

    public IndexModel(ApplicationDbContext context) => _context = context;

    public List<AuditLog> Logs { get; set; } = new();

    [BindProperty(SupportsGet = true)] public string? Module  { get; set; }
    [BindProperty(SupportsGet = true)] public string? Action  { get; set; }
    [BindProperty(SupportsGet = true)] public DateTime? DateFrom { get; set; }
    [BindProperty(SupportsGet = true)] public DateTime? DateTo   { get; set; }
    [BindProperty(SupportsGet = true)] public int Page { get; set; } = 1;
    public int TotalPages { get; set; }
    public const int PageSize = 50;

    public async Task OnGetAsync()
    {
        var query = _context.AuditLogs.AsQueryable();

        if (!string.IsNullOrEmpty(Module))
            query = query.Where(l => l.Module == Module);
        if (!string.IsNullOrEmpty(Action))
            query = query.Where(l => l.Action.Contains(Action));
        if (DateFrom.HasValue)
            query = query.Where(l => l.ActionDate >= DateFrom.Value.ToUniversalTime());
        if (DateTo.HasValue)
            query = query.Where(l => l.ActionDate <= DateTo.Value.ToUniversalTime().AddDays(1));

        var total = await query.CountAsync();
        TotalPages = (int)Math.Ceiling(total / (double)PageSize);

        Logs = await query
            .OrderByDescending(l => l.ActionDate)
            .Skip((Page - 1) * PageSize)
            .Take(PageSize)
            .ToListAsync();
    }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/AuditTrail/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Admin.AuditTrail.IndexModel
@{
    ViewData["Title"] = "Audit Trail";
}

<div class="page-header">
    <div>
        <h2 class="page-title">Audit Trail</h2>
        <p class="text-sm text-gray-500">Rekam jejak semua aktivitas sistem</p>
    </div>
    <a href="/api/reports/audit/export?module=@Model.Module&dateFrom=@Model.DateFrom?.ToString("yyyy-MM-dd")&dateTo=@Model.DateTo?.ToString("yyyy-MM-dd")"
       class="btn-outline">⬇ Export CSV</a>
</div>

<!-- Filter -->
<div class="card mb-4">
    <form method="get" class="flex flex-wrap items-end gap-3">
        <div class="w-36">
            <label class="form-label">Modul</label>
            <select asp-for="Module" class="form-select">
                <option value="">Semua</option>
                <option value="WorkPermit">Permit</option>
                <option value="JSEA">JSEA</option>
                <option value="PAI">PAI</option>
                <option value="User">User</option>
            </select>
        </div>
        <div class="w-40">
            <label class="form-label">Action</label>
            <input asp-for="Action" class="form-input" placeholder="Created, Approved..."/>
        </div>
        <div class="w-40">
            <label class="form-label">Dari</label>
            <input asp-for="DateFrom" type="date" class="form-input"/>
        </div>
        <div class="w-40">
            <label class="form-label">Sampai</label>
            <input asp-for="DateTo" type="date" class="form-input"/>
        </div>
        <button type="submit" class="btn-primary">Filter</button>
        <a asp-page="/Admin/AuditTrail/Index" class="btn-outline">Reset</a>
    </form>
</div>

<!-- Table -->
<div class="card">
    <div class="table-container">
        <table class="table text-xs">
            <thead>
                <tr>
                    <th>Waktu</th>
                    <th>Modul</th>
                    <th>Entity ID</th>
                    <th>Action</th>
                    <th>Dilakukan Oleh</th>
                    <th>IP Address</th>
                    <th>Nilai Lama</th>
                    <th>Nilai Baru</th>
                    <th>Catatan</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var log in Model.Logs)
                {
                    <tr>
                        <td class="text-gray-500 whitespace-nowrap">@log.ActionDate.ToLocalTime().ToString("dd MMM HH:mm:ss")</td>
                        <td><span class="badge-blue">@log.Module</span></td>
                        <td class="text-gray-500">#@log.EntityId</td>
                        <td class="font-medium">@log.Action</td>
                        <td class="text-gray-600">@log.ActionBy</td>
                        <td class="text-gray-400 font-mono">@(log.IPAddress ?? "-")</td>
                        <td class="text-gray-400 max-w-xs truncate" title="@log.OldValue">@(log.OldValue ?? "-")</td>
                        <td class="text-gray-700 max-w-xs truncate" title="@log.NewValue">@(log.NewValue ?? "-")</td>
                        <td class="text-gray-400">@(log.Notes ?? "-")</td>
                    </tr>
                }
            </tbody>
        </table>
    </div>

    <!-- Pagination -->
    @if (Model.TotalPages > 1)
    {
        <div class="flex items-center justify-between mt-4 pt-4 border-t border-gray-100">
            <p class="text-xs text-gray-400">Halaman @Model.Page dari @Model.TotalPages</p>
            <div class="flex gap-1">
                @for (int i = 1; i <= Math.Min(Model.TotalPages, 10); i++)
                {
                    <a asp-page="/Admin/AuditTrail/Index"
                       asp-route-page="@i"
                       asp-route-module="@Model.Module"
                       asp-route-dateFrom="@Model.DateFrom?.ToString("yyyy-MM-dd")"
                       asp-route-dateTo="@Model.DateTo?.ToString("yyyy-MM-dd")"
                       class="px-3 py-1 rounded text-xs @(i == Model.Page ? "bg-blue-600 text-white" : "bg-gray-100 text-gray-600 hover:bg-gray-200")">
                        @i
                    </a>
                }
            </div>
        </div>
    }
</div>
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Reports/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Reports;

[Authorize]
public class IndexModel : PageModel
{
    private readonly ApplicationDbContext _context;

    public IndexModel(ApplicationDbContext context) => _context = context;

    public ReportSummary Summary { get; set; } = new();

    [BindProperty(SupportsGet = true)] public int Month { get; set; } = DateTime.Now.Month;
    [BindProperty(SupportsGet = true)] public int Year  { get; set; } = DateTime.Now.Year;

    public async Task OnGetAsync()
    {
        var from = new DateTime(Year, Month, 1, 0, 0, 0, DateTimeKind.Utc);
        var to   = from.AddMonths(1);

        Summary.TotalPermits = await _context.WorkPermits
            .CountAsync(p => p.CreatedAt >= from && p.CreatedAt < to && !p.IsDeleted);

        Summary.ActivePermits = await _context.WorkPermits
            .CountAsync(p => p.Status == PermitStatus.Active && !p.IsDeleted);

        Summary.ClosedPermits = await _context.WorkPermits
            .CountAsync(p => p.Status == PermitStatus.Closed
                          && p.CreatedAt >= from && p.CreatedAt < to && !p.IsDeleted);

        Summary.RejectedPermits = await _context.WorkPermits
            .CountAsync(p => p.Status == PermitStatus.Rejected
                          && p.CreatedAt >= from && p.CreatedAt < to && !p.IsDeleted);

        Summary.TotalJSEAs = await _context.JSEAs
            .CountAsync(j => j.CreatedAt >= from && j.CreatedAt < to && !j.IsDeleted);

        Summary.ApprovedJSEAs = await _context.JSEAs
            .CountAsync(j => j.Status == JSEAStatus.Approved
                          && j.CreatedAt >= from && j.CreatedAt < to && !j.IsDeleted);

        Summary.NonCompliantAdherence = await _context.AdherenceLogs
            .CountAsync(a => !a.IsCompliant && a.CheckTime >= from && a.CheckTime < to);

        Summary.ExpiredCertificates = await _context.UserCertificates
            .CountAsync(c => c.Status == CertificateStatus.Expired && !c.IsDeleted);

        Summary.ByWorkType = await _context.WorkPermits
            .Where(p => p.CreatedAt >= from && p.CreatedAt < to && !p.IsDeleted)
            .GroupBy(p => p.WorkType)
            .Select(g => new WorkTypeStat { WorkType = g.Key.ToString(), Count = g.Count() })
            .ToListAsync();

        Summary.ByRiskLevel = await _context.WorkPermits
            .Where(p => p.CreatedAt >= from && p.CreatedAt < to && !p.IsDeleted)
            .GroupBy(p => p.RiskLevel)
            .Select(g => new RiskStat { RiskLevel = g.Key.ToString(), Count = g.Count() })
            .ToListAsync();
    }

    public class ReportSummary
    {
        public int TotalPermits          { get; set; }
        public int ActivePermits         { get; set; }
        public int ClosedPermits         { get; set; }
        public int RejectedPermits       { get; set; }
        public int TotalJSEAs            { get; set; }
        public int ApprovedJSEAs         { get; set; }
        public int NonCompliantAdherence { get; set; }
        public int ExpiredCertificates   { get; set; }
        public List<WorkTypeStat> ByWorkType  { get; set; } = new();
        public List<RiskStat>     ByRiskLevel { get; set; } = new();
    }

    public class WorkTypeStat { public string WorkType { get; set; } = ""; public int Count { get; set; } }
    public class RiskStat     { public string RiskLevel { get; set; } = ""; public int Count { get; set; } }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Reports/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Reports.IndexModel
@{
    ViewData["Title"] = "Reports & Analytics";
    var wtJson = System.Text.Json.JsonSerializer.Serialize(Model.Summary.ByWorkType);
    var rlJson = System.Text.Json.JsonSerializer.Serialize(Model.Summary.ByRiskLevel);
}

<div class="page-header mb-6">
    <div>
        <h2 class="page-title">Reports & Analytics</h2>
        <p class="text-sm text-gray-500">Laporan keselamatan kerja bulanan</p>
    </div>
    <form method="get" class="flex items-end gap-2">
        <div>
            <label class="form-label">Bulan</label>
            <select asp-for="Month" class="form-select w-32">
                @for (int m = 1; m <= 12; m++)
                {
                    <option value="@m" @(m == Model.Month ? "selected" : "")>
                        @new DateTime(2000, m, 1).ToString("MMMM")
                    </option>
                }
            </select>
        </div>
        <div>
            <label class="form-label">Tahun</label>
            <select asp-for="Year" class="form-select w-24">
                @for (int y = DateTime.Now.Year; y >= DateTime.Now.Year - 3; y--)
                {
                    <option value="@y" @(y == Model.Year ? "selected" : "")>@y</option>
                }
            </select>
        </div>
        <button type="submit" class="btn-primary">Tampilkan</button>
    </form>
</div>

<!-- Summary KPI -->
<div class="grid grid-cols-2 lg:grid-cols-4 gap-4 mb-6">
    <div class="card text-center border-blue-100">
        <p class="text-3xl font-bold text-blue-600">@Model.Summary.TotalPermits</p>
        <p class="text-sm text-gray-500 mt-1">Total Permit</p>
    </div>
    <div class="card text-center border-green-100">
        <p class="text-3xl font-bold text-green-600">@Model.Summary.ClosedPermits</p>
        <p class="text-sm text-gray-500 mt-1">Selesai</p>
    </div>
    <div class="card text-center border-red-100">
        <p class="text-3xl font-bold text-red-600">@Model.Summary.RejectedPermits</p>
        <p class="text-sm text-gray-500 mt-1">Ditolak</p>
    </div>
    <div class="card text-center border-orange-100">
        <p class="text-3xl font-bold text-orange-600">@Model.Summary.NonCompliantAdherence</p>
        <p class="text-sm text-gray-500 mt-1">Non-Compliant</p>
    </div>
</div>

<div class="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
    <div class="card">
        <h3 class="section-title">Permit per Jenis Pekerjaan</h3>
        <canvas id="workTypeChart" height="220"></canvas>
    </div>
    <div class="card">
        <h3 class="section-title">Permit per Risk Level</h3>
        <canvas id="riskChart" height="220"></canvas>
    </div>
</div>

<!-- Safety metrics -->
<div class="card">
    <h3 class="section-title">Ringkasan Keselamatan</h3>
    <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
        <div class="text-center p-4 bg-gray-50 rounded-xl">
            <p class="text-2xl font-bold text-gray-800">@Model.Summary.TotalJSEAs</p>
            <p class="text-sm text-gray-500">Total JSEA</p>
        </div>
        <div class="text-center p-4 bg-green-50 rounded-xl">
            <p class="text-2xl font-bold text-green-700">@Model.Summary.ApprovedJSEAs</p>
            <p class="text-sm text-gray-500">JSEA Approved</p>
        </div>
        <div class="text-center p-4 bg-red-50 rounded-xl">
            <p class="text-2xl font-bold text-red-700">@Model.Summary.ExpiredCertificates</p>
            <p class="text-sm text-gray-500">Sertifikat Expired</p>
        </div>
        <div class="text-center p-4 bg-blue-50 rounded-xl">
            @{
                var rate = Model.Summary.TotalPermits > 0
                    ? (Model.Summary.ClosedPermits * 100.0 / Model.Summary.TotalPermits)
                    : 0;
            }
            <p class="text-2xl font-bold text-blue-700">@rate.ToString("F0")%</p>
            <p class="text-sm text-gray-500">Completion Rate</p>
        </div>
    </div>
</div>

@section Scripts {
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<script>
const wtData = @Html.Raw(wtJson);
const rlData = @Html.Raw(rlJson);
const palette = ['#3b82f6','#ef4444','#f59e0b','#10b981','#8b5cf6','#06b6d4','#f97316'];

new Chart(document.getElementById('workTypeChart'), {
    type: 'bar',
    data: {
        labels: wtData.map(d => d.workType.replace('WorkingAtHeight','At Height').replace('ConfinedSpaceEntry','Conf.Space')),
        datasets: [{ label: 'Permit', data: wtData.map(d => d.count), backgroundColor: palette, borderRadius: 6 }]
    },
    options: { responsive: true, plugins: { legend: { display: false } }, scales: { y: { beginAtZero: true, ticks: { stepSize: 1 } } } }
});

new Chart(document.getElementById('riskChart'), {
    type: 'doughnut',
    data: {
        labels: rlData.map(d => d.riskLevel),
        datasets: [{ data: rlData.map(d => d.count), backgroundColor: ['#10b981','#f59e0b','#f97316','#ef4444'], borderWidth: 2 }]
    },
    options: { responsive: true, cutout: '60%', plugins: { legend: { position: 'bottom' } } }
});
</script>
}
```

---

# ══════════════════════════════════════════════════════
# MODUL 13 — USER MANAGEMENT & CERTIFICATE MANAGEMENT
# ══════════════════════════════════════════════════════

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Admin.Users;

[Authorize(Roles = "SystemAdmin,SafetyManager")]
public class IndexModel : PageModel
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ApplicationDbContext _context;

    public IndexModel(UserManager<ApplicationUser> userManager, ApplicationDbContext context)
    {
        _userManager = userManager;
        _context     = context;
    }

    public List<UserWithRole> Users { get; set; } = new();
    [BindProperty(SupportsGet = true)] public string? Search { get; set; }

    public async Task OnGetAsync()
    {
        var query = _userManager.Users.AsQueryable();

        if (!string.IsNullOrEmpty(Search))
            query = query.Where(u => u.FullName.Contains(Search) || u.Email!.Contains(Search) || (u.EmployeeId != null && u.EmployeeId.Contains(Search)));

        var users = await query.OrderBy(u => u.FullName).ToListAsync();

        var result = new List<UserWithRole>();
        foreach (var u in users)
        {
            var roles = await _userManager.GetRolesAsync(u);
            var certCount = await _context.UserCertificates.CountAsync(c => c.UserId == u.Id && !c.IsDeleted);
            result.Add(new UserWithRole { User = u, Role = roles.FirstOrDefault() ?? "-", CertificateCount = certCount });
        }
        Users = result;
    }

    public async Task<IActionResult> OnPostToggleActiveAsync(string userId)
    {
        var user = await _userManager.FindByIdAsync(userId);
        if (user == null) return NotFound();
        user.IsActive = !user.IsActive;
        await _userManager.UpdateAsync(user);
        TempData["Success"] = $"User {user.FullName} status diperbarui.";
        return RedirectToPage();
    }

    public class UserWithRole
    {
        public ApplicationUser User { get; set; } = null!;
        public string Role { get; set; } = string.Empty;
        public int CertificateCount { get; set; }
    }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Admin.Users.IndexModel
@{
    ViewData["Title"] = "Manajemen User";
}

<div class="page-header">
    <div>
        <h2 class="page-title">Manajemen User</h2>
        <p class="text-sm text-gray-500">@Model.Users.Count pengguna terdaftar</p>
    </div>
    <a asp-page="/Admin/Users/Create" class="btn-primary">+ Tambah User</a>
</div>

<div class="card mb-4">
    <form method="get" class="flex gap-3">
        <input asp-for="Search" class="form-input flex-1" placeholder="Nama / email / employee ID..."/>
        <button type="submit" class="btn-primary">Cari</button>
        <a asp-page="/Admin/Users/Index" class="btn-outline">Reset</a>
    </form>
</div>

<div class="card">
    <div class="table-container">
        <table class="table">
            <thead>
                <tr>
                    <th>Nama</th>
                    <th>Email</th>
                    <th>Employee ID</th>
                    <th>Departemen</th>
                    <th>Role</th>
                    <th>Sertifikat</th>
                    <th>Status</th>
                    <th>Login Terakhir</th>
                    <th>Aksi</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var item in Model.Users)
                {
                    var u = item.User;
                    <tr>
                        <td class="font-medium text-gray-900">@u.FullName</td>
                        <td class="text-gray-600 text-xs">@u.Email</td>
                        <td class="text-gray-500 text-xs font-mono">@(u.EmployeeId ?? "-")</td>
                        <td class="text-gray-600 text-xs">@(u.Department ?? "-")</td>
                        <td>
                            @{
                                var roleColor = item.Role switch {
                                    "SystemAdmin"   => "badge-red",
                                    "SafetyManager" => "badge-purple",
                                    "HSEOfficer"    => "badge-blue",
                                    "AreaOwner"     => "badge-orange",
                                    "Approver"      => "badge-yellow",
                                    "Requester"     => "badge-gray",
                                    _               => "badge-gray"
                                };
                            }
                            <span class="@roleColor text-xs">@item.Role</span>
                        </td>
                        <td class="text-center text-xs">
                            <a asp-page="/Admin/Certificates/Index" asp-route-userId="@u.Id"
                               class="text-blue-600 hover:underline">@item.CertificateCount cert</a>
                        </td>
                        <td>
                            <span class="@(u.IsActive ? "badge-green" : "badge-red") text-xs">
                                @(u.IsActive ? "Aktif" : "Nonaktif")
                            </span>
                        </td>
                        <td class="text-gray-400 text-xs">
                            @(u.LastLoginAt?.ToLocalTime().ToString("dd MMM HH:mm") ?? "Belum pernah")
                        </td>
                        <td>
                            <div class="flex items-center gap-2">
                                <a asp-page="/Admin/Users/Edit" asp-route-id="@u.Id"
                                   class="text-blue-600 hover:underline text-xs">Edit</a>
                                <form method="post" asp-page-handler="ToggleActive"
                                      asp-route-userId="@u.Id" class="inline">
                                    @Html.AntiForgeryToken()
                                    <button type="submit"
                                            class="@(u.IsActive ? "text-red-500" : "text-green-600") hover:underline text-xs"
                                            onclick="return confirm('Ubah status user ini?')">
                                        @(u.IsActive ? "Nonaktifkan" : "Aktifkan")
                                    </button>
                                </form>
                            </div>
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Create.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.AspNetCore.Mvc.Rendering;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Admin.Users;

[Authorize(Roles = "SystemAdmin")]
public class CreateModel : PageModel
{
    private readonly UserManager<ApplicationUser> _userManager;

    public CreateModel(UserManager<ApplicationUser> userManager) => _userManager = userManager;

    [BindProperty] public UserInputModel Input { get; set; } = new();

    public List<SelectListItem> RoleOptions { get; set; } = DatabaseSeeder.SystemRoles
        .Select(r => new SelectListItem { Value = r, Text = r }).ToList();

    public class UserInputModel
    {
        public string FullName   { get; set; } = string.Empty;
        public string Email      { get; set; } = string.Empty;
        public string? EmployeeId{ get; set; }
        public string? Department{ get; set; }
        public string? Position  { get; set; }
        public string? PhoneWhatsApp { get; set; }
        public string Role       { get; set; } = "Requester";
        public string Password   { get; set; } = string.Empty;
    }

    public void OnGet() { }

    public async Task<IActionResult> OnPostAsync()
    {
        if (!ModelState.IsValid) return Page();

        var user = new ApplicationUser
        {
            UserName      = Input.Email,
            Email         = Input.Email,
            FullName      = Input.FullName,
            EmployeeId    = Input.EmployeeId,
            Department    = Input.Department,
            Position      = Input.Position,
            PhoneWhatsApp = Input.PhoneWhatsApp,
            IsActive      = true,
            EmailConfirmed= true,
            CreatedAt     = DateTime.UtcNow
        };

        var result = await _userManager.CreateAsync(user, Input.Password);
        if (!result.Succeeded)
        {
            foreach (var e in result.Errors)
                ModelState.AddModelError("", e.Description);
            return Page();
        }

        await _userManager.AddToRoleAsync(user, Input.Role);
        TempData["Success"] = $"User {user.FullName} berhasil dibuat.";
        return RedirectToPage("/Admin/Users/Index");
    }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Create.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Admin.Users.CreateModel
@{
    ViewData["Title"] = "Tambah User";
}

<div class="page-header">
    <h2 class="page-title">Tambah User Baru</h2>
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
                <label class="form-label">Email <span class="text-red-500">*</span></label>
                <input asp-for="Input.Email" type="email" class="form-input" required/>
            </div>
            <div>
                <label class="form-label">Employee ID</label>
                <input asp-for="Input.EmployeeId" class="form-input" placeholder="MAT-XXX-001"/>
            </div>
            <div>
                <label class="form-label">Departemen</label>
                <input asp-for="Input.Department" class="form-input" placeholder="HSE, Production..."/>
            </div>
            <div>
                <label class="form-label">Jabatan</label>
                <input asp-for="Input.Position" class="form-input" placeholder="Officer, Supervisor..."/>
            </div>
            <div>
                <label class="form-label">No. WhatsApp</label>
                <input asp-for="Input.PhoneWhatsApp" class="form-input" placeholder="08xx..."/>
            </div>
            <div>
                <label class="form-label">Role <span class="text-red-500">*</span></label>
                <select asp-for="Input.Role" asp-items="Model.RoleOptions" class="form-select"></select>
            </div>
            <div>
                <label class="form-label">Password <span class="text-red-500">*</span></label>
                <input asp-for="Input.Password" type="password" class="form-input"
                       placeholder="Min 8 karakter, huruf besar, angka, simbol"/>
            </div>
        </div>
        <div class="mt-6 flex gap-3">
            <button type="submit" class="btn-primary">Buat User</button>
            <a asp-page="/Admin/Users/Index" class="btn-outline">Batal</a>
        </div>
    </form>
</div>
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Certificates/Index.cshtml.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Application.Services;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Web.Pages.Admin.Certificates;

[Authorize(Roles = "SystemAdmin,SafetyManager,HSEOfficer")]
public class IndexModel : PageModel
{
    private readonly ApplicationDbContext _context;
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly IFileStorageService _fileStorage;

    public IndexModel(ApplicationDbContext context, UserManager<ApplicationUser> userManager,
        IFileStorageService fileStorage)
    {
        _context     = context;
        _userManager = userManager;
        _fileStorage = fileStorage;
    }

    public List<UserCertificate> Certificates { get; set; } = new();
    [BindProperty(SupportsGet = true)] public string? UserId      { get; set; }
    [BindProperty(SupportsGet = true)] public string? StatusFilter{ get; set; }
    [BindProperty] public CertInputModel Input { get; set; } = new();

    public class CertInputModel
    {
        public string UserId            { get; set; } = string.Empty;
        public WorkType WorkType        { get; set; }
        public string CertificateNumber { get; set; } = string.Empty;
        public string CertificateName  { get; set; } = string.Empty;
        public string? IssuingBody     { get; set; }
        public DateTime IssuedDate     { get; set; } = DateTime.Today;
        public DateTime ExpiryDate     { get; set; } = DateTime.Today.AddYears(2);
        public IFormFile? File         { get; set; }
    }

    public async Task OnGetAsync()
    {
        var query = _context.UserCertificates
            .Include(c => c.User)
            .AsQueryable();

        if (!string.IsNullOrEmpty(UserId))
            query = query.Where(c => c.UserId == UserId);

        if (!string.IsNullOrEmpty(StatusFilter) && Enum.TryParse<CertificateStatus>(StatusFilter, out var status))
            query = query.Where(c => c.Status == status);

        Certificates = await query
            .OrderBy(c => c.ExpiryDate)
            .ToListAsync();
    }

    public async Task<IActionResult> OnPostAsync()
    {
        var cert = new UserCertificate
        {
            UserId            = Input.UserId,
            WorkType          = Input.WorkType,
            CertificateNumber = Input.CertificateNumber,
            CertificateName   = Input.CertificateName,
            IssuingBody       = Input.IssuingBody,
            IssuedDate        = Input.IssuedDate.ToUniversalTime(),
            ExpiryDate        = Input.ExpiryDate.ToUniversalTime(),
            Status            = CertificateStatus.Active,
            CreatedAt         = DateTime.UtcNow,
            CreatedBy         = _userManager.GetUserId(User)
        };

        if (Input.File != null)
        {
            var allowed = new[] { ".pdf", ".jpg", ".jpeg", ".png" };
            if (_fileStorage.IsValidFile(Input.File, allowed))
                cert.FilePath = await _fileStorage.SaveFileAsync(Input.File, "certificates");
        }

        _context.UserCertificates.Add(cert);
        await _context.SaveChangesAsync();
        TempData["Success"] = "Sertifikat berhasil ditambahkan.";
        return RedirectToPage();
    }

    public async Task<IActionResult> OnPostRevokeAsync(int certId)
    {
        var cert = await _context.UserCertificates.FindAsync(certId);
        if (cert != null)
        {
            cert.Status    = CertificateStatus.Revoked;
            cert.UpdatedAt = DateTime.UtcNow;
            await _context.SaveChangesAsync();
        }
        TempData["Success"] = "Sertifikat dicabut.";
        return RedirectToPage();
    }
}
```

---

## FILE: `src/PTW.JSEA.Web/Pages/Admin/Certificates/Index.cshtml`

```html
@page
@model PTW.JSEA.Web.Pages.Admin.Certificates.IndexModel
@{
    ViewData["Title"] = "Manajemen Sertifikasi";
}

@if (TempData["Success"] != null)
{
    <div class="alert-success mb-4"><svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/></svg><span>@TempData["Success"]</span></div>
}

<div class="page-header">
    <div>
        <h2 class="page-title">Manajemen Sertifikasi</h2>
        <p class="text-sm text-gray-500">@Model.Certificates.Count sertifikat terdaftar</p>
    </div>
    <button onclick="document.getElementById('addCertModal').classList.remove('hidden')"
            class="btn-primary">+ Tambah Sertifikat</button>
</div>

<!-- Filter -->
<div class="card mb-4">
    <form method="get" class="flex flex-wrap items-end gap-3">
        <div>
            <label class="form-label">Status</label>
            <select asp-for="StatusFilter" class="form-select w-36">
                <option value="">Semua</option>
                <option value="Active">Active</option>
                <option value="Expired">Expired</option>
                <option value="Revoked">Revoked</option>
            </select>
        </div>
        <button type="submit" class="btn-primary">Filter</button>
        <a asp-page="/Admin/Certificates/Index" class="btn-outline">Reset</a>
    </form>
</div>

<!-- Warning: near-expiry -->
@{
    var nearExpiry = Model.Certificates.Where(c => c.Status == CertificateStatus.Active && c.DaysUntilExpiry <= 30 && c.DaysUntilExpiry > 0).ToList();
}
@if (nearExpiry.Any())
{
    <div class="alert-warning mb-4">
        <svg class="w-5 h-5 flex-shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"/></svg>
        <span>⚠️ <strong>@nearExpiry.Count sertifikat</strong> akan expired dalam 30 hari ke depan.</span>
    </div>
}

<!-- Table -->
<div class="card">
    <div class="table-container">
        <table class="table text-xs">
            <thead>
                <tr>
                    <th>User</th>
                    <th>Jenis Pekerjaan</th>
                    <th>Nama Sertifikat</th>
                    <th>No. Sertifikat</th>
                    <th>Penerbit</th>
                    <th>Berlaku Sampai</th>
                    <th>Sisa Hari</th>
                    <th>Status</th>
                    <th>Aksi</th>
                </tr>
            </thead>
            <tbody>
                @foreach (var c in Model.Certificates)
                {
                    var rowBg = c.Status == CertificateStatus.Expired ? "bg-red-50" :
                                c.DaysUntilExpiry <= 30 && c.Status == CertificateStatus.Active ? "bg-yellow-50" : "";
                    <tr class="@rowBg">
                        <td class="font-medium">@(c.User?.FullName ?? c.UserId)</td>
                        <td><span class="badge-blue">@c.WorkType</span></td>
                        <td>@c.CertificateName</td>
                        <td class="font-mono text-gray-500">@c.CertificateNumber</td>
                        <td class="text-gray-500">@(c.IssuingBody ?? "-")</td>
                        <td class="@(c.Status == CertificateStatus.Expired ? "text-red-600 font-bold" : "text-gray-700")">
                            @c.ExpiryDate.ToLocalTime().ToString("dd MMM yyyy")
                        </td>
                        <td class="text-center">
                            @{
                                var days = c.DaysUntilExpiry;
                                var dayClass = c.Status == CertificateStatus.Expired ? "text-red-600 font-bold" :
                                              days <= 30 ? "text-orange-600 font-bold" : "text-gray-600";
                            }
                            <span class="@dayClass">
                                @(c.Status == CertificateStatus.Expired ? "Expired" : $"{days} hari")
                            </span>
                        </td>
                        <td>
                            @{
                                var sc = c.Status switch {
                                    CertificateStatus.Active  => "badge-green",
                                    CertificateStatus.Expired => "badge-red",
                                    CertificateStatus.Revoked => "badge-gray",
                                    _                         => "badge-gray"
                                };
                            }
                            <span class="@sc">@c.Status</span>
                        </td>
                        <td>
                            @if (!string.IsNullOrEmpty(c.FilePath))
                            {
                                <a href="/@c.FilePath" target="_blank" class="text-blue-600 hover:underline mr-2">Lihat</a>
                            }
                            @if (c.Status == CertificateStatus.Active)
                            {
                                <form method="post" asp-page-handler="Revoke" asp-route-certId="@c.Id" class="inline">
                                    @Html.AntiForgeryToken()
                                    <button type="submit" onclick="return confirm('Cabut sertifikat ini?')"
                                            class="text-red-500 hover:underline">Cabut</button>
                                </form>
                            }
                        </td>
                    </tr>
                }
            </tbody>
        </table>
    </div>
</div>

<!-- Add Certificate Modal -->
<div id="addCertModal" class="hidden fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4">
    <div class="bg-white rounded-2xl shadow-2xl p-6 w-full max-w-lg">
        <h3 class="font-bold text-lg mb-5">Tambah Sertifikat</h3>
        <form method="post" enctype="multipart/form-data">
            @Html.AntiForgeryToken()
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                <div class="md:col-span-2">
                    <label class="form-label">User ID <span class="text-red-500">*</span></label>
                    <input asp-for="Input.UserId" class="form-input" placeholder="User ID dari sistem"/>
                    <p class="form-hint">Salin dari halaman Manajemen User</p>
                </div>
                <div>
                    <label class="form-label">Jenis Pekerjaan</label>
                    <select asp-for="Input.WorkType" class="form-select">
                        <option value="HotWork">Hot Work</option>
                        <option value="ColdWork">Cold Work</option>
                        <option value="ElectricalIsolation">Electrical Isolation</option>
                        <option value="ConfinedSpaceEntry">Confined Space</option>
                        <option value="Excavation">Excavation</option>
                        <option value="Lifting">Lifting</option>
                        <option value="WorkingAtHeight">Working at Height</option>
                    </select>
                </div>
                <div>
                    <label class="form-label">No. Sertifikat <span class="text-red-500">*</span></label>
                    <input asp-for="Input.CertificateNumber" class="form-input" required/>
                </div>
                <div class="md:col-span-2">
                    <label class="form-label">Nama Sertifikat <span class="text-red-500">*</span></label>
                    <input asp-for="Input.CertificateName" class="form-input" required/>
                </div>
                <div>
                    <label class="form-label">Penerbit</label>
                    <input asp-for="Input.IssuingBody" class="form-input" placeholder="BNSP, K3..."/>
                </div>
                <div>
                    <label class="form-label">Tanggal Terbit</label>
                    <input asp-for="Input.IssuedDate" type="date" class="form-input"/>
                </div>
                <div>
                    <label class="form-label">Berlaku Sampai <span class="text-red-500">*</span></label>
                    <input asp-for="Input.ExpiryDate" type="date" class="form-input" required/>
                </div>
                <div>
                    <label class="form-label">Upload File (PDF/Foto)</label>
                    <input name="Input.File" type="file" accept=".pdf,.jpg,.jpeg,.png" class="form-input text-xs"/>
                </div>
            </div>
            <div class="flex gap-3 justify-end">
                <button type="button" onclick="document.getElementById('addCertModal').classList.add('hidden')"
                        class="btn-outline">Batal</button>
                <button type="submit" class="btn-primary">Simpan Sertifikat</button>
            </div>
        </form>
    </div>
</div>
```

---

### Stub pages yang masih dibutuhkan:

```bash
# Buat file kosong untuk Edit User
```

### FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Edit.cshtml`
```html
@page "{id}"
@model PTW.JSEA.Web.Pages.Admin.Users.EditModel
@{ ViewData["Title"] = "Edit User"; }
<p class="text-gray-500">Edit user — implementasi serupa dengan Create.</p>
```

### FILE: `src/PTW.JSEA.Web/Pages/Admin/Users/Edit.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
namespace PTW.JSEA.Web.Pages.Admin.Users;
[Authorize(Roles = "SystemAdmin")] public class EditModel : PageModel { public void OnGet(string id) { } }
```

---

# ══════════════════════════════════════════════════════
# MODUL 14 — DEPLOYMENT KE IIS
# ══════════════════════════════════════════════════════

## LANGKAH 14.1 — Persiapan Build Production

### Update `appsettings.Production.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=NAMA-SERVER\\SQLEXPRESS;Database=PTW_JSEA_PROD;Trusted_Connection=True;TrustServerCertificate=True;MultipleActiveResultSets=true"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Warning",
      "Override": {
        "Microsoft": "Error",
        "System": "Error"
      }
    }
  },
  "AppSettings": {
    "SystemName": "PTW & JSEA System",
    "CompanyName": "PT. Mattel Indonesia",
    "PermitAdherenceIntervalHours": 4,
    "CertificateExpiryWarningDays": 30,
    "PermitExpiryWarningHours": 2,
    "FileStorageType": "Local",
    "LocalStoragePath": "wwwroot/uploads",
    "MaxFileSizeMB": 10
  },
  "EmailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": "587",
    "SenderEmail": "noreply@ptw-mattel.com",
    "SenderName": "PTW System Mattel",
    "Username": "ISI_EMAIL_PENGIRIM",
    "Password": "ISI_APP_PASSWORD"
  }
}
```

---

## LANGKAH 14.2 — Build CSS Production

```bash
cd src/PTW.JSEA.Web
npm run build:css
cd ../..
```

---

## LANGKAH 14.3 — Publish Aplikasi

```bash
cd C:\Projects\PTW-JSEA

dotnet publish src/PTW.JSEA.Web/PTW.JSEA.Web.csproj ^
  -c Release ^
  -o C:\Deploy\PTW-JSEA ^
  --self-contained false ^
  -r win-x64
```

---

## LANGKAH 14.4 — Setup di Server Production

### A. Install .NET 8 Hosting Bundle di server:
```
https://dotnet.microsoft.com/en-us/download/dotnet/8.0
→ ASP.NET Core Runtime 8.x — Windows Hosting Bundle
→ Jalankan installer, restart server
```

### B. Buat folder di server:
```
C:\inetpub\wwwroot\PTW-JSEA\
```

### C. Copy hasil publish ke server:
```
Salin semua isi C:\Deploy\PTW-JSEA\ ke C:\inetpub\wwwroot\PTW-JSEA\
```

### D. Buat folder logs dan uploads di server:
```bash
mkdir C:\inetpub\wwwroot\PTW-JSEA\logs
mkdir C:\inetpub\wwwroot\PTW-JSEA\wwwroot\uploads
```

---

## LANGKAH 14.5 — Konfigurasi IIS

### A. Buka IIS Manager → Application Pools:
```
Klik kanan → Add Application Pool
  Name: PTW-JSEA-Pool
  .NET CLR Version: No Managed Code
  Managed Pipeline Mode: Integrated
```

### B. Buat Website baru:
```
Klik kanan Sites → Add Website
  Site name: PTW-JSEA
  Application pool: PTW-JSEA-Pool
  Physical path: C:\inetpub\wwwroot\PTW-JSEA
  Port: 80 (atau 8080 jika port 80 sudah dipakai)
  Host name: ptw.mattel.local (atau IP server)
```

### C. Set Permission folder:
```
Klik kanan C:\inetpub\wwwroot\PTW-JSEA → Properties → Security → Edit → Add
Masukkan: IIS AppPool\PTW-JSEA-Pool
Permission: Full Control (untuk folder logs dan uploads)
Permission: Read & Execute (untuk folder lain)
```

---

## LANGKAH 14.6 — Pastikan `web.config` Ada

File ini otomatis dibuat saat publish. Pastikan isinya:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*"
           modules="AspNetCoreModuleV2" resourceType="Unspecified" />
    </handlers>
    <aspNetCore processPath="dotnet"
                arguments=".\PTW.JSEA.Web.dll"
                stdoutLogEnabled="true"
                stdoutLogFile=".\logs\stdout"
                hostingModel="inprocess">
      <environmentVariables>
        <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
      </environmentVariables>
    </aspNetCore>
  </system.webServer>
</configuration>
```

---

## LANGKAH 14.7 — Jalankan Migration di Production

Di server production, buka Command Prompt sebagai Administrator:

```bash
cd C:\inetpub\wwwroot\PTW-JSEA

set ASPNETCORE_ENVIRONMENT=Production

# Jalankan migration (dari development machine yang punya akses ke DB server)
dotnet ef database update ^
  --project src/PTW.JSEA.Infrastructure ^
  --startup-project src/PTW.JSEA.Web ^
  --connection "Server=NAMA-SERVER\SQLEXPRESS;Database=PTW_JSEA_PROD;Trusted_Connection=True;TrustServerCertificate=True"
```

Atau jalankan SQL migration script yang di-generate:

```bash
dotnet ef migrations script ^
  --project src/PTW.JSEA.Infrastructure ^
  --startup-project src/PTW.JSEA.Web ^
  --output C:\migration.sql
# Kemudian jalankan migration.sql di SSMS
```

---

## LANGKAH 14.8 — Test Deployment

### 1. Buka browser di server:
```
http://localhost
```

### 2. Test dari jaringan lokal (ganti dengan IP server):
```
http://192.168.1.XXX
```

### 3. Cek log jika ada error:
```
C:\inetpub\wwwroot\PTW-JSEA\logs\
```

---

## LANGKAH 14.9 — SQL Server: Beri Akses ke IIS

Jalankan di SSMS di server production:

```sql
-- Buat login untuk IIS AppPool
USE [master]
GO
CREATE LOGIN [IIS APPPOOL\PTW-JSEA-Pool] FROM WINDOWS WITH DEFAULT_DATABASE=[master]
GO

-- Beri akses ke database
USE [PTW_JSEA_PROD]
GO
CREATE USER [IIS APPPOOL\PTW-JSEA-Pool] FOR LOGIN [IIS APPPOOL\PTW-JSEA-Pool]
GO

ALTER ROLE [db_datareader] ADD MEMBER [IIS APPPOOL\PTW-JSEA-Pool]
GO
ALTER ROLE [db_datawriter] ADD MEMBER [IIS APPPOOL\PTW-JSEA-Pool]
GO
ALTER ROLE [db_ddladmin] ADD MEMBER [IIS APPPOOL\PTW-JSEA-Pool]
GO
```

---

## LANGKAH 14.10 — Firewall

```bash
# Buka port HTTP (jalankan di PowerShell sebagai Admin)
netsh advfirewall firewall add rule name="PTW-HTTP" dir=in action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="PTW-HTTPS" dir=in action=allow protocol=TCP localport=443
```

---

## LANGKAH 14.11 — Auto-Start IIS Site

Di IIS Manager:
```
Sites → PTW-JSEA → klik kanan → Manage Website → Start
Application Pools → PTW-JSEA-Pool → klik kanan → Start
```

Set start mode ke Always Running:
```
Application Pools → PTW-JSEA-Pool → Advanced Settings
  Start Mode: AlwaysRunning
  Idle Time-out: 0
```

---

## CHECKLIST FINAL DEPLOYMENT

```
SETUP
[ ] .NET 8 Hosting Bundle terinstall di server
[ ] IIS Application Pool: PTW-JSEA-Pool (No Managed Code)
[ ] Website PTW-JSEA terbuat dan mengarah ke folder publish
[ ] Permission folder logs dan uploads: Full Control
[ ] SQL Server: IIS AppPool user punya akses ke database

DATABASE
[ ] Database PTW_JSEA_PROD terbuat
[ ] Migration berhasil dijalankan
[ ] Seed data (roles + admin user) sudah ada
[ ] Tabel ApprovalMatrix sudah terisi

APLIKASI
[ ] Login berhasil di http://server-ip
[ ] Redirect ke dashboard setelah login
[ ] Buat JSEA berhasil
[ ] Buat Permit berhasil — nomor permit auto-generate
[ ] Approval flow berjalan (HSE → Area → Final)
[ ] Certificate check memblokir approver tanpa sertifikat
[ ] PAI form bisa diisi dan foto tersimpan
[ ] Adherence log tersimpan
[ ] Dashboard counter update real-time (SignalR)
[ ] Notification in-app muncul
[ ] Background service berjalan (cek log setelah 5 menit)

KEAMANAN
[ ] Halaman /Admin hanya bisa diakses SystemAdmin
[ ] Requester tidak bisa approve permit sendiri
[ ] URL manipulation (akses detail permit orang lain) diblokir
[ ] Logout berfungsi dan redirect ke login

LOG
[ ] File log terbuat di folder logs/
[ ] Audit trail tercatat di database
```

---

## AKUN DEFAULT (Login Pertama)

| Email | Password | Role |
|---|---|---|
| admin@ptw-mattel.com | Mattel@123456 | SystemAdmin |
| hse@ptw-mattel.com | Mattel@123456 | HSEOfficer |
| areaowner@ptw-mattel.com | Mattel@123456 | AreaOwner |
| approver@ptw-mattel.com | Mattel@123456 | Approver |
| requester@ptw-mattel.com | Mattel@123456 | Requester |

> ⚠️ **Ganti semua password default segera setelah deployment production!**

---

## URUTAN PENGGUNAAN PERTAMA KALI (Happy Path)

```
1. Login sebagai SystemAdmin
   → Buat user HSE, Area Owner, Approver, Requester sesuai personel PT. Mattel

2. Login sebagai SystemAdmin / HSEOfficer
   → Tambahkan sertifikat untuk semua approver

3. Login sebagai Requester
   → Buat JSEA → Submit

4. Login sebagai HSEOfficer
   → Review JSEA → Approve

5. Login sebagai Requester
   → Buat Permit (pilih JSEA yang sudah approved) → Submit

6. Login sebagai HSEOfficer
   → Approve Step 1 (HSE Review)

7. Login sebagai AreaOwner
   → Approve Step 2 (Area Approval)
   → (Untuk High/Critical) Login sebagai Approver → Approve Step 3

8. Status Permit = ACTIVE ✓

9. Login sebagai HSEOfficer
   → Buka Permit Detail → Lakukan PAI

10. Login sebagai siapa saja
    → Lakukan Adherence Check setiap 4 jam

11. Pekerjaan selesai → Close Permit
```

---

> ✅ SEMUA MODUL (1–14) SELESAI.
>
> Sistem PTW & JSEA PT. Mattel Indonesia siap digunakan.
>
> Jika ada error pada modul tertentu, tunjukkan pesan error-nya dan saya bantu debug.
