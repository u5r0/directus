# Authentication & Security

This document covers authentication mechanisms, authorization systems, and security features in Directus.

## Overview

Directus implements a comprehensive security system with:
- Multiple authentication providers (Local, OAuth2, OpenID, LDAP, SAML)
- JWT-based token authentication
- Policy-based access control
- Role hierarchy with inheritance
- Rate limiting and IP filtering
- Two-factor authentication (TFA)

## Authentication System

### Auth Drivers (`api/src/auth/drivers/`)

Directus uses an extensible driver system for authentication:

```typescript
export abstract class AuthDriver {
  /**
   * Get user ID from provider payload
   */
  abstract getUserID(payload: Record<string, any>): Promise<string>;
  
  /**
   * Verify user credentials
   */
  abstract verify(user: User, password?: string): Promise<void>;
  
  /**
   * Handle login
   */
  async login(user: User, payload: Record<string, any>): Promise<void>;
  
  /**
   * Handle token refresh
   */
  async refresh(user: User): Promise<void>;
  
  /**
   * Handle logout
   */
  async logout(user: User): Promise<void>;
}
```

### Supported Auth Providers

#### 1. Local Authentication (`local.ts`)

Password-based authentication with Argon2 hashing:

```typescript
export class LocalAuthDriver extends AuthDriver {
  async getUserID(payload: Record<string, any>): Promise<string> {
    const user = await this.knex
      .select('id')
      .from('directus_users')
      .whereRaw('LOWER(??) = ?', ['email', payload['email'].toLowerCase()])
      .first();
    
    if (!user) throw new InvalidCredentialsError();
    return user.id;
  }
  
  async verify(user: User, password?: string): Promise<void> {
    if (!user.password || !(await argon2.verify(user.password, password))) {
      throw new InvalidCredentialsError();
    }
  }
}
```

**Configuration:**
```bash
# Local auth is enabled by default
AUTH_DISABLE_DEFAULT=false
```

#### 2. OAuth2 Authentication (`oauth2.ts`)

Generic OAuth 2.0 provider support:

```bash
AUTH_PROVIDERS=google,github

# Google OAuth2
AUTH_GOOGLE_DRIVER=oauth2
AUTH_GOOGLE_CLIENT_ID=your-client-id
AUTH_GOOGLE_CLIENT_SECRET=your-client-secret
AUTH_GOOGLE_AUTHORIZE_URL=https://accounts.google.com/o/oauth2/v2/auth
AUTH_GOOGLE_ACCESS_URL=https://oauth2.googleapis.com/token
AUTH_GOOGLE_PROFILE_URL=https://www.googleapis.com/oauth2/v1/userinfo
```

#### 3. OpenID Connect (`openid.ts`)

OpenID Connect (OIDC) authentication:

```bash
AUTH_PROVIDERS=keycloak

AUTH_KEYCLOAK_DRIVER=openid
AUTH_KEYCLOAK_CLIENT_ID=directus
AUTH_KEYCLOAK_CLIENT_SECRET=your-secret
AUTH_KEYCLOAK_ISSUER_URL=https://keycloak.example.com/realms/master
AUTH_KEYCLOAK_IDENTIFIER_KEY=email
AUTH_KEYCLOAK_ALLOW_PUBLIC_REGISTRATION=true
AUTH_KEYCLOAK_DEFAULT_ROLE_ID=role-uuid
```

#### 4. LDAP Authentication (`ldap.ts`)

Active Directory / LDAP integration:

```bash
AUTH_PROVIDERS=ldap

AUTH_LDAP_DRIVER=ldap
AUTH_LDAP_CLIENT_URL=ldap://ldap.example.com
AUTH_LDAP_BIND_DN=cn=admin,dc=example,dc=com
AUTH_LDAP_BIND_PASSWORD=password
AUTH_LDAP_USER_DN=ou=users,dc=example,dc=com
AUTH_LDAP_USER_ATTRIBUTE=uid
AUTH_LDAP_MAIL_ATTRIBUTE=mail
AUTH_LDAP_FIRST_NAME_ATTRIBUTE=givenName
AUTH_LDAP_LAST_NAME_ATTRIBUTE=sn
```

#### 5. SAML Authentication (`saml.ts`)

SAML 2.0 SSO integration:

