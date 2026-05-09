# Panduan Lengkap Pembangunan Sistem Digital PTW & JSEA
### Stack: ASP.NET Core 8 · Razor Pages · TailwindCSS · SQL Server Express · IIS
### Target: PT. Mattel Indonesia — Windows (VS Code)

---

> **Cara baca panduan ini:**
> Ikuti tahap demi tahap secara berurutan. Jangan skip tahap. Setiap tahap ada **tujuan**, **langkah teknis**, dan **verifikasi** bahwa tahap berhasil sebelum lanjut.

---

# TAHAP 0 — PERSIAPAN ENVIRONMENT

## 0.1 Install Software Yang Dibutuhkan

### A. .NET 8 SDK
```
https://dotnet.microsoft.com/en-us/download/dotnet/8.0
→ Download: .NET 8.0 SDK (Windows x64)
→ Jalankan installer
```
Verifikasi:
```bash
dotnet --version
# Output: 8.x.x
```

### B. SQL Server Express 2022
```
https://www.microsoft.com/en-us/sql-server/sql-server-downloads
→ Download: Express Edition
→ Pilih "Basic" installation type
→ Catat instance name (default: SQLEXPRESS)
```

### C. SQL Server Management Studio (SSMS) — opsional tapi sangat direkomendasikan
```
https://aka.ms/ssmsfullsetup
```

### D. VS Code + Extensions
Install VS Code extensions berikut:
- **C# Dev Kit** (Microsoft) — wajib
- **C#** (Microsoft) — wajib
- **SQL Server (mssql)** — wajib
- **Tailwind CSS IntelliSense** — wajib
- **GitLens** — sangat direkomendasikan
- **Thunder Client** — untuk test API

### E. Git
```
https://git-scm.com/download/win
```
Verifikasi:
```bash
git --version
```

### F. Node.js (untuk TailwindCSS build)
```
https://nodejs.org → LTS version
```
Verifikasi:
```bash
node --version
npm --version
```

---

## 0.2 Setup SQL Server Express

### Aktifkan TCP/IP Protocol:
1. Buka **SQL Server Configuration Manager**
2. SQL Server Network Configuration → Protocols for SQLEXPRESS
3. Enable **TCP/IP**
4. Restart SQL Server service

### Test koneksi di SSMS:
```
Server: localhost\SQLEXPRESS
Auth: Windows Authentication
```

---

## 0.3 Buat Folder Struktur Project

```bash
mkdir C:\Projects\PTW-JSEA
cd C:\Projects\PTW-JSEA
git init
```

---

# TAHAP 1 — PROJECT SETUP & STRUKTUR CLEAN ARCHITECTURE

## 1.1 Buat Solution dan Projects

```bash
cd C:\Projects\PTW-JSEA

# Buat solution
dotnet new sln -n PTW.JSEA

# Buat projects per layer
dotnet new classlib -n PTW.JSEA.Domain -o src/PTW.JSEA.Domain
dotnet new classlib -n PTW.JSEA.Application -o src/PTW.JSEA.Application
dotnet new classlib -n PTW.JSEA.Infrastructure -o src/PTW.JSEA.Infrastructure
dotnet new webapp -n PTW.JSEA.Web -o src/PTW.JSEA.Web

# Tambahkan ke solution
dotnet sln add src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj
dotnet sln add src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet sln add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj
dotnet sln add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj
```

## 1.2 Set Project References

```bash
# Application depends on Domain
dotnet add src/PTW.JSEA.Application/PTW.JSEA.Application.csproj reference src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj

# Infrastructure depends on Application + Domain
dotnet add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj reference src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj reference src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj

# Web depends on semua layer
dotnet add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj reference src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj reference src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj
```

## 1.3 Install NuGet Packages

### Domain (tidak perlu package external)

### Application
```bash
cd src/PTW.JSEA.Application
dotnet add package MediatR
dotnet add package FluentValidation
dotnet add package AutoMapper
dotnet add package Microsoft.Extensions.DependencyInjection.Abstractions
```

### Infrastructure
```bash
cd src/PTW.JSEA.Infrastructure
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.MSSqlServer
dotnet add package MailKit
dotnet add package Azure.Storage.Blobs
dotnet add package SixLabors.ImageSharp
```

### Web
```bash
cd src/PTW.JSEA.Web
dotnet add package Microsoft.AspNetCore.SignalR
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Serilog.AspNetCore
```

## 1.4 Struktur Folder Domain

Buat struktur folder berikut di `src/PTW.JSEA.Domain/`:

```
Domain/
├── Entities/
│   ├── User.cs
│   ├── UserCertificate.cs
│   ├── JSEA.cs
│   ├── JSEADetail.cs
│   ├── WorkPermit.cs
│   ├── PermitApproval.cs
│   ├── PermitWorker.cs
│   ├── AdherenceLog.cs
│   ├── AuditLog.cs
│   ├── ApprovalMatrix.cs
│   └── Notification.cs
├── Enums/
│   ├── PermitStatus.cs
│   ├── JSEAStatus.cs
│   ├── WorkType.cs
│   ├── RiskLevel.cs
│   └── NotificationChannel.cs
├── Interfaces/
│   ├── Repositories/
│   │   ├── IGenericRepository.cs
│   │   ├── IWorkPermitRepository.cs
│   │   ├── IJSEARepository.cs
│   │   └── IUserRepository.cs
│   └── Services/
│       ├── INotificationService.cs
│       ├── IFileStorageService.cs
│       └── IAuditService.cs
└── Common/
    └── BaseEntity.cs
```

---

# TAHAP 2 — DATABASE & ENTITY FRAMEWORK CORE

## 2.1 Buat Entity Classes

### `src/PTW.JSEA.Domain/Common/BaseEntity.cs`
```csharp
namespace PTW.JSEA.Domain.Common;

public abstract class BaseEntity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
    public bool IsDeleted { get; set; } = false;
}
```

