# Phase 1: MVP - Voice Quoting (Months 1-3)

## Overview
Build the core voice-to-quote functionality that allows tradies to record a voice note and receive a professional PDF quote in under 2 minutes.

---

## Week 1-2: Project Setup

### Infrastructure
- [ ] AWS account setup (Sydney region)
- [ ] Terraform/CDK for infrastructure as code
- [ ] VPC, subnets, security groups
- [ ] RDS PostgreSQL instance
- [ ] ElastiCache Redis
- [ ] S3 buckets (audio, PDFs, assets)
- [ ] Route 53 domain setup

### Development Environment
- [ ] GitHub repository structure
- [ ] Docker/Docker Compose for local dev
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Staging environment
- [ ] Production environment

### Documentation
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Architecture decision records (ADRs)
- [ ] Developer onboarding guide

---

## Week 3-4: Database & Auth

### Database Schema
```sql
-- Core tables for MVP
CREATE TABLE businesses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    abn VARCHAR(20),
    trade_type VARCHAR(50),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID REFERENCES businesses(id),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    role VARCHAR(20) DEFAULT 'owner',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID REFERENCES businesses(id),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID REFERENCES businesses(id),
    customer_id UUID REFERENCES customers(id),
    voice_note_url VARCHAR(500),
    transcript TEXT,
    line_items JSONB,
    subtotal DECIMAL(10,2),
    gst DECIMAL(10,2),
    total DECIMAL(10,2),
    status VARCHAR(20) DEFAULT 'draft',
    pdf_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW(),
    sent_at TIMESTAMP,
    accepted_at TIMESTAMP
);
```

### Authentication
- [ ] JWT implementation
- [ ] Password hashing (bcrypt)
- [ ] Registration endpoint
- [ ] Login endpoint
- [ ] Password reset flow
- [ ] Email verification

---

## Week 5-6: Voice Processing

### Audio Upload
```python
# api/routes/voice.py
from fastapi import APIRouter, UploadFile, File, Depends
from services.audio import AudioProcessor
from services.auth import get_current_user

router = APIRouter()

@router.post("/voice/upload")
async def upload_voice(
    file: UploadFile = File(...),
    current_user: User = Depends(get_current_user)
):
    """
    Upload voice note and process to quote
    """
    processor = AudioProcessor()
    
    # Save audio to S3
    audio_url = await processor.save_audio(file)
    
    # Transcribe with Whisper
    transcript = await processor.transcribe(audio_url)
    
    # Store voice note record
    voice_note = await VoiceNote.create(
        business_id=current_user.business_id,
        audio_url=audio_url,
        transcript=transcript,
        status='processing'
    )
    
    return {
        'id': voice_note.id,
        'transcript': transcript,
        'status': 'processing'
    }
```

### Whisper Integration
```python
# services/audio.py
import openai
from config import settings

class AudioProcessor:
    def __init__(self):
        openai.api_key = settings.OPENAI_API_KEY
    
    async def transcribe(self, audio_url: str) -> str:
        """
        Transcribe audio using OpenAI Whisper
        """
        # Download audio from S3
        audio_data = await self.download_from_s3(audio_url)
        
        # Transcribe
        response = await openai.Audio.atranscribe(
            model="whisper-1",
            file=audio_data,
            language="en"
        )
        
        return response['text']
```

### Intent Classification
```python
# services/nlp.py
from openai import AsyncOpenAI

class IntentClassifier:
    def __init__(self):
        self.client = AsyncOpenAI()
    
    async def classify(self, transcript: str) -> dict:
        """
        Classify intent and extract entities from transcript
        """
        prompt = f"""
        Analyze this tradie voice note and extract:
        1. Intent (CREATE_QUOTE, UPDATE_JOB, etc.)
        2. Job type (plumbing, electrical, carpentry, etc.)
        3. Description of work
        4. Materials needed
        5. Estimated hours
        6. Customer info (if mentioned)
        
        Transcript: {transcript}
        
        Return as JSON.
        """
        
        response = await self.client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )
        
        return json.loads(response.choices[0].message.content)
```

---

## Week 7-8: Quote Generation

