# BuildSense AI - Design Document

## Table of Contents
1. [System Architecture Overview](#system-architecture-overview)
2. [Architectural Principles](#architectural-principles)
3. [Component Design](#component-design)
4. [Data Architecture](#data-architecture)
5. [AI/ML Pipeline Design](#aiml-pipeline-design)
6. [API Design](#api-design)
7. [Security Architecture](#security-architecture)
8. [UI/UX Design](#uiux-design)
9. [Deployment Architecture](#deployment-architecture)
10. [Data Flow Diagrams](#data-flow-diagrams)

---

## System Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Web App  │  Mobile App  │  WhatsApp Bot  │  SMS Gateway  │  AR App │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      API GATEWAY LAYER                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Load Balancer (NGINX) + API Gateway (Kong/AWS API Gateway)  │  │
│  │  - Authentication    - Rate Limiting    - Request Routing     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     MICROSERVICES LAYER                              │
├──────────────────┬──────────────────┬──────────────────┬────────────┤
│  User Service    │  Complaint Svc   │  Engineer Svc    │  Admin Svc │
│  - Auth          │  - CRUD          │  - Assignment    │  - Analytics│
│  - Profile       │  - Status Track  │  - Reports       │  - Config   │
│  - Preferences   │  - Search        │  - Scheduling    │  - Users    │
└──────────────────┴──────────────────┴──────────────────┴────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AI/ML SERVICES LAYER                            │
├──────────────┬────────────────┬──────────────┬──────────────────────┤
│  CV Service  │  NLP Service   │  Risk Engine │  Prediction Service  │
│  - Defect    │  - Speech-to-  │  - Scoring   │  - Decay Modeling    │
│    Detection │    Text        │  - Priority  │  - Trend Analysis    │
│  - Severity  │  - Translation │  - Heatmaps  │  - Forecasting       │
│  - Duplicate │  - Sentiment   │  - Analytics │                      │
└──────────────┴────────────────┴──────────────┴──────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                      │
├──────────────┬────────────────┬──────────────┬──────────────────────┤
│  PostgreSQL  │  Redis Cache   │  S3 Storage  │  TimescaleDB         │
│  - User Data │  - Sessions    │  - Images    │  - Time Series       │
│  - Complaints│  - Rate Limits │  - Videos    │  - Analytics         │
│  - Engineers │  - Real-time   │  - Documents │  - Predictions       │
└──────────────┴────────────────┴──────────────┴──────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   INTEGRATION LAYER                                  │
├──────────────┬────────────────┬──────────────┬──────────────────────┤
│  Maps API    │  Payment       │  Messaging   │  Blockchain          │
│  - Google    │  - Razorpay    │  - Twilio    │  - Polygon           │
│  - MapMyIndia│  - Paytm       │  - WhatsApp  │  - Smart Contracts   │
└──────────────┴────────────────┴──────────────┴──────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  MONITORING & OBSERVABILITY                          │
│  Prometheus │ Grafana │ ELK Stack │ Sentry │ New Relic             │
└─────────────────────────────────────────────────────────────────────┘
```

### Architecture Style
- **Microservices Architecture:** Loosely coupled, independently deployable services
- **Event-Driven:** Asynchronous communication via message queues
- **Cloud-Native:** Containerized deployment on Kubernetes
- **API-First:** All functionality exposed via RESTful APIs

---

## Architectural Principles

### 1. Scalability First
- Horizontal scaling for all services
- Stateless services (state in cache/database)
- Auto-scaling based on load

### 2. Resilience & Fault Tolerance
- Circuit breakers for external dependencies
- Retry mechanisms with exponential backoff
- Graceful degradation (AI unavailable → manual processing)

### 3. Security by Design
- Zero trust architecture
- Encryption at rest and in transit
- Principle of least privilege

### 4. Performance Optimization
- Caching at multiple layers
- CDN for static assets
- Database query optimization
- Lazy loading and pagination

### 5. Accessibility First
- Mobile-first responsive design
- Low-bandwidth optimization
- Offline-first architecture

### 6. Data Privacy
- GDPR/DPDP compliance
- Data minimization
- Anonymization options

---

## Component Design

### 1. User Service

**Responsibility:** User authentication, authorization, and profile management

**Tech Stack:**
- Node.js + Express.js
- PostgreSQL for user data
- Redis for session management
- Passport.js for OAuth

**Key Features:**
- OTP-based authentication (via SMS/WhatsApp)
- Social login (Google, Facebook)
- Role-based access control (Citizen, Engineer, Admin)
- Profile management
- Preference settings (language, notifications)

**API Endpoints:**
```
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/verify-otp
GET    /api/users/profile
PUT    /api/users/profile
GET    /api/users/preferences
PUT    /api/users/preferences
```

**Database Schema:**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    phone_number VARCHAR(15) UNIQUE,
    email VARCHAR(255),
    name VARCHAR(255),
    role VARCHAR(20), -- citizen, engineer, admin
    language VARCHAR(10) DEFAULT 'en',
    location GEOGRAPHY(POINT),
    reputation_score INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    token VARCHAR(500),
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 2. Complaint Service

**Responsibility:** Complaint CRUD operations, status tracking, search

**Tech Stack:**
- Node.js + Express.js
- PostgreSQL + PostGIS for spatial queries
- Redis for caching popular queries
- S3 for media storage

**Key Features:**
- Complaint submission with photos/videos
- GPS-based location tagging
- Status lifecycle management
- Full-text search
- Duplicate detection integration

**API Endpoints:**
```
POST   /api/complaints
GET    /api/complaints/:id
GET    /api/complaints (with filters)
PUT    /api/complaints/:id
DELETE /api/complaints/:id
POST   /api/complaints/:id/photos
GET    /api/complaints/:id/timeline
POST   /api/complaints/:id/verify (community)
```

**Database Schema:**
```sql
CREATE TABLE complaints (
    complaint_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    title VARCHAR(255),
    description TEXT,
    category VARCHAR(50), -- pothole, crack, drainage, etc.
    location GEOGRAPHY(POINT) NOT NULL,
    address TEXT,
    risk_score INT, -- 0-100
    risk_category VARCHAR(20), -- critical, high, medium, low
    status VARCHAR(30), -- submitted, verified, assigned, in_progress, completed
    assigned_engineer_id UUID REFERENCES engineers(engineer_id),
    estimated_completion_date DATE,
    actual_completion_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE complaint_photos (
    photo_id UUID PRIMARY KEY,
    complaint_id UUID REFERENCES complaints(complaint_id),
    s3_url VARCHAR(500),
    photo_order INT,
    cv_analysis JSONB, -- CV model results
    uploaded_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE complaint_timeline (
    event_id UUID PRIMARY KEY,
    complaint_id UUID REFERENCES complaints(complaint_id),
    status VARCHAR(30),
    description TEXT,
    actor_id UUID, -- user_id or engineer_id
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_complaints_location ON complaints USING GIST(location);
CREATE INDEX idx_complaints_risk_score ON complaints(risk_score DESC);
CREATE INDEX idx_complaints_status ON complaints(status);
```

### 3. Engineer Service

**Responsibility:** Engineer assignment, inspection reports, task management

**Tech Stack:**
- Node.js + Express.js
- PostgreSQL
- PDF generation library (PDFKit)

**Key Features:**
- Engineer assignment based on location/workload
- Inspection report generation
- Task scheduling
- Performance analytics

**API Endpoints:**
```
GET    /api/engineers
GET    /api/engineers/:id
POST   /api/engineers/:id/assign
GET    /api/engineers/:id/complaints (assigned)
POST   /api/engineers/reports/generate
GET    /api/engineers/:id/performance
```

**Database Schema:**
```sql
CREATE TABLE engineers (
    engineer_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    employee_id VARCHAR(50) UNIQUE,
    specialization VARCHAR(100),
    jurisdiction_ward VARCHAR(50),
    workload_capacity INT DEFAULT 10,
    current_workload INT DEFAULT 0,
    performance_rating DECIMAL(3,2),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE engineer_assignments (
    assignment_id UUID PRIMARY KEY,
    complaint_id UUID REFERENCES complaints(complaint_id),
    engineer_id UUID REFERENCES engineers(engineer_id),
    assigned_at TIMESTAMP,
    deadline TIMESTAMP,
    status VARCHAR(30)
);

CREATE TABLE inspection_reports (
    report_id UUID PRIMARY KEY,
    complaint_id UUID REFERENCES complaints(complaint_id),
    engineer_id UUID REFERENCES engineers(engineer_id),
    findings TEXT,
    recommendations TEXT,
    estimated_cost DECIMAL(12,2),
    materials_needed JSONB,
    pdf_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 4. Computer Vision Service

**Responsibility:** Image analysis for infrastructure defect detection

**Tech Stack:**
- Python 3.10+
- TensorFlow / PyTorch
- OpenCV for preprocessing
- FastAPI for REST endpoints
- GPU instance (AWS p3 / Azure NV series)

**ML Model Architecture:**
- **Object Detection:** YOLOv8 or EfficientDet
- **Segmentation:** U-Net for crack detection
- **Classification:** ResNet-50 for severity classification

**Training Data:**
- Public datasets: CrackForest, RDD2020 (Road Damage Detection)
- Synthetic data augmentation
- Custom labeled dataset from pilot cities

**Key Features:**
- Defect type detection (potholes, cracks, etc.)
- Severity level classification
- Size estimation
- Repair type recommendation

**API Endpoints:**
```
POST   /api/cv/analyze
POST   /api/cv/detect-defect
POST   /api/cv/estimate-severity
POST   /api/cv/check-duplicate
```

**Request/Response Example:**
```json
// Request
POST /api/cv/analyze
{
  "image_url": "s3://bucket/complaints/12345.jpg",
  "complaint_id": "uuid-here"
}

// Response
{
  "defect_type": "pothole",
  "confidence": 0.94,
  "severity": "medium",
  "severity_score": 65,
  "size_estimate": {
    "length_cm": 45,
    "width_cm": 30,
    "depth_cm": 8
  },
  "repair_recommendation": {
    "method": "cold_patching",
    "materials": ["cold_mix_asphalt", "tack_coat"],
    "estimated_cost": 2500
  },
  "affected_area_sqm": 0.135,
  "processing_time_ms": 1200
}
```

**Model Versioning:**
- MLflow for experiment tracking
- Model registry with A/B testing capability
- Continuous training pipeline

### 5. NLP Service

**Responsibility:** Speech-to-text, translation, sentiment analysis

**Tech Stack:**
- Python 3.10+
- Hugging Face Transformers
- Google Speech-to-Text API / Azure Cognitive Services
- IndicTrans for Indian language translation
- FastAPI

**Key Features:**
- Multilingual speech-to-text (10+ languages)
- Auto-translation to English/Hindi
- Sentiment analysis of complaints
- Entity extraction (location, infrastructure type)

**API Endpoints:**
```
POST   /api/nlp/speech-to-text
POST   /api/nlp/translate
POST   /api/nlp/analyze-sentiment
POST   /api/nlp/extract-entities
```

**Request/Response Example:**
```json
// Request
POST /api/nlp/speech-to-text
{
  "audio_url": "s3://bucket/voice/12345.mp3",
  "language": "hi" // Hindi
}

// Response
{
  "transcription": "मेरे घर के सामने बहुत बड़ा गड्ढा है",
  "translation": "There is a very large pothole in front of my house",
  "confidence": 0.91,
  "detected_language": "hi",
  "entities": {
    "defect_type": "pothole",
    "severity": "large",
    "location": "in front of house"
  },
  "sentiment": "negative"
}
```

### 6. Risk Scoring Engine

**Responsibility:** Calculate dynamic risk scores for complaints

**Tech Stack:**
- Python 3.10+
- Scikit-learn / XGBoost
- Redis for caching external data
- FastAPI

**Risk Calculation Algorithm:**

```python
def calculate_risk_score(complaint_data):
    """
    Multi-factor risk scoring algorithm
    Output: Risk score (0-100)
    """
    weights = {
        'severity': 0.25,
        'population_density': 0.20,
        'traffic_load': 0.15,
        'infrastructure_age': 0.10,
        'weather_vulnerability': 0.10,
        'historical_failures': 0.10,
        'public_safety': 0.10
    }
    
    scores = {}
    
    # 1. Severity (from CV model)
    severity_map = {'critical': 100, 'high': 75, 'medium': 50, 'low': 25}
    scores['severity'] = severity_map.get(complaint_data['cv_severity'], 50)
    
    # 2. Population Density
    pop_density = get_population_density(complaint_data['location'])
    scores['population_density'] = min(pop_density / 10000 * 100, 100)
    
    # 3. Traffic Load
    traffic = get_traffic_intensity(complaint_data['location'])
    scores['traffic_load'] = min(traffic / 5000 * 100, 100)
    
    # 4. Infrastructure Age
    age = get_infrastructure_age(complaint_data['location'])
    scores['infrastructure_age'] = min(age / 30 * 100, 100)
    
    # 5. Weather Vulnerability
    weather_risk = get_monsoon_vulnerability(complaint_data['location'])
    scores['weather_vulnerability'] = weather_risk
    
    # 6. Historical Failures
    history = get_failure_history(complaint_data['location'], 
                                   complaint_data['category'])
    scores['historical_failures'] = min(history * 10, 100)
    
    # 7. Public Safety (hospitals, schools nearby)
    safety_factor = calculate_safety_proximity(complaint_data['location'])
    scores['public_safety'] = safety_factor
    
    # Weighted sum
    risk_score = sum(scores[key] * weights[key] for key in weights)
    
    return {
        'risk_score': round(risk_score, 2),
        'risk_category': categorize_risk(risk_score),
        'component_scores': scores,
        'estimated_resolution_days': estimate_resolution_time(risk_score)
    }

def categorize_risk(score):
    if score >= 80:
        return 'critical'
    elif score >= 60:
        return 'high'
    elif score >= 40:
        return 'medium'
    else:
        return 'low'
```

**API Endpoints:**
```
POST   /api/risk/calculate
GET    /api/risk/heatmap
GET    /api/risk/trends
```

### 7. Prediction Service

**Responsibility:** Predictive infrastructure decay modeling

**Tech Stack:**
- Python 3.10+
- Time series models: Prophet, LSTM
- Scikit-learn for feature engineering
- TimescaleDB for historical data
- FastAPI

**ML Model:**
- **Input Features:**
  - Historical complaint frequency
  - Material type and age
  - Traffic load (vehicles/day)
  - Weather patterns (rainfall, temperature)
  - Soil conditions
  - Last maintenance date

- **Output:**
  - Probability of failure (0-1)
  - Predicted failure date
  - Confidence interval

**Training Pipeline:**
```python
class InfrastructureDecayPredictor:
    def __init__(self):
        self.model = XGBRegressor()
        
    def prepare_features(self, location_id):
        return {
            'days_since_last_repair': ...,
            'material_type_encoded': ...,
            'avg_daily_traffic': ...,
            'total_rainfall_30d': ...,
            'infrastructure_age_days': ...,
            'soil_type_encoded': ...,
            'complaint_frequency_30d': ...
        }
    
    def predict(self, location_id, horizon_days=60):
        features = self.prepare_features(location_id)
        failure_probability = self.model.predict(features)
        
        return {
            'location_id': location_id,
            'prediction_horizon_days': horizon_days,
            'failure_probability': failure_probability,
            'confidence_interval': (lower, upper),
            'recommended_action': self.get_recommendation(failure_probability)
        }
```

**API Endpoints:**
```
POST   /api/predict/infrastructure-decay
GET    /api/predict/hotspots (preventive maintenance zones)
GET    /api/predict/trends/:location_id
```

### 8. Notification Service

**Responsibility:** Multi-channel notifications (push, SMS, WhatsApp, email)

**Tech Stack:**
- Node.js + Express.js
- RabbitMQ for message queue
- Twilio for SMS
- WhatsApp Business API
- Firebase Cloud Messaging (FCM) for push

**Notification Types:**
- Status update notifications
- Assignment notifications (engineers)
- Deadline reminders
- Emergency alerts
- Community verification requests

**API Endpoints:**
```
POST   /api/notifications/send
POST   /api/notifications/bulk
GET    /api/notifications/user/:user_id
PUT    /api/notifications/:id/mark-read
```

### 9. Analytics Service

**Responsibility:** Business intelligence, reporting, dashboards

**Tech Stack:**
- Python + FastAPI
- TimescaleDB for time-series data
- Apache Superset for visualization
- Pandas for data processing

**Key Metrics:**
- Average resolution time by ward
- Engineer performance metrics
- Budget utilization analysis
- Complaint trends over time
- Citizen satisfaction scores

**API Endpoints:**
```
GET    /api/analytics/dashboard/public
GET    /api/analytics/dashboard/admin
GET    /api/analytics/reports/monthly
GET    /api/analytics/budget-optimization
```

### 10. Blockchain Service

**Responsibility:** Immutable audit trail, smart contracts

**Tech Stack:**
- Solidity for smart contracts
- Polygon blockchain (low fees)
- ethers.js for Web3 integration
- IPFS for decentralized storage

**Smart Contract Example:**
```solidity
// ComplaintAuditTrail.sol
pragma solidity ^0.8.0;

contract ComplaintAuditTrail {
    struct ComplaintRecord {
        string complaintId;
        uint256 timestamp;
        string status;
        address updatedBy;
        string ipfsHash; // for photos/documents
    }
    
    mapping(string => ComplaintRecord[]) public complaintHistory;
    
    event StatusUpdated(
        string indexed complaintId,
        string newStatus,
        address updatedBy,
        uint256 timestamp
    );
    
    function recordStatusChange(
        string memory complaintId,
        string memory newStatus,
        string memory ipfsHash
    ) public {
        ComplaintRecord memory record = ComplaintRecord({
            complaintId: complaintId,
            timestamp: block.timestamp,
            status: newStatus,
            updatedBy: msg.sender,
            ipfsHash: ipfsHash
        });
        
        complaintHistory[complaintId].push(record);
        emit StatusUpdated(complaintId, newStatus, msg.sender, block.timestamp);
    }
    
    function getComplaintHistory(string memory complaintId) 
        public view returns (ComplaintRecord[] memory) {
        return complaintHistory[complaintId];
    }
}
```

**API Endpoints:**
```
POST   /api/blockchain/record-event
GET    /api/blockchain/audit-trail/:complaint_id
POST   /api/blockchain/smart-contract/deploy
```

---

## Data Architecture

### Database Design Philosophy

1. **Polyglot Persistence:** Right database for right use case
   - PostgreSQL: Relational data (users, complaints)
   - PostGIS: Spatial queries
   - Redis: Caching, sessions
   - TimescaleDB: Time-series analytics
   - S3: Media storage

2. **Data Partitioning:**
   - Complaints table partitioned by month
   - Analytics tables partitioned by date range

3. **Indexing Strategy:**
   - B-tree indexes on primary keys, foreign keys
   - GiST indexes for geospatial queries
   - Full-text indexes for search

### Entity Relationship Diagram

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Users     │────────▶│  Complaints  │◀────────│  Engineers  │
│             │ 1     * │              │ *     1 │             │
│ - user_id   │         │ - complaint_id│         │ - engineer_id│
│ - phone     │         │ - location   │         │ - workload  │
│ - role      │         │ - risk_score │         │ - rating    │
└─────────────┘         └──────────────┘         └─────────────┘
                               │
                               │ 1
                               │
                               │ *
                        ┌──────────────┐
                        │   Photos     │
                        │              │
                        │ - photo_id   │
                        │ - s3_url     │
                        │ - cv_analysis│
                        └──────────────┘
```

### Data Flow: Complaint Submission to Resolution

```
1. Citizen Submits → 2. CV Analysis → 3. Risk Scoring → 4. Assignment →
5. Inspection → 6. Approval → 7. Contractor Assignment → 8. Work →
9. Completion → 10. Verification → 11. Closed
```

Each state transition:
- Recorded in `complaint_timeline` table
- Logged on blockchain
- Triggers notification
- Updates analytics

---

## AI/ML Pipeline Design

### Computer Vision Pipeline

```
┌─────────────┐
│ Raw Image   │
│ Upload      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│Preprocessing│
│- Resize     │
│- Normalize  │
│- Denoise    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│ Object Detection        │
│ (YOLOv8)                │
│ → Bounding boxes        │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Segmentation            │
│ (U-Net for cracks)      │
│ → Pixel-level masks     │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Classification          │
│ (ResNet-50)             │
│ → Defect type & severity│
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Size Estimation         │
│ (Geometry calculation)  │
│ → Dimensions in cm      │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Recommendation Engine   │
│ → Repair method         │
│ → Materials             │
│ → Cost estimate         │
└──────┬──────────────────┘
       │
       ▼
┌─────────────┐
│ Store JSON  │
│ Results     │
└─────────────┘
```

### NLP Pipeline

```
┌─────────────┐
│ Voice Note  │
│ (Hindi)     │
└──────┬──────┘
       │
       ▼
┌─────────────────────────┐
│ Speech-to-Text          │
│ (Google Cloud Speech)   │
│ → Hindi text            │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Translation             │
│ (IndicTrans/mBART)      │
│ → English text          │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Entity Extraction       │
│ (spaCy/BERT)            │
│ → Location, defect type │
└──────┬──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ Sentiment Analysis      │
│ → Urgency level         │
└──────┬──────────────────┘
       │
       ▼
┌─────────────┐
│ Structured  │
│ Complaint   │
└─────────────┘
```

### MLOps Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Data         │───▶│ Model        │───▶│ Model        │
│ Collection   │    │ Training     │    │ Validation   │
└──────────────┘    └──────────────┘    └──────────────┘
                            │
                            ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Monitoring   │◀───│ Deployment   │◀───│ Model        │
│ & Retraining │    │ (A/B Test)   │    │ Registry     │
└──────────────┘    └──────────────┘    └──────────────┘
```

**Tools:**
- **Versioning:** DVC (Data Version Control)
- **Experiment Tracking:** MLflow
- **Model Registry:** MLflow Model Registry
- **Serving:** TensorFlow Serving / TorchServe
- **Monitoring:** Prometheus + Grafana

---

## API Design

### RESTful API Standards

**Base URL:** `https://api.buildsense.ai/v1`

**Authentication:** JWT Bearer Token

**Request Format:**
```json
{
  "data": { ... },
  "metadata": {
    "request_id": "uuid",
    "timestamp": "2026-02-15T10:30:00Z"
  }
}
```

**Response Format:**
```json
{
  "success": true,
  "data": { ... },
  "error": null,
  "metadata": {
    "request_id": "uuid",
    "timestamp": "2026-02-15T10:30:01Z",
    "processing_time_ms": 250
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "INVALID_INPUT",
    "message": "Location coordinates are required",
    "details": { ... }
  },
  "metadata": { ... }
}
```

### Key API Endpoints Summary

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | /auth/register | Register new user | No |
| POST | /auth/login | Login | No |
| POST | /complaints | Create complaint | Yes |
| GET | /complaints/:id | Get complaint details | Yes |
| GET | /complaints | Search/filter complaints | Yes |
| PUT | /complaints/:id/status | Update status | Engineer/Admin |
| POST | /cv/analyze | Analyze image | Internal |
| POST | /risk/calculate | Calculate risk score | Internal |
| GET | /analytics/dashboard | Get dashboard data | Admin |
| POST | /notifications/send | Send notification | Internal |

### WebSocket API (Real-time Updates)

**Connection:** `wss://api.buildsense.ai/ws`

**Subscribe to Complaint Updates:**
```json
{
  "action": "subscribe",
  "channel": "complaint:12345-uuid",
  "auth_token": "jwt-token"
}
```

**Receive Update:**
```json
{
  "event": "status_changed",
  "data": {
    "complaint_id": "12345-uuid",
    "old_status": "assigned",
    "new_status": "in_progress",
    "engineer_name": "Rajesh Kumar",
    "timestamp": "2026-02-15T11:00:00Z"
  }
}
```

---

## Security Architecture

### Authentication & Authorization

**Authentication Flow:**
```
1. User requests OTP → SMS/WhatsApp sent
2. User submits OTP → Server validates
3. Server issues JWT (15-day expiry)
4. User includes JWT in Authorization header
5. Server validates JWT on each request
```

**JWT Payload:**
```json
{
  "user_id": "uuid",
  "role": "citizen",
  "permissions": ["create_complaint", "view_own_complaints"],
  "iat": 1676454000,
  "exp": 1677750000
}
```

**Role-Based Access Control (RBAC):**

| Role | Permissions |
|------|-------------|
| Citizen | Create complaint, view own complaints, verify completion |
| Engineer | View assigned complaints, update status, generate reports |
| Admin | View all data, manage users, access analytics, system config |

### Data Encryption

- **In Transit:** TLS 1.3 for all API calls
- **At Rest:** AES-256 encryption for database (PostgreSQL transparent encryption)
- **S3 Storage:** Server-side encryption (SSE-S3)

### API Security

- **Rate Limiting:** 100 requests/minute per user
- **Input Validation:** All inputs sanitized and validated
- **SQL Injection Prevention:** Parameterized queries
- **XSS Prevention:** Content Security Policy (CSP) headers
- **CSRF Protection:** CSRF tokens for state-changing requests

### Privacy Measures

- **Data Minimization:** Collect only necessary data
- **Anonymization:** Option to submit anonymous complaints
- **Data Retention:** 7-year retention, then auto-delete
- **Right to Deletion:** Users can request data deletion
- **Audit Logs:** All data access logged

---

## UI/UX Design

### Design Principles

1. **Mobile-First:** 80% users on mobile
2. **Accessibility:** WCAG 2.1 AA compliance
3. **Simplicity:** Max 3 taps to key actions
4. **Localization:** 10+ language support
5. **Offline-First:** Core features work offline

### Citizen Mobile App Design

#### Home Screen
```
┌────────────────────────────────────┐
│  BuildSense AI          [🔔] [☰]  │
├────────────────────────────────────┤
│                                    │
│   🎤  Report by Voice              │
│   ┌──────────────────────────┐    │
│   │  Tap and hold to record  │    │
│   └──────────────────────────┘    │
│                                    │
│   📷  Report with Photo            │
│                                    │
│   📝  Type Complaint               │
│                                    │
├────────────────────────────────────┤
│  My Complaints (3 Active)          │
│                                    │
│  🔴 Pothole - Main St              │
│      Assigned to Rajesh            │
│      Expected: Feb 18              │
│                                    │
│  🟡 Streetlight - Park Road        │
│      Under Inspection              │
│                                    │
└────────────────────────────────────┘
```

#### Complaint Submission Flow
```
Screen 1: Choose Method
┌─────────────────┐
│ 🎤 Voice        │
│ 📷 Photo        │
│ 📝 Text         │
│ 💬 WhatsApp     │
└─────────────────┘

Screen 2: Capture/Input
[Camera interface or voice recorder]

Screen 3: Location
[Map with auto-detected pin, option to adjust]

Screen 4: Review
┌─────────────────────────────┐
│ Type: Pothole (AI detected) │
│ Location: Main St, Ward 5   │
│ Risk Score: 72/100 (High)   │
│                             │
│ [Edit] [Submit]             │
└─────────────────────────────┘

Screen 5: Success
✅ Complaint Submitted
   ID: #BC-2026-12345
   Risk: High Priority
   Expected: 2-3 days
   [Track Status]
```

#### Complaint Tracking Screen (Amazon-style)
```
┌────────────────────────────────────┐
│  ← Complaint #BC-2026-12345        │
├────────────────────────────────────┤
│  📍 Main Street, Ward 5            │
│  🔴 Risk Score: 72 (High Priority) │
│                                    │
│  Timeline:                         │
│                                    │
│  ✅ Feb 15, 10:00 AM               │
│     Submitted                      │
│                                    │
│  ✅ Feb 15, 10:05 AM               │
│     AI Verified - Pothole Detected │
│     Size: 45cm × 30cm              │
│                                    │
│  ✅ Feb 15, 11:30 AM               │
│     Assigned to Rajesh Kumar       │
│     📞 +91-XXXX-XXXX               │
│                                    │
│  🔵 Feb 15, 3:00 PM (Current)      │
│     Site Inspection Scheduled      │
│                                    │
│  ⚪ Expected: Feb 18               │
│     Repair Work                    │
│                                    │
│  [📸 Add Update] [💬 Comment]     │
└────────────────────────────────────┘
```

### Engineer Dashboard Design

#### Dashboard Overview
```
┌─────────────────────────────────────────────────┐
│ 👷 Rajesh Kumar            [Settings] [Logout]  │
├─────────────────────────────────────────────────┤
│                                                 │
│  📊 My Workload: 8/10                          │
│  ⏱️ Avg Resolution: 4.2 days                   │
│  ⭐ Rating: 4.7/5                              │
│                                                 │
├─────────────────────────────────────────────────┤
│  Assigned Complaints (8)   [Sort: Priority ▼]  │
│                                                 │
│  🔴 #BC-12345 - Main St (Risk: 85)             │
│     ⏰ Due: Today                               │
│     [Inspect] [Report]                          │
│                                                 │
│  🟠 #BC-12346 - Park Road (Risk: 68)           │
│     ⏰ Due: Tomorrow                            │
│     [Inspect] [Report]                          │
│                                                 │
│  🟡 #BC-12347 - School St (Risk: 45)           │
│     ⏰ Due: Feb 20                              │
│     [Inspect] [Report]                          │
│                                                 │
├─────────────────────────────────────────────────┤
│  🤖 AI Co-Pilot                                │
│  "Generate inspection report for #BC-12345"    │
│  [Ask AI]                                       │
└─────────────────────────────────────────────────┘
```

#### AI Co-Pilot Interface
```
┌─────────────────────────────────────────────────┐
│  🤖 AI Co-Pilot            [History] [Settings] │
├─────────────────────────────────────────────────┤
│                                                 │
│  You: Generate inspection report for #BC-12345 │
│                                                 │
│  AI: I've analyzed complaint #BC-12345:        │
│      - Pothole: 45cm × 30cm × 8cm deep         │
│      - Location: Main St (high traffic)        │
│      - Recommended: Cold patching              │
│      - Materials: 50kg cold mix, tack coat     │
│      - Estimated cost: ₹2,500                  │
│      - Timeline: 4 hours work                  │
│                                                 │
│      [📄 Generate PDF Report]                   │
│      [📋 Create Work Order]                     │
│                                                 │
│  You: What's IRC standard for road width?      │
│                                                 │
│  AI: IRC 73-1980 specifies:                    │
│      - Arterial roads: 45-60m                  │
│      - Sub-arterial: 30-45m                    │
│      - Collector: 20-30m                       │
│      - Local streets: 15-20m                   │
│      [View Full Standard]                      │
│                                                 │
│  ┌─────────────────────────────────────────┐  │
│  │ Ask anything...                          │  │
│  └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Admin Analytics Dashboard

```
┌─────────────────────────────────────────────────────────┐
│  BuildSense AI - City Dashboard  [Date: Feb 1-15, 2026] │
├─────────────────────────────────────────────────────────┤
│  📊 Overview                                            │
│  ┌────────────┬────────────┬────────────┬──────────┐   │
│  │Total       │ Resolved   │ Avg Time   │ Budget   │   │
│  │Complaints  │            │            │ Used     │   │
│  │   1,247    │    856     │   5.2 days │  ₹23L    │   │
│  │  +12% ↑    │  +8% ↑     │  -15% ↓    │  68%     │   │
│  └────────────┴────────────┴────────────┴──────────┘   │
│                                                         │
│  🗺️ Risk Heatmap                                       │
│  [Interactive map showing risk zones in colors]        │
│                                                         │
│  📈 Trends                                             │
│  [Line chart: Complaints over time by category]       │
│                                                         │
│  🏆 Top Performers (Engineers)                         │
│  1. Rajesh Kumar    - 4.8 ⭐ - 45 resolved            │
│  2. Priya Sharma    - 4.7 ⭐ - 42 resolved            │
│  3. Amit Patel      - 4.6 ⭐ - 38 resolved            │
│                                                         │
│  🚨 Critical Alerts (3)                                │
│  - Bridge risk near School St (Score: 92)              │
│  - Recurring potholes in Ward 12                       │
│  - Budget overrun alert for drainage                   │
│                                                         │
│  🔮 Predictive Insights                                │
│  - 15 locations likely to fail in next 60 days        │
│  - Recommended preventive budget: ₹12L                 │
└─────────────────────────────────────────────────────────┘
```

### Color Scheme & Branding

**Primary Colors:**
- Critical Red: #DC2626
- High Orange: #EA580C
- Medium Yellow: #CA8A04
- Low Green: #16A34A
- Primary Blue: #2563EB (Brand color)

**Typography:**
- Headings: Inter Bold
- Body: Inter Regular
- Monospace (code): Fira Code

**Accessibility:**
- Minimum contrast ratio: 4.5:1
- Touch targets: Minimum 44×44px
- Screen reader labels on all interactive elements

---

## Deployment Architecture

### Cloud Infrastructure (AWS Example)

```
┌─────────────────────────────────────────────────────┐
│                  Route 53 (DNS)                     │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│            CloudFront (CDN)                         │
│  - Static assets (images, JS, CSS)                 │
│  - Edge caching for global users                   │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│        Application Load Balancer (ALB)             │
└───────────┬──────────────────────┬──────────────────┘
            │                      │
            ▼                      ▼
┌─────────────────────┐  ┌─────────────────────┐
│  EKS Cluster        │  │  EKS Cluster        │
│  (Availability      │  │  (Availability      │
│   Zone A)           │  │   Zone B)           │
│                     │  │                     │
│  ┌───────────────┐ │  │  ┌───────────────┐  │
│  │ Microservices │ │  │  │ Microservices │  │
│  │ - User Svc    │ │  │  │ - User Svc    │  │
│  │ - Complaint   │ │  │  │ - Complaint   │  │
│  │ - Engineer    │ │  │  │ - Engineer    │  │
│  └───────────────┘ │  │  └───────────────┘  │
└─────────────────────┘  └─────────────────────┘
            │                      │
            └──────────┬───────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│               RDS PostgreSQL                        │
│  - Multi-AZ deployment                              │
│  - Read replicas for analytics                      │
└─────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────┐
│         ElastiCache (Redis)                         │
│  - Session management                               │
│  - API response caching                             │
└─────────────────────────────────────────────────────┘
```

### Kubernetes Architecture

```yaml
# Deployment example for Complaint Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complaint-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complaint-service
  template:
    metadata:
      labels:
        app: complaint-service
    spec:
      containers:
      - name: complaint-service
        image: buildsense/complaint-service:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: complaint-service
spec:
  selector:
    app: complaint-service
  ports:
  - port: 80
    targetPort: 3000
  type: LoadBalancer
```

### CI/CD Pipeline

```
┌──────────────┐
│ Git Push     │
│ (GitHub)     │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────┐
│ GitHub Actions               │
│ 1. Run tests                 │
│ 2. Build Docker images       │
│ 3. Push to ECR               │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ ArgoCD                       │
│ - Sync with Git              │
│ - Deploy to K8s staging      │
└──────┬───────────────────────┘
       │
       ▼ (manual approval)
┌──────────────────────────────┐
│ Production Deployment        │
│ - Canary release             │
│ - Gradual rollout            │
│ - Monitor metrics            │
└──────────────────────────────┘
```

### Monitoring Stack

```
Application Metrics → Prometheus → Grafana → Alerts (PagerDuty)
Application Logs → FluentD → Elasticsearch → Kibana
Error Tracking → Sentry → Slack Notifications
APM → New Relic/Datadog → Performance Dashboards
```

---

## Data Flow Diagrams

### Complaint Submission Flow

```
┌──────────┐
│ Citizen  │
│ (Mobile  │
│  App)    │
└────┬─────┘
     │ 1. Upload photo + location
     ▼
┌─────────────────┐
│ API Gateway     │
│ - Auth check    │
│ - Rate limit    │
└────┬────────────┘
     │ 2. Validate & route
     ▼
┌─────────────────┐         ┌─────────────────┐
│ Complaint Svc   │───────▶│ S3 Storage      │
│ - Create record │  3.     │ - Store image   │
└────┬────────────┘  Upload └─────────────────┘
     │ 4. Trigger CV
     ▼
┌─────────────────┐
│ Message Queue   │
│ (RabbitMQ)      │
└────┬────────────┘
     │ 5. Async process
     ▼
┌─────────────────┐         ┌─────────────────┐
│ CV Service      │────────▶│ Risk Engine     │
│ - Analyze image │  6.     │ - Calculate     │
│ - Detect defect │  Results│   risk score    │
└─────────────────┘         └────┬────────────┘
                                 │ 7. Score
                                 ▼
┌─────────────────────────────────────────────┐
│ Complaint Service                           │
│ - Update with CV results                    │
│ - Update risk score                         │
│ - Determine priority                        │
└────┬────────────────────────────────────────┘
     │ 8. Assignment logic
     ▼
┌─────────────────┐         ┌─────────────────┐
│ Engineer Svc    │◀────────│ Assignment      │
│ - Find engineer │  10.    │ Algorithm       │
│   by location   │ Assign  │ - Load balance  │
│ - Check workload│         │ - Location      │
└────┬────────────┘         └─────────────────┘
     │ 11. Notify
     ▼
┌─────────────────┐         ┌─────────────────┐
│ Notification    │────────▶│ Twilio/FCM      │
│ Service         │  12.    │ - Send SMS      │
└─────────────────┘  Queue  │ - Push notif    │
                             └─────────────────┘
                                   │ 13. Deliver
                                   ▼
                             ┌─────────────────┐
                             │ Engineer        │
                             │ (Mobile)        │
                             └─────────────────┘
```

### Real-time Status Update Flow

```
┌──────────┐
│ Engineer │
│ updates  │
│ status   │
└────┬─────┘
     │ 1. PUT /complaints/:id/status
     ▼
┌─────────────────┐
│ API Gateway     │
│ - Auth (JWT)    │
│ - RBAC check    │
└────┬────────────┘
     │ 2. Authorized
     ▼
┌─────────────────┐         ┌─────────────────┐
│ Complaint Svc   │────────▶│ Database        │
│ - Update status │  3.     │ - ACID update   │
│ - Log timeline  │  Write  └─────────────────┘
└────┬────────────┘
     │ 4. Publish event
     ▼
┌─────────────────┐
│ Event Bus       │
│ (Redis Pub/Sub) │
└────┬────────────┘
     │ 5. Broadcast
     ├─────────────────┬─────────────────┬──────────────────┐
     ▼                 ▼                 ▼                  ▼
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌──────────────┐
│Notification│   │ Blockchain │   │ Analytics  │   │ WebSocket    │
│  Service   │   │  Service   │   │  Service   │   │  Server      │
└─────┬──────┘   └─────┬──────┘   └────┬───────┘   └─────┬────────┘
      │                │                │                  │
      │ 6. Send        │ 7. Record      │ 8. Update        │ 9. Push
      │    notif       │    on chain    │    metrics       │    to client
      ▼                ▼                ▼                  ▼
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌──────────────┐
│  Citizen   │   │  Polygon   │   │ TimescaleDB│   │  Citizen     │
│  (Push)    │   │ Blockchain │   │            │   │  (Real-time) │
└────────────┘   └────────────┘   └────────────┘   └──────────────┘
```

---

## Scalability Considerations

### Horizontal Scaling Strategy

1. **Stateless Services:** All services are stateless, enabling easy replication
2. **Database Sharding:** Complaints sharded by geographic region
3. **Caching Layers:** 
   - L1: Application-level cache (in-memory)
   - L2: Redis cache
   - L3: CDN for static assets
4. **Async Processing:** Heavy tasks (CV, NLP) processed asynchronously via queues
5. **Read Replicas:** Database read replicas for analytics queries

### Performance Optimization

- **Database Indexing:** Strategic indexes on frequently queried fields
- **Query Optimization:** N+1 query prevention, pagination
- **Image Optimization:** Compress images before storage, serve via CDN
- **API Response Caching:** Cache GET requests for 5-10 minutes
- **Lazy Loading:** Load data on-demand, not upfront

### Load Testing

- **Tool:** Apache JMeter / Locust
- **Scenarios:**
  - 10,000 concurrent users submitting complaints
  - 1,000 images analyzed simultaneously
  - 50,000 dashboard requests/hour
- **Target Metrics:**
  - P95 latency < 500ms
  - Error rate < 0.1%
  - Throughput: 1,000 requests/second

---

## Future Enhancements

### Phase 2 Features
1. **IoT Integration:** Real-time sensor data from bridges, roads
2. **Drone Scanning:** Automated aerial infrastructure surveys
3. **AR for Citizens:** AR overlay showing infrastructure status while walking
4. **Predictive Budgeting:** AI-powered multi-year budget forecasting
5. **Carbon Footprint Tracker:** Environmental impact of repairs

### Phase 3 Features
1. **Quantum Optimization:** Quantum-inspired algorithms for large-scale scheduling
2. **Digital Twin:** Virtual city model with real-time infrastructure health
3. **International Expansion:** Adapt for Southeast Asia, Africa markets
4. **Public-Private Partnership Portal:** Enable private contractors bidding
5. **Citizen Science:** Crowdsourced data collection programs

---

## Conclusion

BuildSense AI's architecture is designed for:

✅ **Scalability** - Handle millions of users  
✅ **Reliability** - 99.9% uptime  
✅ **Security** - Enterprise-grade protection  
✅ **Performance** - Sub-second response times  
✅ **Maintainability** - Modular microservices  
✅ **Extensibility** - Easy to add new features  
✅ **Accessibility** - Voice-first, multilingual  
✅ **Social Impact** - Transparent, inclusive governance  

**We're building the future of civic infrastructure management.**

