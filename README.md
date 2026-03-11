# pg-simple-migrate

A simple, standalone PostgreSQL migration tool with zero dependencies (except `pg`).

No ORM required. No complex configuration. Just SQL files and a tracking table.

## Features

- ✅ **Simple** - One JavaScript file, minimal dependencies
- ✅ **SQL-based** - Write pure SQL migration files
- ✅ **Transaction-safe** - Automatic rollback on errors
- ✅ **Idempotent** - Tracks applied migrations, won't re-run them
- ✅ **No ORM** - Works with your existing raw SQL queries
- ✅ **Configurable** - Environment variables for all settings
- ✅ **Zero magic** - Understandable, hackable code

## Installation

### Option 1: Use in your project

```bash
npm install pg-simple-migrate
```

### Option 2: Copy the file

Just copy `migrate.js` to your project and run it with Node.js.

```bash
curl -o migrate.js https://raw.githubusercontent.com/jonathan-ross/pg-simple-migrate/main/migrate.js
npm install pg
```

## Quick Start

### 1. Set environment variables

```bash
export DB_HOST=localhost
export DB_PORT=5432
export DB_NAME=mydb
export DB_USER=postgres
export DB_PASSWORD=password

# Or use a connection string
export DATABASE_URL="postgresql://user:password@localhost:5432/mydb"
```

### 2. Create your first migration

```bash
node migrate.js create add_users_table
```

This creates `migrations/YYYY-MM-DD_add_users_table.sql`

### 3. Edit the migration file

```sql
-- migrations/2026-03-09_add_users_table.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

### 4. Run migrations

```bash
node migrate.js up
```

## Commands

```bash
# Run all pending migrations
node migrate.js up

# Show migration status
node migrate.js status

# Create a new migration file
node migrate.js create <migration_name>

# Rollback last migration (removes record only, doesn't undo SQL)
node migrate.js down

# Show help
node migrate.js help
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `localhost` | Database host |
| `DB_PORT` | `5432` | Database port |
| `DB_NAME` | `postgres` | Database name |
| `DB_USER` | `postgres` | Database user |
| `DB_PASSWORD` | - | Database password |
| `DATABASE_URL` | - | Full connection string (overrides individual vars) |
| `MIGRATIONS_DIR` | `./migrations` | Path to migrations directory |
| `MIGRATIONS_TABLE` | `public.migrations` | Table name for tracking migrations |

## Migration File Format

Migration files should:
- Be named with a date prefix: `YYYY-MM-DD_description.sql`
- Contain valid SQL statements
- Be idempotent when possible (use `IF NOT EXISTS`, etc.)

Example:
```sql
-- migrations/2026-03-09_add_user_phone.sql
ALTER TABLE users
ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

CREATE INDEX IF NOT EXISTS idx_users_phone ON users(phone);
```

## Usage with npm scripts

Add to your `package.json`:

```json
{
  "scripts": {
    "migrate": "node migrate.js up",
    "migrate:status": "node migrate.js status",
    "migrate:create": "node migrate.js create",
    "migrate:down": "node migrate.js down"
  }
}
```

Then run:
```bash
npm run migrate
npm run migrate:status
npm run migrate:create add_feature
```

## AWS Deployment Example

Create a deployment script that fetches credentials from AWS SSM:

```bash
#!/bin/bash
# deploy-migrate.sh

ENV=$1
AWS_PROFILE=$2

# Fetch DB credentials from SSM
export DB_HOST=$(aws ssm get-parameter --name "/${ENV}/db/host" --query 'Parameter.Value' --output text --profile $AWS_PROFILE)
export DB_NAME=$(aws ssm get-parameter --name "/${ENV}/db/name" --query 'Parameter.Value' --output text --profile $AWS_PROFILE)
export DB_USER=$(aws ssm get-parameter --name "/${ENV}/db/user" --query 'Parameter.Value' --output text --profile $AWS_PROFILE)
export DB_PASSWORD=$(aws ssm get-parameter --name "/${ENV}/db/password" --with-decryption --query 'Parameter.Value' --output text --profile $AWS_PROFILE)

# Run migrations
node migrate.js up
```

## CI/CD Integration

### GitHub Actions

```yaml
- name: Run Database Migrations
  env:
    DB_HOST: ${{ secrets.DB_HOST }}
    DB_USER: ${{ secrets.DB_USER }}
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    DB_NAME: ${{ secrets.DB_NAME }}
  run: |
    npm install
    npm run migrate
```

### GitLab CI

```yaml
migrate:
  stage: deploy
  script:
    - npm install
    - npm run migrate
  variables:
    DB_HOST: $DB_HOST
    DB_USER: $DB_USER
    DB_PASSWORD: $DB_PASSWORD
    DB_NAME: $DB_NAME
```

## How It Works

