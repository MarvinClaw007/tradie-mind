# TradieMind AI - Technical Specification

## System Architecture

### High-Level Overview
```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                          │
├─────────────────┬─────────────────┬─────────────────────────┤
│  iOS App        │  Android App    │  Web Dashboard          │
│  (React Native) │  (React Native) │  (Next.js)              │
└────────┬────────┴────────┬────────┴──────────┬──────────────┘
         │                 │                   │
         └─────────────────┼───────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    API GATEWAY (FastAPI)                     │
│  - Authentication (JWT)                                      │
│  - Rate limiting                                             │
│  - Request routing                                           │
└──────────┬───────────────┬───────────────┬───────────────────┘
           │               │               │
┌──────────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
│   AI ENGINE     │ │  BUSINESS   │ │ INTEGRATION │
│   (Python)      │ │   LOGIC     │ │   LAYER     │
├─────────────────┤ ├─────────────┤ ├─────────────┤
│ • Speech-to-Text│ │ • Quotes    │ │ • Xero      │
│ • NLP/LLM       │ │ • Scheduling│ │ • MYOB      │
│ • Quote Gen     │ │ • Invoicing │ │ • Stripe    │
│ • Scheduling AI │ │ • Payments  │ │ • Twilio    │
└────────┬────────┘ └──────┬──────┘ └──────┬──────┘
         │                 │               │
         └─────────────────┼───────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                    DATA LAYER                                │
├─────────────────┬─────────────────┬─────────────────────────┤
│  PostgreSQL     │  Redis          │  S3/Cloud Storage       │
│  (Primary DB)   │  (Cache/Queue)  │  (Files/Audio)          │
└─────────────────┴─────────────────┴─────────────────────────┘
```

---

## Core Components

### 1. Voice Processing Pipeline

```python
class VoiceProcessor:
    """
    Handles voice input from tradies in the field
    """
    
    async def process_voice_note(self, audio_file: bytes) -> dict:
        # Step 1: Speech-to-text (Whisper)
        transcript = await self.transcribe(audio_file)
        
        # Step 2: Intent classification
        intent = await self.classify_intent(transcript)
        
        # Step 3: Entity extraction
        entities = await self.extract_entities(transcript, intent)
        
        # Step 4: Action execution
        result = await self.execute_action(intent, entities)
        
        return {
            'transcript': transcript,
            'intent': intent,
            'entities': entities,
            'result': result
        }
```

**Intents:**
- `CREATE_QUOTE` - Generate new quote
- `SCHEDULE_JOB` - Book a job
- `LOG_EXPENSE` - Record expense
- `UPDATE_JOB_STATUS` - Mark job complete
- `FOLLOW_UP_PAYMENT` - Chase payment

---

### 2. Quote Generation System

```python
class QuoteGenerator:
    """
    AI-powered quote generation from voice descriptions
    """
    
    async def generate_quote(self, voice_description: str, customer_id: str) -> Quote:
        # Extract job details
        details = await self.extract_job_details(voice_description)
        
        # Calculate pricing
        pricing = await self.calculate_pricing(
            trade_type=details.trade,
            materials=details.materials,
            labor_hours=details.estimated_hours,
            complexity=details.complexity
        )
        
        # Generate PDF
        pdf = await self.generate_pdf_quote(
            customer=customer_id,
            items=pricing.line_items,
            total=pricing.total,
            terms=pricing.terms
        )
        
        return Quote(
            id=generate_uuid(),
            customer_id=customer_id,
            items=pricing.line_items,
            total=pricing.total,
            pdf_url=pdf.url,
            status='sent'
        )
```

**Quote Template:**
- Company branding (logo, colors)
- Itemized breakdown
- Terms & conditions
- Expiry date (14 days default)
- Accept/Decline buttons

---

### 3. AI Scheduling Engine

```python
class SchedulingEngine:
    """
    Optimizes job scheduling based on multiple factors
    """
    
    def optimize_schedule(self, jobs: List[Job], constraints: Constraints) -> Schedule:
        # Factors to optimize:
        # - Geographic proximity (minimize drive time)
        # - Job urgency/priority
        # - Staff availability and skills
        # - Customer preferences
        # - Equipment requirements
        
        optimizer = RouteOptimizer()
        
        return optimizer.solve(
            jobs=jobs,
            staff=constraints.staff,
            vehicles=constraints.vehicles,
            time_windows=constraints.time_windows,
            objective='minimize_drive_time'
        )
```