### `src/PTW.JSEA.Domain/Enums/PermitStatus.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum PermitStatus
{
    Draft = 0,
    Submitted = 1,
    WaitingHSEReview = 2,
    WaitingPAI = 3,
    WaitingAreaApproval = 4,
    WaitingFinalApproval = 5,
    Active = 6,
    Suspended = 7,
    Closed = 8,
    Rejected = 9,
    Archived = 10
}
```

### `src/PTW.JSEA.Domain/Enums/WorkType.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum WorkType
{
    HotWork = 1,
    ColdWork = 2,
    ElectricalIsolation = 3,
    ConfinedSpaceEntry = 4,
    Excavation = 5,
    Lifting = 6,
    WorkingAtHeight = 7
}
```

### `src/PTW.JSEA.Domain/Enums/RiskLevel.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum RiskLevel
{
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4
}
```

### `src/PTW.JSEA.Domain/Entities/WorkPermit.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class WorkPermit : BaseEntity
{
    public string PermitNumber { get; set; } = string.Empty;
    public int JSEAId { get; set; }
    public string Location { get; set; } = string.Empty;
    public WorkType WorkType { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public PermitStatus Status { get; set; } = PermitStatus.Draft;
    public string? SuspendReason { get; set; }
    public DateTime? SuspendedAt { get; set; }
    public string RequesterId { get; set; } = string.Empty;
    public string? Description { get; set; }

    // Navigation properties
    public JSEA JSEA { get; set; } = null!;
    public ApplicationUser Requester { get; set; } = null!;
    public ICollection<PermitApproval> Approvals { get; set; } = new List<PermitApproval>();
    public ICollection<PermitWorker> Workers { get; set; } = new List<PermitWorker>();
    public ICollection<AdherenceLog> AdherenceLogs { get; set; } = new List<AdherenceLog>();
    public ICollection<PAIChecklist> PAIChecklists { get; set; } = new List<PAIChecklist>();
}
```

### `src/PTW.JSEA.Domain/Entities/JSEA.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class JSEA : BaseEntity
{
    public string WorkTitle { get; set; } = string.Empty;
    public string Location { get; set; } = string.Empty;
    public WorkType WorkType { get; set; }
    public RiskLevel RiskLevel { get; set; }
    public JSEAStatus Status { get; set; } = JSEAStatus.Draft;
    public string RequesterId { get; set; } = string.Empty;
    public string? ApprovedById { get; set; }
    public DateTime? ApprovedAt { get; set; }
    public int Version { get; set; } = 1;
    public int? ParentJSEAId { get; set; }

    // Navigation
    public ApplicationUser Requester { get; set; } = null!;
    public ApplicationUser? ApprovedBy { get; set; }
    public ICollection<JSEADetail> Details { get; set; } = new List<JSEADetail>();
    public ICollection<JSEAAttachment> Attachments { get; set; } = new List<JSEAAttachment>();
    public ICollection<WorkPermit> WorkPermits { get; set; } = new List<WorkPermit>();
}
```

> **CATATAN:** Buat semua entity lainnya dengan pola yang sama: UserCertificate, JSEADetail, PermitApproval, PermitWorker, AdherenceLog, AuditLog, ApprovalMatrix, Notification, PAIChecklist, JSEAAttachment.

## 2.2 Buat ApplicationDbContext

### `src/PTW.JSEA.Infrastructure/Data/ApplicationDbContext.cs`
```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;

namespace PTW.JSEA.Infrastructure.Data;

public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    public DbSet<JSEA> JSEAs => Set<JSEA>();
    public DbSet<JSEADetail> JSEADetails => Set<JSEADetail>();
    public DbSet<JSEAAttachment> JSEAAttachments => Set<JSEAAttachment>();
    public DbSet<WorkPermit> WorkPermits => Set<WorkPermit>();
    public DbSet<PermitApproval> PermitApprovals => Set<PermitApproval>();
    public DbSet<PermitWorker> PermitWorkers => Set<PermitWorker>();
    public DbSet<AdherenceLog> AdherenceLogs => Set<AdherenceLog>();
    public DbSet<AuditLog> AuditLogs => Set<AuditLog>();
    public DbSet<ApprovalMatrix> ApprovalMatrices => Set<ApprovalMatrix>();
    public DbSet<Notification> Notifications => Set<Notification>();
    public DbSet<UserCertificate> UserCertificates => Set<UserCertificate>();
    public DbSet<PAIChecklist> PAIChecklists => Set<PAIChecklist>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        // Soft delete global filter
        builder.Entity<WorkPermit>().HasQueryFilter(x => !x.IsDeleted);
        builder.Entity<JSEA>().HasQueryFilter(x => !x.IsDeleted);
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return base.SaveChangesAsync(cancellationToken);
    }
}
```

## 2.3 Setup Connection String

### `src/PTW.JSEA.Web/appsettings.json`
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost\\SQLEXPRESS;Database=PTW_JSEA_DB;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "AppSettings": {
    "PermitAdherenceIntervalHours": 4,
    "CertificateExpiryWarningDays": 30,
    "FileStorageType": "Local",
    "LocalStoragePath": "wwwroot/uploads"
  }
}
```

## 2.4 Setup Program.cs

### `src/PTW.JSEA.Web/Program.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Identity;
using PTW.JSEA.Infrastructure.Data;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Infrastructure;
using PTW.JSEA.Application;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/ptw-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

builder.Host.UseSerilog();

// Database
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Identity
builder.Services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();

// Session & Cookie
builder.Services.ConfigureApplicationCookie(options =>
{
    options.LoginPath = "/Account/Login";
    options.LogoutPath = "/Account/Logout";
    options.AccessDeniedPath = "/Account/AccessDenied";
    options.ExpireTimeSpan = TimeSpan.FromHours(8);
    options.SlidingExpiration = true;
});

// SignalR
builder.Services.AddSignalR();

// Application Layer DI
builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

// Razor Pages
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/");
    options.Conventions.AllowAnonymousToPage("/Account/Login");
});

builder.Services.AddAntiforgery();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();

// SignalR Hub
app.MapHub<PTW.JSEA.Web.Hubs.PermitHub>("/hubs/permit");

app.MapRazorPages();

// Seed database
using (var scope = app.Services.CreateScope())
{
    await DatabaseSeeder.SeedAsync(scope.ServiceProvider);
}