1. Creates a `migrations` table (default: `public.migrations`) to track applied migrations
2. Scans the migrations directory for `.sql` files
3. Compares files against the migrations table to find pending migrations
4. Runs each pending migration in a transaction
5. Records successful migrations in the tracking table

## Migration Table

The tool automatically creates this table:

```sql
CREATE TABLE public.migrations (
  name VARCHAR(255) PRIMARY KEY,
  created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

You can customize the table name and schema with the `MIGRATIONS_TABLE` environment variable:

```bash
export MIGRATIONS_TABLE=myschema.my_migrations
```

## Transaction Safety

Each migration runs in a transaction:
- If the migration succeeds, it's committed and recorded
- If the migration fails, it's rolled back and nothing is recorded
- You can safely re-run migrations after fixing errors

## Rollbacks

The `down` command removes the last migration record but **does not** automatically undo SQL changes.

```bash
node migrate.js down
```

⚠️ **Warning**: You must manually write and execute SQL to revert schema changes.

For safer rollbacks, consider:
1. Writing reversible migrations (separate up/down files)
2. Taking database backups before migrations
3. Testing migrations in dev/staging first

## Best Practices

1. **Test locally first** - Always test migrations in development before production
2. **One change per file** - Keep migrations focused and atomic
3. **Use descriptive names** - `add_user_email_column` not `update_users`
4. **Make migrations idempotent** - Use `IF NOT EXISTS`, `IF EXISTS`, etc.
5. **Never edit applied migrations** - Create a new migration instead
6. **Backup before big changes** - Especially in production
7. **Review migrations in PRs** - Treat schema changes carefully

## Examples

### Creating a table
```sql
-- migrations/2026-03-09_create_posts_table.sql
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

### Adding a column
```sql
-- migrations/2026-03-10_add_user_avatar.sql
ALTER TABLE users
ADD COLUMN avatar_url VARCHAR(500);
```

### Data migration
```sql
-- migrations/2026-03-11_migrate_user_status.sql
-- Migrate old status integers to new status strings

ALTER TABLE users ADD COLUMN status_new VARCHAR(20);

UPDATE users SET status_new = 'active' WHERE status = 1;
UPDATE users SET status_new = 'inactive' WHERE status = 2;
UPDATE users SET status_new = 'banned' WHERE status = 3;

ALTER TABLE users DROP COLUMN status;
ALTER TABLE users RENAME COLUMN status_new TO status;
```

### Complex migration with multiple statements
```sql
-- migrations/2026-03-12_refactor_orders.sql
BEGIN;

-- Create new table
CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER REFERENCES products(id),
  quantity INTEGER NOT NULL,
  price DECIMAL(10,2) NOT NULL
);

-- Migrate data from old structure
INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT id, product_id, quantity, price
FROM orders;

-- Drop old columns
ALTER TABLE orders DROP COLUMN product_id;
ALTER TABLE orders DROP COLUMN quantity;
ALTER TABLE orders DROP COLUMN price;

COMMIT;
```

## Troubleshooting

### Migration fails mid-execution

The transaction is automatically rolled back. Fix the SQL and run again:

```bash
# Fix the SQL file, then:
node migrate.js up
```

### Need to skip a migration

Manually insert a record:

```sql
INSERT INTO public.migrations (name, created_at)
VALUES ('2026-03-09_skip_this.sql', CURRENT_TIMESTAMP);
```

### Reset all migrations (dev only!)

⚠️ **Destructive operation** - Only do this in development:

```sql
DROP TABLE public.migrations;
```

Then run migrations again to reapply everything.

### Check database connection

```bash
# Test connection
psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "SELECT version();"
```

## Comparison with Other Tools

| Feature | pg-simple-migrate | node-pg-migrate | Flyway | Knex |
|---------|------------------|-----------------|--------|------|
| Dependencies | 1 (pg) | Many | Java | Many |
| SQL files | ✅ | ✅ | ✅ | Via raw |
| JavaScript API | ❌ | ✅ | ❌ | ✅ |
| Query builder | ❌ | ❌ | ❌ | ✅ |
| File size | ~5KB | ~500KB | ~20MB | ~1MB |
| Learning curve | Minimal | Low | Medium | Medium |
| Hackable | ✅ | ⚠️ | ❌ | ⚠️ |

## Why Use This?

Use `pg-simple-migrate` if you:
- Want a simple, understandable migration tool
- Don't want to learn a complex migration framework
- Prefer writing pure SQL
- Need something you can easily modify
- Don't want heavy dependencies

Use something else if you:
- Need complex JavaScript migrations
- Want a query builder (use Knex or Kysely)
- Need programmatic rollbacks
- Want an ORM (use TypeORM, Prisma, etc.)

## License

MIT

## Contributing

Issues and PRs welcome!

## Author

Created for simple, ORM-free PostgreSQL projects that just need migration tracking.
