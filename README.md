# Task-mobile-app

An offline-first construction task tracking app with two parts:

- **`TaskApi/`** — ASP.NET Core (.NET 10) Web API, EF Core + SQL Server, JWT auth.
- **`TaskMobile/`** — Expo / React Native app, WatermelonDB (SQLite) for offline, Axios for API calls.

Users log in to a specific project, browse "Look Ahead" activities, enter and sync daily execution quantities. Writes performed offline are queued locally and replayed when the device is back online.

---

## Repository layout

```
TaskApp/
├── TaskApi/          ← ASP.NET Core API (backend)
│   ├── Controllers/
│   ├── Models/
│   ├── Services/
│   ├── appsettings.json              ← placeholders only, safe to commit
│   └── appsettings.Development.json  ← local secrets, git-ignored
├── TaskMobile/       ← Expo React Native app (frontend)
│   ├── src/
│   ├── .env                          ← local env vars, git-ignored
│   └── .env.example                  ← template for other devs
├── .gitignore
└── README.md
```

---

## Prerequisites

Install these once on your machine:

| Tool | Minimum version | How to install |
|---|---|---|
| [.NET SDK](https://dotnet.microsoft.com/download) | 10.0 (preview) | Download the installer |
| [Node.js](https://nodejs.org/) | 20 LTS or newer | Download the LTS installer |
| [Git](https://git-scm.com/download/win) | any recent | Download for Windows |
| SQL Server | any edition (Developer / Express / full) | [SQL Server Downloads](https://www.microsoft.com/sql-server/sql-server-downloads) |
| Android Studio (for Android emulator) OR a physical device | — | [Android Studio](https://developer.android.com/studio) |

After installing Node.js, install the Expo CLI once:

```powershell
npm install -g expo
```

---

## 1 — Clone the repo

```powershell
git clone https://github.com/omarZrayka23/Task-mobile-app.git
cd Task-mobile-app
```

---

## 2 — Backend (`TaskApi`)

### 2.1 — Create your local secrets file

The committed `TaskApi/appsettings.json` has placeholders only. Create `TaskApi/appsettings.Development.json` (git-ignored) with your real database connection string and a JWT key:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=YOUR_SERVER;Database=YOUR_DB;User ID=YOUR_USER;Password=YOUR_PASSWORD;Trusted_Connection=false;Encrypt=False;TrustServerCertificate=True;"
  },
  "Jwt": {
    "Key": "at-least-32-random-characters-here-replace-this"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Generate a safe JWT key with any password generator (32+ characters) or in PowerShell:

```powershell
-join ((48..57) + (65..90) + (97..122) | Get-Random -Count 48 | ForEach-Object {[char]$_})
```

### 2.2 — Restore and run

```powershell
cd TaskApi
dotnet restore
dotnet run
```

You should see:

```
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
```

### 2.3 — Verify it's up

Open [http://localhost:5000/swagger](http://localhost:5000/swagger) in a browser. You should see the Swagger UI with the `Activity`, `Auth`, and `Lookup` endpoints.

### 2.4 — Troubleshooting

| Problem | Fix |
|---|---|
| `Could not connect to SQL Server` | Check the server address + credentials in `appsettings.Development.json`. Make sure SQL Server is running and reachable. |
| `IDX10720: Unable to create KeyedHashAlgorithm` | Your JWT key is too short. Use at least 32 characters. |
| `error MSB3027: file is locked by TaskApi.exe` | The API is already running in another window. Close it and rebuild. |
| Port 5000 already in use | Change the port in `TaskApi/Properties/launchSettings.json`. |

---

## 3 — Frontend (`TaskMobile`)

### 3.1 — Install dependencies

```powershell
cd TaskMobile
npm install
```

(This creates `node_modules/` — large, git-ignored.)

### 3.2 — Create your local `.env`

Copy the template and edit it:

```powershell
Copy-Item .env.example .env
```

Open `.env` and set `EXPO_PUBLIC_API_URL` to match where your backend is running:

| Where you're running the app | Value |
|---|---|
| Android emulator (same machine as API) | `http://10.0.2.2:5000` |
| iOS simulator (Mac only) | `http://localhost:5000` |
| Physical phone on same Wi-Fi | `http://<YOUR_DEV_PC_LAN_IP>:5000` |
| Production API | `https://api.your-domain.com` |

> **Physical device tip** — find your dev PC's LAN IP with `ipconfig` (look for `IPv4 Address` under your active Wi-Fi adapter). Your phone must be on the same Wi-Fi. Windows Firewall may need to allow inbound TCP 5000.

### 3.3 — Run the app

Make sure the backend from Step 2 is still running, then:

```powershell
npm start
```

Expo DevTools opens. Options:

- Press **`a`** → launch on an Android emulator (requires Android Studio).
- Press **`i`** → launch on iOS simulator (Mac only).
- Scan the QR code with the **Expo Go** app on your phone (install from Play Store / App Store).

### 3.4 — Dev build vs. Expo Go

This project uses `expo-dev-client` and native modules (`@nozbe/watermelondb`, `expo-secure-store`), which **don't work with the public Expo Go app**. For a real device test, build a dev client once:

```powershell
npx expo run:android       # produces an .apk and installs on a connected device/emulator
# or
npx expo run:ios           # Mac only
```

After that first native build, you can keep using `npm start` for day-to-day JS iteration.

### 3.5 — Troubleshooting

| Problem | Fix |
|---|---|
| `Network Error` on login | Backend not reachable from the device. Confirm `EXPO_PUBLIC_API_URL`, same Wi-Fi, firewall. |
| `.env` changes not picked up | Restart Metro with cache cleared: `npx expo start -c` |
| `Invalid key provided to SecureStore` | Keys must be `[A-Za-z0-9._-]` only. See `TaskMobile/src/api/offline/projectBackfill.js` for an example. |
| `Cannot update a record with pending changes` (WatermelonDB) | Batch your `prepareUpdate` calls inside a single `database.write(...)` and don't reuse a prepared record. |
| Metro bundler stuck | Delete the cache: `Remove-Item -Recurse -Force $env:TEMP\metro-*` and restart with `npx expo start -c` |

---

## 4 — Typical dev workflow

Open two terminal windows:

**Terminal 1 — backend**
```powershell
cd TaskApi
dotnet watch run      # auto-restarts on C# file save
```

**Terminal 2 — mobile**
```powershell
cd TaskMobile
npm start             # Metro bundler; auto-reloads on JS save
```

Then launch the app on your emulator / device. Edit code → it hot-reloads.

---

## 5 — First login (seeding a test user)

The API doesn't ship with a seeded user — you need at least one row in your `User` table. Insert one manually:

```sql
INSERT INTO Users (UsrID, UsrPWD, UsrFullName, UsrType)
VALUES ('your.email@example.com', 'YourTempPassword', 'Your Name', 'SiteEng');
```

> ⚠️ Passwords are currently stored in plaintext — this is a known issue tracked for the next security pass. Do not reuse any real password.

Then in the mobile app:
1. Pick a project from the dropdown.
2. Enter the email + password you just inserted.
3. Tap **Login**.

---

## 6 — Build for production

### Backend

```powershell
cd TaskApi
dotnet publish -c Release -o ./publish
```

Deploy the contents of `./publish` behind IIS, Kestrel, or a container. Set `ASPNETCORE_ENVIRONMENT=Production` and provide secrets via environment variables or a secret manager — not `appsettings.Production.json` in the published folder.

### Mobile

For a release APK / AAB via EAS:

```powershell
cd TaskMobile
npx eas build --platform android --profile production
```

(Requires an [Expo account](https://expo.dev/signup) and `eas-cli` installed: `npm install -g eas-cli`.)

---

## 7 — What's NOT in this repo

The `.gitignore` keeps these out on purpose. If you clone and something's missing, create it locally:

| Missing file | How to recreate |
|---|---|
| `TaskApi/appsettings.Development.json` | See Step 2.1 above |
| `TaskMobile/.env` | `Copy-Item TaskMobile/.env.example TaskMobile/.env` and edit |
| `TaskMobile/node_modules/` | `cd TaskMobile && npm install` |
| `TaskApi/bin/`, `TaskApi/obj/` | `cd TaskApi && dotnet build` |

---

## 8 — License

Private project — not open-sourced.