### Pricing Engine
```python
# services/pricing.py
class PricingEngine:
    """
    Calculate pricing based on trade type, materials, labor
    """
    
    TRADE_RATES = {
        'plumbing': {'hourly': 120, 'callout': 95},
        'electrical': {'hourly': 110, 'callout': 85},
        'carpentry': {'hourly': 95, 'callout': 75},
        'hvac': {'hourly': 130, 'callout': 110},
        'general': {'hourly': 85, 'callout': 65}
    }
    
    async def calculate_quote(
        self,
        trade_type: str,
        description: str,
        materials: list,
        estimated_hours: float
    ) -> dict:
        """
        Generate line items and totals
        """
        rates = self.TRADE_RATES.get(trade_type, self.TRADE_RATES['general'])
        
        line_items = []
        
        # Callout fee
        line_items.append({
            'description': 'Callout fee',
            'quantity': 1,
            'unit_price': rates['callout'],
            'total': rates['callout']
        })
        
        # Labor
        labor_cost = estimated_hours * rates['hourly']
        line_items.append({
            'description': f'Labor ({estimated_hours} hours)',
            'quantity': estimated_hours,
            'unit_price': rates['hourly'],
            'total': labor_cost
        })
        
        # Materials (with markup)
        for material in materials:
            cost_price = await self.get_material_cost(material)
            sell_price = cost_price * 1.3  # 30% markup
            line_items.append({
                'description': material,
                'quantity': 1,
                'unit_price': sell_price,
                'total': sell_price
            })
        
        # Calculate totals
        subtotal = sum(item['total'] for item in line_items)
        gst = subtotal * 0.10  # 10% GST
        total = subtotal + gst
        
        return {
            'line_items': line_items,
            'subtotal': subtotal,
            'gst': gst,
            'total': total
        }
```

### PDF Generation
```python
# services/pdf.py
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph, Spacer
from reportlab.lib.styles import getSampleStyleSheet

class PDFGenerator:
    async def generate_quote_pdf(
        self,
        business: Business,
        customer: Customer,
        quote: Quote
    ) -> str:
        """
        Generate branded PDF quote
        """
        filename = f"quote_{quote.id}.pdf"
        filepath = f"/tmp/{filename}"
        
        doc = SimpleDocTemplate(filepath, pagesize=A4)
        styles = getSampleStyleSheet()
        story = []
        
        # Header
        story.append(Paragraph(business.name, styles['Heading1']))
        story.append(Paragraph(f"ABN: {business.abn}", styles['Normal']))
        story.append(Spacer(1, 20))
        
        # Quote details
        story.append(Paragraph("QUOTE", styles['Heading2']))
        story.append(Paragraph(f"Quote #: {quote.id}", styles['Normal']))
        story.append(Paragraph(f"Date: {quote.created_at.strftime('%d/%m/%Y')}", styles['Normal']))
        story.append(Spacer(1, 20))
        
        # Customer info
        story.append(Paragraph("To:", styles['Heading3']))
        story.append(Paragraph(customer.name, styles['Normal']))
        story.append(Paragraph(customer.address, styles['Normal']))
        story.append(Spacer(1, 20))
        
        # Line items table
        data = [['Description', 'Qty', 'Unit Price', 'Total']]
        for item in quote.line_items:
            data.append([
                item['description'],
                str(item['quantity']),
                f"${item['unit_price']:.2f}",
                f"${item['total']:.2f}"
            ])
        
        # Totals
        data.append(['', '', 'Subtotal:', f"${quote.subtotal:.2f}"])
        data.append(['', '', 'GST (10%):', f"${quote.gst:.2f}"])
        data.append(['', '', 'TOTAL:', f"${quote.total:.2f}"])
        
        table = Table(data)
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 14),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('BACKGROUND', (0, -3), (-1, -1), colors.beige),
            ('GRID', (0, 0), (-1, -1), 1, colors.black)
        ]))
        
        story.append(table)
        
        # Terms
        story.append(Spacer(1, 30))
        story.append(Paragraph("Terms & Conditions", styles['Heading3']))
        story.append(Paragraph("Valid for 14 days. Payment due within 14 days of invoice.", styles['Normal']))
        
        doc.build(story)
        
        # Upload to S3
        s3_url = await self.upload_to_s3(filepath, filename)
        return s3_url
```

---

## Week 9-10: Mobile App

### React Native Setup
```bash
npx react-native init TradieMindMobile --template react-native-template-typescript
cd TradieMindMobile
npm install @react-navigation/native @react-navigation/stack
npm install react-native-screens react-native-safe-area-context
npm install @react-native-voice/voice
npm install react-native-fs
npm install axios
```

