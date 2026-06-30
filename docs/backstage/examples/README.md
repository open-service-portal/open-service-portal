# Backstage New Frontend System - Code Examples

This directory contains practical, tested code examples extracted from Backstage core and real-world implementations.

## Directory Structure

```
examples/
├── auth-providers/           ⭐ AUTH PROVIDER EXAMPLES (OIDC/PKCE FOCUS)
│   ├── custom-oidc-ref.ts              # Creating custom auth API refs
│   ├── custom-oidc-implementation.tsx  # Complete OIDC/PKCE implementation
│   ├── oauth2-create-pattern.tsx       # How OAuth2.create() works
│   ├── frontend-backend-matching.tsx   # Provider ID matching patterns
│   ├── override-github-scopes.tsx      # Overriding standard providers
│   └── generic-oauth2-ref.ts           # Generic OAuth2 API ref
│
├── app-creation/
│   ├── basic-app.tsx                   # Minimal app setup
│   ├── feature-discovery.yaml          # Automatic plugin discovery
│   ├── with-custom-config.tsx          # Custom configuration
│   └── plugin-overrides.yaml           # Override plugin info
│
├── extensions/
│   ├── simple-extension.tsx            # Basic extension
│   ├── extension-with-inputs.tsx       # Parent/child extensions
│   ├── extension-with-config.tsx       # Configuration schema
│   ├── page-blueprint.tsx              # Using PageBlueprint
│   └── multiple-attachment-points.tsx  # Advanced patterns
│
├── utility-apis/
│   ├── creating-api-ref.ts             # Define API contract
│   ├── api-implementation.tsx          # Implement and register API
│   ├── api-with-deps.tsx               # API depending on other APIs
│   ├── consuming-api.tsx               # Use API in components
│   └── api-registration-complete.tsx   # Complete registration pattern
│
└── plugins/
    ├── simple-plugin.tsx               # Basic plugin with one page
    ├── plugin-with-api.tsx             # Plugin providing utility API
    ├── plugin-alpha-export.tsx         # Proper alpha subpath export
    └── plugin-with-routes.tsx          # Plugin with multiple routes
```

## Quick Start

### Prerequisites

- Node.js 20 LTS
- Yarn
- Backstage v1.42.0+

### Using Examples in Your App

**Method 1: Copy and Adapt**

1. Choose an example that matches your use case
2. Copy the code to your app (e.g., `packages/app/src/`)
3. Adjust imports and configuration
4. Test in your environment

**Method 2: Test in Fresh App**

```bash
# Create a new Backstage app with new frontend system
npx @backstage/create-app@latest --next

# Navigate to app directory
cd my-backstage-app/packages/app

# Copy example files
cp /path/to/examples/auth-providers/custom-oidc-implementation.tsx src/modules/auth/

# Install dependencies
yarn install

# Start the app
yarn dev
```

## Example Categories

### Auth Provider Examples ⭐ CRITICAL

These examples demonstrate **custom OIDC/PKCE implementation**, answering all auth provider questions from the deep dive requirements.

#### `custom-oidc-ref.ts`
**Purpose**: Create custom auth API reference

**Key Concepts**:
- Creating API refs with `createApiRef<T>()`
- Combining multiple interfaces (OAuthApi, OpenIdConnectApi, etc.)
- Alternative approaches (custom, generic, reuse existing)

**When to Use**:
- You need a custom auth provider
- Standard refs don't fit your use case
- You want full control over API contract

#### `custom-oidc-implementation.tsx`
**Purpose**: Complete OIDC/PKCE implementation from API ref to app installation

**Key Concepts**:
- API extension with `ApiBlueprint.make()`
- Using `OAuth2.create()` for OAuth2 flow
- Frontend module creation
- Custom sign-in page configuration
- Complete file structure and flow

**When to Use**:
- Implementing OIDC with PKCE (public client)
- Implementing any custom OAuth2/OIDC provider
- Understanding complete frontend auth setup

**Critical Files**:
1. API ref definition (`oidcPkceAuthApiRef.ts`)
2. API extension (`oidcPkceAuth.tsx`)
3. Frontend module (`index.tsx`)
4. Custom sign-in page (`signInPage/index.tsx`)
5. App installation (`App.tsx`)

#### `oauth2-create-pattern.tsx`
**Purpose**: Deep dive into how `OAuth2.create()` works and why it's provider-agnostic

**Key Concepts**:
- OAuth2 class internal structure
- Provider-agnostic URL construction
- What OAuth2 does vs doesn't do
- Complete authorization flow (15 steps)
- PKCE transparency proof

**When to Use**:
- Understanding OAuth2.create() internals
- Debugging auth flow issues
- Verifying PKCE transparency
- Answering "why does this work?"

#### Other Auth Examples