app.Run();
```

## 2.5 Migration & Buat Database

```bash
cd C:\Projects\PTW-JSEA

# Tambah migration pertama
dotnet ef migrations add InitialCreate --project src/PTW.JSEA.Infrastructure --startup-project src/PTW.JSEA.Web

# Update database (buat database di SQL Server)
dotnet ef database update --project src/PTW.JSEA.Infrastructure --startup-project src/PTW.JSEA.Web
```

**Verifikasi:** Buka SSMS → database `PTW_JSEA_DB` sudah terbuat dengan semua tabel.

---

# TAHAP 3 — AUTHENTICATION & ROLE MANAGEMENT

## 3.1 ApplicationUser & ApplicationRole

### `src/PTW.JSEA.Domain/Entities/ApplicationUser.cs`
```csharp
using Microsoft.AspNetCore.Identity;

namespace PTW.JSEA.Domain.Entities;

public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; } = string.Empty;
    public string? Department { get; set; }
    public string? Position { get; set; }
    public string? EmployeeId { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation
    public ICollection<UserCertificate> Certificates { get; set; } = new List<UserCertificate>();
}
```

## 3.2 Seed Roles & Admin User

### `src/PTW.JSEA.Infrastructure/Data/DatabaseSeeder.cs`
```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.DependencyInjection;
using PTW.JSEA.Domain.Entities;

namespace PTW.JSEA.Infrastructure.Data;

public static class DatabaseSeeder
{
    public static readonly string[] Roles = new[]
    {
        "SystemAdmin",
        "SafetyManager",
        "HSEOfficer",
        "AreaOwner",
        "Approver",
        "Requester",
        "Auditor"
    };

    public static async Task SeedAsync(IServiceProvider serviceProvider)
    {
        var roleManager = serviceProvider.GetRequiredService<RoleManager<ApplicationRole>>();
        var userManager = serviceProvider.GetRequiredService<UserManager<ApplicationUser>>();

        // Seed roles
        foreach (var role in Roles)
        {
            if (!await roleManager.RoleExistsAsync(role))
                await roleManager.CreateAsync(new ApplicationRole { Name = role });
        }

        // Seed admin user
        var adminEmail = "admin@ptw-system.com";
        if (await userManager.FindByEmailAsync(adminEmail) == null)
        {
            var admin = new ApplicationUser
            {
                UserName = adminEmail,
                Email = adminEmail,
                FullName = "System Administrator",
                Department = "HSE",
                IsActive = true,
                EmailConfirmed = true
            };
            var result = await userManager.CreateAsync(admin, "Admin@123456");
            if (result.Succeeded)
                await userManager.AddToRoleAsync(admin, "SystemAdmin");
        }
    }
}
```

## 3.3 Buat Login Page

### `src/PTW.JSEA.Web/Pages/Account/Login.cshtml`
```html
@page
@model PTW.JSEA.Web.Pages.Account.LoginModel
@{
    Layout = "_AuthLayout";
    ViewData["Title"] = "Login";
}

<div class="min-h-screen bg-gradient-to-br from-blue-900 to-blue-700 flex items-center justify-center">
    <div class="bg-white rounded-2xl shadow-2xl p-8 w-full max-w-md">
        <div class="text-center mb-8">
            <div class="inline-flex items-center justify-center w-16 h-16 bg-blue-600 rounded-full mb-4">
                <svg class="w-8 h-8 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                          d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
                </svg>
            </div>
            <h1 class="text-2xl font-bold text-gray-900">PTW & JSEA System</h1>
            <p class="text-gray-500 text-sm mt-1">PT. Mattel Indonesia</p>
        </div>

        <form method="post">
            <div asp-validation-summary="ModelOnly" class="bg-red-50 text-red-600 p-3 rounded-lg mb-4 text-sm"></div>

            <div class="mb-4">
                <label asp-for="Input.Email" class="block text-sm font-medium text-gray-700 mb-1">Email</label>
                <input asp-for="Input.Email" type="email" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="email@mattel.com" />
                <span asp-validation-for="Input.Email" class="text-red-500 text-xs"></span>
            </div>

            <div class="mb-6">
                <label asp-for="Input.Password" class="block text-sm font-medium text-gray-700 mb-1">Password</label>
                <input asp-for="Input.Password" type="password" class="w-full border border-gray-300 rounded-lg px-3 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" />
                <span asp-validation-for="Input.Password" class="text-red-500 text-xs"></span>
            </div>

            <div class="flex items-center justify-between mb-6">
                <label class="flex items-center text-sm text-gray-600">
                    <input asp-for="Input.RememberMe" type="checkbox" class="mr-2" />
                    Remember me
                </label>
            </div>

            <button type="submit" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-semibold py-2.5 rounded-lg transition duration-200">
                Sign In
            </button>
        </form>
    </div>
</div>
```

---

# TAHAP 4 — TAILWIND CSS SETUP

## 4.1 Install TailwindCSS

```bash
cd src/PTW.JSEA.Web

# Init npm
npm init -y

# Install tailwind
npm install -D tailwindcss @tailwindcss/forms @tailwindcss/typography

# Init tailwind config
npx tailwindcss init
```

## 4.2 Update `tailwind.config.js`
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './Pages/**/*.cshtml',
    './Views/**/*.cshtml',
    './wwwroot/js/**/*.js'
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: '#eff6ff',
          500: '#3b82f6',
          600: '#2563eb',
          700: '#1d4ed8',
          900: '#1e3a8a',
        },
        danger: '#ef4444',
        warning: '#f59e0b',
        success: '#10b981',
      }
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
  ],
}
```

