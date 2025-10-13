# Authentik OAuth Setup for Open WebUI

**Status**: OAuth is configured in Open WebUI but needs Authentik provider setup

## Quick Setup Steps

### Step 1: Access Authentik Admin Panel
1. Go to: https://authentik.homehub.dk/if/admin/
2. Login with your admin credentials

### Step 2: Create OAuth2/OpenID Provider
1. Navigate to **Applications** → **Providers** (left sidebar)
2. Click **Create** button
3. Select **OAuth2/OpenID Provider**

### Step 3: Configure Provider Settings
Fill in these values:

**Basic Settings:**
- **Name**: `openwebui`
- **Authorization flow**: `default-provider-authorization-implicit-consent` (or your default)
- **Client type**: `Confidential`
- **Client ID**: (auto-generated - copy this!)
- **Client Secret**: (auto-generated - click "Show" and copy this!)

**Redirect URIs (IMPORTANT):**
```
https://chat.homehub.dk/oauth/oidc/callback
```

**Signing Key:**
- Select: `authentik Self-signed Certificate` (or your existing signing key)

**Advanced protocol settings (Optional but recommended):**
- **Scopes**: `openid`, `email`, `profile`
- **Subject mode**: `Based on the User's hashed ID`
- **Include claims in id_token**: ✓ (checked)

Click **Finish** to create the provider.

### Step 4: Create Application
1. Navigate to **Applications** → **Applications**
2. Click **Create**
3. Fill in:
   - **Name**: `Open WebUI`
   - **Slug**: `openwebui`
   - **Provider**: Select `openwebui` (the provider you just created)
   - **Launch URL**: `https://chat.homehub.dk`
4. Click **Create**

### Step 5: Copy Credentials and Update .env

**Important**: Copy the Client ID and Client Secret from the provider.

To view them again:
1. Go to **Applications** → **Providers**
2. Click on `openwebui` provider
3. Scroll to **Protocol settings**
4. Client ID is visible, click **Show** next to Client Secret

**Now update the .env file on the server:**

```bash
# SSH into lxc-prod-2
ssh root@10.2.1.130

# Edit the .env file
nano /opt/stacks/ai/.env

# Replace these lines with your actual values:
OAUTH_CLIENT_ID=<paste-your-client-id-here>
OAUTH_CLIENT_SECRET=<paste-your-client-secret-here>

# Save and exit (Ctrl+X, Y, Enter)

# Restart Open WebUI to apply changes
cd /opt/stacks/ai
docker compose restart open-webui

# Wait 10 seconds for it to restart
sleep 10
docker logs open-webui | tail -20
```

### Step 6: Test OAuth Login

1. Go to https://chat.homehub.dk
2. You should now see **two login options**:
   - Sign in with email/password (local)
   - Sign in with Authentik (OAuth)
3. Click **Sign in with Authentik**
4. You'll be redirected to Authentik
5. Login with your Authentik credentials
6. Authorize the application
7. You'll be redirected back to Open WebUI, logged in!

## Verification Commands

After setup, verify the configuration:

```bash
# Check if environment variables are loaded
ssh root@10.2.1.130 "docker exec open-webui env | grep OAUTH"

# Check Open WebUI logs for OAuth errors
ssh root@10.2.1.130 "docker logs open-webui 2>&1 | grep -i oauth"

# Test the OIDC discovery endpoint
curl -s https://authentik.homehub.dk/application/o/openwebui/.well-known/openid-configuration | jq
```

## Troubleshooting

### "Invalid redirect URI"
- Check that redirect URI in Authentik exactly matches: `https://chat.homehub.dk/oauth/oidc/callback`
- No trailing slashes!

### "OAuth provider not showing up"
- Verify .env has correct Client ID and Secret (no quotes, no spaces)
- Restart Open WebUI: `docker compose restart open-webui`

### "Failed to fetch user info"
- Ensure scopes include: `openid`, `email`, `profile`
- Check Authentik logs: Applications → Providers → openwebui → View Logs

### "Can't login after OAuth setup"
- Local login should still work
- If locked out, reset admin: 
  ```bash
  docker exec -it open-webui python -c "from apps.webui.models.users import Users; Users.update_user_by_id(1, {'password': Users.hash_password('newpassword')})"
  ```

## Current Configuration Summary

**Open WebUI Environment Variables:**
```
OLLAMA_BASE_URLS=http://10.2.0.190:11434
QDRANT_URI=http://10.2.0.190:6333
ENABLE_OAUTH_SIGNUP=true
OAUTH_MERGE_ACCOUNTS_BY_EMAIL=false
OAUTH_PROVIDER_NAME=authentik
OPENID_PROVIDER_URL=https://authentik.homehub.dk/application/o/openwebui/.well-known/openid-configuration
OAUTH_CLIENT_ID=${OAUTH_CLIENT_ID}
OAUTH_CLIENT_SECRET=${OAUTH_CLIENT_SECRET}
OAUTH_SCOPES=openid email profile
OPENID_REDIRECT_URI=https://chat.homehub.dk/oauth/oidc/callback
```

**File Location:** `/opt/stacks/ai/.env` on lxc-prod-2

## Security Notes

✅ OAuth uses HTTPS for all communications  
✅ Client Secret should be kept confidential  
✅ Redirect URI prevents OAuth hijacking  
⚠️ First local user created is admin - secure this account!  
⚠️ Consider disabling local signup after OAuth is working: Set `ENABLE_SIGNUP=false` in .env

## References

- Authentik OAuth2 Provider Docs: https://docs.goauthentik.io/docs/providers/oauth2/
- Open WebUI OAuth Docs: https://docs.openwebui.com/getting-started/env-configuration#oauth


