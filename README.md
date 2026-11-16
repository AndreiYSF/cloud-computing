# TaskFlow: gestionar de task-uri pe Azure

Proiect pentru cursul de Cloud Computing (FMI). O aplicație web de gestionare a proiectelor și task-urilor în echipă. Are autentificare Microsoft Entra ID, autorizare pe roluri, o bază de date privată, notificări prin email trimise asincron printr-o coadă de mesaje și infrastructură descrisă complet ca cod.

> Status: **planificare**. Pentru moment repo-ul conține doar acest README. Structura de mai jos este cea care urmează să fie implementată.

---

## 1. Cerințe și cum sunt acoperite

| Cerință curs | Acoperire |
|---|---|
| Cel puțin o resursă compute | **Azure App Service** (API), **Azure Functions** (worker notificări), **Static Web Apps** (frontend) |
| Bază de date / object storage | **Azure SQL Database** (serverless) + **Blob Storage** (atașamente) |
| Aplicație securizată, DB neexpusă în internet | SQL cu acces public dezactivat, accesibil doar prin **Private Endpoint** din VNet. Conectare cu **Managed Identity**, fără parole. Secretele stau în **Key Vault**. |
| Autentificare | **Microsoft Entra ID** (OAuth2 / OpenID Connect, token JWT validat în API) |
| Autorizare | **App roles** în Entra (`Admin`, `Member`) + reguli la nivel de resursă (doar membrii unui proiect îi văd task-urile) |
| Coadă de mesaje (ideal) | **Azure Service Bus** (queue). API-ul publică evenimente, iar Function-ul le consumă și trimite email. |
| Deployment scriptat | **Bicep** + scripturi `az cli` (`deploy.sh` / `teardown.sh`) |
| Nu „Hello World” | CRUD proiecte, task-uri, comentarii, atașamente, statusuri, filtrare, roluri, remindere programate |

---

## 2. Funcționalități

**Utilizator `Member`**
- Login cu cont Microsoft (Entra ID)
- Vede proiectele în care este membru
- Vede, filtrează (status, prioritate, deadline) și actualizează statusul task-urilor asignate lui
- Adaugă comentarii și atașamente la task-uri
- Primește email când i se asignează un task și cu 24h înainte de deadline

**Utilizator `Admin`** (toate drepturile `Member`, plus)
- Creează, editează și șterge proiecte
- Adaugă sau scoate membri din proiecte
- Creează task-uri și le asignează
- Vede toate proiectele

**Statusuri task:** `To Do` → `In Progress` → `Done`

---

## 3. Arhitectură

```
                    ┌──────────────────────┐
                    │  Microsoft Entra ID  │  (login, token JWT, app roles)
                    └──────────┬───────────┘
                               │
┌──────────────┐   HTTPS + JWT │
│ Static Web   │───────────────▼──────────────────────────────────────────┐
│ Apps (React) │        ┌─────────────────┐     VNet                      │
└──────────────┘        │  App Service    │  (VNet integration)           │
                        │  FastAPI        │──────────┐                    │
                        └──┬──────┬───────┘          │                    │
                Managed    │      │                  ▼                    │
                Identity   │      │         ┌──────────────────┐          │
                           │      │         │ Private Endpoint │          │
                           │      │         │  → Azure SQL     │ (fără    │
                           │      │         └──────────────────┘ acces    │
                           │      │                             public)   │
                           │      └──► Blob Storage (atașamente, privat)  │
                           ▼                                              │
                 ┌──────────────────┐                                     │
                 │ Service Bus      │  queue: notifications               │
                 └────────┬─────────┘                                     │
                          ▼                                               │
                 ┌──────────────────┐      ┌────────────────────────┐     │
                 │ Azure Function   │─────►│ Communication Services │──► email
                 │ (SB trigger)     │      │ Email                  │     │
                 └──────────────────┘      └────────────────────────┘     │
                                                                          │
      Key Vault (secrete)   ·   Application Insights (loguri, monitorizare)
```

