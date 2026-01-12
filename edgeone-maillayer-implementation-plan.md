# Maillayer on EdgeOne Implementation Plan

## Project Overview
Deploy a self-hosted email marketing platform using:
- **Frontend/Backend**: Maillayer (GitHub clone)
- **Hosting**: EdgeOne Pages + Edge Functions
- **Storage**: EdgeOne KV + D1
- **Authentication**: Supabase Auth (Google OAuth)
- **Email Delivery**: Amazon SES
- **Domain**: EdgeOne domain registration

---

## Phase 1: Environment Setup

### 1.1 Account Creation
- [ ] Create EdgeOne account at [edgeone.ai](https://edgeone.ai)
- [ ] Create/login to Supabase account at [supabase.com](https://supabase.com)
- [ ] Create/login to AWS account for SES at [aws.amazon.com](https://aws.amazon.com/ses/)

### 1.2 Domain Setup
- [ ] Purchase domain through EdgeOne domain registration
- [ ] Configure DNS settings in EdgeOne console
- [ ] Wait for DNS propagation (24-48 hours)

### 1.3 AWS SES Configuration
- [ ] Navigate to AWS SES console (Singapore region: ap-southeast-1)
- [ ] Verify domain ownership (add TXT/CNAME records to EdgeOne DNS)
- [ ] Create SMTP credentials or API access keys
- [ ] Request production access (move out of sandbox mode)
- [ ] Configure SPF, DKIM, and DMARC records in EdgeOne DNS
- [ ] Save SES credentials securely (SMTP/API keys)

---

## Phase 2: Authentication Setup (Supabase)

### 2.1 Create Supabase Project
- [ ] Create new Supabase project (choose Singapore region if available)
- [ ] Note down project URL and anon key
- [ ] Navigate to Authentication settings

### 2.2 Configure Google OAuth
- [ ] Go to [Google Cloud Console](https://console.cloud.google.com)
- [ ] Create new project or select existing one
- [ ] Enable Google+ API
- [ ] Create OAuth 2.0 credentials (Web application)
- [ ] Add authorized redirect URIs:
  - `https://<your-supabase-project>.supabase.co/auth/v1/callback`
  - `https://<your-edgeone-domain>/auth/callback`
- [ ] Save Client ID and Client Secret

### 2.3 Configure Supabase Auth Provider
- [ ] In Supabase dashboard → Authentication → Providers
- [ ] Enable Google provider
- [ ] Add Google Client ID and Client Secret
- [ ] Configure redirect URLs
- [ ] Save Supabase environment variables:
  ```
  SUPABASE_URL=https://<project>.supabase.co
  SUPABASE_ANON_KEY=<your-anon-key>
  ```

---

## Phase 3: Maillayer Setup

### 3.1 Clone Repository
```bash
# Clone Maillayer repository
git clone https://github.com/TencentEdgeOne/maillayer.git
cd maillayer

# Or fork to your own GitHub account first
gh repo fork https://github.com/TencentEdgeOne/maillayer.git
cd maillayer
```

### 3.2 Local Development Setup
```bash
# Install dependencies
npm install

# Create .env.local file
cp .env.example .env.local
```

### 3.3 Configure Environment Variables
Edit `.env.local`:
```env
# Supabase Auth
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_ANON_KEY=<your-anon-key>

# Amazon SES
AWS_REGION=ap-southeast-1
AWS_SES_ACCESS_KEY_ID=<your-access-key>
AWS_SES_SECRET_ACCESS_KEY=<your-secret-key>
AWS_SES_FROM_EMAIL=noreply@yourdomain.com

# EdgeOne (will be auto-configured on deployment)
EDGEONE_KV_NAMESPACE=<namespace-id>
EDGEONE_D1_DATABASE=<database-id>

# Application
APP_URL=http://localhost:3000
NODE_ENV=development
```

### 3.4 Test Locally
```bash
# Run development server
npm run dev

# Test authentication flow
# Test email sending with SES
# Verify KV/D1 connections (if using local emulator)
```

---

## Phase 4: EdgeOne Configuration

### 4.1 Create EdgeOne KV Namespace
- [ ] Log into EdgeOne console
- [ ] Navigate to Storage → KV
- [ ] Create new KV namespace: `maillayer-kv-staging`
- [ ] Create production namespace: `maillayer-kv-production`
- [ ] Note down namespace IDs

### 4.2 Create EdgeOne D1 Database
- [ ] Navigate to Storage → D1
- [ ] Create new D1 database: `maillayer-db-staging`
- [ ] Create production database: `maillayer-db-production`
- [ ] Note down database IDs

### 4.3 Initialize Database Schema
```bash
# Create migration file
mkdir -p migrations
nano migrations/001_initial_schema.sql
```

Add initial schema:
```sql
-- Users table (minimal, Supabase handles auth)
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Email campaigns
CREATE TABLE campaigns (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  name TEXT NOT NULL,
  subject TEXT,
  content TEXT,
  status TEXT DEFAULT 'draft',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Subscribers
CREATE TABLE subscribers (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  email TEXT NOT NULL,
  status TEXT DEFAULT 'subscribed',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Email logs
CREATE TABLE email_logs (
  id TEXT PRIMARY KEY,
  campaign_id TEXT,
  subscriber_id TEXT,
  status TEXT,
  sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (campaign_id) REFERENCES campaigns(id),
  FOREIGN KEY (subscriber_id) REFERENCES subscribers(id)
);
```

Apply migration:
```bash
# Using EdgeOne CLI
edgeone d1 migrations apply maillayer-db-staging
```

---

## Phase 5: Deployment (Staging)

### 5.1 Install EdgeOne CLI
```bash
npm install -g @edgeone/cli
edgeone login
```

### 5.2 Initialize EdgeOne Project
```bash
edgeone init

# Select:
# - Framework: Next.js (or appropriate framework)
# - Build command: npm run build
# - Output directory: .next
```

### 5.3 Configure edgeone.config.js
```javascript
module.exports = {
  projectName: 'maillayer-staging',
  bindings: {
    kv: {
      MAILLAYER_KV: '<staging-kv-namespace-id>'
    },
    d1: {
      MAILLAYER_DB: '<staging-d1-database-id>'
    }
  },
  env: {
    SUPABASE_URL: process.env.SUPABASE_URL,
    SUPABASE_ANON_KEY: process.env.SUPABASE_ANON_KEY,
    AWS_REGION: 'ap-southeast-1',
    AWS_SES_ACCESS_KEY_ID: process.env.AWS_SES_ACCESS_KEY_ID,
    AWS_SES_SECRET_ACCESS_KEY: process.env.AWS_SES_SECRET_ACCESS_KEY,
    AWS_SES_FROM_EMAIL: 'staging@yourdomain.com'
  }
}
```

### 5.4 Deploy to Staging
```bash
# Deploy
edgeone deploy --env staging

# Note the deployment URL
# Example: https://maillayer-staging.pages.edgeone.ai
```

### 5.5 Configure Staging Environment Variables in EdgeOne Console
- [ ] Go to EdgeOne dashboard → Project Settings → Environment Variables
- [ ] Add all sensitive variables (don't commit to Git):
  - `SUPABASE_URL`
  - `SUPABASE_ANON_KEY`
  - `AWS_SES_ACCESS_KEY_ID`
  - `AWS_SES_SECRET_ACCESS_KEY`
  - `AWS_SES_FROM_EMAIL`

### 5.6 Update Supabase Redirect URLs
- [ ] In Supabase dashboard → Authentication → URL Configuration
- [ ] Add staging URL to allowed redirect URLs:
  - `https://maillayer-staging.pages.edgeone.ai/auth/callback`

### 5.7 Test Staging Deployment
- [ ] Visit staging URL
- [ ] Test Google OAuth login
- [ ] Create test campaign
- [ ] Send test email via SES
- [ ] Verify email delivery
- [ ] Check EdgeOne KV data storage
- [ ] Verify D1 database queries

---

## Phase 6: Production Deployment

### 6.1 Create Production Project
```bash
edgeone init --name maillayer-production
```

### 6.2 Configure Production Bindings
Update `edgeone.config.production.js`:
```javascript
module.exports = {
  projectName: 'maillayer-production',
  bindings: {
    kv: {
      MAILLAYER_KV: '<production-kv-namespace-id>'
    },
    d1: {
      MAILLAYER_DB: '<production-d1-database-id>'
    }
  },
  env: {
    // Same structure as staging but production values
    AWS_SES_FROM_EMAIL: 'noreply@yourdomain.com'
  }
}
```

### 6.3 Apply Database Migrations to Production
```bash
edgeone d1 migrations apply maillayer-db-production
```

### 6.4 Configure Custom Domain
- [ ] In EdgeOne console → Domains
- [ ] Add custom domain: `mail.yourdomain.com`
- [ ] Point domain to EdgeOne nameservers or add CNAME record
- [ ] Wait for SSL certificate provisioning (automatic)

### 6.5 Deploy to Production
```bash
edgeone deploy --env production --domain mail.yourdomain.com
```

### 6.6 Update Supabase Production URLs
- [ ] Add production domain to Supabase allowed redirect URLs:
  - `https://mail.yourdomain.com/auth/callback`

### 6.7 Production Testing Checklist
- [ ] SSL certificate active (HTTPS working)
- [ ] Google OAuth login functioning
- [ ] Email sending via SES operational
- [ ] User data persisting in D1
- [ ] KV storage working (sessions, cache)
- [ ] Error logging configured
- [ ] Performance monitoring active

---

## Phase 7: Code Implementation Details

### 7.1 Authentication Integration (Supabase)

Create `lib/supabase.js`:
```javascript
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_ANON_KEY
)

// Auth helpers
export async function signInWithGoogle() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: `${process.env.APP_URL}/auth/callback`
    }
  })
  return { data, error }
}

export async function getSession() {
  const { data: { session } } = await supabase.auth.getSession()
  return session
}

export async function signOut() {
  const { error } = await supabase.auth.signOut()
  return { error }
}
```

### 7.2 EdgeOne KV Integration

Create `lib/kv.js`:
```javascript
// For EdgeOne Edge Functions
export async function getFromKV(key, env) {
  return await env.MAILLAYER_KV.get(key, { type: 'json' })
}

export async function setToKV(key, value, env, ttl = null) {
  const options = ttl ? { expirationTtl: ttl } : {}
  return await env.MAILLAYER_KV.put(key, JSON.stringify(value), options)
}

export async function deleteFromKV(key, env) {
  return await env.MAILLAYER_KV.delete(key)
}
```

### 7.3 EdgeOne D1 Integration

Create `lib/db.js`:
```javascript
export async function queryDB(sql, params, env) {
  const result = await env.MAILLAYER_DB.prepare(sql)
    .bind(...params)
    .all()
  return result.results
}

export async function insertCampaign(userId, name, subject, content, env) {
  const id = crypto.randomUUID()
  const sql = `INSERT INTO campaigns (id, user_id, name, subject, content) 
               VALUES (?, ?, ?, ?, ?)`
  await env.MAILLAYER_DB.prepare(sql)
    .bind(id, userId, name, subject, content)
    .run()
  return id
}

export async function getCampaigns(userId, env) {
  const sql = `SELECT * FROM campaigns WHERE user_id = ? ORDER BY created_at DESC`
  return await queryDB(sql, [userId], env)
}
```

### 7.4 Amazon SES Integration

Create `lib/ses.js`:
```javascript
import { SESClient, SendEmailCommand } from '@aws-sdk/client-ses'

const sesClient = new SESClient({
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_SES_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SES_SECRET_ACCESS_KEY
  }
})

export async function sendEmail({ to, subject, htmlBody, textBody }) {
  const params = {
    Source: process.env.AWS_SES_FROM_EMAIL,
    Destination: {
      ToAddresses: Array.isArray(to) ? to : [to]
    },
    Message: {
      Subject: { Data: subject },
      Body: {
        Html: { Data: htmlBody },
        Text: { Data: textBody || '' }
      }
    }
  }

  try {
    const command = new SendEmailCommand(params)
    const response = await sesClient.send(command)
    return { success: true, messageId: response.MessageId }
  } catch (error) {
    console.error('SES Error:', error)
    return { success: false, error: error.message }
  }
}
```

### 7.5 Edge Function Example (API Route)

Create `pages/api/campaigns/send.js`:
```javascript
import { getSession } from '../../../lib/supabase'
import { getCampaigns } from '../../../lib/db'
import { sendEmail } from '../../../lib/ses'

export default async function handler(req, res, env) {
  // Verify authentication
  const session = await getSession()
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' })
  }

  const { campaignId } = req.body

  // Fetch campaign from D1
  const sql = `SELECT * FROM campaigns WHERE id = ? AND user_id = ?`
  const campaigns = await env.MAILLAYER_DB.prepare(sql)
    .bind(campaignId, session.user.id)
    .all()

  if (campaigns.results.length === 0) {
    return res.status(404).json({ error: 'Campaign not found' })
  }

  const campaign = campaigns.results[0]

  // Fetch subscribers from D1
  const subscribersSql = `SELECT email FROM subscribers WHERE user_id = ? AND status = 'subscribed'`
  const subscribers = await env.MAILLAYER_DB.prepare(subscribersSql)
    .bind(session.user.id)
    .all()

  // Send emails via SES
  const results = await Promise.all(
    subscribers.results.map(sub =>
      sendEmail({
        to: sub.email,
        subject: campaign.subject,
        htmlBody: campaign.content
      })
    )
  )

  return res.json({ 
    sent: results.filter(r => r.success).length,
    failed: results.filter(r => !r.success).length
  })
}
```

---

## Phase 8: Monitoring & Maintenance

### 8.1 Setup Monitoring
- [ ] Configure EdgeOne analytics dashboard
- [ ] Monitor Edge Function invocations
- [ ] Track KV storage usage
- [ ] Monitor D1 query performance
- [ ] Setup AWS SES sending statistics alerts
- [ ] Monitor Supabase Auth usage (approaching limits)

### 8.2 Cost Monitoring
- [ ] Setup AWS SES spending alerts (>$5/month)
- [ ] Monitor Supabase MAU count (approaching 10,000 limit)
- [ ] Track EdgeOne usage against free tier limits

### 8.3 Backup Strategy
- [ ] Export D1 database regularly: `edgeone d1 export maillayer-db-production`
- [ ] Backup campaign templates to GitHub
- [ ] Document environment variables in secure password manager

### 8.4 Security Checklist
- [ ] Rotate AWS SES credentials every 90 days
- [ ] Review Supabase Auth logs monthly
- [ ] Enable Supabase email verification for new signups
- [ ] Configure rate limiting on Edge Functions
- [ ] Setup CORS policies correctly
- [ ] Enable HTTPS only (no HTTP fallback)

---

## Phase 9: Scaling Considerations

### When to Upgrade (Monitor These Metrics)

**EdgeOne Free Tier Limits:**
- Edge Function requests: 3M/month
- KV storage: 1GB
- D1 operations: Check current limits

**Supabase Free Tier Limits:**
- Monthly Active Users: 10,000
- Database size: 500MB (not used in this setup)

**AWS SES Free Tier:**
- First 12 months: 3,000 emails/month free
- After 12 months: $0.10 per 1,000 emails

### Scaling Actions
1. **>10K MAUs**: Upgrade Supabase to Pro ($25/month)
2. **>3M Edge Function requests**: Upgrade EdgeOne (pricing TBA)
3. **>100K emails/month**: Budget $10/month for SES
4. **Database performance issues**: Consider D1 optimization or external database

---

## Troubleshooting Guide

### Issue: Google OAuth Not Working
- Verify redirect URIs match exactly in Google Console and Supabase
- Check Supabase logs in dashboard
- Ensure environment variables are set correctly in EdgeOne

### Issue: Emails Not Sending
- Verify AWS SES is out of sandbox mode
- Check SES credentials are correct
- Verify sender email is verified in SES
- Check SES sending limits (200 emails/day in sandbox)

### Issue: EdgeOne KV/D1 Connection Errors
- Verify namespace/database IDs in edgeone.config.js
- Check bindings are correctly configured
- Review Edge Function logs in EdgeOne console

### Issue: CORS Errors
- Add proper CORS headers in Edge Functions
- Verify allowed origins in Supabase dashboard
- Check EdgeOne domain configuration

---

## Resources & Documentation

### Official Documentation
- [EdgeOne Pages Docs](https://pages.edgeone.ai/document)
- [Supabase Auth Docs](https://supabase.com/docs/guides/auth)
- [AWS SES Developer Guide](https://docs.aws.amazon.com/ses/)
- [Maillayer GitHub](https://github.com/TencentEdgeOne/maillayer) *(placeholder - replace with actual repo)*

### Community Support
- EdgeOne Discord/Community Forum
- Supabase Discord
- AWS SES Support Forums

### Estimated Timeline
- **Phase 1-2 (Setup)**: 2-3 hours
- **Phase 3-4 (Configuration)**: 3-4 hours
- **Phase 5 (Staging)**: 2-3 hours
- **Phase 6 (Production)**: 1-2 hours
- **Phase 7 (Code Implementation)**: 8-12 hours (depending on Maillayer complexity)
- **Phase 8 (Monitoring)**: 1-2 hours
- **Total**: 17-26 hours

---

## Success Criteria

### Staging Success
- [ ] Google OAuth login works
- [ ] Can create campaigns in UI
- [ ] Can send test emails via SES
- [ ] Data persists in EdgeOne D1
- [ ] Sessions stored in EdgeOne KV

### Production Success
- [ ] Custom domain active with SSL
- [ ] All staging tests pass in production
- [ ] Performance: Page load <2s
- [ ] Email delivery rate >95%
- [ ] Zero authentication errors
- [ ] Cost: Within budget ($0-3/month initially)

---

## Notes
- Replace all placeholder URLs and IDs with actual values during implementation
- Store all credentials securely (use password manager, never commit to Git)
- Test thoroughly in staging before production deployment
- Document any custom modifications to Maillayer codebase
- Keep this plan updated as implementation progresses