## 4.3 Buat `wwwroot/css/app.css`
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .btn-primary {
    @apply bg-blue-600 hover:bg-blue-700 text-white font-semibold py-2 px-4 rounded-lg transition duration-200 focus:outline-none focus:ring-2 focus:ring-blue-500;
  }
  .btn-danger {
    @apply bg-red-600 hover:bg-red-700 text-white font-semibold py-2 px-4 rounded-lg transition duration-200;
  }
  .btn-secondary {
    @apply bg-gray-100 hover:bg-gray-200 text-gray-700 font-semibold py-2 px-4 rounded-lg transition duration-200;
  }
  .card {
    @apply bg-white rounded-xl shadow-sm border border-gray-100 p-6;
  }
  .badge-active {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800;
  }
  .badge-pending {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-yellow-100 text-yellow-800;
  }
  .badge-rejected {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-red-100 text-red-800;
  }
  .badge-draft {
    @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-gray-100 text-gray-700;
  }
}
```

## 4.4 Update `package.json` Scripts
```json
{
  "scripts": {
    "build:css": "tailwindcss -i ./wwwroot/css/app.css -o ./wwwroot/css/output.css",
    "watch:css": "tailwindcss -i ./wwwroot/css/app.css -o ./wwwroot/css/output.css --watch"
  }
}
```

## 4.5 Build CSS
```bash
npm run build:css
```

## 4.6 Referensikan di `_Layout.cshtml`
```html
<link rel="stylesheet" href="~/css/output.css" asp-append-version="true" />
```

---

# TAHAP 5 — LAYOUT & NAVIGATION

## 5.1 Main Layout `_Layout.cshtml`

Buat layout utama dengan sidebar navigation:

```html
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] — PTW & JSEA System</title>
    <link rel="stylesheet" href="~/css/output.css" asp-append-version="true" />
</head>
<body class="bg-gray-50 font-sans">
    <div class="flex h-screen overflow-hidden">

        <!-- SIDEBAR -->
        <aside class="w-64 bg-blue-900 text-white flex flex-col flex-shrink-0">
            <div class="p-5 border-b border-blue-800">
                <h1 class="font-bold text-lg">PTW & JSEA</h1>
                <p class="text-blue-300 text-xs">PT. Mattel Indonesia</p>
            </div>

            <nav class="flex-1 overflow-y-auto p-4 space-y-1">
                <a asp-page="/Dashboard/Index" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    🏠 Dashboard
                </a>
                <div class="pt-3 pb-1">
                    <p class="text-blue-400 text-xs font-semibold uppercase tracking-wider px-3">JSEA</p>
                </div>
                <a asp-page="/JSEA/Index" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    📋 Daftar JSEA
                </a>
                <a asp-page="/JSEA/Create" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    ➕ Buat JSEA
                </a>
                <div class="pt-3 pb-1">
                    <p class="text-blue-400 text-xs font-semibold uppercase tracking-wider px-3">PERMIT</p>
                </div>
                <a asp-page="/Permits/Index" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    📝 Daftar Permit
                </a>
                <a asp-page="/Permits/Create" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    ➕ Buat Permit
                </a>
                <a asp-page="/Permits/MyApprovals" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    ✅ Approval Queue
                </a>
                <div class="pt-3 pb-1">
                    <p class="text-blue-400 text-xs font-semibold uppercase tracking-wider px-3">MONITORING</p>
                </div>
                <a asp-page="/Monitoring/Active" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    🟢 Active Work
                </a>
                <a asp-page="/Reports/Index" class="flex items-center gap-3 px-3 py-2 rounded-lg hover:bg-blue-800 text-sm transition">
                    📊 Reports
                </a>
            </nav>

            <!-- User info bottom -->
            <div class="p-4 border-t border-blue-800">
                <p class="text-sm font-medium">@User.Identity?.Name</p>
                <form asp-page="/Account/Logout" method="post">
                    <button type="submit" class="text-blue-300 text-xs hover:text-white transition mt-1">
                        Sign out
                    </button>
                </form>
            </div>
        </aside>

        <!-- MAIN CONTENT -->
        <main class="flex-1 flex flex-col overflow-hidden">
            <header class="bg-white border-b border-gray-200 px-6 py-4 flex items-center justify-between">
                <h2 class="text-lg font-semibold text-gray-800">@ViewData["Title"]</h2>
                <div id="notification-bell" class="relative">
                    <button class="relative p-2 text-gray-500 hover:text-blue-600">
                        🔔
                        <span id="notif-count" class="absolute top-0 right-0 w-4 h-4 bg-red-500 text-white text-xs rounded-full hidden flex items-center justify-center"></span>
                    </button>
                </div>
            </header>

            <div class="flex-1 overflow-y-auto p-6">
                @RenderBody()
            </div>
        </main>
    </div>

    <script src="~/js/signalr/dist/browser/signalr.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

---

# TAHAP 6 — REPOSITORY PATTERN & SERVICE LAYER

## 6.1 Generic Repository Interface

### `src/PTW.JSEA.Domain/Interfaces/Repositories/IGenericRepository.cs`
```csharp
using System.Linq.Expressions;

namespace PTW.JSEA.Domain.Interfaces.Repositories;

public interface IGenericRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate);
    Task<T> AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(T entity);
    Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null);
    IQueryable<T> Query();
}
```

## 6.2 Generic Repository Implementation

### `src/PTW.JSEA.Infrastructure/Repositories/GenericRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;
using System.Linq.Expressions;

namespace PTW.JSEA.Infrastructure.Repositories;

public class GenericRepository<T> : IGenericRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public GenericRepository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id) => await _dbSet.FindAsync(id);
    public async Task<IEnumerable<T>> GetAllAsync() => await _dbSet.ToListAsync();
    public async Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate)
        => await _dbSet.Where(predicate).ToListAsync();
    public async Task<T> AddAsync(T entity) { await _dbSet.AddAsync(entity); await _context.SaveChangesAsync(); return entity; }
    public async Task UpdateAsync(T entity) { _dbSet.Update(entity); await _context.SaveChangesAsync(); }
    public async Task DeleteAsync(T entity) { _dbSet.Remove(entity); await _context.SaveChangesAsync(); }
    public async Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null)
        => predicate == null ? await _dbSet.CountAsync() : await _dbSet.CountAsync(predicate);
    public IQueryable<T> Query() => _dbSet.AsQueryable();
}
```

## 6.3 Work Permit Service

### `src/PTW.JSEA.Application/Services/WorkPermitService.cs`
```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Application.Services;

