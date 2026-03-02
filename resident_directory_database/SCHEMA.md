# Resident Directory Database Schema (PostgreSQL)

This document describes the **applied** PostgreSQL schema for:
- residents
- users / roles (RBAC)
- resident privacy preferences
- audit logs

> Applied via `psql` using `db_connection.txt` (one statement at a time).

## Extensions
- `citext` (case-insensitive text), used for `email` and `roles.name`.

## Tables

### `roles`
Role definitions for RBAC.

Columns:
- `id` BIGSERIAL PK
- `name` CITEXT NOT NULL UNIQUE
- `description` TEXT NULL
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()

Seeded roles:
- `admin`
- `resident`
- `manager`

### `users`
Authentication users.

Columns:
- `id` BIGSERIAL PK
- `email` CITEXT NOT NULL UNIQUE
- `password_hash` TEXT NOT NULL
- `is_active` BOOLEAN NOT NULL DEFAULT TRUE
- `display_name` TEXT NULL
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `last_login_at` TIMESTAMPTZ NULL

Constraints:
- `users_email_format` CHECK (position('@' in email) > 1)

Indexes:
- `idx_users_email` on `(email)` (redundant with UNIQUE but kept explicitly for clarity)

### `user_roles`
Many-to-many mapping between `users` and `roles`.

Columns:
- `user_id` BIGINT NOT NULL FK → `users(id)` ON DELETE CASCADE
- `role_id` BIGINT NOT NULL FK → `roles(id)` ON DELETE RESTRICT
- `assigned_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `assigned_by` BIGINT NULL FK → `users(id)` ON DELETE SET NULL

Constraints:
- PK (`user_id`, `role_id`)

Indexes:
- `idx_user_roles_role_id` on `(role_id)`

### `residents`
Core resident profile.

Columns:
- `id` BIGSERIAL PK
- `unit_identifier` TEXT NOT NULL
- `first_name` TEXT NOT NULL
- `last_name` TEXT NOT NULL
- `preferred_name` TEXT NULL
- `email` CITEXT NULL
- `phone` TEXT NULL
- `building` TEXT NULL
- `floor` TEXT NULL
- `unit_number` TEXT NULL
- `address_line1` TEXT NULL
- `address_line2` TEXT NULL
- `city` TEXT NULL
- `state` TEXT NULL
- `postal_code` TEXT NULL
- `country` TEXT NULL
- `notes` TEXT NULL
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `created_by` BIGINT NULL FK → `users(id)` ON DELETE SET NULL
- `updated_by` BIGINT NULL FK → `users(id)` ON DELETE SET NULL
- `is_active` BOOLEAN NOT NULL DEFAULT TRUE

Constraints:
- `residents_email_format` CHECK (email IS NULL OR position('@' in email) > 1)
- UNIQUE (`unit_identifier`, `email`)

Indexes:
- `idx_residents_name` on `(last_name, first_name)`
- `idx_residents_unit_identifier` on `(unit_identifier)`
- `idx_residents_email` on `(email)` WHERE email IS NOT NULL

### `resident_privacy_preferences`
1:1 privacy preferences for a resident.

Columns:
- `resident_id` BIGINT PK FK → `residents(id)` ON DELETE CASCADE
- `hide_email` BOOLEAN NOT NULL DEFAULT FALSE
- `hide_phone` BOOLEAN NOT NULL DEFAULT FALSE
- `hide_address` BOOLEAN NOT NULL DEFAULT FALSE
- `hide_profile_from_directory` BOOLEAN NOT NULL DEFAULT FALSE
- `show_preferred_name` BOOLEAN NOT NULL DEFAULT TRUE
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT now()
- `updated_by` BIGINT NULL FK → `users(id)` ON DELETE SET NULL

### `audit_logs`
Immutable audit trail for security and compliance.

Columns:
- `id` BIGSERIAL PK
- `actor_user_id` BIGINT NULL FK → `users(id)` ON DELETE SET NULL
- `actor_email` CITEXT NULL
- `action` TEXT NOT NULL
- `entity_type` TEXT NOT NULL
- `entity_id` TEXT NULL
- `ip_address` INET NULL
- `user_agent` TEXT NULL
- `request_id` TEXT NULL
- `details` JSONB NOT NULL DEFAULT '{}'::jsonb
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT now()

Indexes:
- `idx_audit_logs_created_at` on `(created_at DESC)`
- `idx_audit_logs_entity` on `(entity_type, entity_id)`
- `idx_audit_logs_actor_user_id` on `(actor_user_id)` WHERE actor_user_id IS NOT NULL