```bash
AUTH_PROVIDERS=saml

AUTH_SAML_DRIVER=saml
AUTH_SAML_SP_ENTITY_ID=https://directus.example.com
AUTH_SAML_SP_ACS_URL=https://directus.example.com/auth/login/saml/acs
AUTH_SAML_SP_SLS_URL=https://directus.example.com/auth/login/saml/sls
AUTH_SAML_IDP_ENTITY_ID=https://idp.example.com
AUTH_SAML_IDP_SSO_URL=https://idp.example.com/sso
AUTH_SAML_IDP_SLS_URL=https://idp.example.com/sls
AUTH_SAML_IDP_CERT=certificate-content
```

### Provider Registration (`api/src/auth.ts`)

```typescript
export async function registerAuthProviders(): Promise<void> {
  const env = useEnv();
  const providerNames = toArray(env['AUTH_PROVIDERS']);
  
  // Register default local provider
  if (!env['AUTH_DISABLE_DEFAULT']) {
    const defaultProvider = new LocalAuthDriver(options);
    providers.set(DEFAULT_AUTH_PROVIDER, defaultProvider);
  }
  
  // Register configured providers
  providerNames.forEach((name) => {
    const { driver, ...config } = getConfigFromEnv(`AUTH_${name.toUpperCase()}_`);
    const provider = getProviderInstance(driver, options, config);
    providers.set(name, provider);
  });
}
```

## JWT Token System

### Token Types

**Access Token** - Short-lived token for API requests
```typescript
interface DirectusTokenPayload {
  id: string;           // User ID
  role: string;         // Primary role ID
  app_access: boolean;  // Can access admin app
  admin_access: boolean;// Has admin privileges
  iat: number;          // Issued at
  exp: number;          // Expires at
  iss: string;          // Issuer
}
```

**Refresh Token** - Long-lived token for obtaining new access tokens
```typescript
interface RefreshTokenPayload {
  id: string;           // User ID
  session: string;      // Session ID
  iat: number;
  exp: number;
  iss: string;
}
```

### Token Configuration

```bash
# Secret key (min 32 characters)
SECRET=your-secret-key-min-32-bytes

# Token expiration
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL=7d
REFRESH_TOKEN_COOKIE_SECURE=true
REFRESH_TOKEN_COOKIE_SAME_SITE=lax
REFRESH_TOKEN_COOKIE_NAME=directus_refresh_token

# Session cookie
SESSION_COOKIE_TTL=1d
SESSION_COOKIE_SECURE=true
SESSION_COOKIE_SAME_SITE=lax
SESSION_COOKIE_NAME=directus_session_token
```

### Token Extraction (`api/src/middleware/extract-token.ts`)

Tokens can be provided via:

1. **Authorization Header**
```http
Authorization: Bearer <token>
```

2. **Query Parameter**
```http
GET /items/articles?access_token=<token>
```

3. **Session Cookie**
```http
Cookie: directus_session_token=<token>
```

```typescript
const extractToken: RequestHandler = (req, _res, next) => {
  let token: string | null = null;
  
  // Check query parameter
  if (req.query?.['access_token']) {
    token = req.query['access_token'] as string;
  }
  
  // Check Authorization header
  if (req.headers?.authorization) {
    const parts = req.headers.authorization.split(' ');
    if (parts.length === 2 && parts[0].toLowerCase() === 'bearer') {
      if (token !== null) {
        // RFC6750 compliance - only one method allowed
        throw new InvalidPayloadError();
      }
      token = parts[1];
    }
  }
  
  // Check session cookie (doesn't conflict with other methods)
  if (req.cookies?.[env['SESSION_COOKIE_NAME']]) {
    if (token === null) {
      token = req.cookies[env['SESSION_COOKIE_NAME']];
    }
  }
  
  req.token = token;
  next();
};
```

### Authentication Middleware (`api/src/middleware/authenticate.ts`)