**Fluxuri principale**
1. **Login:** frontend-ul (MSAL) autentifică utilizatorul în Entra ID și primește un access token, pe care îl trimite în header-ul `Authorization: Bearer` către API.
2. **Request API:** FastAPI validează JWT-ul (semnătură, audience, issuer) și citește rolurile din claim-ul `roles`. Apoi aplică regulile de autorizare.
3. **Acces date:** API-ul se conectează la Azure SQL prin Private Endpoint, autentificat cu Managed Identity (Entra auth pe SQL), fără connection string cu parolă.
4. **Notificare la asignare:** API-ul publică un mesaj `TaskAssigned` în Service Bus, iar Function-ul îl consumă și trimite email prin ACS.
5. **Reminder deadline:** la crearea sau modificarea unui task, API-ul programează un mesaj Service Bus (*scheduled message*) pentru `deadline − 24h`. `sequence_number`-ul se salvează în DB, ca mesajul să poată fi anulat dacă task-ul e finalizat sau deadline-ul se schimbă.

**De ce Function-ul nu e în VNet:** mesajul conține deja tot ce trebuie pentru email (destinatar, titlu task, link). Function-ul nu accesează DB-ul, deci poate rula pe planul Consumption (gratuit), fără integrare VNet.

---

## 4. Stack tehnic

| Componentă | Tehnologie |
|---|---|
| Backend API | Python 3.12, **FastAPI**, SQLAlchemy, Alembic (migrări), `pyodbc` |
| Auth backend | Validare JWT cu `PyJWT` + cheile JWKS ale Entra |
| Frontend | **React + Vite + TypeScript**, `@azure/msal-react` |
| Worker | **Azure Functions** (Python), trigger Service Bus |
| Email | Azure Communication Services Email (domeniu gestionat Azure) |
| IaC | **Bicep** (module) + `az cli` |
| Monitorizare | Application Insights |
| CI/CD (opțional) | GitHub Actions |

---

## 5. Resurse Azure

Toate resursele stau într-un **singur resource group** (`rg-taskflow-dev`), ca ștergerea să fie o singură comandă.

| Resursă | SKU / tier | Rol |
|---|---|---|
| App Service Plan + Web App | **B1** Linux (minim necesar pentru VNet integration) | API FastAPI |
| Azure SQL Database | **Serverless**, General Purpose, auto-pause 1h | Date relaționale |
| Private Endpoint + Private DNS Zone | — | Acces privat la SQL |
| Virtual Network | 2 subnet-uri: `snet-app` (delegat App Service), `snet-pe` | Izolare rețea |
| Storage Account | Standard LRS, acces public blocat | Atașamente + storage-ul Function-ului |
| Service Bus Namespace | **Basic** (queue + scheduled messages) | Coada `notifications` |
| Function App | **Consumption** | Consumator notificări |
| Communication Services + Email Service | Pay-as-you-go | Trimitere email |
| Key Vault | Standard, RBAC | Secrete (ex. connection string ACS) |
| Static Web App | **Free** | Frontend |
| Application Insights + Log Analytics | Pay-as-you-go | Loguri |
| User-assigned Managed Identity | — | Identitate API/Function → SQL, Service Bus, Storage, Key Vault |

**App Registration-urile din Entra ID** (API + SPA, cu app roles) se creează o singură dată, cu script `az ad`, și **nu se șterg** între iterații. Nu costă nimic, iar așa utilizatorii și rolurile asignate rămân.

---

## 6. Securitate

- **Rețea:** SQL are `publicNetworkAccess = Disabled` și e accesibil doar prin Private Endpoint din VNet. Storage-ul are acces public la blob-uri dezactivat.
- **Identitate în loc de parole:** API-ul și Function-ul folosesc Managed Identity, cu RBAC pe Service Bus (`Data Sender` / `Data Receiver`), Storage (`Blob Data Contributor`) și Key Vault (`Secrets User`). Pe SQL, identitatea e creată ca user Entra cu drepturi minime (`db_datareader`, `db_datawriter`).
- **SQL:** doar autentificare Entra (`azureADOnlyAuthentication = true`), fără user SQL cu parolă.
- **Atașamente:** descărcarea se face prin **SAS cu durată scurtă** (user delegation SAS), generat de API doar după verificarea dreptului de acces.
- **API:** doar HTTPS, CORS restrâns la domeniul Static Web App, validare strictă a token-ului (audience, issuer, expirare), validare input cu Pydantic.
- **Autorizare pe două niveluri:**
  - *Rol* (`Admin` / `Member`) din claim-ul `roles` pentru operații de administrare
  - *Resursă*: un `Member` vede doar proiectele în care e membru, verificat în query