public interface IWorkPermitService
{
    Task<WorkPermit> CreatePermitAsync(WorkPermit permit, string requesterId);
    Task<bool> SubmitPermitAsync(int permitId, string userId);
    Task<bool> ApprovePermitAsync(int permitId, string approverId, string comments);
    Task<bool> RejectPermitAsync(int permitId, string approverId, string reason);
    Task<bool> SuspendPermitAsync(int permitId, string userId, string reason);
    Task<bool> ResumePermitAsync(int permitId, string userId);
    Task<bool> ClosePermitAsync(int permitId, string userId);
    Task<string> GeneratePermitNumberAsync(WorkType workType);
    Task<bool> CheckApproverEligibilityAsync(string approverId, int permitId);
}

public class WorkPermitService : IWorkPermitService
{
    private readonly IWorkPermitRepository _permitRepo;
    private readonly IUserCertificateRepository _certRepo;
    private readonly IAuditService _auditService;
    private readonly INotificationService _notificationService;

    public WorkPermitService(
        IWorkPermitRepository permitRepo,
        IUserCertificateRepository certRepo,
        IAuditService auditService,
        INotificationService notificationService)
    {
        _permitRepo = permitRepo;
        _certRepo = certRepo;
        _auditService = auditService;
        _notificationService = notificationService;
    }

    public async Task<string> GeneratePermitNumberAsync(WorkType workType)
    {
        var prefix = workType switch
        {
            WorkType.HotWork => "HW",
            WorkType.ColdWork => "CW",
            WorkType.ElectricalIsolation => "EI",
            WorkType.ConfinedSpaceEntry => "CS",
            WorkType.Excavation => "EX",
            WorkType.Lifting => "LF",
            WorkType.WorkingAtHeight => "WH",
            _ => "WP"
        };
        var date = DateTime.Now.ToString("yyyyMMdd");
        var seq = await _permitRepo.GetTodaySequenceAsync(workType) + 1;
        return $"{prefix}-{date}-{seq:D3}";
    }

    public async Task<bool> CheckApproverEligibilityAsync(string approverId, int permitId)
    {
        var permit = await _permitRepo.GetByIdAsync(permitId);
        if (permit == null) return false;

        // Cek sertifikat approver untuk jenis pekerjaan ini
        var certs = await _certRepo.GetValidCertificatesAsync(approverId, permit.WorkType);
        return certs.Any();
    }

    public async Task<bool> ApprovePermitAsync(int permitId, string approverId, string comments)
    {
        // Pastikan approver eligible
        var eligible = await CheckApproverEligibilityAsync(approverId, permitId);
        if (!eligible) throw new InvalidOperationException("Approver tidak memiliki sertifikasi yang valid untuk jenis pekerjaan ini.");

        var permit = await _permitRepo.GetWithApprovalsAsync(permitId);
        if (permit == null) return false;

        // Dapatkan approval step berikutnya
        var nextApproval = permit.Approvals
            .Where(a => a.Status == ApprovalStatus.Pending)
            .OrderBy(a => a.Sequence)
            .FirstOrDefault();

        if (nextApproval == null) return false;

        nextApproval.Status = ApprovalStatus.Approved;
        nextApproval.ApproverId = approverId;
        nextApproval.ApprovedAt = DateTime.UtcNow;
        nextApproval.Comments = comments;

        // Cek apakah semua approval selesai
        var allApproved = permit.Approvals.All(a => a.Status == ApprovalStatus.Approved);
        if (allApproved)
            permit.Status = PermitStatus.Active;
        else
            permit.Status = PermitStatus.WaitingAreaApproval; // next step

        await _permitRepo.UpdateAsync(permit);
        await _auditService.LogAsync("WorkPermit", permitId, "Approved", approverId);
        await _notificationService.SendPermitApprovedAsync(permit);

        return true;
    }
    
    // Implementasi method lainnya...
    public Task<WorkPermit> CreatePermitAsync(WorkPermit permit, string requesterId) => throw new NotImplementedException();
    public Task<bool> SubmitPermitAsync(int permitId, string userId) => throw new NotImplementedException();
    public Task<bool> RejectPermitAsync(int permitId, string approverId, string reason) => throw new NotImplementedException();
    public Task<bool> SuspendPermitAsync(int permitId, string userId, string reason) => throw new NotImplementedException();
    public Task<bool> ResumePermitAsync(int permitId, string userId) => throw new NotImplementedException();
    public Task<bool> ClosePermitAsync(int permitId, string userId) => throw new NotImplementedException();
}
```

---

# TAHAP 7 — SIGNALR REAL-TIME SYSTEM

## 7.1 Buat PermitHub

### `src/PTW.JSEA.Web/Hubs/PermitHub.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

namespace PTW.JSEA.Web.Hubs;

[Authorize]
public class PermitHub : Hub
{
    public async Task JoinDashboardGroup()
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, "Dashboard");
    }

    public async Task LeaveDashboardGroup()
    {
        await Groups.RemoveFromGroupAsync(Context.ConnectionId, "Dashboard");
    }

    public async Task JoinPermitGroup(int permitId)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, $"Permit_{permitId}");
    }

    public override async Task OnConnectedAsync()
    {
        await base.OnConnectedAsync();
    }
}
```

## 7.2 SignalR Service untuk Push Update

### `src/PTW.JSEA.Application/Services/IDashboardNotifier.cs`
```csharp
namespace PTW.JSEA.Application.Services;

public interface IDashboardNotifier
{
    Task NotifyPermitStatusChanged(int permitId, string newStatus);
    Task NotifyDashboardCounterUpdated(int activePermits, int activeWorkers);
    Task NotifyEmergencyStop(int permitId, string location, string reason);
    Task NotifyAdherenceOverdue(int permitId, string permitNumber);
}
```

### Implementation dengan SignalR:
```csharp
using Microsoft.AspNetCore.SignalR;
using PTW.JSEA.Web.Hubs;

public class SignalRDashboardNotifier : IDashboardNotifier
{
    private readonly IHubContext<PermitHub> _hubContext;

    public SignalRDashboardNotifier(IHubContext<PermitHub> hubContext)
        => _hubContext = hubContext;