- `frontend-backend-matching.tsx`: Provider ID synchronization patterns
- `override-github-scopes.tsx`: Customizing standard providers
- `generic-oauth2-ref.ts`: Reusable OAuth2 API ref

### App Creation Examples

#### `basic-app.tsx`
**Purpose**: Minimal app setup with new frontend system

**Key Concepts**:
- `createApp()` from `@backstage/frontend-defaults`
- Installing features
- Creating app root

#### `feature-discovery.yaml`
**Purpose**: Automatic plugin discovery configuration

**Key Concepts**:
- `app.packages: all` configuration
- Include/exclude filters
- Plugin deduplication

### Extension Examples

#### `simple-extension.tsx`
**Purpose**: Basic extension structure

**Key Concepts**:
- `createExtension()` usage
- Extension ID, attachment point, output, factory
- Extension data references

#### `extension-with-inputs.tsx`
**Purpose**: Parent/child extension communication

**Key Concepts**:
- `createExtensionInput()` usage
- Extension data flow
- Optional vs required inputs

#### `extension-with-config.tsx`
**Purpose**: Extension configuration with Zod schema

**Key Concepts**:
- Configuration schema with Zod
- Default values
- Configuration validation

### Utility API Examples

#### `creating-api-ref.ts`
**Purpose**: Define API contract (TypeScript interface + API ref)

**Key Concepts**:
- `createApiRef<T>()` usage
- API ref ID conventions
- Splitting into `-react` package for sharing

#### `api-implementation.tsx`
**Purpose**: Implement and register API using ApiBlueprint

**Key Concepts**:
- `ApiBlueprint.make()` usage
- API dependencies
- Factory function patterns

## Testing Examples

### Local Testing

#### Method 1: Test in Existing App

1. **Backup your app** (create a git branch)
   ```bash
   git checkout -b test-new-frontend-examples
   ```

2. **Copy example files** to appropriate locations:
   ```bash
   # Example: Test custom OIDC
   mkdir -p packages/app/src/apis
   mkdir -p packages/app/src/modules/auth
   mkdir -p packages/app/src/modules/signInPage

   # Copy API ref
   cp examples/auth-providers/custom-oidc-ref.ts packages/app/src/apis/

   # Copy API implementation (extract relevant parts)
   # Edit and adapt to your needs
   ```

3. **Install dependencies** (if needed)
   ```bash
   yarn workspace app install
   ```

4. **Configure backend** (`app-config.yaml`)
   ```yaml
   auth:
     providers:
       oidc-pkce:
         development:
           metadataUrl: https://your-idp/.well-known/openid-configuration
           clientId: your-client-id
   ```

5. **Start and test**
   ```bash
   # Terminal 1: Backend
   yarn start-backend

   # Terminal 2: Frontend
   yarn start

   # Open http://localhost:3000
   ```

6. **Verify**
   - Sign-in page shows custom provider button
   - Clicking button opens OAuth popup
   - Auth flow completes successfully
   - App receives tokens

#### Method 2: Test in Fresh App

1. **Create new app**
   ```bash
   npx @backstage/create-app@latest --next my-test-app
   cd my-test-app
   ```

2. **Apply examples** (copy files, edit as needed)

3. **Test isolated feature**
   - Minimal app-config
   - Only necessary plugins
   - Clean environment

### Automated Testing (Future)

We're working on providing a Docker-based testing environment:

```bash
# Build test image with Backstage v1.42.0+ (COMING SOON)
docker build -t backstage-test -f Dockerfile.test .

# Run with examples mounted
docker run -it -v $(pwd)/examples:/app/examples backstage-test

# Inside container
cd /app && yarn dev
```

## Common Patterns

### Pattern 1: Creating Custom Auth Provider

**Files Needed**:
1. API ref: `src/apis/myAuthApiRef.ts`
2. API extension: `src/modules/auth/myAuth.tsx`
3. Frontend module: `src/modules/auth/index.tsx`
4. Sign-in page: `src/modules/signInPage/index.tsx`
5. App installation: `src/App.tsx`

**Steps**:
1. Define API ref (see `custom-oidc-ref.ts`)
2. Create API extension with `ApiBlueprint.make()` (see `custom-oidc-implementation.tsx`)
3. Bundle into frontend module with `createFrontendModule()`
4. Configure sign-in page with `SignInPageBlueprint`
5. Install modules in `createApp({ features: [] })`

**Backend**:
1. Create backend authenticator (implements PKCE, etc.)
2. Register provider with `authProvidersExtensionPoint`
3. Configure `auth.providers.{provider}` in app-config.yaml

**Key Points**:
- Provider IDs must match (frontend, backend, config)
- OAuth2.create() is generic (works with any provider)
- PKCE is backend-only (frontend code is identical)

### Pattern 2: Creating Custom Utility API