- **Secrete:** niciun secret în repo. Ce nu se poate face prin Managed Identity stă în Key Vault, referit din App Settings (`@Microsoft.KeyVault(...)`).

---

## 7. Model de date

```
User          (id = Entra object id, email, display_name, created_at)
Project       (id, name, description, created_by → User, created_at)
ProjectMember (project_id → Project, user_id → User)            PK compus
Task          (id, project_id → Project, title, description, status,
               priority, deadline, assignee_id → User, created_by → User,
               reminder_seq_number, created_at, updated_at)
Comment       (id, task_id → Task, author_id → User, body, created_at)
Attachment    (id, task_id → Task, blob_name, file_name, size, uploaded_by, created_at)
```

User-ul e creat sau actualizat automat la primul request autentificat (din claim-urile `oid`, `email`, `name`).

---

## 8. API (FastAPI)

| Metodă | Endpoint | Acces |
|---|---|---|
| GET | `/me` | autentificat |
| GET | `/projects` | Member (doar ale lui) / Admin (toate) |
| POST | `/projects` | Admin |
| PATCH / DELETE | `/projects/{id}` | Admin |
| POST / DELETE | `/projects/{id}/members/{userId}` | Admin |
| GET | `/projects/{id}/tasks?status=&priority=&assignee=` | membru proiect |
| POST | `/projects/{id}/tasks` | Admin |
| PATCH | `/tasks/{id}` | Admin (orice câmp) / assignee (doar `status`) |
| DELETE | `/tasks/{id}` | Admin |
| GET / POST | `/tasks/{id}/comments` | membru proiect |
| POST | `/tasks/{id}/attachments` | membru proiect |
| GET | `/attachments/{id}/download` | membru proiect → SAS URL |
| GET | `/health` | public |

Documentație automată la `/docs` (OpenAPI), activă doar în mediul dev.

---

## 9. Mesaje în coadă

Coada: `notifications`

```json
{
  "type": "TaskAssigned | DeadlineReminder",
  "taskId": 42,
  "taskTitle": "Configurare Private Endpoint",
  "projectName": "Cloud Computing",
  "recipientEmail": "user@example.com",
  "deadline": "2027-01-10T12:00:00Z",
  "link": "https://<swa>.azurestaticapps.net/tasks/42"
}
```

- `TaskAssigned`: trimis imediat la asignare sau re-asignare
- `DeadlineReminder`: programat (`scheduled_enqueue_time`) la `deadline − 24h`, anulat la `Done` sau la schimbarea deadline-ului
- După 10 încercări eșuate, mesajele ajung în **dead-letter queue**, inspectabilă pentru debugging

---

## 10. Structura repo-ului (planificată)

```
cloud computing/
├── README.md
├── infra/
│   ├── main.bicep                 # orchestrare module
│   ├── main.bicepparam            # parametri (location, prefix, ...)
│   └── modules/
│       ├── network.bicep          # VNet, subnet-uri, Private DNS
│       ├── sql.bicep              # SQL server + DB + Private Endpoint
│       ├── storage.bicep
│       ├── servicebus.bicep
│       ├── appservice.bicep       # plan + web app + VNet integration
│       ├── function.bicep
│       ├── communication.bicep
│       ├── keyvault.bicep
│       ├── staticwebapp.bicep
│       ├── monitoring.bicep
│       └── identity.bicep         # managed identity + role assignments
├── scripts/
│   ├── setup-entra.sh             # app registrations + app roles (o singură dată)
│   ├── deploy.sh                  # infra + cod + migrări + SQL user pt MI
│   └── teardown.sh                # az group delete
├── backend/                       # FastAPI
│   ├── app/ (main.py, auth.py, models.py, routers/, services/)
│   ├── alembic/
│   └── requirements.txt
├── functions/                     # Azure Function (Service Bus trigger)
│   └── notifications/
└── frontend/                      # React + Vite + MSAL
```

---

## 11. Deployment

**Prerechizite:** Azure CLI (+ extensia Bicep), Python 3.12, Node 20, Azure Functions Core Tools, abonament Azure for Students.

```bash
# o singură dată: app registrations + roluri în Entra ID
./scripts/setup-entra.sh

# la fiecare iterație de lucru
./scripts/deploy.sh

# la finalul iterației: ȘTERGE TOT
./scripts/teardown.sh
```

