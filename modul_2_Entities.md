# MODUL 2 — DATABASE, ENTITIES & EF CORE

> Buat semua file berikut sesuai path yang tertera.
> Semua path relatif dari `C:\Projects\PTW-JSEA\`

---

## BAGIAN A — DOMAIN LAYER

### FILE: `src/PTW.JSEA.Domain/Common/BaseEntity.cs`
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

---

### FILE: `src/PTW.JSEA.Domain/Enums/PermitStatus.cs`
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

---

### FILE: `src/PTW.JSEA.Domain/Enums/JSEAStatus.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum JSEAStatus
{
    Draft = 0,
    Submitted = 1,
    UnderReview = 2,
    RevisionRequired = 3,
    Approved = 4,
    Archived = 5,
    Rejected = 6
}
```

---

### FILE: `src/PTW.JSEA.Domain/Enums/WorkType.cs`
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

---

### FILE: `src/PTW.JSEA.Domain/Enums/RiskLevel.cs`
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

---

### FILE: `src/PTW.JSEA.Domain/Enums/ApprovalStatus.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum ApprovalStatus
{
    Pending = 0,
    Approved = 1,
    Rejected = 2,
    Skipped = 3
}
```

---

### FILE: `src/PTW.JSEA.Domain/Enums/CertificateStatus.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum CertificateStatus
{
    Active = 0,
    Expired = 1,
    Suspended = 2,
    Revoked = 3
}
```

---

### FILE: `src/PTW.JSEA.Domain/Enums/NotificationType.cs`
```csharp
namespace PTW.JSEA.Domain.Enums;

public enum NotificationType
{
    PermitApproved = 1,
    PermitRejected = 2,
    PermitExpiring = 3,
    PermitExpired = 4,
    AdherenceOverdue = 5,
    EmergencyStop = 6,
    CertificateExpiring = 7,
    CertificateExpired = 8,
    NewPermitForApproval = 9,
    JSEAApproved = 10,
    JSEARevisionRequired = 11
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/ApplicationUser.cs`
```csharp
using Microsoft.AspNetCore.Identity;

namespace PTW.JSEA.Domain.Entities;

public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; } = string.Empty;
    public string? Department { get; set; }
    public string? Position { get; set; }
    public string? EmployeeId { get; set; }
    public string? PhoneWhatsApp { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }

    // Navigation
    public ICollection<UserCertificate> Certificates { get; set; } = new List<UserCertificate>();
    public ICollection<Notification> Notifications { get; set; } = new List<Notification>();
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/ApplicationRole.cs`
```csharp
using Microsoft.AspNetCore.Identity;

namespace PTW.JSEA.Domain.Entities;