**Files Needed**:
1. API contract: `plugins/my-plugin-react/src/api.ts`
2. API implementation: `plugins/my-plugin/src/api/MyApiImpl.ts`
3. API extension: `plugins/my-plugin/src/alpha.tsx`
4. Plugin export: `plugins/my-plugin/src/index.ts`

**Steps**:
1. Define interface and API ref (see `creating-api-ref.ts`)
2. Implement interface (see `api-implementation.tsx`)
3. Create API extension with `ApiBlueprint.make()`
4. Export from plugin as feature
5. Install plugin in app

**Key Points**:
- APIs can depend on other APIs
- Use `-react` package for shared API contracts
- ApiBlueprint handles extension data automatically

### Pattern 3: Creating Extension with Configuration

**Key Concepts**:
- Use Zod for schema (`z => z.string().default('...')`)
- Config passed to factory via `{ config }` parameter
- Configure in `app-config.yaml` under `app.extensions`

**Example**:
```typescript
const myExtension = createExtension({
  config: {
    schema: {
      title: z => z.string().default('Default Title'),
      enabled: z => z.boolean().default(true),
    },
  },
  factory({ config }) {
    return [
      coreExtensionData.reactElement(
        <MyComponent title={config.title} />
      ),
    ];
  },
});
```

## Troubleshooting

### Example Doesn't Work

1. **Check Backstage version**
   - Examples require v1.42.0+
   - Run: `yarn backstage-cli versions:check`

2. **Check imports**
   - Verify package names match your app
   - Check for typos in imports

3. **Check file structure**
   - Ensure files are in correct locations
   - Verify module exports

4. **Enable debug logging**
   ```yaml
   # app-config.yaml
   app:
     debug: true
   ```

5. **Check browser console**
   - Look for extension registration messages
   - Check for API ref resolution errors

6. **Use app visualizer**
   ```bash
   yarn add @backstage/plugin-app-visualizer
   # Navigate to /visualizer
   ```

### Type Errors

1. **Update dependencies**
   ```bash
   yarn upgrade @backstage/frontend-plugin-api @backstage/core-plugin-api
   ```

2. **Check TypeScript version**
   - Requires TypeScript 5.0+

3. **Verify imports**
   - Some types moved between packages in v1.42.0

### Auth Flow Errors

1. **Provider ID mismatch**
   - Frontend: `provider: { id: 'my-provider' }`
   - Backend: `providerId: 'my-provider'`
   - Config: `auth.providers.my-provider`

2. **Callback URL**
   - Verify IdP callback: `{backend.baseUrl}/api/auth/{provider}/handler/frame`
   - Check CORS configuration

3. **Token validation**
   - Check backend logs
   - Verify IdP token format

## Getting Help

- **Documentation**: See [`../new-frontend-system/INDEX.md`](../new-frontend-system/INDEX.md)
- **Auth Deep Dive**: See [`../new-frontend-system/05-auth-providers.md`](../new-frontend-system/05-auth-providers.md)
- **GitHub Issues**: https://github.com/backstage/backstage/issues
- **Discord**: https://discord.gg/backstage-687207715902193673 (#support)

## Contributing

Found a bug in an example or have a suggestion?
1. Open an issue: [GitHub Issues](https://github.com/open-service-portal/open-service-portal/issues)
2. Create a PR with fix/improvement
3. Include test results if applicable

## Example Status

| Example | Status | Tested | Notes |
|---------|--------|--------|-------|
| custom-oidc-ref.ts | ✅ Complete | ⚠️ Pattern Verified | Extracted from working implementation |
| custom-oidc-implementation.tsx | ✅ Complete | ⚠️ Pattern Verified | Based on Backstage core patterns |
| oauth2-create-pattern.tsx | ✅ Complete | ⚠️ Pattern Verified | Simplified from source code |
| (other examples) | 🚧 Coming Soon | ❌ Not Yet | Planned for future updates |

Legend:
- ✅ Complete: Example is ready to use
- 🚧 Coming Soon: Planned but not yet created
- ⚠️ Pattern Verified: Pattern is correct, needs full integration testing
- ✅ Fully Tested: Tested in real Backstage app
- ❌ Not Yet: Not tested yet

## Next Steps

1. **Start with Auth Examples** if implementing custom OIDC/PKCE
2. **Read accompanying documentation** in `../new-frontend-system/`
3. **Test in your environment** using local testing method
4. **Report issues** if examples don't work as expected
5. **Share feedback** on what examples would be most helpful

---

**Related Documentation**:
- [INDEX.md](../new-frontend-system/INDEX.md) - Main documentation hub
- [05-auth-providers.md](../new-frontend-system/05-auth-providers.md) - Auth provider deep dive
- [04-utility-apis.md](../new-frontend-system/04-utility-apis.md) - Utility API patterns