    public async Task NotifyPermitStatusChanged(int permitId, string newStatus)
    {
        await _hubContext.Clients.Group("Dashboard")
            .SendAsync("PermitStatusChanged", new { permitId, newStatus, timestamp = DateTime.UtcNow });
        await _hubContext.Clients.Group($"Permit_{permitId}")
            .SendAsync("StatusUpdated", newStatus);
    }

    public async Task NotifyEmergencyStop(int permitId, string location, string reason)
    {
        await _hubContext.Clients.All
            .SendAsync("EmergencyStop", new { permitId, location, reason, timestamp = DateTime.UtcNow });
    }

    public async Task NotifyDashboardCounterUpdated(int activePermits, int activeWorkers)
    {
        await _hubContext.Clients.Group("Dashboard")
            .SendAsync("CounterUpdated", new { activePermits, activeWorkers });
    }

    public async Task NotifyAdherenceOverdue(int permitId, string permitNumber)
    {
        await _hubContext.Clients.Group("Dashboard")
            .SendAsync("AdherenceOverdue", new { permitId, permitNumber });
    }
}
```

## 7.3 Client-Side SignalR JavaScript

### `src/PTW.JSEA.Web/wwwroot/js/site.js`
```javascript
// SignalR connection
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/permit")
    .withAutomaticReconnect()
    .build();

connection.on("PermitStatusChanged", (data) => {
    console.log("Permit status changed:", data);
    // Update UI tanpa refresh
    const badge = document.getElementById(`permit-status-${data.permitId}`);
    if (badge) badge.textContent = data.newStatus;

    showToast(`Permit #${data.permitId} status: ${data.newStatus}`, 'info');
});

connection.on("EmergencyStop", (data) => {
    // Tampilkan modal emergency
    showEmergencyAlert(data.location, data.reason);
});

connection.on("CounterUpdated", (data) => {
    const activeEl = document.getElementById("active-permits-count");
    const workersEl = document.getElementById("active-workers-count");
    if (activeEl) activeEl.textContent = data.activePermits;
    if (workersEl) workersEl.textContent = data.activeWorkers;
});

connection.on("AdherenceOverdue", (data) => {
    showToast(`⚠️ Permit ${data.permitNumber} adherence overdue!`, 'warning');
});

// Start connection
connection.start()
    .then(() => connection.invoke("JoinDashboardGroup"))
    .catch(err => console.error("SignalR error:", err));

// Toast utility
function showToast(message, type = 'info') {
    const colors = {
        info: 'bg-blue-500',
        warning: 'bg-yellow-500',
        error: 'bg-red-500',
        success: 'bg-green-500'
    };
    const toast = document.createElement('div');
    toast.className = `fixed bottom-4 right-4 ${colors[type]} text-white px-4 py-3 rounded-lg shadow-lg z-50 transition-all`;
    toast.textContent = message;
    document.body.appendChild(toast);
    setTimeout(() => toast.remove(), 4000);
}
```

---

# TAHAP 8 — DASHBOARD & CORE PAGES

## 8.1 Dashboard Page Model

### `src/PTW.JSEA.Web/Pages/Dashboard/Index.cshtml.cs`
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc.RazorPages;
using PTW.JSEA.Application.Services;

namespace PTW.JSEA.Web.Pages.Dashboard;

[Authorize]
public class IndexModel : PageModel
{
    private readonly IDashboardService _dashboardService;

    public IndexModel(IDashboardService dashboardService)
        => _dashboardService = dashboardService;

    public int ActivePermitCount { get; set; }
    public int HighRiskPermitCount { get; set; }
    public int ActiveWorkerCount { get; set; }
    public int OverdueAdherenceCount { get; set; }
    public int PendingApprovalCount { get; set; }
    public List<ActivePermitDto> ActivePermits { get; set; } = new();
    public List<RecentActivityDto> RecentActivities { get; set; } = new();

    public async Task OnGetAsync()
    {
        var stats = await _dashboardService.GetDashboardStatsAsync(User);
        ActivePermitCount = stats.ActivePermitCount;
        HighRiskPermitCount = stats.HighRiskPermitCount;
        ActiveWorkerCount = stats.ActiveWorkerCount;
        OverdueAdherenceCount = stats.OverdueAdherenceCount;
        PendingApprovalCount = stats.PendingApprovalCount;
        ActivePermits = stats.ActivePermits;
        RecentActivities = stats.RecentActivities;
    }
}
```

## 8.2 Dashboard HTML