### Voice Recording Screen
```typescript
// src/screens/VoiceScreen.tsx
import React, { useState, useEffect } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import Voice from '@react-native-voice/voice';

export const VoiceScreen: React.FC = () => {
  const [isRecording, setIsRecording] = useState(false);
  const [transcript, setTranscript] = useState('');
  
  useEffect(() => {
    Voice.onSpeechResults = (e) => {
      setTranscript(e.value[0]);
    };
    
    return () => {
      Voice.destroy().then(Voice.removeAllListeners);
    };
  }, []);
  
  const startRecording = async () => {
    try {
      await Voice.start('en-AU');
      setIsRecording(true);
    } catch (e) {
      console.error(e);
    }
  };
  
  const stopRecording = async () => {
    try {
      await Voice.stop();
      setIsRecording(false);
      
      // Upload to server
      await uploadVoiceNote(transcript);
    } catch (e) {
      console.error(e);
    }
  };
  
  return (
    <View style={styles.container}>
      <Text style={styles.title}>Record Voice Note</Text>
      
      <TouchableOpacity
        style={[styles.recordButton, isRecording && styles.recording]}
        onPressIn={startRecording}
        onPressOut={stopRecording}
      >
        <Text style={styles.buttonText}>
          {isRecording ? 'Recording...' : 'Hold to Record'}
        </Text>
      </TouchableOpacity>
      
      {transcript ? (
        <View style={styles.transcriptContainer}>
          <Text style={styles.transcriptLabel}>Transcript:</Text>
          <Text style={styles.transcript}>{transcript}</Text>
        </View>
      ) : null}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#1a1a2e'
  },
  title: {
    fontSize: 24,
    color: '#fff',
    marginBottom: 30
  },
  recordButton: {
    width: 150,
    height: 150,
    borderRadius: 75,
    backgroundColor: '#7c5dff',
    justifyContent: 'center',
    alignItems: 'center',
    alignSelf: 'center'
  },
  recording: {
    backgroundColor: '#ff6b6b'
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: 'bold'
  },
  transcriptContainer: {
    marginTop: 30,
    padding: 15,
    backgroundColor: 'rgba(255,255,255,0.1)',
    borderRadius: 10
  },
  transcriptLabel: {
    color: '#888',
    marginBottom: 10
  },
  transcript: {
    color: '#fff',
    fontSize: 16
  }
});
```

---

## Week 11-12: Integration & Testing

### Xero Integration
```python
# services/integrations/xero.py
from xero_python.accounting import AccountingApi
from xero_python.api_client import ApiClient

class XeroIntegration:
    def __init__(self, access_token: str, tenant_id: str):
        self.client = ApiClient(
            configuration={'access_token': access_token}
        )
        self.accounting_api = AccountingApi(self.client)
        self.tenant_id = tenant_id
    
    async def create_contact(self, customer: Customer):
        """Create or update customer in Xero"""
        contacts = {
            'contacts': [{
                'name': customer.name,
                'emailAddress': customer.email,
                'phones': [{
                    'phoneType': 'MOBILE',
                    'phoneNumber': customer.phone
                }]
            }]
        }
        
        response = await self.accounting_api.create_contacts(
            self.tenant_id,
            contacts
        )
        return response.contacts[0].contact_id
    
    async def create_invoice(self, quote: Quote, customer: Customer):
        """Create invoice from quote"""
        line_items = [
            {
                'description': item['description'],
                'quantity': item['quantity'],
                'unitAmount': item['unit_price'],
                'accountCode': '200'  # Sales
            }
            for item in quote.line_items
        ]
        
        invoices = {
            'invoices': [{
                'type': 'ACCREC',
                'contact': {'name': customer.name},
                'lineItems': line_items,
                'date': quote.created_at.strftime('%Y-%m-%d'),
                'dueDate': (quote.created_at + timedelta(days=14)).strftime('%Y-%m-%d'),
                'status': 'AUTHORISED'
            }]
        }
        
        response = await self.accounting_api.create_invoices(
            self.tenant_id,
            invoices
        )
        return response.invoices[0].invoice_id
```

### Testing Checklist
- [ ] Voice upload works on iOS
- [ ] Voice upload works on Android
- [ ] Transcription accuracy > 80%
- [ ] Quote generation < 2 minutes
- [ ] PDF looks professional
- [ ] Email delivery works
- [ ] Xero sync works
- [ ] 5 beta users onboarded

---

## Beta Program

### Recruiting Beta Users
**Target:** 5 tradies (mix of trades)
- 2 plumbers
- 2 electricians
- 1 carpenter/general

**Incentives:**
- Free lifetime Solo plan
- Direct access to founders
- Shape product direction

### Feedback Collection
- Daily usage logs
- Weekly check-in calls
- In-app feedback button
- NPS survey

### Success Criteria
- 80%+ use it for real quotes
- < 5 min average quote time
- 90%+ satisfaction score
- Zero critical bugs

---

## Deliverables

By end of Month 3:
1. ✅ Working iOS & Android apps
2. ✅ Voice-to-quote in < 2 minutes
3. ✅ Professional PDF generation
4. ✅ Xero/MYOB integration
5. ✅ 5 active beta users
6. ✅ Demo video for investors

---

*Next: Phase 2 - V1.0 Full Admin Automation*
