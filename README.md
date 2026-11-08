# ClassroomHub

Proyecto Django de los Laboratorios 6 a 9: catálogo de recursos de aprendizaje guardado en MySQL/MariaDB, con registro, login, panel privado, una página de revisión protegida por permisos y un CRUD de recursos (detalle, crear, editar y eliminar) protegido por permisos.

## Requisitos

- Python 3.12 (Django 5.2 LTS admite Python 3.10 a 3.13)
- MariaDB 10.11 (o MySQL 8) encendido
- Las dependencias están en `requirements.txt` (Django 5.2.19, mysqlclient, python-dotenv)

## Puesta en marcha

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env      # y escribe tu contraseña real en DB_PASSWORD
```

Crea la base y la cuenta SQL una sola vez con `mysql -u root -p`:

```sql
CREATE DATABASE classroom_hub CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'classroomhub_user'@'localhost' IDENTIFIED BY 'TU_CONTRASEÑA';
GRANT ALL PRIVILEGES ON classroom_hub.* TO 'classroomhub_user'@'localhost';
GRANT ALL PRIVILEGES ON test_classroom_hub.* TO 'classroomhub_user'@'localhost';
FLUSH PRIVILEGES;
```

Después:

```powershell
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

## Rutas

| Ruta | Acceso | Descripción |
|---|---|---|
| `/` | Público | Inicio |
| `/resources/` | Público | Catálogo de recursos (viene de la base de datos). Búsqueda de solo lectura por título: `/resources/?q=django` |
| `/resources/<pk>/` | Público | Detalle de un recurso (404 si no existe) |
| `/resources/new/` | Permiso `pages.add_resource` | Crear recurso (GET formulario, POST guarda) |
| `/resources/<pk>/edit/` | Permiso `pages.change_resource` | Editar recurso (misma PK, mismo `ResourceForm`) |
| `/resources/<pk>/delete/` | Permiso `pages.delete_resource` | GET muestra la confirmación; **solo el POST elimina** |
| `/about/` | Público | Acerca de |
| `/accounts/signup/` | Público | Registro de cuentas comunes (sin privilegios) |
| `/accounts/login/` | Público | Inicio de sesión |
| `/accounts/logout/` | Solo POST | Cierre de sesión (desde el botón del menú) |
| `/dashboard/` | Requiere sesión | Panel privado con contador de visitas de la sesión |
| `/dashboard/summary/` | Requiere sesión | Resumen de solo lectura: total de recursos y disponibles |
| `/resources/review/` | Requiere permiso `pages.view_resource` | Revisión del catálogo (HTTP 403 si falta el permiso) |
| `/admin/` | Solo `is_staff` | Administración de Django |

## Cuentas y grupos

1. Crea una cuenta en `/accounts/signup/` o desde `/admin/`.
2. Entra a `/admin/` con el superusuario y abre **Groups → Add group**.
3. Crea el grupo **Catalog reviewers** y asígnale solo el permiso `pages | resource | Can view resource`.
4. Abre la cuenta en **Users** y agrégala al grupo. No marques *Staff status*: el revisor no necesita entrar al administrador.

Resultado esperado en `/resources/review/`: anónimo → redirige al login; cuenta común → 403; miembro del grupo → 200.

### Categorías de ejemplo y grupo editor (CRUD)

Las categorías se gestionan solo desde Django Admin (el CRUD público es únicamente de recursos).

1. En `/admin/` abre **Categories → Add** y crea, por ejemplo, `Programming` y `Databases`.
2. Abre **Groups → Add group** y crea **Catalog editors** con estos cuatro permisos de `pages | resource`: *Can view*, *Can add*, *Can change* y *Can delete resource*.
3. En **Users** abre una **cuenta de demostración** y agrégala a **Catalog editors**. No marques *Staff status*: el CRUD usa permisos, no `is_staff`.

| Operación | Anónimo | Cuenta común | Catalog reviewers | Catalog editors |
|---|---|---|---|---|
| Ver lista y detalle | Sí | Sí | Sí | Sí |
| Revisar (`/resources/review/`) | No | No | Sí | Sí |
| Crear / editar / eliminar | No | No | No | Sí |

**Los permisos nunca se asignan mediante el formulario público de registro**: `/accounts/signup/` solo crea cuentas sin grupos ni privilegios; los grupos los asigna una cuenta administradora desde Admin.

Anónimo → redirige al login; autenticado sin permiso → 403.

### Rutas del CRUD

```text
GET  /resources/                 GET  /resources/<pk>/
GET  /resources/new/             POST /resources/new/
GET  /resources/<pk>/edit/       POST /resources/<pk>/edit/
GET  /resources/<pk>/delete/     POST /resources/<pk>/delete/
```

## Pruebas

```powershell
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py test pages
```

Hay 68 pruebas (10 del catálogo, 12 de autenticación, 6 de permisos de revisión, 5 del resumen, 30 del CRUD y 5 de la búsqueda). Se ejecutan sobre la base descartable `test_classroom_hub`, que Django crea y destruye; no tocan los datos de `classroom_hub`. Estas pruebas cubren los flujos implementados, no son una auditoría de seguridad.

## Seguridad

- `.env` y `.venv/` están en `.gitignore`: no se suben a GitHub. Solo se versiona `.env.example`.
- `SECRET_KEY` y `DEBUG = True` son de desarrollo; cámbialos antes de publicar el proyecto.
- No compartas contraseñas, cookies de sesión ni exportaciones SQL con datos reales.