**Optimization Goals:**
1. Minimize total drive time
2. Maximize jobs per day
3. Respect time windows
4. Balance workload across team

---

### 4. Autonomous Invoicing

```python
class InvoicingSystem:
    """
    Automatic invoice generation and payment chasing
    """
    
    async def on_job_complete(self, job_id: str):
        job = await self.get_job(job_id)
        
        # Generate invoice
        invoice = await self.generate_invoice(job)
        
        # Send to customer
        await self.send_invoice(invoice)
        
        # Schedule follow-ups
        await self.schedule_payment_reminders(invoice.id)
        
        # Sync to accounting
        await self.sync_to_xero(invoice)
    
    async def payment_reminder(self, invoice_id: str, days_overdue: int):
        templates = {
            7: 'friendly_reminder',
            14: 'firm_reminder', 
            21: 'final_notice',
            30: 'collections_handoff'
        }
        
        template = templates.get(days_overdue, 'friendly_reminder')
        await self.send_reminder_email(invoice_id, template)
```

**Payment Schedule:**
- Day 0: Invoice sent
- Day 7: Friendly reminder
- Day 14: Firm reminder
- Day 21: Final notice
- Day 30: Collections/handoff

---

### 5. AI Phone Answering

```python
class PhoneAnsweringSystem:
    """
    24/7 AI phone receptionist
    """
    
    async def handle_call(self, phone_number: str, call_sid: str):
        # Answer with Australian accent
        greeting = self.generate_greeting()
        
        # Conversational AI
        conversation = ConversationManager(
            voice='en-AU-Neural2-B',  # Australian accent
            personality='professional_friendly'
        )
        
        # Handle common scenarios
        intent = await conversation.detect_intent()
        
        handlers = {
            'NEW_ENQUIRY': self.handle_new_enquiry,
            'BOOK_JOB': self.handle_booking,
            'CHECK_STATUS': self.handle_status_check,
            'LEAVE_MESSAGE': self.handle_message,
            'EMERGENCY': self.handle_emergency
        }
        
        handler = handlers.get(intent, self.handle_general)
        return await handler(conversation)
```

**Capabilities:**
- Answer common questions
- Qualify leads (budget, timeline, scope)
- Book jobs directly into calendar
- Take messages and notify tradie
- Handle emergencies (escalate to on-call)

---

## Database Schema

### Core Tables

```sql
-- Tradies/Businesses
CREATE TABLE businesses (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    abn VARCHAR(20),
    trade_type VARCHAR(50),
    subscription_tier VARCHAR(20),
    created_at TIMESTAMP
);

-- Staff/Team Members
CREATE TABLE staff (
    id UUID PRIMARY KEY,
    business_id UUID REFERENCES businesses(id),
    name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    role VARCHAR(50),
    trade_skills JSONB,
    availability JSONB
);

-- Customers
CREATE TABLE customers (
    id UUID PRIMARY KEY,
    business_id UUID REFERENCES businesses(id),
    name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    location GEOGRAPHY(POINT),
    payment_terms INTEGER DEFAULT 14
);

-- Jobs
CREATE TABLE jobs (
    id UUID PRIMARY KEY,
    business_id UUID REFERENCES businesses(id),
    customer_id UUID REFERENCES customers(id),
    title VARCHAR(255),
    description TEXT,
    status VARCHAR(20),
    priority INTEGER,
    scheduled_date DATE,
    scheduled_time TIME,
    estimated_hours DECIMAL,
    actual_hours DECIMAL,
    location GEOGRAPHY(POINT),
    assigned_staff UUID[],
    materials JSONB,
    created_at TIMESTAMP
);

-- Quotes
CREATE TABLE quotes (
    id UUID PRIMARY KEY,
    job_id UUID REFERENCES jobs(id),
    customer_id UUID REFERENCES customers(id),
    line_items JSONB,
    subtotal DECIMAL,
    gst DECIMAL,
    total DECIMAL,
    status VARCHAR(20),
    expiry_date DATE,
    pdf_url VARCHAR(500),
    accepted_at TIMESTAMP
);

-- Invoices
CREATE TABLE invoices (
    id UUID PRIMARY KEY,
    job_id UUID REFERENCES jobs(id),
    customer_id UUID REFERENCES customers(id),
    quote_id UUID REFERENCES quotes(id),
    invoice_number VARCHAR(50),
    amount DECIMAL,
    gst DECIMAL,
    total DECIMAL,
    status VARCHAR(20),
    sent_at TIMESTAMP,
    paid_at TIMESTAMP,
    due_date DATE
);

-- Voice Notes
CREATE TABLE voice_notes (
    id UUID PRIMARY KEY,
    business_id UUID REFERENCES businesses(id),
    staff_id UUID REFERENCES staff(id),
    audio_url VARCHAR(500),
    transcript TEXT,
    intent VARCHAR(50),
    entities JSONB,
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP
);

-- Call Logs (AI Phone)
CREATE TABLE call_logs (
    id UUID PRIMARY KEY,
    business_id UUID REFERENCES businesses(id),
    phone_number VARCHAR(20),
    call_sid VARCHAR(100),
    direction VARCHAR(10),
    duration INTEGER,
    transcript TEXT,
    outcome VARCHAR(50),
    recording_url VARCHAR(500),
    created_at TIMESTAMP
);
```