public class ApplicationRole : IdentityRole
{
    public string? Description { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/UserCertificate.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class UserCertificate : BaseEntity
{
    public string UserId { get; set; } = string.Empty;
    public WorkType WorkType { get; set; }
    public string CertificateNumber { get; set; } = string.Empty;
    public string CertificateName { get; set; } = string.Empty;
    public string? IssuingBody { get; set; }
    public DateTime IssuedDate { get; set; }
    public DateTime ExpiryDate { get; set; }
    public CertificateStatus Status { get; set; } = CertificateStatus.Active;
    public string? FilePath { get; set; }
    public string? Notes { get; set; }

    // Navigation
    public ApplicationUser User { get; set; } = null!;

    // Computed
    public bool IsValid => Status == CertificateStatus.Active && ExpiryDate > DateTime.UtcNow;
    public int DaysUntilExpiry => (ExpiryDate - DateTime.UtcNow).Days;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/JSEA.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class JSEA : BaseEntity
{
    public string WorkTitle { get; set; } = string.Empty;
    public string Location { get; set; } = string.Empty;
    public string? WorkDescription { get; set; }
    public WorkType WorkType { get; set; }
    public RiskLevel RiskLevel { get; set; }
    public JSEAStatus Status { get; set; } = JSEAStatus.Draft;
    public string RequesterId { get; set; } = string.Empty;
    public string? ReviewedById { get; set; }
    public string? ApprovedById { get; set; }
    public DateTime? ReviewedAt { get; set; }
    public DateTime? ApprovedAt { get; set; }
    public string? ReviewComments { get; set; }
    public string? RejectionReason { get; set; }
    public int Version { get; set; } = 1;
    public int? ParentJSEAId { get; set; }

    // Navigation
    public ApplicationUser Requester { get; set; } = null!;
    public ApplicationUser? ReviewedBy { get; set; }
    public ApplicationUser? ApprovedBy { get; set; }
    public JSEA? ParentJSEA { get; set; }
    public ICollection<JSEADetail> Details { get; set; } = new List<JSEADetail>();
    public ICollection<JSEAAttachment> Attachments { get; set; } = new List<JSEAAttachment>();
    public ICollection<WorkPermit> WorkPermits { get; set; } = new List<WorkPermit>();
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/JSEADetail.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class JSEADetail : BaseEntity
{
    public int JSEAId { get; set; }
    public int StepNumber { get; set; }
    public string StepDescription { get; set; } = string.Empty;
    public string Hazard { get; set; } = string.Empty;
    public string HazardCategory { get; set; } = string.Empty;
    public string ControlMeasure { get; set; } = string.Empty;
    public string PPERequired { get; set; } = string.Empty;
    public int Severity { get; set; }       // 1-5
    public int Likelihood { get; set; }     // 1-5
    public int RiskScore => Severity * Likelihood;
    public int ResidualRisk { get; set; }
    public string? ResponsiblePerson { get; set; }

    // Navigation
    public JSEA JSEA { get; set; } = null!;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/JSEAAttachment.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class JSEAAttachment : BaseEntity
{
    public int JSEAId { get; set; }
    public string FileName { get; set; } = string.Empty;
    public string OriginalFileName { get; set; } = string.Empty;
    public string FilePath { get; set; } = string.Empty;
    public string FileType { get; set; } = string.Empty;  // SOP, Certificate, Drawing, etc
    public long FileSizeBytes { get; set; }
    public string UploadedBy { get; set; } = string.Empty;

    // Navigation
    public JSEA JSEA { get; set; } = null!;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/WorkPermit.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class WorkPermit : BaseEntity
{
    public string PermitNumber { get; set; } = string.Empty;
    public int JSEAId { get; set; }
    public string Location { get; set; } = string.Empty;
    public string? SpecificArea { get; set; }
    public WorkType WorkType { get; set; }
    public RiskLevel RiskLevel { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime { get; set; }
    public PermitStatus Status { get; set; } = PermitStatus.Draft;
    public string? Description { get; set; }
    public string RequesterId { get; set; } = string.Empty;

    // Suspend info
    public string? SuspendReason { get; set; }
    public DateTime? SuspendedAt { get; set; }
    public string? SuspendedById { get; set; }
    public DateTime? ResumedAt { get; set; }

    // Close info
    public string? CloseReason { get; set; }
    public DateTime? ClosedAt { get; set; }
    public string? ClosedById { get; set; }

    // PAI
    public bool PAICompleted { get; set; } = false;
    public DateTime? PAICompletedAt { get; set; }

    // Navigation
    public JSEA JSEA { get; set; } = null!;
    public ApplicationUser Requester { get; set; } = null!;
    public ICollection<PermitApproval> Approvals { get; set; } = new List<PermitApproval>();
    public ICollection<PermitWorker> Workers { get; set; } = new List<PermitWorker>();
    public ICollection<AdherenceLog> AdherenceLogs { get; set; } = new List<AdherenceLog>();
    public ICollection<PAIChecklist> PAIChecklists { get; set; } = new List<PAIChecklist>();
    public ICollection<PermitAttachment> Attachments { get; set; } = new List<PermitAttachment>();

    // Computed
    public bool IsExpired => EndTime < DateTime.UtcNow && Status == PermitStatus.Active;
    public bool IsExpiringSoon => EndTime < DateTime.UtcNow.AddHours(2) && Status == PermitStatus.Active;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/PermitApproval.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class PermitApproval : BaseEntity
{
    public int PermitId { get; set; }
    public string? ApproverId { get; set; }
    public string RequiredRole { get; set; } = string.Empty;
    public ApprovalStatus Status { get; set; } = ApprovalStatus.Pending;
    public int Sequence { get; set; }
    public DateTime? ApprovedAt { get; set; }
    public string? Comments { get; set; }
    public string? RejectionReason { get; set; }
    public string? DigitalSignatureHash { get; set; }

    // Navigation
    public WorkPermit Permit { get; set; } = null!;
    public ApplicationUser? Approver { get; set; }
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/PermitWorker.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class PermitWorker : BaseEntity
{
    public int PermitId { get; set; }
    public string WorkerName { get; set; } = string.Empty;
    public string? WorkerId { get; set; }
    public string? Company { get; set; }
    public string? Position { get; set; }
    public bool FitnessStatus { get; set; } = true;
    public string? FitnessNotes { get; set; }
    public DateTime? CheckInTime { get; set; }
    public DateTime? CheckOutTime { get; set; }

    // Navigation
    public WorkPermit Permit { get; set; } = null!;
    public ApplicationUser? Worker { get; set; }
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/PAIChecklist.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class PAIChecklist : BaseEntity
{
    public int PermitId { get; set; }
    public bool AreaConditionSafe { get; set; }
    public bool FireExtinguisherAvailable { get; set; }
    public bool GasTestValid { get; set; }
    public bool PPEComplete { get; set; }
    public bool IsolationVerified { get; set; }
    public bool BarricadeInstalled { get; set; }
    public bool EmergencyExitClear { get; set; }
    public bool CommunicationReady { get; set; }
    public string? InspectorId { get; set; }
    public string? InspectorName { get; set; }
    public DateTime InspectionDate { get; set; } = DateTime.UtcNow;
    public string? Notes { get; set; }
    public string? PhotoPath1 { get; set; }
    public string? PhotoPath2 { get; set; }
    public string? PhotoPath3 { get; set; }

    // Navigation
    public WorkPermit Permit { get; set; } = null!;

    // Computed
    public bool IsAllPassed =>
        AreaConditionSafe && FireExtinguisherAvailable && GasTestValid &&
        PPEComplete && IsolationVerified && EmergencyExitClear && CommunicationReady;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/PermitAttachment.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class PermitAttachment : BaseEntity
{
    public int PermitId { get; set; }
    public string FileName { get; set; } = string.Empty;
    public string OriginalFileName { get; set; } = string.Empty;
    public string FilePath { get; set; } = string.Empty;
    public string FileType { get; set; } = string.Empty;
    public long FileSizeBytes { get; set; }
    public string UploadedBy { get; set; } = string.Empty;

    // Navigation
    public WorkPermit Permit { get; set; } = null!;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/AdherenceLog.cs`
```csharp
using PTW.JSEA.Domain.Common;

namespace PTW.JSEA.Domain.Entities;

public class AdherenceLog : BaseEntity
{
    public int PermitId { get; set; }
    public DateTime CheckTime { get; set; } = DateTime.UtcNow;
    public string ObserverName { get; set; } = string.Empty;
    public string? ObserverId { get; set; }
    public bool IsCompliant { get; set; }
    public string? UnsafeObservation { get; set; }
    public string? CorrectiveAction { get; set; }
    public string? Notes { get; set; }

    // Navigation
    public WorkPermit Permit { get; set; } = null!;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/ApprovalMatrix.cs`
```csharp
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class ApprovalMatrix : BaseEntity
{
    public WorkType WorkType { get; set; }
    public RiskLevel RiskLevel { get; set; }
    public string RequiredRole { get; set; } = string.Empty;
    public int Sequence { get; set; }
    public string? Description { get; set; }
    public bool IsActive { get; set; } = true;
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/AuditLog.cs`
```csharp
namespace PTW.JSEA.Domain.Entities;

public class AuditLog
{
    public int Id { get; set; }
    public string Module { get; set; } = string.Empty;
    public int EntityId { get; set; }
    public string Action { get; set; } = string.Empty;
    public string? OldValue { get; set; }
    public string? NewValue { get; set; }
    public string ActionBy { get; set; } = string.Empty;
    public DateTime ActionDate { get; set; } = DateTime.UtcNow;
    public string? IPAddress { get; set; }
    public string? Device { get; set; }
    public string? Notes { get; set; }
}
```

---

### FILE: `src/PTW.JSEA.Domain/Entities/Notification.cs`
```csharp
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Entities;

public class Notification
{
    public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public string Title { get; set; } = string.Empty;
    public string Message { get; set; } = string.Empty;
    public NotificationType Type { get; set; }
    public bool IsRead { get; set; } = false;
    public string? Link { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? ReadAt { get; set; }

    // Navigation
    public ApplicationUser User { get; set; } = null!;
}
```

---

## BAGIAN B — REPOSITORY INTERFACES

### FILE: `src/PTW.JSEA.Domain/Interfaces/Repositories/IGenericRepository.cs`
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
    Task SoftDeleteAsync(int id);
    Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null);
    Task<bool> ExistsAsync(Expression<Func<T, bool>> predicate);
    IQueryable<T> Query();
}
```

---

### FILE: `src/PTW.JSEA.Domain/Interfaces/Repositories/IWorkPermitRepository.cs`
```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Interfaces.Repositories;

public interface IWorkPermitRepository : IGenericRepository<WorkPermit>
{
    Task<WorkPermit?> GetWithDetailsAsync(int id);
    Task<IEnumerable<WorkPermit>> GetActivePermitsAsync();
    Task<IEnumerable<WorkPermit>> GetPermitsByUserAsync(string userId);
    Task<IEnumerable<WorkPermit>> GetPermitsByStatusAsync(PermitStatus status);
    Task<IEnumerable<WorkPermit>> GetExpiringPermitsAsync(int withinHours);
    Task<int> GetTodaySequenceAsync(WorkType workType);
    Task<int> GetActiveCountAsync();
    Task<int> GetHighRiskCountAsync();
}
```

---

### FILE: `src/PTW.JSEA.Domain/Interfaces/Repositories/IJSEARepository.cs`
```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Interfaces.Repositories;

public interface IJSEARepository : IGenericRepository<JSEA>
{
    Task<JSEA?> GetWithDetailsAsync(int id);
    Task<IEnumerable<JSEA>> GetByUserAsync(string userId);
    Task<IEnumerable<JSEA>> GetByStatusAsync(JSEAStatus status);
    Task<IEnumerable<JSEA>> GetApprovedTemplatesAsync();
}
```

---

### FILE: `src/PTW.JSEA.Domain/Interfaces/Repositories/IUserRepository.cs`
```csharp
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;

namespace PTW.JSEA.Domain.Interfaces.Repositories;

public interface IUserRepository
{
    Task<ApplicationUser?> GetByIdAsync(string id);
    Task<IEnumerable<ApplicationUser>> GetByRoleAsync(string roleName);
    Task<IEnumerable<UserCertificate>> GetValidCertificatesAsync(string userId, WorkType workType);
    Task<bool> HasValidCertificateAsync(string userId, WorkType workType);
    Task<IEnumerable<UserCertificate>> GetExpiringCertificatesAsync(int withinDays);
}
```

---

## BAGIAN C — INFRASTRUCTURE: DbContext

### FILE: `src/PTW.JSEA.Infrastructure/Data/ApplicationDbContext.cs`
```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Common;
using PTW.JSEA.Domain.Entities;

namespace PTW.JSEA.Infrastructure.Data;

public class ApplicationDbContext : IdentityDbContext<ApplicationUser, ApplicationRole, string>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options) { }

    public DbSet<JSEA> JSEAs => Set<JSEA>();
    public DbSet<JSEADetail> JSEADetails => Set<JSEADetail>();
    public DbSet<JSEAAttachment> JSEAAttachments => Set<JSEAAttachment>();
    public DbSet<WorkPermit> WorkPermits => Set<WorkPermit>();
    public DbSet<PermitApproval> PermitApprovals => Set<PermitApproval>();
    public DbSet<PermitWorker> PermitWorkers => Set<PermitWorker>();
    public DbSet<PermitAttachment> PermitAttachments => Set<PermitAttachment>();
    public DbSet<PAIChecklist> PAIChecklists => Set<PAIChecklist>();
    public DbSet<AdherenceLog> AdherenceLogs => Set<AdherenceLog>();
    public DbSet<AuditLog> AuditLogs => Set<AuditLog>();
    public DbSet<ApprovalMatrix> ApprovalMatrices => Set<ApprovalMatrix>();
    public DbSet<Notification> Notifications => Set<Notification>();
    public DbSet<UserCertificate> UserCertificates => Set<UserCertificate>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Rename ASP.NET Identity tables
        builder.Entity<ApplicationUser>().ToTable("Users");
        builder.Entity<ApplicationRole>().ToTable("Roles");
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityUserRole<string>>().ToTable("UserRoles");
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityUserClaim<string>>().ToTable("UserClaims");
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityUserLogin<string>>().ToTable("UserLogins");
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityRoleClaim<string>>().ToTable("RoleClaims");
        builder.Entity<Microsoft.AspNetCore.Identity.IdentityUserToken<string>>().ToTable("UserTokens");

        // Soft delete global filters
        builder.Entity<WorkPermit>().HasQueryFilter(x => !x.IsDeleted);
        builder.Entity<JSEA>().HasQueryFilter(x => !x.IsDeleted);
        builder.Entity<JSEADetail>().HasQueryFilter(x => !x.IsDeleted);
        builder.Entity<UserCertificate>().HasQueryFilter(x => !x.IsDeleted);

        // WorkPermit indexes
        builder.Entity<WorkPermit>()
            .HasIndex(x => x.PermitNumber).IsUnique();
        builder.Entity<WorkPermit>()
            .HasIndex(x => x.Status);
        builder.Entity<WorkPermit>()
            .HasIndex(x => x.RequesterId);

        // JSEA indexes
        builder.Entity<JSEA>()
            .HasIndex(x => x.Status);
        builder.Entity<JSEA>()
            .HasIndex(x => x.RequesterId);

        // PermitApproval
        builder.Entity<PermitApproval>()
            .HasIndex(x => new { x.PermitId, x.Sequence });

        // AuditLog
        builder.Entity<AuditLog>()
            .HasIndex(x => new { x.Module, x.EntityId });
        builder.Entity<AuditLog>()
            .HasIndex(x => x.ActionDate);

        // Notification
        builder.Entity<Notification>()
            .HasIndex(x => new { x.UserId, x.IsRead });

        // WorkPermit relationships
        builder.Entity<WorkPermit>()
            .HasOne(x => x.Requester)
            .WithMany()
            .HasForeignKey(x => x.RequesterId)
            .OnDelete(DeleteBehavior.Restrict);

        // JSEA relationships
        builder.Entity<JSEA>()
            .HasOne(x => x.Requester)
            .WithMany()
            .HasForeignKey(x => x.RequesterId)
            .OnDelete(DeleteBehavior.Restrict);

        builder.Entity<JSEA>()
            .HasOne(x => x.ApprovedBy)
            .WithMany()
            .HasForeignKey(x => x.ApprovedById)
            .OnDelete(DeleteBehavior.Restrict);

        builder.Entity<JSEA>()
            .HasOne(x => x.ReviewedBy)
            .WithMany()
            .HasForeignKey(x => x.ReviewedById)
            .OnDelete(DeleteBehavior.Restrict);

        // PermitApproval
        builder.Entity<PermitApproval>()
            .HasOne(x => x.Approver)
            .WithMany()
            .HasForeignKey(x => x.ApproverId)
            .OnDelete(DeleteBehavior.Restrict);

        // Seed ApprovalMatrix data
        SeedApprovalMatrix(builder);
    }

    private void SeedApprovalMatrix(ModelBuilder builder)
    {
        var matrices = new List<ApprovalMatrix>();
        int id = 1;

        var workTypes = Enum.GetValues<Domain.Enums.WorkType>();
        var riskLevels = Enum.GetValues<Domain.Enums.RiskLevel>();

        foreach (var wt in workTypes)
        {
            foreach (var rl in riskLevels)
            {
                // Step 1: HSE Officer selalu ada
                matrices.Add(new ApprovalMatrix
                {
                    Id = id++,
                    WorkType = wt,
                    RiskLevel = rl,
                    RequiredRole = "HSEOfficer",
                    Sequence = 1,
                    Description = "HSE Review",
                    IsActive = true,
                    CreatedAt = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc)
                });

                // Step 2: Area Owner
                matrices.Add(new ApprovalMatrix
                {
                    Id = id++,
                    WorkType = wt,
                    RiskLevel = rl,
                    RequiredRole = "AreaOwner",
                    Sequence = 2,
                    Description = "Area Owner Approval",
                    IsActive = true,
                    CreatedAt = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc)
                });

                // Step 3: Final Approval - hanya untuk High & Critical
                if (rl == Domain.Enums.RiskLevel.High || rl == Domain.Enums.RiskLevel.Critical)
                {
                    matrices.Add(new ApprovalMatrix
                    {
                        Id = id++,
                        WorkType = wt,
                        RiskLevel = rl,
                        RequiredRole = "Approver",
                        Sequence = 3,
                        Description = "Final Manager Approval",
                        IsActive = true,
                        CreatedAt = new DateTime(2024, 1, 1, 0, 0, 0, DateTimeKind.Utc)
                    });
                }
            }
        }

        builder.Entity<ApprovalMatrix>().HasData(matrices);
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = DateTime.UtcNow;
            }
        }
        return base.SaveChangesAsync(cancellationToken);
    }
}
```

---

## BAGIAN D — INFRASTRUCTURE: Repositories

### FILE: `src/PTW.JSEA.Infrastructure/Repositories/GenericRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Common;
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

    public async Task<T> AddAsync(T entity)
    {
        await _dbSet.AddAsync(entity);
        await _context.SaveChangesAsync();
        return entity;
    }

    public async Task UpdateAsync(T entity)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync();
    }

    public async Task SoftDeleteAsync(int id)
    {
        var entity = await GetByIdAsync(id);
        if (entity is BaseEntity baseEntity)
        {
            baseEntity.IsDeleted = true;
            baseEntity.UpdatedAt = DateTime.UtcNow;
            await _context.SaveChangesAsync();
        }
    }

    public async Task<int> CountAsync(Expression<Func<T, bool>>? predicate = null)
        => predicate == null
            ? await _dbSet.CountAsync()
            : await _dbSet.CountAsync(predicate);

    public async Task<bool> ExistsAsync(Expression<Func<T, bool>> predicate)
        => await _dbSet.AnyAsync(predicate);

    public IQueryable<T> Query() => _dbSet.AsQueryable();
}
```

---

### FILE: `src/PTW.JSEA.Infrastructure/Repositories/WorkPermitRepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.Repositories;

public class WorkPermitRepository : GenericRepository<WorkPermit>, IWorkPermitRepository
{
    public WorkPermitRepository(ApplicationDbContext context) : base(context) { }

    public async Task<WorkPermit?> GetWithDetailsAsync(int id)
        => await _context.WorkPermits
            .Include(x => x.JSEA).ThenInclude(j => j.Details)
            .Include(x => x.Requester)
            .Include(x => x.Approvals).ThenInclude(a => a.Approver)
            .Include(x => x.Workers)
            .Include(x => x.PAIChecklists)
            .Include(x => x.AdherenceLogs)
            .Include(x => x.Attachments)
            .FirstOrDefaultAsync(x => x.Id == id);

    public async Task<IEnumerable<WorkPermit>> GetActivePermitsAsync()
        => await _context.WorkPermits
            .Include(x => x.Requester)
            .Include(x => x.Workers)
            .Where(x => x.Status == PermitStatus.Active)
            .OrderBy(x => x.EndTime)
            .ToListAsync();

    public async Task<IEnumerable<WorkPermit>> GetPermitsByUserAsync(string userId)
        => await _context.WorkPermits
            .Include(x => x.JSEA)
            .Where(x => x.RequesterId == userId)
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync();

    public async Task<IEnumerable<WorkPermit>> GetPermitsByStatusAsync(PermitStatus status)
        => await _context.WorkPermits
            .Include(x => x.Requester)
            .Where(x => x.Status == status)
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync();

    public async Task<IEnumerable<WorkPermit>> GetExpiringPermitsAsync(int withinHours)
    {
        var threshold = DateTime.UtcNow.AddHours(withinHours);
        return await _context.WorkPermits
            .Include(x => x.Requester)
            .Where(x => x.Status == PermitStatus.Active
                     && x.EndTime <= threshold
                     && x.EndTime > DateTime.UtcNow)
            .ToListAsync();
    }

    public async Task<int> GetTodaySequenceAsync(WorkType workType)
    {
        var today = DateTime.UtcNow.Date;
        return await _context.WorkPermits
            .CountAsync(x => x.WorkType == workType
                          && x.CreatedAt.Date == today);
    }

    public async Task<int> GetActiveCountAsync()
        => await _context.WorkPermits.CountAsync(x => x.Status == PermitStatus.Active);

    public async Task<int> GetHighRiskCountAsync()
        => await _context.WorkPermits.CountAsync(x =>
            x.Status == PermitStatus.Active &&
            (x.RiskLevel == RiskLevel.High || x.RiskLevel == RiskLevel.Critical));
}
```

---

### FILE: `src/PTW.JSEA.Infrastructure/Repositories/JSEARepository.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.Repositories;

public class JSEARepository : GenericRepository<JSEA>, IJSEARepository
{
    public JSEARepository(ApplicationDbContext context) : base(context) { }

    public async Task<JSEA?> GetWithDetailsAsync(int id)
        => await _context.JSEAs
            .Include(x => x.Requester)
            .Include(x => x.ReviewedBy)
            .Include(x => x.ApprovedBy)
            .Include(x => x.Details.OrderBy(d => d.StepNumber))
            .Include(x => x.Attachments)
            .FirstOrDefaultAsync(x => x.Id == id);

    public async Task<IEnumerable<JSEA>> GetByUserAsync(string userId)
        => await _context.JSEAs
            .Include(x => x.Requester)
            .Where(x => x.RequesterId == userId)
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync();

    public async Task<IEnumerable<JSEA>> GetByStatusAsync(JSEAStatus status)
        => await _context.JSEAs
            .Include(x => x.Requester)
            .Where(x => x.Status == status)
            .OrderByDescending(x => x.CreatedAt)
            .ToListAsync();

    public async Task<IEnumerable<JSEA>> GetApprovedTemplatesAsync()
        => await _context.JSEAs
            .Include(x => x.Requester)
            .Where(x => x.Status == JSEAStatus.Approved)
            .OrderByDescending(x => x.ApprovedAt)
            .ToListAsync();
}
```

---

### FILE: `src/PTW.JSEA.Infrastructure/Repositories/UserRepository.cs`
```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using PTW.JSEA.Domain.Entities;
using PTW.JSEA.Domain.Enums;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;

namespace PTW.JSEA.Infrastructure.Repositories;

public class UserRepository : IUserRepository
{
    private readonly ApplicationDbContext _context;
    private readonly UserManager<ApplicationUser> _userManager;

    public UserRepository(ApplicationDbContext context, UserManager<ApplicationUser> userManager)
    {
        _context = context;
        _userManager = userManager;
    }

    public async Task<ApplicationUser?> GetByIdAsync(string id)
        => await _context.Users.Include(x => x.Certificates).FirstOrDefaultAsync(x => x.Id == id);

    public async Task<IEnumerable<ApplicationUser>> GetByRoleAsync(string roleName)
    {
        var users = await _userManager.GetUsersInRoleAsync(roleName);
        return users.Where(x => x.IsActive);
    }

    public async Task<IEnumerable<UserCertificate>> GetValidCertificatesAsync(string userId, WorkType workType)
        => await _context.UserCertificates
            .Where(x => x.UserId == userId
                     && x.WorkType == workType
                     && x.Status == CertificateStatus.Active
                     && x.ExpiryDate > DateTime.UtcNow)
            .ToListAsync();

    public async Task<bool> HasValidCertificateAsync(string userId, WorkType workType)
        => await _context.UserCertificates
            .AnyAsync(x => x.UserId == userId
                        && x.WorkType == workType
                        && x.Status == CertificateStatus.Active
                        && x.ExpiryDate > DateTime.UtcNow);

    public async Task<IEnumerable<UserCertificate>> GetExpiringCertificatesAsync(int withinDays)
    {
        var threshold = DateTime.UtcNow.AddDays(withinDays);
        return await _context.UserCertificates
            .Include(x => x.User)
            .Where(x => x.Status == CertificateStatus.Active
                     && x.ExpiryDate <= threshold
                     && x.ExpiryDate > DateTime.UtcNow)
            .ToListAsync();
    }
}
```

---

## BAGIAN E — INFRASTRUCTURE: DI Registration

### FILE: `src/PTW.JSEA.Infrastructure/DependencyInjection.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using PTW.JSEA.Domain.Interfaces.Repositories;
using PTW.JSEA.Infrastructure.Data;
using PTW.JSEA.Infrastructure.Repositories;

namespace PTW.JSEA.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(
                configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly("PTW.JSEA.Infrastructure")
            ));

        // Repositories
        services.AddScoped(typeof(IGenericRepository<>), typeof(GenericRepository<>));
        services.AddScoped<IWorkPermitRepository, WorkPermitRepository>();
        services.AddScoped<IJSEARepository, JSEARepository>();
        services.AddScoped<IUserRepository, UserRepository>();

        // HttpContextAccessor for audit
        services.AddHttpContextAccessor();

        return services;
    }
}
```

---

## BAGIAN F — KONFIGURASI

### FILE: `src/PTW.JSEA.Web/appsettings.json`
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost\\SQLEXPRESS;Database=PTW_JSEA_DB;Trusted_Connection=True;MultipleActiveResultSets=true;TrustServerCertificate=True"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    }
  },
  "AllowedHosts": "*",
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
    "SmtpPort": 587,
    "SenderEmail": "noreply@ptw-mattel.com",
    "SenderName": "PTW System Mattel",
    "Username": "",
    "Password": ""
  }
}
```

---

## BAGIAN G — MIGRATION

### Jalankan perintah berikut:

```bash
cd C:\Projects\PTW-JSEA

dotnet ef migrations add InitialCreate \
  --project src/PTW.JSEA.Infrastructure \
  --startup-project src/PTW.JSEA.Web \
  --output-dir Data/Migrations

dotnet ef database update \
  --project src/PTW.JSEA.Infrastructure \
  --startup-project src/PTW.JSEA.Web
```

### Jika ada error "Unable to create migrations":
Pastikan `Program.cs` sudah ada `AddDbContext` (ada di Modul 3).
Jalankan migration SETELAH menyelesaikan Modul 3.

---

> ✅ Modul 2 selesai. Lanjut ke Modul 3.
