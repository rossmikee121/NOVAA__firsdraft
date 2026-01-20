# NOVAA PROJECT CHARTER

**Document Version**: 1.0  
**Date**: January 20, 2026  
**Status**: DRAFT - Ready for Stakeholder Review  

---

## 1. PROJECT OVERVIEW

### 1.1 Project Name & Acronym
**NOVAA** - Next-Generation Operations & Value-Based Academic Analytics

### 1.2 Vision Statement
To eliminate administrative friction in Indian higher education by creating a secure, compliant, multi-tenant SaaS platform that transforms routine college operations into strategic advantages for institutions navigating India's rapidly evolving educational landscape post-NEP 2020.

### 1.3 Mission Statement
Deliver a pragmatic college management system that:
- ✅ Reduces manual administrative work by 70%
- ✅ Ensures regulatory compliance (GST, DPDPA, state-specific rules)
- ✅ Provides transparent access for students and parents
- ✅ Scales from 5 pilot colleges to 500+ institutions across India

---

## 2. BUSINESS CASE

### 2.1 Problem Statement
**Current State**: Indian colleges rely on Excel sheets, manual data entry, and fragmented systems to manage:
- Student admissions with conflicting state-specific reservation rules
- Fee collection without proper GST compliance
- Attendance tracking via physical sign-ups
- Scattered reporting for NAAC/NIRF audits

**Consequences**:
- Data entry errors leading to admission disputes
- Revenue leakage due to failed payment reconciliation
- 40+ hours/month wasted on manual reporting
- Compliance violations resulting in ₹10-50 lakh fines per incident
- Physical protests when payment/admission issues arise

### 2.2 Value Proposition
| Stakeholder | Value Delivered | Quantified Benefit |
|-------------|------------------|--------------------|
| **Colleges** | Operational efficiency | 95% reduction in admission processing time |
| **Colleges** | Compliance assurance | 100% GST-compliant receipts |
| **Colleges** | Student retention | 30% fewer inquiries via proactive alerts |
| **Students** | Transparency | Real-time admission status tracking |
| **Students** | Convenience | QR-based attendance (no paper tokens) |
| **Parents** | Trust | GST-compliant digital receipts |

### 2.3 Market Opportunity
- **TAM** (Total Addressable Market): 45,000+ colleges in India
- **SAM** (Serviceable Available Market): 12,000 colleges in metro/tier-2 cities
- **SOM** (Serviceable Obtainable Market): 500 colleges within 3 years
- **Price Point**: ₹50,000-200,000/year per college (based on size)
- **Projected ARR (Year 3)**: ₹2.5-5 crore

---

## 3. SCOPE DEFINITION

### 3.1 In Scope (MVP - Launch Ready)

#### Phase 1: Admissions (Weeks 1-4)
- [ ] College registration & tenant isolation
- [ ] Student digital application form
- [ ] Document upload interface
- [ ] Admin verification workflow
- [ ] Caste-based reservation (Maharashtra rules only)

#### Phase 2: Fee Management (Weeks 5-8)
- [ ] Fee structure configuration per course
- [ ] Razorpay payment gateway integration
- [ ] GST split calculation & invoice generation
- [ ] Failed payment recovery workflow
- [ ] SMS/email notifications

#### Phase 3: Attendance (Weeks 9-10)
- [ ] QR code generation & scanning
- [ ] Staff marking interface
- [ ] Student attendance dashboard
- [ ] Basic reports & exports

#### Phase 4: Reporting (Weeks 11-12)
- [ ] Admission funnel analytics
- [ ] Fee collection summaries
- [ ] Attendance trend reports
- [ ] Excel/PDF export capabilities

### 3.2 Out of Scope (Future Versions)

| Feature | Why Not in MVP | Planned For |
|---------|----------------|-------------|
| APAAR ID integration | Requires government API setup | V2.0 |
| Multi-state rules | Only Maharashtra pilot colleges | V1.5 |
| Mobile app | Responsive web sufficient for MVP | V2.0 |
| Offline attendance | All pilot colleges have WiFi | V1.1 |
| Biometric attendance | QR codes sufficient initially | V1.5 |
| Parent portal | Phase 2 requirement | V1.1 |
| Scholarship management | Complex state-specific rules | V1.5 |

---

## 4. SUCCESS CRITERIA

### 4.1 MVP Success Metrics (End of Month 3)

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Adoption** | 5 pilot colleges live | Signed contracts + data migration complete |
| **Performance** | 95% uptime | Monitoring dashboard average |
| **User Satisfaction** | 4.5/5 stars | Post-launch survey with 50+ college users |
| **Data Quality** | 99% accuracy | Automated validation tests pass |
| **Compliance** | 100% GST-correct | External audit by CA firm |
| **Support** | <4hr resolution | Tracked in support ticketing system |