---

## API Endpoints

### Authentication
```
POST /auth/register
POST /auth/login
POST /auth/refresh
POST /auth/forgot-password
```

### Voice
```
POST /voice/upload
GET  /voice/:id/transcript
POST /voice/:id/process
```

### Quotes
```
POST /quotes
GET  /quotes
GET  /quotes/:id
PUT  /quotes/:id
POST /quotes/:id/send
POST /quotes/:id/accept
POST /quotes/:id/decline
```

### Jobs
```
POST /jobs
GET  /jobs
GET  /jobs/:id
PUT  /jobs/:id
DELETE /jobs/:id
POST /jobs/:id/complete
```

### Scheduling
```
GET  /schedule/optimize
POST /schedule/batch
GET  /schedule/availability
```

### Invoicing
```
POST /invoices
GET  /invoices
GET  /invoices/:id
POST /invoices/:id/send
POST /invoices/:id/mark-paid
```

### Phone
```
POST /phone/webhook
GET  /phone/calls
GET  /phone/calls/:id
```

---

## Integrations

### Xero
```python
class XeroIntegration:
    async def sync_invoice(self, invoice: Invoice):
        xero_invoice = {
            'Type': 'ACCREC',
            'Contact': {'Name': invoice.customer.name},
            'LineItems': [
                {
                    'Description': item.description,
                    'Quantity': item.quantity,
                    'UnitAmount': item.unit_price,
                    'AccountCode': item.account_code
                }
                for item in invoice.line_items
            ]
        }
        
        return await self.xero_api.invoices.put(xero_invoice)
```

### Stripe
```python
class StripeIntegration:
    async def create_payment_intent(self, invoice: Invoice):
        return await stripe.payment_intents.create(
            amount=int(invoice.total * 100),  # cents
            currency='aud',
            customer=invoice.customer.stripe_id,
            metadata={'invoice_id': invoice.id}
        )
```

### Twilio
```python
class TwilioIntegration:
    async def make_call(self, phone_number: str, script: str):
        return await self.twilio.calls.create(
            to=phone_number,
            from_=self.twilio_number,
            twiml=self.generate_twiml(script)
        )
```

---

## Security

### Authentication
- JWT tokens with 24h expiry
- Refresh token rotation
- MFA for admin accounts

### Data Protection
- AES-256 encryption at rest
- TLS 1.3 in transit
- Field-level encryption for sensitive data (ABN, bank details)

### Compliance
- Australian Privacy Principles (APP)
- GDPR compliance for international
- SOC 2 Type II certification (target)

---

## Deployment

### Infrastructure
- **Primary:** AWS Sydney (ap-southeast-2)
- **DR:** AWS Melbourne (ap-southeast-4)
- **CDN:** CloudFront
- **DNS:** Route 53

### CI/CD
```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to AWS
        run: |
          docker build -t tradiemind:${{ github.sha }} .
          aws ecs update-service --cluster tradiemind --service api --force-new-deployment
```

### Monitoring
- **APM:** Datadog or New Relic
- **Logging:** CloudWatch Logs
- **Error Tracking:** Sentry
- **Uptime:** Pingdom

---

## Performance Targets

| Metric | Target |
|--------|--------|
| Voice-to-Quote | < 2 minutes |
| API Response | < 200ms (p95) |
| App Launch | < 3 seconds |
| Invoice Generation | < 5 seconds |
| AI Phone Answer | < 10 seconds |
| Uptime | 99.9% |

---

*This specification is a living document. Update as requirements evolve.*
