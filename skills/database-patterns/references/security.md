# Database Security

## SQL Injection Prevention

### The Problem

```typescript
// VULNERABLE - Never do this!
const query = `SELECT * FROM users WHERE email = '${email}'`;

// Attack: email = "'; DROP TABLE users; --"
// Results in: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
```

### The Solution: Parameterized Queries

```typescript
// Drizzle (safe by default)
const user = await db
  .select()
  .from(users)
  .where(eq(users.email, email));

// Prisma (safe by default)
const user = await prisma.user.findUnique({
  where: { email },
});

// Raw SQL with parameters
const result = await db.execute(
  sql`SELECT * FROM users WHERE email = ${email}`
);

// PostgreSQL directly
const result = await client.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);
```

### OWASP Prevention Checklist

1. **Use parameterized queries** - Always
2. **Use stored procedures** - With parameterized calls
3. **Validate input** - Whitelist allowed values
4. **Escape user input** - Only as last resort
5. **Least privilege** - App user has minimal permissions
6. **Keep software updated** - Patch vulnerabilities

---

## Password Storage

### Use bcrypt or Argon2

```typescript
import bcrypt from 'bcrypt';

// Hash password
const SALT_ROUNDS = 12;
const passwordHash = await bcrypt.hash(password, SALT_ROUNDS);

// Verify password
const isValid = await bcrypt.compare(password, passwordHash);
```

```typescript
import argon2 from 'argon2';

// Hash password (Argon2id recommended)
const passwordHash = await argon2.hash(password, {
  type: argon2.argon2id,
  memoryCost: 65536,  // 64 MB
  timeCost: 3,
  parallelism: 4,
});

// Verify password
const isValid = await argon2.verify(passwordHash, password);
```

### Never Do

```typescript
// NEVER store plaintext
user.password = password;  // ❌

// NEVER use weak hashing
user.passwordHash = md5(password);   // ❌
user.passwordHash = sha256(password); // ❌

// NEVER use reversible encryption for passwords
user.passwordHash = encrypt(password); // ❌
```

---

## Data Encryption

### Encrypt Sensitive Fields

```typescript
import crypto from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex'); // 32 bytes

function encrypt(text: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, KEY, iv);

  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');

  const authTag = cipher.getAuthTag();

  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

function decrypt(encryptedData: string): string {
  const [ivHex, authTagHex, encrypted] = encryptedData.split(':');

  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');

  const decipher = crypto.createDecipheriv(ALGORITHM, KEY, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}

// Usage
const encryptedSSN = encrypt(ssn);
const decryptedSSN = decrypt(encryptedSSN);
```

### Database-Level Encryption (PostgreSQL)

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt
INSERT INTO sensitive_data (encrypted_value)
VALUES (pgp_sym_encrypt('secret data', 'encryption_key'));

-- Decrypt
SELECT pgp_sym_decrypt(encrypted_value, 'encryption_key')
FROM sensitive_data;
```

---

## Least Privilege Principle

### Create Restricted Database Users

```sql
-- Create application user with minimal permissions
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Grant only necessary permissions
GRANT CONNECT ON DATABASE myapp TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;

-- Table-specific permissions
GRANT SELECT, INSERT, UPDATE ON users TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON posts TO app_user;
GRANT SELECT ON roles TO app_user;  -- Read-only

-- Prevent schema modifications
REVOKE CREATE ON SCHEMA public FROM app_user;

-- Create read-only user for analytics
CREATE USER analytics_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE myapp TO analytics_user;
GRANT USAGE ON SCHEMA public TO analytics_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics_user;
```

### Role-Based Access

```sql
-- Create roles
CREATE ROLE app_read;
CREATE ROLE app_write;
CREATE ROLE app_admin;

-- Grant permissions to roles
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_admin;

-- Assign roles to users
GRANT app_read, app_write TO app_user;
GRANT app_read TO analytics_user;
```

---

## Connection Security

### SSL/TLS Configuration

```typescript
// PostgreSQL with SSL
import postgres from 'postgres';

const sql = postgres(process.env.DATABASE_URL!, {
  ssl: {
    rejectUnauthorized: true,  // Verify server certificate
    ca: fs.readFileSync('/path/to/ca-certificate.crt'),
  },
});

// Prisma
// In DATABASE_URL: postgresql://user:pass@host:5432/db?sslmode=require
```

```sql
-- Force SSL connections (PostgreSQL)
ALTER SYSTEM SET ssl = on;
ALTER SYSTEM SET ssl_cert_file = '/path/to/server.crt';
ALTER SYSTEM SET ssl_key_file = '/path/to/server.key';

-- Require SSL for specific user
ALTER USER app_user SET require_ssl = true;
```

### Connection Pooling with Credentials

```typescript
// Use environment variables (never hardcode)
const pool = postgres({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT!),
  database: process.env.DB_NAME,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  ssl: process.env.DB_SSL === 'true',
  max: 20,
  idle_timeout: 30,
});
```

---

## Audit Logging

### Automatic Audit Trail

```sql
-- Audit log table
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name VARCHAR(100) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by UUID,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address INET,
    user_agent TEXT
);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_func()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, record_id, action, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'INSERT', to_jsonb(NEW), current_setting('app.current_user_id', true)::UUID);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_setting('app.current_user_id', true)::UUID);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, record_id, action, old_data, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, 'DELETE', to_jsonb(OLD), current_setting('app.current_user_id', true)::UUID);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Apply to tables
CREATE TRIGGER users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

### Set User Context in App

```typescript
// Before queries, set current user
await db.execute(sql`
  SELECT set_config('app.current_user_id', ${userId}, false)
`);
```

---

## Data Masking

### Mask Sensitive Data in Logs

```typescript
// Never log sensitive data
console.log(`User ${user.id} logged in`);  // ✅
console.log(`User ${user.email} with password ${password}`);  // ❌

// Mask in error messages
function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  return `${local.slice(0, 2)}***@${domain}`;
}

// maskEmail('john@example.com') => 'jo***@example.com'
```

### Row-Level Security (PostgreSQL)

```sql
-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own documents
CREATE POLICY documents_user_policy ON documents
    FOR ALL
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Policy: Admins can see all
CREATE POLICY documents_admin_policy ON documents
    FOR ALL
    USING (current_setting('app.current_role') = 'admin');
```

---

## Security Checklist

### Development
- [ ] Use parameterized queries only
- [ ] Passwords hashed with bcrypt/argon2
- [ ] Sensitive data encrypted at rest
- [ ] No credentials in code/logs
- [ ] Environment variables for secrets

### Database Configuration
- [ ] SSL/TLS enabled
- [ ] Application user has minimal privileges
- [ ] Row-level security where needed
- [ ] Audit logging enabled
- [ ] Regular security updates applied

### Operations
- [ ] Backups encrypted
- [ ] Access logs monitored
- [ ] Failed login attempts tracked
- [ ] Regular security audits
- [ ] Incident response plan ready