### 4.2 Financial Success Metrics

| Year | Target Users | Projected ARR | Break-even Point |
|------|--------------|---------------|------------------|
| Year 1 | 5 colleges | ₹25-50 lakhs | Not expected |
| Year 2 | 50 colleges | ₹1.25-2.5 crores | Q4 2027 |
| Year 3 | 150+ colleges | ₹3.75+ crores | Profitable |

---

## 5. PROJECT TEAM

### 5.1 Governance Structure
```
Project Sponsor (Founder/CTO)
├── Project Manager (Overall coordination)
├── Technical Lead (Architecture & code quality)
├── Compliance Officer (Regulatory adherence)
└── Product Owner (User needs & prioritization)

Development Team:
├── Backend Intern (API development)
├── Frontend Intern (UI/UX implementation)
└── Database Intern (Schema & optimization)

Support Team:
├── Quality Assurance (Testing)
└── Customer Success (Pilot college liaison)
```

### 5.2 Key Responsibilities

| Role | Key Responsibilities | Decision Authority |
|------|---------------------|-------------------|
| **PM** | Schedule, deliverables, risk management | Escalation decisions |
| **Tech Lead** | Architecture approval, code reviews, tech decisions | Technology choices |
| **Compliance Officer** | GST, DPDPA, state rules verification | Compliance approval |
| **Product Owner** | Feature prioritization, pilot college feedback | Scope decisions |

---

## 6. RISK MANAGEMENT

### 6.1 Top 5 Project Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| **Payment failures lose admissions** | High | Critical | • Daily reconciliation tests<br>• 24/7 monitoring<br>• Manual override process |
| **Data isolation breach** | Medium | Critical | • Mandatory collegeId in every query<br>• Automated code scanning<br>• Weekly security audits |
| **Pilot colleges demand state-specific rules** | Medium | High | • Clear MVP scope documentation<br>• Feature roadmap communication<br>• Early feedback loops |
| **GST compliance violation** | Low | Critical | • CA review of calculations<br>• External audit before launch<br>• Automated validation tests |
| **Staff turnover of interns** | Medium | High | • Clear documentation of all work<br>• Weekly knowledge transfers<br>• Code review standards |

### 6.2 Risk Response Plan
- **Payment failures**: Maintain backup payment processor, manual reconciliation process
- **Data isolation**: Code review checklist, automated scanning in CI/CD pipeline
- **Pilot college demands**: Product roadmap communication, clear MVP boundaries
- **GST compliance**: External CA verification, test cases for all tax scenarios
- **Staff turnover**: Daily standups, documented code, pair programming

---

## 7. TIMELINE & MILESTONES

```
WEEK 1-2: Foundation Setup
├── GitHub repo setup
├── CI/CD pipeline configuration
├── Database schema finalized
└── Local development environment ready

WEEK 3-4: Admissions Module
├── User authentication & college context
├── Application form builder
├── Document upload & verification
└── Status tracking dashboard

WEEK 5-6: Payments Module
├── Razorpay integration
├── GST calculation engine
├── Receipt generation & delivery
└── Failed payment recovery

WEEK 7-8: Attendance Module
├── QR code generation
├── QR scanning interface
├── Real-time dashboard
└── Basic reports

WEEK 9-10: Testing & Polish
├── Full system testing
├── Pilot college feedback integration
├── Performance optimization
└── Security audit

WEEK 11-12: Launch
├── Final deployment
├── Pilot college training
├── 24/7 monitoring setup
└── Go-live support
```

---

## 8. APPROVAL & SIGN-OFF

### 8.1 Stakeholder Sign-Off

| Stakeholder | Role | Signature | Date |
|-------------|------|-----------|------|
| [Name] | Founder/Project Sponsor | _____ | _____ |
| [Name] | Technical Lead | _____ | _____ |
| [Name] | Compliance Officer | _____ | _____ |
| [Name] | Product Owner | _____ | _____ |
| [Name] | Pilot College Representative | _____ | _____ |

---

## 9. DOCUMENT REFERENCES

- **Project Vision**: check this as well.md
- **Knowledge Base**: ANALYSIS_REPORT.md
- **Chat Export**: chat-export-1768911149788.json
- **Next Documents**: 
  - 02_PRODUCT_REQUIREMENTS_DOCUMENT.md
  - 03_TECHNICAL_ARCHITECTURE.md
  - 04_DATABASE_DESIGN.md
  - 05_API_SPECIFICATIONS.md
  - 06_IMPLEMENTATION_ROADMAP.md

---

**Document Owner**: Project Manager  
**Last Updated**: 2026-01-20  
**Next Review**: Weekly with stakeholders