### `src/PTW.JSEA.Web/Pages/Dashboard/Index.cshtml`
```html
@page
@model PTW.JSEA.Web.Pages.Dashboard.IndexModel
@{
    ViewData["Title"] = "Dashboard";
}

<!-- KPI Cards -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-5 gap-4 mb-6">
    <div class="card">
        <p class="text-sm text-gray-500">Active Permits</p>
        <p id="active-permits-count" class="text-3xl font-bold text-blue-600">@Model.ActivePermitCount</p>
    </div>
    <div class="card">
        <p class="text-sm text-gray-500">High Risk Work</p>
        <p class="text-3xl font-bold text-red-600">@Model.HighRiskPermitCount</p>
    </div>
    <div class="card">
        <p class="text-sm text-gray-500">Active Workers</p>
        <p id="active-workers-count" class="text-3xl font-bold text-green-600">@Model.ActiveWorkerCount</p>
    </div>
    <div class="card">
        <p class="text-sm text-gray-500">Overdue Adherence</p>
        <p class="text-3xl font-bold text-orange-600">@Model.OverdueAdherenceCount</p>
    </div>
    <div class="card">
        <p class="text-sm text-gray-500">Pending Approval</p>
        <p class="text-3xl font-bold text-yellow-600">@Model.PendingApprovalCount</p>
    </div>
</div>

<!-- Active Permits Table -->
<div class="card mb-6">
    <div class="flex items-center justify-between mb-4">
        <h3 class="text-lg font-semibold text-gray-800">Active Permits</h3>
        <a asp-page="/Permits/Index" class="text-blue-600 hover:underline text-sm">Lihat semua →</a>
    </div>
    <div class="overflow-x-auto">
        <table class="w-full text-sm">
            <thead>
                <tr class="border-b border-gray-200 text-gray-500 text-left">
                    <th class="pb-3 font-medium">Permit No.</th>
                    <th class="pb-3 font-medium">Jenis Pekerjaan</th>
                    <th class="pb-3 font-medium">Lokasi</th>
                    <th class="pb-3 font-medium">Pekerja</th>
                    <th class="pb-3 font-medium">Berlaku Hingga</th>
                    <th class="pb-3 font-medium">Status</th>
                    <th class="pb-3 font-medium">Aksi</th>
                </tr>
            </thead>
            <tbody class="divide-y divide-gray-100">
                @foreach (var permit in Model.ActivePermits)
                {
                    <tr class="hover:bg-gray-50">
                        <td class="py-3 font-mono font-medium text-blue-600">@permit.PermitNumber</td>
                        <td class="py-3">@permit.WorkType</td>
                        <td class="py-3">@permit.Location</td>
                        <td class="py-3">@permit.WorkerCount orang</td>
                        <td class="py-3">
                            <span class="@(permit.IsExpiringSoon ? "text-red-600 font-semibold" : "text-gray-700")">
                                @permit.EndTime.ToString("dd MMM HH:mm")
                            </span>
                        </td>
                        <td class="py-3">
                            <span id="permit-status-@permit.Id" class="badge-active">Active</span>
                        </td>
                        <td class="py-3">
                            <a asp-page="/Permits/Detail" asp-route-id="@permit.Id" class="text-blue-600 hover:underline">Detail</a>
                        </td>
                    </tr>
                }
                @if (!Model.ActivePermits.Any())
                {
                    <tr><td colspan="7" class="py-8 text-center text-gray-400">Tidak ada permit aktif saat ini</td></tr>
                }
            </tbody>
        </table>
    </div>
</div>

<!-- Recent Activity -->
<div class="card">
    <h3 class="text-lg font-semibold text-gray-800 mb-4">Aktivitas Terbaru</h3>
    <div class="space-y-3">
        @foreach (var activity in Model.RecentActivities)
        {
            <div class="flex items-start gap-3 text-sm">
                <span class="text-gray-400 text-xs w-32 flex-shrink-0 mt-0.5">@activity.Timestamp.ToString("dd MMM HH:mm")</span>
                <span class="text-gray-700">@activity.Description</span>
            </div>
        }
    </div>
</div>
```

---

# TAHAP 9 — BACKGROUND SERVICES

## 9.1 Permit Auto-Expire Service

### `src/PTW.JSEA.Infrastructure/BackgroundServices/PermitExpiryService.cs`
```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using PTW.JSEA.Application.Services;

namespace PTW.JSEA.Infrastructure.BackgroundServices;

public class PermitExpiryService : BackgroundService
{
    private readonly IServiceProvider _services;
    private readonly ILogger<PermitExpiryService> _logger;

    public PermitExpiryService(IServiceProvider services, ILogger<PermitExpiryService> logger)
    {
        _services = services;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("PermitExpiryService running at: {time}", DateTimeOffset.Now);
            try
            {
                using var scope = _services.CreateScope();
                var permitService = scope.ServiceProvider.GetRequiredService<IWorkPermitService>();
                var notifier = scope.ServiceProvider.GetRequiredService<IDashboardNotifier>();

                // Auto-expire permits yang sudah melewati EndTime
                // Implementasi auto-expire logic di sini

                _logger.LogInformation("Permit expiry check completed.");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in PermitExpiryService");
            }

            // Jalankan setiap 5 menit
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

## 9.2 Adherence Reminder Service

```csharp
public class AdherenceReminderService : BackgroundService
{
    // Cek setiap 30 menit apakah ada permit aktif yang adherence-nya sudah 4 jam belum dicek
    // Jika ada: kirim notifikasi ke HSE Officer & Observer

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var scope = _services.CreateScope();
                var adherenceRepo = scope.ServiceProvider.GetRequiredService<IAdherenceRepository>();
                var notifier = scope.ServiceProvider.GetRequiredService<IDashboardNotifier>();

                var overduePermits = await adherenceRepo.GetOverdueAdherencePermitsAsync(4);
                foreach (var permit in overduePermits)
                {
                    await notifier.NotifyAdherenceOverdue(permit.Id, permit.PermitNumber);
                    // Kirim email/WhatsApp ke PIC
                }
            }
            catch (Exception ex) { /* log */ }

            await Task.Delay(TimeSpan.FromMinutes(30), stoppingToken);
        }
    }
}
```

## 9.3 Register Background Services di Program.cs

```csharp
builder.Services.AddHostedService<PermitExpiryService>();
builder.Services.AddHostedService<AdherenceReminderService>();
builder.Services.AddHostedService<CertificateExpiryReminderService>();
```

---

# TAHAP 10 — AUDIT TRAIL SYSTEM

## 10.1 Audit Service

### `src/PTW.JSEA.Application/Services/AuditService.cs`
```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Interfaces.Repositories;

namespace PTW.JSEA.Application.Services;

public interface IAuditService
{
    Task LogAsync(string module, int entityId, string action, string userId,
        string? oldValue = null, string? newValue = null);
    Task<IEnumerable<AuditLog>> GetLogsAsync(string module, int entityId);
}

