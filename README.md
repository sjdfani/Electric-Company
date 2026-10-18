# Electric Company

A REST API backend for an electricity distribution company, built with Django and Django REST Framework. Customers register with their phone number, submit damage reports (for example, appliances damaged by a power outage) together with supporting documents, track the status of their claims, and send suggestions. Operators and administrators review and process reports, record payments, and manage users through a panel API.

## Features

- **Phone-number based authentication** with JWT (access and refresh tokens).
- **Three roles**: user (customer), operator (staff) and admin (superuser).
- **Password recovery** via a 6-digit verification code stored in Redis (expires after 6 minutes).
- **Damage reports** with bill, national card, ownership document and any number of additional document uploads.
- **Report tracking** by billing ID and the report's unique ID.
- **Report workflow**: status (TODO, In progress, Done, Failure, Lack of sufficient information), assigned operator, operator notes and payment details (bank, branch, payment code, paid date, payment status).
- **Suggestions** submitted by authenticated users.
- **Management panel** for users, operators, damage reports, damage types, additional documents and suggestions.

## Tech Stack

- Python 3, Django 3.2, Django REST Framework 3.13
- Simple JWT for authentication
- PostgreSQL as the database
- Redis for verification codes
- Docker and Docker Compose

## Project Structure

```
prop_project/   Django project settings and root URL configuration
users/          Custom user model, registration, login, password recovery, profile update
damage/         Damage reports, damage types, additional documents, report tracking
suggestion/     User suggestions
panel/          Management API for operators and admins
```

## Getting Started

### 1. Configure environment variables

Create a `.env` file in the project root:

```env
SECRET_KEY=change-me
DEBUG=True

DATABASE_NAME=electric
DATABASE_USER=electric
DATABASE_PASSWORD=change-me
DATABASE_PORT=5432

REDIS_PASSWORD=change-me
REDIS_PORT=6379

APP_PORT=8000

# Token lifetimes: access token in hours, refresh token in days
ACCESS_TOKEN_LIFETIME=24
REFRESH_TOKEN_LIFETIME=7
```

| Variable | Description |
| --- | --- |
| `SECRET_KEY` | Django secret key |
| `DEBUG` | `True` to enable debug mode, anything else disables it |
| `DATABASE_NAME` | PostgreSQL database name |
| `DATABASE_USER` | PostgreSQL user |
| `DATABASE_PASSWORD` | PostgreSQL password |
| `DATABASE_PORT` | PostgreSQL port inside the Docker network (use `5432`) |
| `REDIS_PASSWORD` | Redis password |
| `REDIS_PORT` | Host port mapped to Redis |
| `APP_PORT` | Host port mapped to the Django app |
| `ACCESS_TOKEN_LIFETIME` | JWT access token lifetime, in hours |
| `REFRESH_TOKEN_LIFETIME` | JWT refresh token lifetime, in days |

> **Note:** The official `postgres` image creates a database named after `POSTGRES_USER` by default. Keep `DATABASE_NAME` equal to `DATABASE_USER`, or create the database manually.

### 2. Start the services

```bash
docker compose up -d --build
```

This starts three containers: `web` (Django), `db` (PostgreSQL) and `redis`.

### 3. Run migrations and create an admin

```bash
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
```

The superuser logs in with a phone number instead of a username.

The API is now available at `http://localhost:<APP_PORT>/` and the Django admin at `http://localhost:<APP_PORT>/admin/`.

## Authentication

Phone numbers are sent in international format, 13 characters long (for example `+989123456789`).

Log in through `POST /api/v1/users/login/` with `phone_number` and `password`. The response contains the user's role (`admin`, `operator` or `user`), the user data and a token pair:

```json
{
  "role": "user",
  "user": { "...": "..." },
  "tokens": {
    "refresh": "<refresh-token>",
    "access": "<access-token>"
  }
}
```

Send the access token on authenticated requests:

```
Authorization: Bearer <access-token>
```

Date and time values in responses are returned as Unix timestamps.

## API Endpoints

### Auth tokens

| Method | Endpoint | Description |
| --- | --- | --- |
| POST | `/api/token/` | Obtain a token pair with `phone_number` and `password` |
| POST | `/api/token/refresh/` | Get a new access token from a refresh token |

### Users — `/api/v1/users/`

| Method | Endpoint | Access | Description |
| --- | --- | --- | --- |
| POST | `register-user/` | Public | Register a customer (`full_name`, `phone_number`, `password`) |
| POST | `register-operator/` | Admin | Register an operator |
| POST | `login/` | Public | Log in and receive role, user data and tokens |
| POST | `forgot-password/` | Public | Send a verification code for `phone_number` |
| POST | `verify-forgot-password/` | Public | Verify the code (`phone_number`, `code`) |
| POST | `confirm-forgot-password/` | Public | Set a new password (`phone_number`, `password`) |
| GET, PUT, PATCH | `update-information/<id>/` | Authenticated | View or update the current user's profile |

Updatable profile fields: `full_name`, `email`, `national_code`, `province`, `city`, `postal_code`, `address`, `card_number`, `account_number`, `shaba_number`.

> The SMS step is not implemented yet: the verification code is printed to the server console.

### Damage reports — `/api/v1/damage/`

| Method | Endpoint | Access | Description |
| --- | --- | --- | --- |
| POST | `create/` | Authenticated | Submit a damage report (multipart form) |
| GET | `type-of-damages/` | Authenticated | List damage types |
| POST | `track/` | Authenticated | Look up a report by `billing_id` and `unique_id` |

Main fields for `create/`: `billing_id`, `type_of_damage`, `description`, `amount_of_damages`, and the optional files `last_bill_image`, `national_card_image` and `ownership_doc_image`. Any other uploaded file is saved as an additional document for the report.

### Suggestions — `/api/v1/suggestion/`

| Method | Endpoint | Access | Description |
| --- | --- | --- | --- |
| POST | `create/` | Authenticated | Submit a suggestion |

### Panel — `/api/v1/panel/`

| Resource | Methods | Access | Notes |
| --- | --- | --- | --- |
| `users/` | List, create, retrieve, update, delete | Operator, Admin | Customers only (non-staff users) |
| `operators/` | List, create, retrieve, update, delete | Admin | |
| `damage-reports/` | List, retrieve, update, delete | Authenticated | Admins see all reports, operators see unassigned reports and their own, users see their own |
| `additional-documents/` | Create, retrieve, delete | Admin | |
| `type-of-damages/` | List, create, retrieve, update, delete | Admin | |
| `suggestions/` | List, retrieve, delete | Authenticated | Admins see all, users see their own |

Routes follow the standard DRF router layout: `GET` and `POST` on `/<resource>/`, and `GET`, `PUT`, `PATCH` and `DELETE` on `/<resource>/<id>/`, limited to the methods listed above.

## Media Files

Uploaded files are stored in `media_root/` and served under `/media/`.
