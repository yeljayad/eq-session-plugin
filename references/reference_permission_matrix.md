---
name: Permission Matrix
description: Complete RBAC permission matrix — 9 roles x resources with CRUD/action permissions (source: docs/Users_matrix_21June - matrix.csv)
type: reference
---

## Roles (9 total)

- **Owner** — full access to everything
- **workspace_admin** — full workspace + downstream
- **workspace_user** — read workspace, limited downstream
- **space_admin** — full space + downstream
- **space_user** — read space, limited downstream
- **retailer_admin** — full retailer + devices
- **retailer_user** — read retailer, limited
- **customer_admin** — full customer + cards + accounts
- **customer_user** — limited customer actions (cards, devices)

## Resource Permissions

| Resource | Owner | WS Admin | WS User | Space Admin | Space User | Ret Admin | Ret User | Cust Admin | Cust User |
|---|---|---|---|---|---|---|---|---|---|
| Workspace CRUD | CRUD | R | - | - | - | - | - | - | - |
| Space CRUD | CRUD | CRUD | R | R | - | - | - | - | - |
| Retailer CRUD | CRUD | CRUD | CRUD | CRUD | R | R | - | - | - |
| Account CRUD | CRUD | CRUD | CRUD | CRUD | CRUD | R | CRUD | R | - |
| Customer CRUD | CRUD | CRUD | CRUD | CRUD | - | - | R | - | - |
| Card CRUD | R | R | R | R | - | - | CRUD | CRUD | - |
| Device CRUD | R | R | R | R | CRUD | R | R | R | - |
| User CRUD | CRUD | CRUD | CRUD | CRUD | CRUD | R | CRUD | - | - |
| Transaction | R | R | R | R | R | R | R | R | R |

## Key patterns
- Permissions cascade downward: owner > workspace > space > retailer > customer
- "ok" in CSV = permission granted, empty = denied
- Some entries marked "a supprimer" (to remove) or "a implementer" (to implement)
- Card operations are primarily customer_admin/customer_user scope
- Device operations are primarily retailer_admin scope
- Transaction is read-only for all roles

Source file: `docs/Users_matrix_21June - matrix.csv`
