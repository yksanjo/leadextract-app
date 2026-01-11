# LeadExtract - LinkedIn Sales Navigator Scraper

## Project Overview
Web application that allows B2B sales teams to export unlimited leads from LinkedIn Sales Navigator, bypassing LinkedIn's 2,500/month export limit.

## Core Value Proposition
Sales Navigator costs $100/mo and limits exports to 2,500 leads/month. We offer unlimited exports for $299/mo - still cheaper than Sales Navigator Advanced ($1,600/year) with no export limits.

## Technical Architecture

### Backend (Node.js/Express)
- **Proxy Management**: Bright Data residential proxies
- **Scraping Engine**: Puppeteer/Playwright with stealth plugins
- **Queue System**: Bull/BullMQ with Redis

### Frontend (Next.js 14+ App Router)
- Landing page with pricing, features, testimonials
- Dashboard with job management
- Lead database with search/filter
- Export functionality

## Database Schema (PostgreSQL)

```sql
-- Users table
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255) UNIQUE NOT NULL,
  stripe_customer_id VARCHAR(255),
  subscription_status VARCHAR(50) DEFAULT 'free',
  subscription_tier VARCHAR(50), -- 'starter', 'pro', 'enterprise'
  linkedin_session_cookie TEXT, -- encrypted
  exports_this_month INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Scrape Jobs table
CREATE TABLE scrape_jobs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id),
  search_url TEXT NOT NULL,
  status VARCHAR(50) DEFAULT 'pending', -- pending, running, completed, failed
  total_results INTEGER,
  scraped_count INTEGER DEFAULT 0,
  error_message TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);

-- Leads table
CREATE TABLE leads (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  job_id UUID REFERENCES scrape_jobs(id),
  user_id UUID REFERENCES users(id),
  linkedin_url VARCHAR(500),
  full_name VARCHAR(255),
  headline TEXT,
  company_name VARCHAR(255),
  company_linkedin_url VARCHAR(500),
  location VARCHAR(255),
  profile_image_url TEXT,
  current_title VARCHAR(255),
  email VARCHAR(255), -- if found via email finder
  phone VARCHAR(50), -- if found
  connection_degree INTEGER,
  scraped_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, linkedin_url)
);

-- Export History
CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES users(id),
  job_id UUID REFERENCES scrape_jobs(id),
  format VARCHAR(20), -- csv, xlsx, json
  record_count INTEGER,
  file_url TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
```

## API Endpoints

```
POST /api/auth/register - Create account
POST /api/auth/login - Login
POST /api/auth/linkedin-session - Store LinkedIn session cookie

POST /api/jobs - Create new scrape job
GET /api/jobs - List user's jobs
GET /api/jobs/:id - Get job status and progress
DELETE /api/jobs/:id - Cancel job

GET /api/leads - List leads with pagination/filters
POST /api/leads/export - Export leads to CSV/XLSX
DELETE /api/leads - Bulk delete leads

POST /api/billing/create-checkout - Stripe checkout session
POST /api/billing/webhook - Stripe webhook handler
GET /api/billing/subscription - Get subscription status
POST /api/billing/portal - Create Stripe billing portal session
```

## Key Code Examples

### Scraping Logic
```javascript
async function scrapeSearchResults(searchUrl, sessionCookie, jobId) {
  const browser = await puppeteer.launch({
    args: ['--proxy-server=...'] // Bright Data proxy
  });

  const page = await browser.newPage();
  await page.setCookie({ name: 'li_at', value: sessionCookie, domain: '.linkedin.com' });

  // Navigate to search
  await page.goto(searchUrl);
  await randomDelay(2000, 4000);

  // Get total results count
  const totalResults = await page.$eval('.search-results-count', el => parseInt(el.textContent));

  // Paginate through results (Sales Nav shows 25 per page)
  for (let pageNum = 1; pageNum <= Math.ceil(totalResults / 25); pageNum++) {
    // Scroll to load lazy content
    await autoScroll(page);

    // Extract profile cards
    const profiles = await page.$$eval('.search-result__wrapper', cards => {
      return cards.map(card => ({
        name: card.querySelector('.result-lockup__name').textContent,
        headline: card.querySelector('.result-lockup__highlight-keyword').textContent,
        company: card.querySelector('.result-lockup__position-company').textContent,
        location: card.querySelector('.result-lockup__misc-list-item').textContent,
        linkedinUrl: card.querySelector('a.result-lockup__name-link').href
      }));
    });

    // Save to database
    await saveleads(profiles, jobId);

    // Update progress
    await updateJobProgress(jobId, pageNum * 25);

    // Navigate to next page with human-like delay
    await page.click('.search-results__pagination-next');
    await randomDelay(3000, 6000);
  }
}
```

## Pricing Tiers

| Tier | Price | Limits |
|------|-------|--------|
| Starter | $99/mo | 5,000 exports/month |
| Pro | $299/mo | Unlimited exports |
| Enterprise | $599/mo | Unlimited + API access + priority support |

## Launch Strategy
1. Deploy to Railway/Vercel
2. Set up Stripe products and pricing
3. Create landing page with clear value prop
4. Write Twitter thread announcing tool
5. Post in r/sales, r/b2bsales, r/salesforce
6. DM 50 salespeople on LinkedIn with free trial offer
7. Collect testimonials from first 5 customers

## Build Time: 3 days
## First Revenue: Week 1