```typescript
export const handler = async (req: Request, res: Response, next: NextFunction) => {
  // Create default accountability
  const defaultAccountability: Accountability = createDefaultAccountability({
    ip: getIPFromReq(req),
    userAgent: req.get('user-agent'),
    origin: req.get('origin'),
  });
  
  // Allow custom authentication via filter hook
  const customAccountability = await emitter.emitFilter(
    'authenticate',
    defaultAccountability,
    { req },
    { database, schema: null, accountability: null }
  );
  
  if (customAccountability && !isEqual(customAccountability, defaultAccountability)) {
    req.accountability = customAccountability;
    return next();
  }
  
  // Get accountability from token
  try {
    req.accountability = await getAccountabilityForToken(
      req.token,
      defaultAccountability
    );
  } catch (err) {
    // Clear invalid session cookie
    if (req.cookies[env['SESSION_COOKIE_NAME']] === req.token) {
      res.clearCookie(env['SESSION_COOKIE_NAME'], SESSION_COOKIE_OPTIONS);
    }
    throw err;
  }
  
  return next();
};
```

## Authorization System

### Policy-Based Access Control

Directus uses a policy-based system with role hierarchy:

```
User → Access → Policy → Permissions
       ↓
     Role → Access → Policy → Permissions
```

### Accountability Object

```typescript
interface Accountability {
  user: string | null;          // User ID
  role: string | null;          // Primary role ID
  roles: string[];              // All role IDs (including inherited)
  admin: boolean;               // Admin flag
  app: boolean;                 // App access flag
  ip: string | null;            // IP address
  userAgent: string | null;     // User agent
  origin: string | null;        // Request origin
  share: string | null;         // Share ID (for public shares)
  share_scope: any | null;      // Share scope
}
```

### Fetching Policies (`api/src/permissions/lib/fetch-policies.ts`)

```typescript
export async function fetchPolicies(
  { roles, user, ip }: Pick<Accountability, 'user' | 'roles' | 'ip'>,
  context: Context
): Promise<string[]> {
  const accessService = new AccessService(context);
  
  // Build filter for role-based or user-based policies
  let roleFilter: Filter;
  if (roles.length === 0) {
    // Public role
    roleFilter = { _and: [{ role: { _null: true } }, { user: { _null: true } }] };
  } else {
    roleFilter = { role: { _in: roles } };
  }
  
  // Include user-specific policies
  const filter = user
    ? { _or: [{ user: { _eq: user } }, roleFilter] }
    : roleFilter;
  
  const accessRows = await accessService.readByQuery({
    filter,
    fields: ['policy.id', 'policy.ip_access', 'role'],
    limit: -1,
  });
  
  // Filter by IP address
  const filteredAccessRows = filterPoliciesByIp(accessRows, ip);
  
  // Sort by priority: parent roles → child roles → user policies
  filteredAccessRows.sort((a, b) => {
    if (!a.role && !b.role) return 0;
    if (!a.role) return 1;
    if (!b.role) return -1;
    return roles.indexOf(a.role) - roles.indexOf(b.role);
  });
  
  return filteredAccessRows.map(({ policy }) => policy.id);
}
```

### Fetching Permissions (`api/src/permissions/lib/fetch-permissions.ts`)

```typescript
export async function fetchPermissions(
  options: FetchPermissionsOptions,
  context: Context
) {
  // Fetch raw permissions from database
  const permissions = await fetchRawPermissions(options, context);
  
  if (options.accountability && !options.bypassDynamicVariableProcessing) {
    // Extract dynamic variables ($CURRENT_USER, $NOW, etc.)
    const dynamicVariableContext = extractRequiredDynamicVariableContextForPermissions(permissions);
    
    // Fetch data for dynamic variables
    const permissionsContext = await fetchDynamicVariableData({
      accountability: options.accountability,
      policies: options.policies,
      dynamicVariableContext,
    }, context);
    
    // Replace dynamic variables with actual values
    return processPermissions({
      permissions,
      accountability: options.accountability,
      permissionsContext,
    });
  }
  
  return permissions;
}
```

### Permission Structure

```typescript
interface Permission {
  id: string;
  policy: string;
  collection: string;
  action: 'create' | 'read' | 'update' | 'delete' | 'share';
  permissions: Filter | null;  // Row-level filter
  validation: Filter | null;   // Validation rules
  presets: Record<string, any> | null;  // Default values
  fields: string[] | null;     // Allowed fields
}
```

### Dynamic Variables

Permissions support dynamic variables that are resolved at runtime:

```typescript
// User context
$CURRENT_USER          // Current user ID
$CURRENT_USER.email    // Current user's email
$CURRENT_USER.role     // Current user's role
$CURRENT_ROLE          // Current role ID

// Time context
$NOW                   // Current timestamp
$NOW(+1 day)          // Relative time

// Request context
$REQ.ip               // Request IP
$REQ.userAgent        // User agent
```

## Rate Limiting

### Global Rate Limiter (`api/src/middleware/rate-limiter-global.ts`)

Limits total requests across all IPs:

```bash
RATE_LIMITER_ENABLED=true
RATE_LIMITER_STORE=memory  # or redis

# Global rate limiter
RATE_LIMITER_GLOBAL_ENABLED=true
RATE_LIMITER_GLOBAL_POINTS=1000
RATE_LIMITER_GLOBAL_DURATION=1
```

```typescript
export let rateLimiterGlobal: RateLimiterRedis | RateLimiterMemory;

if (env['RATE_LIMITER_GLOBAL_ENABLED']) {
  rateLimiterGlobal = createRateLimiter('RATE_LIMITER_GLOBAL');
  
  checkRateLimit = asyncHandler(async (_req, res, next) => {
    try {
      await rateLimiterGlobal.consume(RATE_LIMITER_GLOBAL_KEY, 1);
    } catch (rateLimiterRes) {
      res.set('Retry-After', String(Math.round(rateLimiterRes.msBeforeNext / 1000)));
      throw new HitRateLimitError({
        limit: env['RATE_LIMITER_GLOBAL_POINTS'],
        reset: new Date(Date.now() + rateLimiterRes.msBeforeNext),
      });
    }
    next();
  });
}
```

### IP-Based Rate Limiter (`api/src/middleware/rate-limiter-ip.ts`)

Limits requests per IP address:

```bash
RATE_LIMITER_ENABLED=true
RATE_LIMITER_POINTS=50
RATE_LIMITER_DURATION=1
```

### Login Rate Limiter

Prevents brute-force attacks:

```bash
RATE_LIMITER_ENABLED=true
RATE_LIMITER_DURATION=0  # No time window
RATE_LIMITER_POINTS=25   # Max failed attempts
```

### Rate Limiter Configuration (`api/src/rate-limiter.ts`)

```typescript
export function createRateLimiter(
  configPrefix = 'RATE_LIMITER',
  configOverrides?: IRateLimiterOptionsOverrides
): RateLimiterAbstract {
  const env = useEnv();
  
  switch (env['RATE_LIMITER_STORE']) {
    case 'redis':
      return new RateLimiterRedis(getConfig('redis', configPrefix, configOverrides));
    case 'memory':
    default:
      return new RateLimiterMemory(getConfig('memory', configPrefix, configOverrides));
  }
}
```

## Two-Factor Authentication (TFA)

### TFA Setup

```typescript
// Enable TFA for user
const tfaService = new TFAService({ schema, knex: database });

// Generate TFA secret
const { url, secret } = await tfaService.generateTFA(userId);

// Enable TFA with verification
await tfaService.enableTFA(userId, secret, otp);

// Disable TFA
await tfaService.disableTFA(userId);
```

### TFA Login Flow

```typescript
// 1. Login with credentials
const result = await authService.login(provider, payload);

// 2. If TFA enabled, require OTP
if (user.tfa_secret) {
  // Verify OTP
  await tfaService.verifyOTP(userId, otp);
}

// 3. Generate tokens
const tokens = await authService.login(provider, payload, { otp });
```

## Security Features

### Password Hashing

Uses Argon2 for password hashing:

```typescript
import argon2 from 'argon2';

// Hash password
const hash = await argon2.hash(password);

// Verify password
const valid = await argon2.verify(hash, password);
```

### CORS Configuration

```bash
CORS_ENABLED=true
CORS_ORIGIN=true  # or specific origins
CORS_METHODS=GET,POST,PATCH,DELETE
CORS_ALLOWED_HEADERS=Content-Type,Authorization
CORS_EXPOSED_HEADERS=Content-Range
CORS_CREDENTIALS=true
CORS_MAX_AGE=18000
```

### IP Filtering

Policies can restrict access by IP:

```typescript
interface Policy {
  id: string;
  name: string;
  ip_access: string[] | null;  // CIDR notation
  // ...
}

// Example: Allow only from specific IPs
ip_access: ['192.168.1.0/24', '10.0.0.1']
```

### Security Headers

Helmet.js integration for security headers:

```typescript
// Automatically applied headers:
- X-DNS-Prefetch-Control
- X-Frame-Options
- X-Content-Type-Options
- X-XSS-Protection
- Strict-Transport-Security
- Content-Security-Policy
```

### Query Sanitization

Prevents SQL injection and invalid queries:

```typescript
export async function sanitizeQuery(
  rawQuery: Query,
  schema: SchemaOverview,
  accountability: Accountability | null
): Promise<Query> {
  // Validate field names
  // Sanitize filter values
  // Check permissions
  // Remove invalid operators
  return sanitizedQuery;
}
```

### Input Validation

Uses Joi for input validation:

```typescript
import Joi from 'joi';

const schema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
});

const { error, value } = schema.validate(input);
```

## Authentication Service

### Login Flow (`api/src/services/authentication.ts`)

```typescript
export class AuthenticationService {
  async login(
    providerName: string,
    payload: Record<string, any>,
    options?: { otp?: string; session?: boolean }
  ): Promise<LoginResult> {
    const STALL_TIME = env['LOGIN_STALL_TIME'];
    const timeStart = performance.now();
    
    // Get auth provider
    const provider = getAuthProvider(providerName);
    
    // Get user ID from provider
    let userId;
    try {
      userId = await provider.getUserID(payload);
    } catch (err) {
      await stall(STALL_TIME, timeStart);  // Prevent timing attacks
      throw err;
    }
    
    // Load user from database
    const user = await this.knex
      .select('*')
      .from('directus_users')
      .where({ id: userId })
      .first();
    
    // Check user status
    if (user.status !== 'active') {
      throw new UserSuspendedError();
    }
    
    // Verify credentials
    await provider.verify(user, payload['password']);
    
    // Check TFA
    if (user.tfa_secret && !options?.otp) {
      throw new InvalidOtpError();
    }
    
    if (user.tfa_secret) {
      await tfaService.verifyOTP(userId, options.otp);
    }
    
    // Generate tokens
    const accessToken = jwt.sign(tokenPayload, getSecret(), {
      expiresIn: env['ACCESS_TOKEN_TTL'],
      issuer: 'directus',
    });
    
    const refreshToken = jwt.sign(refreshPayload, getSecret(), {
      expiresIn: env['REFRESH_TOKEN_TTL'],
      issuer: 'directus',
    });
    
    // Create session
    await this.knex.insert({
      token: refreshToken,
      user: userId,
      expires: new Date(Date.now() + ms(env['REFRESH_TOKEN_TTL'])),
    }).into('directus_sessions');
    
    // Log activity
    await this.activityService.createOne({
      action: Action.LOGIN,
      user: userId,
      ip: this.accountability?.ip,
      user_agent: this.accountability?.userAgent,
      collection: 'directus_users',
      item: userId,
    });
    
    return {
      access_token: accessToken,
      refresh_token: refreshToken,
      expires: ms(env['ACCESS_TOKEN_TTL']),
    };
  }
}
```

## Best Practices

### Security Checklist

- ✅ Use strong SECRET key (min 32 characters)
- ✅ Enable HTTPS in production
- ✅ Configure CORS properly
- ✅ Enable rate limiting
- ✅ Use TFA for admin accounts
- ✅ Implement IP filtering for sensitive operations
- ✅ Regular security audits
- ✅ Keep dependencies updated
- ✅ Use environment variables for secrets
- ✅ Enable security headers

### Password Policy

```bash
# Enforce password requirements
AUTH_PASSWORD_POLICY=/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$/
```

### Session Management

```bash
# Session configuration
SESSION_COOKIE_TTL=1d
SESSION_COOKIE_SECURE=true
SESSION_COOKIE_SAME_SITE=lax

# Refresh token configuration
REFRESH_TOKEN_TTL=7d
REFRESH_TOKEN_COOKIE_SECURE=true
```

## Next Steps

- [Database Layer](./05-database-layer.md) - Database architecture
- [Backend Architecture](./04-backend-architecture.md) - API structure
- [Real-time Features](./16-realtime-features.md) - WebSocket security