**Ce face `deploy.sh`:**
1. `az group create` pentru resource group
2. `az deployment group create` cu `infra/main.bicep`
3. Creează user-ul SQL pentru Managed Identity (`CREATE USER ... FROM EXTERNAL PROVIDER`), cu acces public temporar sau dintr-un runner din VNet
4. Rulează migrările Alembic
5. Deploy backend (`az webapp deploy`), Function (`func azure functionapp publish`) și frontend (SWA CLI)
6. Afișează URL-urile finale

**Ce face `teardown.sh`:** `az group delete --name rg-taskflow-dev --yes`, plus purge la Key Vault (soft-delete), ca numele să poată fi refolosit.

---

## 12. Cost și managementul creditului

| Resursă | Cost aproximativ |
|---|---|
| App Service B1 | ~13 $/lună (se plătește pe oră) |
| Private Endpoint | ~7 $/lună + trafic |
| SQL Serverless (auto-pause) | câțiva $, doar cât rulează |
| Service Bus Basic, Functions Consumption, SWA Free, Key Vault | ~0 la volumul ăsta |
| ACS Email | cenți |

Prețurile sunt orientative și trebuie verificate în Azure Pricing Calculator.

**Reguli:**
- Resursele trăiesc **doar cât lucrez**: `deploy.sh` → lucru → `teardown.sh`
- **Budget alert** pe abonament (ex. 10 $) setat de la început
- Fallback dacă se termină creditul: Azure Free Account

---

## 13. Plan de lucru (noiembrie 2026 – ianuarie 2027)

| Iterație | Perioadă | Livrabil |
|---|---|---|
| **0. Setup** | 16–22 nov | Repo, Entra app registrations + roluri, budget alert, schelet Bicep (RG + App Service) |
| **1. API + DB local** | 23–29 nov | FastAPI + SQLAlchemy + Alembic local (SQL în Docker), CRUD proiecte/task-uri, validare JWT |
| **2. Cloud core** | 30 nov – 6 dec | Bicep: VNet, SQL privat + Private Endpoint, App Service cu VNet integration, Managed Identity. API rulând în cloud. |
| **3. Frontend + auth** | 7–13 dec | React + MSAL, login, liste/filtre, roluri vizibile în UI, Static Web App deploy |
| **4. Coadă + email** | 14–20 dec | Service Bus, Function, ACS Email, mesaje `TaskAssigned` + remindere programate |
| **5. Atașamente + hardening** | 4–10 ian | Blob + SAS, Key Vault, App Insights, CORS, dead-letter |
| **6. Finalizare** | 11–17 ian | `deploy.sh` testat end-to-end de la zero, README final, diagramă, demo |

Fiecare iterație se încheie cu `teardown.sh`.

---

## 14. Riscuri

| Risc | Mitigare |
|---|---|
| Tenant-ul universității nu permite app registrations / app roles | Folosesc tenant-ul propriu (Default Directory al abonamentului) |
| Crearea user-ului SQL pentru MI cere acces la DB, dar DB-ul e privat | Acces public deschis temporar pe IP-ul propriu în `deploy.sh`, apoi închis, sau script rulat din VNet |
| Private Endpoint + DNS greu de depanat | Fallback temporar: firewall SQL permis doar din subnet-ul App Service (service endpoint) |
| Domeniul de email ACS are limite de trimitere | Domeniu gestionat Azure e suficient pentru demo |
| Se consumă creditul | Teardown disciplinat + budget alert + SQL serverless cu auto-pause |

---

## 15. Checklist

- [ ] Entra ID: app registrations API + SPA, app roles `Admin` / `Member`
- [ ] Budget alert pe abonament
- [ ] Bicep: rețea, SQL privat, App Service, Storage, Service Bus, Function, ACS, Key Vault, SWA, monitoring
- [ ] `deploy.sh` / `teardown.sh` funcționale de la zero
- [ ] Backend: auth JWT, autorizare pe rol + resursă, CRUD complet, migrări
- [ ] Frontend: login MSAL, proiecte, task-uri, comentarii, atașamente
- [ ] Notificări: asignare + reminder deadline, dead-letter
- [ ] Verificare: SQL inaccesibil din internet (test din afara VNet)
- [ ] Demo + README final