public class AuditService : IAuditService
{
    private readonly IGenericRepository<AuditLog> _auditRepo;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public AuditService(IGenericRepository<AuditLog> auditRepo, IHttpContextAccessor httpContextAccessor)
    {
        _auditRepo = auditRepo;
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task LogAsync(string module, int entityId, string action, string userId,
        string? oldValue = null, string? newValue = null)
    {
        var context = _httpContextAccessor.HttpContext;
        var log = new AuditLog
        {
            Module = module,
            EntityId = entityId,
            Action = action,
            ActionBy = userId,
            ActionDate = DateTime.UtcNow,
            OldValue = oldValue,
            NewValue = newValue,
            IPAddress = context?.Connection.RemoteIpAddress?.ToString() ?? "Unknown",
            Device = context?.Request.Headers["User-Agent"].ToString() ?? "Unknown"
        };
        await _auditRepo.AddAsync(log);
    }

    public async Task<IEnumerable<AuditLog>> GetLogsAsync(string module, int entityId)
        => await _auditRepo.FindAsync(x => x.Module == module && x.EntityId == entityId);
}
```

---

# TAHAP 11 — NOTIFICATION ENGINE

## 11.1 Notification Service Interface

```csharp
namespace PTW.JSEA.Application.Services;

public interface INotificationService
{
    Task SendInAppAsync(string userId, string title, string message);
    Task SendEmailAsync(string toEmail, string subject, string body);
    Task SendWhatsAppAsync(string phone, string message);
    Task SendPermitApprovedAsync(WorkPermit permit);
    Task SendPermitRejectedAsync(WorkPermit permit, string reason);
    Task SendPermitExpiringAsync(WorkPermit permit);
    Task SendCertificateExpiringAsync(UserCertificate certificate);
}
```

## 11.2 Email Notification (MailKit)

```csharp
using MailKit.Net.Smtp;
using MimeKit;

public class EmailNotificationService
{
    public async Task SendEmailAsync(string toEmail, string subject, string body)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress("PTW System", "noreply@ptw-system.com"));
        message.To.Add(MailboxAddress.Parse(toEmail));
        message.Subject = subject;
        message.Body = new TextPart("html") { Text = body };

        using var client = new SmtpClient();
        await client.ConnectAsync("smtp.gmail.com", 587, false);
        await client.AuthenticateAsync("your-email@gmail.com", "your-app-password");
        await client.SendAsync(message);
        await client.DisconnectAsync(true);
    }
}
```

---

# TAHAP 12 — DEPLOYMENT KE IIS

## 12.1 Persiapan Build Production

### Update `appsettings.Production.json`
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=PRODUCTION-SERVER\\SQLEXPRESS;Database=PTW_JSEA_PROD;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
```

### Build untuk publish:
```bash
cd C:\Projects\PTW-JSEA
dotnet publish src/PTW.JSEA.Web/PTW.JSEA.Web.csproj -c Release -o ./publish
```

## 12.2 Setup IIS

### 1. Install .NET Hosting Bundle di server:
```
https://dotnet.microsoft.com/en-us/download/dotnet/8.0
→ Download: ASP.NET Core Runtime 8.x — Windows Hosting Bundle
```

### 2. Buat Application Pool di IIS:
- Application Pool Name: `PTW-JSEA-Pool`
- .NET CLR Version: **No Managed Code**
- Managed Pipeline Mode: Integrated

### 3. Buat Website di IIS:
- Site Name: `PTW-JSEA`
- Physical Path: path ke folder publish
- Port: 80 (atau port yang diinginkan)
- Application Pool: `PTW-JSEA-Pool`

### 4. Set permission folder:
```
Klik kanan folder publish → Properties → Security
Tambahkan user: IIS AppPool\PTW-JSEA-Pool
Permission: Read & Execute, List, Read
Untuk folder logs dan uploads: tambahkan Write
```

### 5. Buat `web.config` (auto-generated saat publish, pastikan ada):
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
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

## 12.3 SSL Certificate

Untuk production, install SSL:
- **Option A:** Let's Encrypt via win-acme: `https://www.win-acme.com`
- **Option B:** Corporate certificate dari IT Mattel

```bash
# win-acme (jika ada domain)
wacs.exe --source manual --host ptw.mattel.co.id --installation iis
```

## 12.4 Firewall & Network

```bash
# Buka port 80 dan 443 di Windows Firewall
netsh advfirewall firewall add rule name="HTTP" dir=in action=allow protocol=TCP localport=80
netsh advfirewall firewall add rule name="HTTPS" dir=in action=allow protocol=TCP localport=443
```

---

# TAHAP 13 — VERIFIKASI AKHIR & CHECKLIST

## ✅ Checklist Sebelum Go-Live

### Database
- [ ] Semua tabel terbuat dengan benar
- [ ] Seed data roles sudah ada
- [ ] Admin user default sudah ada
- [ ] Index database sudah dioptimalkan

### Authentication
- [ ] Login berhasil
- [ ] Role-based access berfungsi
- [ ] Session timeout aktif
- [ ] Password policy diterapkan

### Core Features
- [ ] JSEA bisa dibuat dan disubmit
- [ ] Permit bisa dibuat dengan permit number otomatis
- [ ] Approval workflow berjalan sesuai urutan
- [ ] Authorization check sertifikat berfungsi
- [ ] Suspend/resume permit berfungsi
- [ ] PAI checklist bisa diisi dengan foto

### Real-Time
- [ ] SignalR terhubung
- [ ] Status update realtime berfungsi
- [ ] Emergency stop notification berfungsi

### Background Services
- [ ] Permit auto-expire berjalan
- [ ] Adherence reminder dikirim setiap 4 jam
- [ ] Certificate expiry reminder berjalan

### Notification
- [ ] In-app notification muncul
- [ ] Email notification terkirim

### Deployment
- [ ] IIS berjalan tanpa error
- [ ] HTTPS aktif
- [ ] Log file terbuat di folder logs
- [ ] File upload berhasil ke folder wwwroot/uploads

---

# URUTAN PENGERJAAN YANG DISARANKAN

```
Minggu 1-2  : Tahap 0, 1, 2, 3 (Setup + DB + Auth)
Minggu 3-4  : Tahap 4, 5, 6 (UI + Repository + Service)
Minggu 5-6  : JSEA Module lengkap
Minggu 7-8  : Permit Module + Approval Workflow
Minggu 9    : Tahap 7 (SignalR) + Dashboard
Minggu 10   : Tahap 9, 10, 11 (Background + Audit + Notif)
Minggu 11   : Testing, bug fix, polish UI
Minggu 12   : Tahap 12 (Deployment ke IIS)
```

---

> **Catatan penting:** Setiap kali akan mengerjakan satu modul baru, tanyakan kepada saya dan saya akan bantu buatkan kode lengkap untuk modul tersebut. Panduan ini adalah kerangka besar — setiap bagian bisa kita kerjakan detail bersama-sama.
