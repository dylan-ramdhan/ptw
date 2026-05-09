# MODUL 1 — PROJECT SETUP & CLEAN ARCHITECTURE

> Jalankan semua perintah ini secara berurutan di terminal VS Code.
> Buka terminal: `Ctrl + `` ` (backtick)

---

## LANGKAH 1.1 — Buat Folder & Solution

```bash
mkdir C:\Projects\PTW-JSEA
cd C:\Projects\PTW-JSEA
dotnet new sln -n PTW.JSEA
```

---

## LANGKAH 1.2 — Buat 4 Layer Project

```bash
dotnet new classlib -n PTW.JSEA.Domain      -o src/PTW.JSEA.Domain
dotnet new classlib -n PTW.JSEA.Application -o src/PTW.JSEA.Application
dotnet new classlib -n PTW.JSEA.Infrastructure -o src/PTW.JSEA.Infrastructure
dotnet new webapp   -n PTW.JSEA.Web         -o src/PTW.JSEA.Web
```

---

## LANGKAH 1.3 — Daftarkan ke Solution

```bash
dotnet sln add src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj
dotnet sln add src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet sln add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj
dotnet sln add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj
```

---

## LANGKAH 1.4 — Set Project References

```bash
dotnet add src/PTW.JSEA.Application/PTW.JSEA.Application.csproj reference src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj

dotnet add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj reference src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet add src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj reference src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj

dotnet add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj reference src/PTW.JSEA.Application/PTW.JSEA.Application.csproj
dotnet add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj reference src/PTW.JSEA.Infrastructure/PTW.JSEA.Infrastructure.csproj
dotnet add src/PTW.JSEA.Web/PTW.JSEA.Web.csproj reference src/PTW.JSEA.Domain/PTW.JSEA.Domain.csproj
```

---

## LANGKAH 1.5 — Install NuGet Packages

### Domain (tidak perlu package)
Biarkan apa adanya.

### Application
```bash
cd src/PTW.JSEA.Application
dotnet add package Microsoft.Extensions.DependencyInjection.Abstractions --version 8.0.0
dotnet add package Microsoft.Extensions.Logging.Abstractions --version 8.0.0
cd ../..
```

### Infrastructure
```bash
cd src/PTW.JSEA.Infrastructure
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 8.0.0
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 8.0.0
dotnet add package Serilog.AspNetCore --version 8.0.0
dotnet add package Serilog.Sinks.File --version 5.0.0
dotnet add package MailKit --version 4.3.0
cd ../..
```

### Web
```bash
cd src/PTW.JSEA.Web
dotnet add package Microsoft.AspNetCore.SignalR --version 8.0.0
dotnet add package Microsoft.EntityFrameworkCore.Design --version 8.0.0
dotnet add package Serilog.AspNetCore --version 8.0.0
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design --version 8.0.0
cd ../..
```

---

## LANGKAH 1.6 — Buat Struktur Folder Domain

Buat folder-folder berikut secara manual di VS Code Explorer, atau jalankan perintah ini:

```bash
cd C:\Projects\PTW-JSEA

# Domain folders
mkdir src\PTW.JSEA.Domain\Common
mkdir src\PTW.JSEA.Domain\Entities
mkdir src\PTW.JSEA.Domain\Enums
mkdir src\PTW.JSEA.Domain\Interfaces
mkdir src\PTW.JSEA.Domain\Interfaces\Repositories
mkdir src\PTW.JSEA.Domain\Interfaces\Services

# Application folders
mkdir src\PTW.JSEA.Application\Common
mkdir src\PTW.JSEA.Application\DTOs
mkdir src\PTW.JSEA.Application\Services
mkdir src\PTW.JSEA.Application\Services\Interfaces

# Infrastructure folders
mkdir src\PTW.JSEA.Infrastructure\Data
mkdir src\PTW.JSEA.Infrastructure\Data\Configurations
mkdir src\PTW.JSEA.Infrastructure\Repositories
mkdir src\PTW.JSEA.Infrastructure\Services
mkdir src\PTW.JSEA.Infrastructure\BackgroundServices

# Web folders
mkdir src\PTW.JSEA.Web\Hubs
mkdir src\PTW.JSEA.Web\Pages\Account
mkdir src\PTW.JSEA.Web\Pages\Dashboard
mkdir src\PTW.JSEA.Web\Pages\JSEA
mkdir src\PTW.JSEA.Web\Pages\Permits
mkdir src\PTW.JSEA.Web\Pages\Monitoring
mkdir src\PTW.JSEA.Web\Pages\Reports
mkdir src\PTW.JSEA.Web\Pages\Admin
mkdir src\PTW.JSEA.Web\Pages\Shared
```

---

## LANGKAH 1.7 — Setup TailwindCSS

```bash
cd src/PTW.JSEA.Web
npm init -y
npm install -D tailwindcss @tailwindcss/forms
npx tailwindcss init
```

Buat file `src/PTW.JSEA.Web/wwwroot/css/app.css`:
(isi kode ada di file MODUL-4 nanti, untuk sekarang buat file kosong dulu)

```bash
mkdir wwwroot\css
echo. > wwwroot\css\app.css
```

Update `package.json` — ganti bagian `"scripts"`:
```json
"scripts": {
  "build:css": "tailwindcss -i ./wwwroot/css/app.css -o ./wwwroot/css/output.css",
  "watch:css": "tailwindcss -i ./wwwroot/css/app.css -o ./wwwroot/css/output.css --watch"
}
```

---

## LANGKAH 1.8 — Verifikasi Build

```bash
cd C:\Projects\PTW-JSEA
dotnet build
```

✅ Harus muncul: `Build succeeded` tanpa error.

---

## STRUKTUR AKHIR MODUL 1

```
C:\Projects\PTW-JSEA\
├── PTW.JSEA.sln
└── src/
    ├── PTW.JSEA.Domain/
    │   ├── Common/
    │   ├── Entities/
    │   ├── Enums/
    │   └── Interfaces/
    │       ├── Repositories/
    │       └── Services/
    ├── PTW.JSEA.Application/
    │   ├── Common/
    │   ├── DTOs/
    │   └── Services/
    ├── PTW.JSEA.Infrastructure/
    │   ├── Data/
    │   ├── Repositories/
    │   ├── Services/
    │   └── BackgroundServices/
    └── PTW.JSEA.Web/
        ├── Hubs/
        ├── Pages/
        │   ├── Account/
        │   ├── Dashboard/
        │   ├── JSEA/
        │   ├── Permits/
        │   ├── Monitoring/
        │   ├── Reports/
        │   └── Admin/
        └── wwwroot/
            └── css/
```

> ✅ Modul 1 selesai. Lanjut ke Modul 2.
