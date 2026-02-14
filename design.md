# Design Document: GramScore AI

## Overview

GramScore AI is a cloud-native, AI-powered credit scoring platform designed specifically for rural India. The system integrates alternative data sources (climate, crop cycles, mandi prices) with traditional borrower information to generate accurate, explainable credit risk scores. Built on AWS infrastructure, the platform provides a scalable, secure, and compliant solution for Microfinance Institutions, Regional Rural Banks, and NBFCs.

### Design Philosophy

1. **Rural-First**: Every component is designed considering rural India's unique challenges—seasonal income, weather dependency, informal economies
2. **Explainability-First**: AI decisions are transparent and interpretable for regulatory compliance and user trust
3. **API-First**: All functionality is exposed via REST APIs for seamless integration with existing loan management systems
4. **Cloud-Native**: Leverages AWS managed services for scalability, reliability, and reduced operational overhead
5. **Security-First**: Implements defense-in-depth with encryption, RBAC, and comprehensive audit logging

### Key Design Decisions

- **XGBoost for Risk Scoring**: Chosen for superior performance on tabular data, built-in handling of missing values, and compatibility with SHAP explainability
- **SHAP for Explainability**: Industry-standard method providing consistent, mathematically rigorous feature importance
- **Serverless Architecture**: Lambda functions for data ingestion and API endpoints to minimize costs and maximize scalability
- **SageMaker for ML**: Managed ML platform for model training, deployment, and monitoring
- **RDS PostgreSQL**: Relational database for structured borrower data with ACID guarantees
- **S3 for Data Lake**: Cost-effective storage for raw alternative data, model artifacts, and audit logs

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          External Data Sources                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ Weather APIs │  │ Mandi Price  │  │  Synthetic   │                  │
│  │ (IMD, etc.)  │  │     APIs     │  │  Data Gen    │                  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                  │
└─────────┼──────────────────┼──────────────────┼──────────────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
┌────────────────────────────┼─────────────────────────────────────────────┐
│                            ▼                                              │
│                   Data Ingestion Layer                                    │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Lambda Functions (Scheduled)                                      │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │   Weather    │  │ Mandi Price  │  │  Synthetic   │            │  │
│  │  │   Ingestion  │  │  Ingestion   │  │Data Ingestion│            │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │  │
│  └─────────┼──────────────────┼──────────────────┼────────────────────┘  │
│            │                  │                  │                        │
│            └──────────────────┼──────────────────┘                        │
│                               ▼                                           │
│                        S3 Data Lake                                       │
│                  (Raw Alternative Data)                                   │
└───────────────────────────────┬───────────────────────────────────────────┘

                                │
┌───────────────────────────────┼───────────────────────────────────────────┐
│                               ▼                                           │
│                   Data Processing Layer                                   │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Lambda Functions (Event-Driven)                                   │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │   Feature    │  │  Seasonal    │  │     Risk     │            │  │
│  │  │ Engineering  │  │Normalization │  │  Feature     │            │  │
│  │  │              │  │              │  │  Extraction  │            │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │  │
│  └─────────┼──────────────────┼──────────────────┼────────────────────┘  │
│            │                  │                  │                        │
│            └──────────────────┼──────────────────┘                        │
│                               ▼                                           │
│                        Feature Store                                      │
│                    (Processed Features)                                   │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │
┌───────────────────────────────┼───────────────────────────────────────────┐
│                               ▼                                           │
│                       AI Model Layer                                      │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │              SageMaker Endpoints                                   │  │
│  │  ┌──────────────────────────────────────────────────────────────┐ │  │
│  │  │  XGBoost Risk Scoring Model                                  │ │  │
│  │  │  - Gradient Boosting                                         │ │  │
│  │  │  - Time-series adjustment for crop seasonality              │ │  │
│  │  └──────────────────────────────────────────────────────────────┘ │  │
│  │  ┌──────────────────────────────────────────────────────────────┐ │  │
│  │  │  SHAP Explainability Module                                  │ │  │
│  │  │  - TreeExplainer for XGBoost                                 │ │  │
│  │  │  - Feature importance calculation                            │ │  │
│  │  └──────────────────────────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │
┌───────────────────────────────┼───────────────────────────────────────────┐
│                               ▼                                           │
│                     Application Layer                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                    API Gateway                                     │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │   Borrower   │  │  Risk Score  │  │ Simulation   │            │  │
│  │  │     API      │  │     API      │  │     API      │            │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘            │  │
│  └─────────┼──────────────────┼──────────────────┼────────────────────┘  │
│            │                  │                  │                        │
│  ┌─────────┼──────────────────┼──────────────────┼────────────────────┐  │
│  │         ▼                  ▼                  ▼                    │  │
│  │              Lambda Functions (API Handlers)                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │              Web Application (React)                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │  │
│  │  │ Loan Officer │  │ Credit Mgr   │  │   Reports    │            │  │
│  │  │  Dashboard   │  │  Dashboard   │  │  & Analytics │            │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │  │
│  │              Hosted on S3 + CloudFront                            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬───────────────────────────────────────────┘
                                │
┌───────────────────────────────┼───────────────────────────────────────────┐
│                               ▼                                           │
│                       Storage Layer                                       │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  RDS PostgreSQL                                                    │  │
│  │  - Borrower profiles                                               │  │
│  │  - Risk scores                                                     │  │
│  │  - User accounts                                                   │  │
│  │  - Audit logs                                                      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  S3 Buckets                                                        │  │
│  │  - Raw alternative data                                            │  │
│  │  - Processed features                                              │  │
│  │  - Model artifacts                                                 │  │
│  │  - Explainability reports (PDF)                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                  Security & Governance Layer                              │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AWS IAM + Cognito                                                 │  │
│  │  - Role-based access control                                       │  │
│  │  - User authentication                                             │  │
│  │  - API key management                                              │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AWS KMS                                                           │  │
│  │  - Encryption key management                                       │  │
│  │  - Data encryption at rest                                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  CloudWatch + CloudTrail                                           │  │
│  │  - Monitoring and alerting                                         │  │
│  │  - Audit logging                                                   │  │
│  │  - Performance metrics                                             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

### Component Descriptions

#### 1. Data Ingestion Layer

**Purpose**: Collect and store alternative data from external sources

**Components**:

- **Weather Ingestion Lambda**: Scheduled function (daily) that fetches climate data from IMD (India Meteorological Department) or OpenWeather APIs
  - Retrieves rainfall, temperature, humidity data
  - Calculates rainfall anomalies (deviation from historical averages)
  - Stores raw data in S3 with partitioning by date and region
  
- **Mandi Price Ingestion Lambda**: Scheduled function (weekly) that fetches agricultural commodity prices from government mandi APIs
  - Retrieves prices for major crops (wheat, rice, cotton, sugarcane, pulses)
  - Filters prices within 50km radius of borrower locations
  - Stores raw data in S3 with partitioning by date and commodity
  
- **Synthetic Data Ingestion Lambda**: On-demand function for hackathon/testing that generates realistic synthetic borrower data
  - Generates borrower profiles with realistic distributions
  - Creates correlated loan outcomes (default/repayment)
  - Stores synthetic data in S3 with clear labeling

**Data Flow**:
1. Lambda functions triggered by CloudWatch Events (scheduled) or API calls (on-demand)
2. Functions authenticate with external APIs using secrets stored in AWS Secrets Manager
3. Raw data written to S3 in JSON format with metadata (timestamp, source, version)
4. S3 event notifications trigger downstream processing

**Error Handling**:
- Retry logic with exponential backoff for API failures
- Dead letter queues (SQS) for failed ingestion attempts
- CloudWatch alarms for consecutive failures
- Fallback to cached data when external sources unavailable

#### 2. Data Processing Layer

**Purpose**: Transform raw data into ML-ready features

**Components**:

- **Feature Engineering Lambda**: Event-driven function triggered by S3 uploads
  - Joins borrower data with climate and mandi price data based on location and time
  - Creates derived features (rainfall deviation %, price volatility, seasonal indicators)
  - Handles missing values using regional averages with confidence penalties
  - Outputs processed features to Feature Store
  
- **Seasonal Normalization Lambda**: Adjusts features for crop cycle seasonality
  - Normalizes income patterns based on crop calendar (Kharif/Rabi seasons)
  - Adjusts risk scores for harvest timing (higher risk pre-harvest, lower post-harvest)
  - Creates time-series features (days until harvest, days since planting)
  
- **Risk Feature Extraction Lambda**: Calculates domain-specific risk indicators
  - Climate risk score (based on rainfall anomalies, temperature extremes)
  - Market risk score (based on mandi price volatility)
  - Livelihood diversity score (single crop vs. mixed income sources)
  - Geographic risk score (drought-prone regions, flood zones)

**Feature Store**:
- S3-based storage for processed features in Parquet format
- Partitioned by borrower_id and timestamp for efficient retrieval
- Versioned features for model reproducibility
- Metadata catalog in RDS for feature discovery

**Data Quality Checks**:
- Schema validation against predefined feature definitions
- Outlier detection using statistical thresholds (±3 standard deviations)
- Completeness checks (flag records with >30% missing features)
- Data quality metrics logged to CloudWatch

#### 3. AI Model Layer

**Purpose**: Generate risk scores and explanations using machine learning

**Components**:

- **XGBoost Risk Scoring Model**:
  - **Algorithm**: Gradient Boosted Decision Trees (XGBoost)
  - **Input Features** (30+ features):
    - Borrower demographics: age, gender, family size, education
    - Livelihood: primary occupation, secondary income sources, years of experience
    - Location: state, district, village, geographic coordinates
    - Climate: rainfall (current, historical average, anomaly %), temperature, drought indicators
    - Crop: primary crop, crop cycle stage, expected harvest date, land size
    - Market: mandi prices (current, 3-month average, volatility), price trends
    - Seasonal: days until harvest, season indicator (Kharif/Rabi), month
  - **Output**: Risk score (0-1000) and probability of default (0-1)
  - **Training**: Supervised learning on historical loan outcomes (default/repayment)
  - **Hyperparameters**: 
    - max_depth: 6 (prevent overfitting)
    - learning_rate: 0.1
    - n_estimators: 100
    - subsample: 0.8 (row sampling)
    - colsample_bytree: 0.8 (column sampling)
  - **Time-Series Adjustment**: Custom objective function that weights recent data more heavily and accounts for seasonal patterns
  
- **SHAP Explainability Module**:
  - **Algorithm**: TreeExplainer for XGBoost (fast, exact Shapley values)
  - **Output**: SHAP values for each feature showing contribution to prediction
  - **Visualization**: Waterfall plots showing how features push score up/down from baseline
  - **Top-K Features**: Identifies top 5 contributing factors for each prediction
  - **Natural Language Generation**: Converts SHAP values to human-readable explanations
    - Example: "Rainfall 25% below average reduced score by 45 points"
    - Example: "Wheat prices 15% above average increased score by 30 points"

**Model Deployment**:
- SageMaker real-time endpoint for synchronous predictions (API requests)
- SageMaker batch transform for bulk scoring (nightly batch jobs)
- Model versioning with A/B testing capability
- Auto-scaling based on request volume (min 1 instance, max 10 instances)

**Model Monitoring**:
- Data drift detection comparing input feature distributions over time
- Model performance tracking (accuracy, AUC-ROC, precision, recall)
- Prediction latency monitoring (target: <1 second for model inference)
- Automated retraining triggers when accuracy drops below threshold

#### 4. Application Layer

**Purpose**: Provide user interfaces and APIs for system interaction

**Components**:

- **API Gateway**:
  - RESTful API endpoints with OpenAPI specification
  - Authentication via API keys (for external integrations) and JWT tokens (for web app)
  - Rate limiting (100 requests/minute per client)
  - Request/response validation against JSON schemas
  - CORS configuration for web app access
  
- **Lambda API Handlers**:
  - **Borrower API**: CRUD operations for borrower profiles
    - POST /borrowers: Create new borrower
    - GET /borrowers/{id}: Retrieve borrower details
    - PUT /borrowers/{id}: Update borrower profile
    - DELETE /borrowers/{id}: Soft delete borrower
  - **Risk Score API**: Generate and retrieve risk scores
    - POST /risk-scores: Generate new risk score for borrower
    - GET /risk-scores/{id}: Retrieve existing risk score
    - GET /risk-scores/borrower/{borrower_id}: Get all scores for borrower
  - **Explainability API**: Generate and retrieve explainability reports
    - GET /explainability/{risk_score_id}: Get SHAP-based explanation
    - GET /explainability/{risk_score_id}/pdf: Download PDF report
  - **Simulation API**: Run risk scenario simulations
    - POST /simulations: Run simulation with parameter changes
    - GET /simulations/{id}: Retrieve simulation results
  - **Reports API**: Generate analytics and reports
    - GET /reports/portfolio: Portfolio risk distribution
    - GET /reports/geographic: Geographic risk heatmap
    - GET /reports/trends: Risk score trends over time
  
- **Web Application (React)**:
  - **Loan Officer Dashboard**:
    - Borrower search and filtering
    - Borrower profile creation/editing forms
    - Risk score display with color-coded risk bands
    - Explainability report viewer
    - Application status tracking (pending, approved, rejected)
  - **Credit Manager Dashboard**:
    - Portfolio overview with risk distribution charts
    - Risk simulation interface with parameter sliders
    - Approval workflow (review, approve, reject applications)
    - Analytics dashboards with drill-down capabilities
  - **Reports & Analytics**:
    - Interactive charts (Chart.js or Recharts)
    - Geographic heatmaps (Leaflet or Mapbox)
    - Export functionality (PDF, CSV)
    - Scheduled report configuration
  - **Hosting**: Static files on S3, distributed via CloudFront CDN
  - **Authentication**: AWS Cognito for user management and JWT tokens

#### 5. Storage Layer

**Purpose**: Persist structured and unstructured data

**Components**:

- **RDS PostgreSQL**:
  - **Schema Design**:
    - `users`: User accounts with roles and permissions
    - `borrowers`: Borrower profiles with demographics and livelihood info
    - `risk_scores`: Generated risk scores with model version and timestamp
    - `explainability_reports`: SHAP values and natural language explanations
    - `simulations`: Simulation parameters and results
    - `audit_logs`: Immutable log of all system actions
  - **Configuration**:
    - Multi-AZ deployment for high availability
    - Automated backups with 7-day retention
    - Read replicas for analytics queries
    - Encryption at rest using AWS KMS
    - Connection pooling via RDS Proxy
  
- **S3 Buckets**:
  - **raw-data-bucket**: Raw alternative data from external sources
    - Partitioned by source/date (e.g., s3://bucket/weather/2024/01/15/)
    - Lifecycle policy: transition to Glacier after 90 days
  - **feature-store-bucket**: Processed features in Parquet format
    - Partitioned by borrower_id (e.g., s3://bucket/features/borrower_123/)
    - Versioned for reproducibility
  - **model-artifacts-bucket**: Trained model files and metadata
    - Model binaries, hyperparameters, training metrics
    - Versioned with semantic versioning (v1.0.0, v1.1.0)
  - **reports-bucket**: Generated PDF explainability reports
    - Partitioned by date and borrower_id
    - Lifecycle policy: delete after 1 year
  - **All buckets**: Server-side encryption (SSE-S3), versioning enabled, access logging

#### 6. Security & Governance Layer

**Purpose**: Ensure data security, access control, and compliance

**Components**:

- **AWS IAM + Cognito**:
  - **IAM Roles**: Least-privilege roles for Lambda functions, SageMaker, and EC2
  - **Cognito User Pools**: User authentication with password policies
  - **Cognito Identity Pools**: Federated access for web app
  - **RBAC Implementation**:
    - Loan_Officer: Read/write borrower profiles, read risk scores
    - Credit_Manager: All Loan_Officer permissions + approve/reject loans, run simulations
    - System_Administrator: All permissions + user management, system configuration
    - API_Client: Programmatic access via API keys with rate limiting
  
- **AWS KMS**:
  - Customer-managed keys for encryption at rest
  - Separate keys for different data classifications (PII, financial, operational)
  - Key rotation enabled (annual)
  - Key usage audited via CloudTrail
  
- **CloudWatch + CloudTrail**:
  - **CloudWatch Logs**: Centralized logging for all Lambda functions and API Gateway
  - **CloudWatch Metrics**: Custom metrics for business KPIs (risk scores generated, API latency)
  - **CloudWatch Alarms**: Alerts for errors, performance degradation, security events
  - **CloudTrail**: Audit trail of all AWS API calls for compliance
  - **Log Retention**: 7 years for audit logs, 90 days for operational logs

## Data Models

### Borrower Profile

```json
{
  "borrower_id": "string (UUID)",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "personal_info": {
    "name": "string",
    "age": "integer (18-75)",
    "gender": "enum (Male, Female, Other)",
    "mobile": "string (10 digits)",
    "education": "enum (None, Primary, Secondary, Higher Secondary, Graduate)"
  },
  "location": {
    "state": "string",
    "district": "string",
    "village": "string",
    "latitude": "float",
    "longitude": "float"
  },
  "livelihood": {
    "primary_occupation": "enum (Agriculture, Dairy, Poultry, Small Business, Wage Labor, Mixed)",
    "secondary_income_sources": "array of enums",
    "years_of_experience": "integer",
    "land_size_acres": "float (nullable)",
    "primary_crop": "string (nullable)",
    "livestock_count": "integer (nullable)"
  },
  "loan_request": {
    "amount": "float (5000-500000)",
    "purpose": "string",
    "tenure_months": "integer (6-60)"
  },
  "metadata": {
    "created_by_user_id": "string (UUID)",
    "data_quality_score": "float (0-1)",
    "is_synthetic": "boolean"
  }
}
```

### Risk Score

```json
{
  "risk_score_id": "string (UUID)",
  "borrower_id": "string (UUID)",
  "created_at": "timestamp",
  "score": "integer (0-1000)",
  "risk_band": "enum (Excellent, Good, Fair, Poor, High Risk)",
  "default_probability": "float (0-1)",
  "model_version": "string (e.g., v1.2.0)",
  "features_used": {
    "borrower_age": "integer",
    "livelihood_type": "string",
    "rainfall_anomaly_percent": "float",
    "mandi_price_volatility": "float",
    "days_until_harvest": "integer",
    "climate_risk_score": "float",
    "market_risk_score": "float",
    "... (30+ features)": "..."
  },
  "confidence_score": "float (0-1)",
  "data_completeness": "float (0-1)"
}
```

### Explainability Report

```json
{
  "explainability_id": "string (UUID)",
  "risk_score_id": "string (UUID)",
  "created_at": "timestamp",
  "baseline_score": "float",
  "predicted_score": "float",
  "top_factors": [
    {
      "feature_name": "string",
      "feature_value": "string",
      "shap_value": "float",
      "contribution_percent": "float",
      "direction": "enum (Positive, Negative)",
      "explanation": "string (natural language)"
    }
  ],
  "all_shap_values": {
    "feature_name": "float",
    "...": "..."
  },
  "pdf_report_url": "string (S3 URL)"
}
```

### Alternative Data (Climate)

```json
{
  "climate_data_id": "string (UUID)",
  "location": {
    "latitude": "float",
    "longitude": "float",
    "district": "string"
  },
  "date": "date",
  "rainfall_mm": "float",
  "rainfall_historical_avg_mm": "float",
  "rainfall_anomaly_percent": "float",
  "temperature_celsius": "float",
  "humidity_percent": "float",
  "drought_indicator": "boolean",
  "flood_indicator": "boolean",
  "source": "string (e.g., IMD, OpenWeather)",
  "ingested_at": "timestamp"
}
```

### Alternative Data (Mandi Prices)

```json
{
  "mandi_price_id": "string (UUID)",
  "mandi_name": "string",
  "location": {
    "district": "string",
    "state": "string",
    "latitude": "float",
    "longitude": "float"
  },
  "date": "date",
  "commodity": "string (e.g., Wheat, Rice, Cotton)",
  "variety": "string",
  "price_per_quintal": "float",
  "price_3month_avg": "float",
  "price_volatility": "float",
  "source": "string (e.g., Agmarknet)",
  "ingested_at": "timestamp"
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

Before defining the correctness properties, let me analyze the acceptance criteria from the requirements document to determine which are testable as properties, examples, or edge cases.

### Property Reflection

After analyzing all 120+ acceptance criteria, I've identified several areas where properties can be consolidated:

**Consolidation Opportunities**:
1. Multiple logging properties (9.1, 9.2, 9.3, 9.4, 7.9) can be combined into a comprehensive "all actions are logged" property
2. Multiple validation properties (1.2, 1.3, 1.5, 11.2, 11.3) can be combined into an "input validation" property
3. Multiple data storage properties (1.8, 2.8, 3.7) can be combined into a "metadata persistence" property
4. Multiple export properties (4.7, 6.7, 14.7) can be combined into a "format export" property
5. Multiple CRUD properties (1.1, 1.7, 15.1, 15.2) share similar patterns and can be consolidated

**Redundancy Elimination**:
- Property 9.2 (API logging) is subsumed by property 9.1 (all user actions logging) if we consider API calls as user actions
- Property 2.8 (timestamp storage) is implied by property 9.3 (ingestion logging with timestamp)
- Property 3.7 (model version storage) is implied by property 9.4 (prediction logging with model version)

After reflection, I'll focus on unique, high-value properties that provide comprehensive coverage without redundancy.

### Correctness Properties

#### Property 1: Borrower Profile Creation Uniqueness
*For any* valid borrower profile submitted by a Loan Officer, the System should create a new borrower record with a unique identifier that does not conflict with any existing borrower identifier.
**Validates: Requirements 1.1**

#### Property 2: Input Validation Consistency
*For any* input field with defined validation rules (age range, mobile format, livelihood enum, numeric ranges, geographic bounds), the System should accept all valid inputs and reject all invalid inputs with specific error messages indicating which validation rule failed.
**Validates: Requirements 1.2, 1.3, 1.5, 11.2, 11.3**

#### Property 3: Data Persistence Completeness
*For any* data entity created or updated (borrower profile, risk score, alternative data), the System should store all required metadata including timestamps, user IDs, model versions, and data sources.
**Validates: Requirements 1.8, 2.8, 3.7**

#### Property 4: Incomplete Profile Rejection
*For any* borrower profile with one or more mandatory fields missing, the System should prevent submission and return error messages listing all specific missing fields.
**Validates: Requirements 1.6**

#### Property 5: Version History Preservation
*For any* borrower profile update, the System should maintain complete version history such that all previous versions remain accessible with their original timestamps and values.
**Validates: Requirements 1.8**

#### Property 6: Spatial Query Accuracy
*For any* borrower location, the System should fetch only mandi prices from markets within 50km radius, and should not include prices from markets beyond 50km.
**Validates: Requirements 2.2**

#### Property 7: Graceful Degradation with Cached Data
*For any* external data source failure, the System should use cached data with staleness indicators rather than failing the request, and should log the failure for monitoring.
**Validates: Requirements 2.6**

#### Property 8: Risk Score Range Invariant
*For any* complete borrower profile and alternative data, the generated risk score should always be within the range [0, 1000] inclusive.
**Validates: Requirements 3.1**

#### Property 9: Risk Band Classification Correctness
*For any* risk score value, the System should assign exactly one risk band according to the defined thresholds: Excellent (750-1000), Good (650-749), Fair (550-649), Poor (450-549), High Risk (0-449).
**Validates: Requirements 3.6**

#### Property 10: Seasonal Weighting for Agricultural Livelihoods
*For any* pair of borrowers with similar profiles where one has agricultural livelihood and the other has non-agricultural livelihood, the System should assign higher weight to seasonal income patterns for the agricultural borrower, reflected in greater score sensitivity to crop cycle and climate factors.
**Validates: Requirements 3.4**

#### Property 11: Climate Risk Adjustment
*For any* borrower in a location with rainfall anomaly exceeding 30% deviation from historical average, the System should adjust the risk score to reflect increased climate risk compared to the same borrower with normal rainfall.
**Validates: Requirements 3.5**

#### Property 12: Missing Data Handling with Confidence Penalty
*For any* borrower profile with missing alternative data fields, the System should generate a risk score using regional averages for missing fields and should assign a lower confidence score compared to profiles with complete data.
**Validates: Requirements 3.8**

#### Property 13: Explainability Report Completeness
*For any* generated risk score, the System should produce an explainability report containing exactly 5 top contributing factors, each with feature name, value, SHAP contribution, percentage contribution, and natural language explanation.
**Validates: Requirements 4.1, 4.2**

#### Property 14: Explainability Percentage Sum
*For any* explainability report, the sum of absolute percentage contributions of all factors should equal 100% (within rounding tolerance of ±1%).
**Validates: Requirements 4.2**

#### Property 15: Factor Direction Classification
*For any* explainability report, all factors should be correctly classified as either positive (increasing score) or negative (decreasing score) based on the sign of their SHAP values.
**Validates: Requirements 4.4**

#### Property 16: Climate Anomaly Explanation Detail
*For any* risk score where climate factors appear in the top 5 contributing factors, the explainability report should specify the type of climate anomaly (rainfall, temperature, drought, flood) and its magnitude.
**Validates: Requirements 4.5**

#### Property 17: Data Source Attribution
*For any* contributing factor in an explainability report, the System should include a reference to the data source (borrower profile, climate API, mandi API, synthetic data).
**Validates: Requirements 4.6**

#### Property 18: Search Functionality Correctness
*For any* search query (name, mobile number, or application ID), the System should return all borrowers matching the query and should not return borrowers that do not match the query.
**Validates: Requirements 5.2**

#### Property 19: Risk Band Color Coding Consistency
*For any* risk score displayed on the dashboard, the System should apply color coding consistent with the risk band: Excellent (green), Good (light green), Fair (yellow), Poor (orange), High Risk (red).
**Validates: Requirements 5.3**

#### Property 20: Complete Borrower Data Retrieval
*For any* borrower selected on the dashboard, the System should display the complete profile, all associated risk scores, and all explainability reports without omitting any data.
**Validates: Requirements 5.4**

#### Property 21: Multi-Criteria Filtering Correctness
*For any* combination of filters (risk band, livelihood type, location, application date), the System should return only borrowers matching all selected filter criteria.
**Validates: Requirements 5.5**

#### Property 22: Portfolio Statistics Accuracy
*For any* loan officer's portfolio, the displayed summary statistics (total applications, average risk score, approval rate) should match the actual calculated values from the underlying data.
**Validates: Requirements 5.6**

#### Property 23: Simulation Score Recalculation
*For any* borrower and simulation parameters (rainfall change, mandi price change), the System should recalculate the risk score incorporating the parameter changes and should return both original and simulated scores.
**Validates: Requirements 6.1, 6.2, 6.4**

#### Property 24: Risk Band Change Detection
*For any* simulation result where the simulated risk score falls into a different risk band than the original score, the System should highlight the borrower as having a band change.
**Validates: Requirements 6.5**

#### Property 25: Portfolio-Level Simulation Aggregation
*For any* portfolio simulation, the aggregate risk exposure should equal the sum of individual borrower risk exposures under the simulation parameters.
**Validates: Requirements 6.6**

#### Property 26: API Authentication and Authorization
*For any* API request without a valid API key or with an API key lacking required permissions, the System should return HTTP 401 (unauthorized) and should not process the request.
**Validates: Requirements 7.2, 7.5**

#### Property 27: API Response Format Consistency
*For any* valid API request for risk score generation, the System should return a JSON response containing risk score, risk band, default probability, and explainability report in the documented schema.
**Validates: Requirements 7.3**

#### Property 28: API Error Response Specificity
*For any* invalid API request (malformed JSON, missing required fields, invalid field values), the System should return HTTP 400 with a detailed error message specifying which validation failed.
**Validates: Requirements 7.4**

#### Property 29: Rate Limiting Enforcement
*For any* API client exceeding 100 requests per minute, the System should return HTTP 429 (too many requests) for subsequent requests until the rate limit window resets.
**Validates: Requirements 7.6**

#### Property 30: Password Security
*For any* user password stored in the System, the password should be hashed using bcrypt or stronger algorithm and should never be stored or retrievable in plaintext.
**Validates: Requirements 8.2**

#### Property 31: Password Complexity Enforcement
*For any* password submission (creation or reset), the System should accept only passwords meeting complexity requirements (minimum 12 characters, uppercase, lowercase, number, special character) and should reject non-compliant passwords.
**Validates: Requirements 8.3**

#### Property 32: Data Encryption at Rest
*For any* borrower personal data stored in the database, the data should be encrypted using AES-256 and should not be readable without decryption.
**Validates: Requirements 8.4**

#### Property 33: Session Timeout Enforcement
*For any* user session inactive for 30 minutes, the System should automatically invalidate the session and require re-authentication for subsequent requests.
**Validates: Requirements 8.6**

#### Property 34: Unauthorized Access Denial and Logging
*For any* user attempting to access a resource without required permissions, the System should deny the request and should log the attempt with user ID, resource, and timestamp.
**Validates: Requirements 8.7**

#### Property 35: Comprehensive Audit Logging
*For any* system action (user login, profile creation, score generation, API request, data ingestion, model prediction), the System should create an immutable audit log entry containing action type, timestamp, user/client ID, and relevant parameters.
**Validates: Requirements 9.1, 9.2, 9.3, 9.4, 7.9**

#### Property 36: Audit Log Immutability
*For any* audit log entry created, the entry should be immutable such that no user or system process can modify or delete the entry after creation.
**Validates: Requirements 9.5**

#### Property 37: Audit Log Export Filtering
*For any* audit log export request with filters (date range, user, action type), the exported logs should contain all and only the log entries matching the filter criteria.
**Validates: Requirements 9.7**

#### Property 38: Model Fairness Metrics Calculation
*For any* compliance report generation, the System should calculate and display fairness metrics (demographic parity, equal opportunity) across demographic groups (age, gender, location, livelihood).
**Validates: Requirements 9.8**

#### Property 39: Cache Consistency
*For any* frequently accessed data (mandi prices, climate data) that is cached, repeated requests within the cache validity period should return identical data from cache without fetching from source.
**Validates: Requirements 10.7**

#### Property 40: Mandatory Field Validation
*For any* borrower data submission, the System should validate that all mandatory fields (name, age, mobile, location, livelihood, loan amount) are present and non-empty, rejecting submissions with any missing mandatory field.
**Validates: Requirements 11.1**

#### Property 41: Geographic Boundary Validation
*For any* geographic coordinates submitted, the System should accept coordinates within India's bounding box (approximately 8°N to 37°N latitude, 68°E to 97°E longitude) and should reject coordinates outside this box.
**Validates: Requirements 11.3**

#### Property 42: Statistical Outlier Detection
*For any* alternative data ingested (climate, mandi prices), the System should flag values exceeding ±3 standard deviations from historical mean as outliers for review.
**Validates: Requirements 11.4**

#### Property 43: Data Quality Confidence Scoring
*For any* data entity (borrower profile, alternative data), the System should calculate and assign a confidence score (0-1) based on completeness, validity, and consistency, with higher scores for higher quality data.
**Validates: Requirements 11.5**

#### Property 44: Critical Field Threshold Rejection
*For any* borrower profile with more than 30% of critical fields missing, the System should reject the profile and should not generate a risk score.
**Validates: Requirements 11.6**

#### Property 45: Mandi Price Anomaly Detection
*For any* mandi price ingested, the System should validate the price against historical ranges for that commodity and location, flagging prices outside the expected range as anomalies.
**Validates: Requirements 11.7**

#### Property 46: Data Quality Report Metrics
*For any* data quality report generated, the report should include calculated metrics for completeness (% non-null fields), validity (% passing validation), and consistency (% matching expected patterns).
**Validates: Requirements 11.8**

#### Property 47: Model Performance Tracking
*For any* loan outcome (default or repayment) recorded, the System should update model performance metrics (accuracy, precision, recall, AUC-ROC) to reflect the actual outcome versus predicted risk.
**Validates: Requirements 12.1, 12.2**

#### Property 48: Model Accuracy Alert Trigger
*For any* model performance calculation where accuracy drops below 70%, the System should trigger an alert to Data Scientists with details of the performance degradation.
**Validates: Requirements 12.3**

#### Property 49: Data Drift Detection
*For any* input feature, the System should compare the current distribution (last 30 days) with the training distribution and should flag features with >15% distribution deviation as experiencing drift.
**Validates: Requirements 12.4, 12.5**

#### Property 50: Model Version Rollback Capability
*For any* new model version deployed, the System should maintain the previous version such that rollback to the previous version is possible without data loss or service interruption.
**Validates: Requirements 12.7**

#### Property 51: Model Lineage Completeness
*For any* model version, the System should maintain complete lineage metadata including training dataset reference, hyperparameters, training date, performance metrics, and deployment date.
**Validates: Requirements 12.8**

#### Property 52: Synthetic Data Distribution Realism
*For any* batch of synthetic borrower profiles generated, the distributions of age, livelihood type, location, and income should match realistic rural India demographics within statistical tolerance (Chi-square test p-value > 0.05).
**Validates: Requirements 13.1, 13.6**

#### Property 53: Synthetic Climate Data Historical Consistency
*For any* synthetic climate data generated for a region, the rainfall and temperature patterns should match historical seasonal patterns for that region within ±20% deviation.
**Validates: Requirements 13.2**

#### Property 54: Synthetic Mandi Price Seasonality
*For any* synthetic mandi price data generated for a crop, the prices should exhibit seasonal variation consistent with that crop's harvest cycles (higher prices pre-harvest, lower post-harvest).
**Validates: Requirements 13.3**

#### Property 55: Synthetic Outcome Correlation
*For any* batch of synthetic loan outcomes generated, the default rate should be negatively correlated with risk scores (higher scores = lower default rate) with correlation coefficient < -0.5.
**Validates: Requirements 13.4**

#### Property 56: Synthetic Data Volume Minimum
*For any* synthetic data generation request, the System should produce at least 1000 borrower records with complete profiles and associated alternative data.
**Validates: Requirements 13.5**

#### Property 57: Synthetic Data Labeling
*For any* synthetic data record created, the record should have a clearly marked flag (is_synthetic = true) to distinguish it from real borrower data.
**Validates: Requirements 13.7**

#### Property 58: Synthetic Data Parameterization
*For any* synthetic data generation with custom parameters (sample size, default rate, feature distributions), the generated data should reflect the specified parameters within ±5% tolerance.
**Validates: Requirements 13.8**

#### Property 59: Portfolio Risk Distribution Accuracy
*For any* portfolio risk distribution report, the count of borrowers in each risk band should match the actual count of borrowers with scores in that band's range.
**Validates: Requirements 14.1**

#### Property 60: Geographic Risk Aggregation
*For any* geographic risk heatmap, the risk level for each region should equal the average risk score of all borrowers in that region.
**Validates: Requirements 14.2**

#### Property 61: Livelihood Risk Analysis Calculation
*For any* livelihood-based risk analysis report, the average risk score for each livelihood type should match the calculated mean of all borrowers with that livelihood type.
**Validates: Requirements 14.3**

#### Property 62: Trend Report Time-Series Accuracy
*For any* trend report showing risk score changes over time, the data points should match the actual average risk scores calculated for each time period.
**Validates: Requirements 14.4**

#### Property 63: Climate Impact Correlation Calculation
*For any* climate impact report, the correlation between weather anomalies and risk scores should be calculated using Pearson correlation coefficient on the actual data.
**Validates: Requirements 14.5**

#### Property 64: Multi-Format Export Validity
*For any* report or data export, the System should produce valid files in the requested format (PDF, CSV) that can be opened by standard applications without errors.
**Validates: Requirements 4.7, 6.7, 14.7**

#### Property 65: User Account Creation with Role Assignment
*For any* user account created by a System Administrator, the account should be created with the specified role and should have exactly the permissions associated with that role.
**Validates: Requirements 15.1**

#### Property 66: Account Deactivation Access Revocation
*For any* user account deactivated, all active sessions for that user should be immediately invalidated and subsequent authentication attempts should fail.
**Validates: Requirements 15.2, 15.3**

#### Property 67: Password Reset Functionality
*For any* password reset operation, the old password should be invalidated and the new password should be required for subsequent authentication.
**Validates: Requirements 15.4**

#### Property 68: Username and Email Uniqueness
*For any* user account creation attempt with a username or email that already exists, the System should reject the creation and return an error indicating the duplicate field.
**Validates: Requirements 15.5**

#### Property 69: Account Modification Notification
*For any* user account creation or modification, the System should send (or queue for sending) an email notification to the user's email address.
**Validates: Requirements 15.6**

#### Property 70: User Activity Log Visibility
*For any* user, the System should display an activity log showing all actions performed by that user with timestamps, ordered by most recent first.
**Validates: Requirements 15.7**

#### Property 71: Role Modification Audit Trail
*For any* user role modification, the System should create an audit log entry recording the old role, new role, timestamp, and administrator who made the change.
**Validates: Requirements 15.8**

## Error Handling

### Error Categories

The system implements comprehensive error handling across four categories:

#### 1. Validation Errors (HTTP 400)
- **Trigger**: Invalid input data (missing fields, out-of-range values, format violations)
- **Response**: Detailed error message specifying which validation failed
- **Example**: "Borrower age must be between 18 and 75. Provided: 16"
- **Logging**: Log validation errors with input data (sanitized of PII) for debugging
- **User Action**: Correct input and resubmit

#### 2. Authentication/Authorization Errors (HTTP 401/403)
- **Trigger**: Missing/invalid credentials, insufficient permissions
- **Response**: Generic error message to prevent information leakage
- **Example**: "Authentication required" or "Insufficient permissions"
- **Logging**: Log authentication failures with user ID and attempted resource for security monitoring
- **User Action**: Authenticate or request access from administrator

#### 3. External Service Errors (HTTP 502/503)
- **Trigger**: External API failures (weather, mandi price sources)
- **Response**: Error message indicating service unavailability
- **Fallback**: Use cached data with staleness indicator
- **Example**: "Weather data temporarily unavailable. Using cached data from 2024-01-15."
- **Logging**: Log external service failures with retry attempts and response codes
- **User Action**: Retry later or proceed with cached data

#### 4. Internal Server Errors (HTTP 500)
- **Trigger**: Unexpected system errors (database failures, model inference errors)
- **Response**: Generic error message with incident ID for tracking
- **Example**: "An unexpected error occurred. Incident ID: abc-123-def"
- **Logging**: Log full stack trace, request context, and system state for debugging
- **User Action**: Contact support with incident ID

### Error Handling Patterns

#### Retry Logic with Exponential Backoff
```python
def fetch_with_retry(api_call, max_retries=3):
    for attempt in range(max_retries):
        try:
            return api_call()
        except TransientError as e:
            if attempt == max_retries - 1:
                raise
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            time.sleep(wait_time)
```

#### Circuit Breaker for External Services
- **Closed State**: Normal operation, requests pass through
- **Open State**: After 5 consecutive failures, stop calling external service for 60 seconds
- **Half-Open State**: After timeout, allow one test request to check if service recovered

#### Graceful Degradation
- **Primary Path**: Fetch fresh data from external APIs
- **Fallback Path 1**: Use cached data from last successful fetch
- **Fallback Path 2**: Use regional averages with confidence penalty
- **Last Resort**: Return error if no fallback data available

#### Dead Letter Queues
- Failed data ingestion jobs sent to SQS dead letter queue
- Manual review and reprocessing of failed jobs
- Alerts triggered after 10 messages in DLQ

### Error Monitoring and Alerting

#### CloudWatch Alarms
- **High Error Rate**: Alert if error rate exceeds 5% of requests
- **External Service Failures**: Alert if external API fails for >5 minutes
- **Model Inference Failures**: Alert immediately on any model inference error
- **Database Connection Failures**: Alert immediately on connection pool exhaustion

#### Error Dashboards
- Real-time error rate by endpoint and error type
- Error trends over time (hourly, daily, weekly)
- Top error messages and their frequencies
- Mean time to resolution (MTTR) tracking

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

#### Unit Tests
Unit tests verify specific examples, edge cases, and error conditions. They are helpful for:
- Specific examples that demonstrate correct behavior
- Integration points between components
- Edge cases and boundary conditions
- Error handling scenarios

**Focus Areas**:
- API endpoint integration tests
- Database query correctness
- External API mocking and error simulation
- Authentication and authorization flows
- Specific calculation examples (e.g., risk score for a known borrower profile)

**Avoid**: Writing too many unit tests for scenarios that property tests can cover. Property tests handle comprehensive input coverage through randomization.

#### Property-Based Tests
Property tests verify universal properties across all inputs. They are essential for:
- Universal properties that hold for all inputs
- Comprehensive input coverage through randomization
- Discovering edge cases not anticipated by developers

**Configuration**:
- **Library**: Use `hypothesis` (Python), `fast-check` (TypeScript/JavaScript), or equivalent for the implementation language
- **Iterations**: Minimum 100 iterations per property test
- **Tagging**: Each property test must reference its design document property
- **Tag Format**: `# Feature: gramscore-ai, Property {number}: {property_text}`

**Focus Areas**:
- Input validation properties (Properties 2, 31, 40, 41, 42, 44, 45)
- Data transformation properties (Properties 9, 23, 39, 52-58)
- Calculation accuracy properties (Properties 22, 38, 46, 47, 59-63)
- Invariant properties (Properties 1, 3, 8, 36, 51, 57, 71)
- Error handling properties (Properties 7, 12, 28, 34)

**Example Property Test**:
```python
from hypothesis import given, strategies as st
import pytest

# Feature: gramscore-ai, Property 2: Input Validation Consistency
@given(age=st.integers())
def test_age_validation_property(age):
    """
    For any age input, the System should accept ages in [18, 75]
    and reject ages outside this range with specific error message.
    """
    borrower = create_borrower_with_age(age)
    
    if 18 <= age <= 75:
        # Valid age should be accepted
        result = validate_borrower(borrower)
        assert result.is_valid
    else:
        # Invalid age should be rejected with specific error
        result = validate_borrower(borrower)
        assert not result.is_valid
        assert "age must be between 18 and 75" in result.error_message.lower()
```

### Test Coverage Requirements

- **Overall Code Coverage**: Minimum 80%
- **Critical Paths Coverage**: Minimum 95% (risk scoring, explainability, authentication)
- **Property Test Coverage**: All 71 correctness properties must have corresponding property tests
- **Integration Test Coverage**: All API endpoints, database operations, external service integrations

### Testing Environments

#### Local Development
- Docker Compose setup with PostgreSQL, LocalStack (S3/Lambda emulation)
- Synthetic data generation for testing
- Fast feedback loop for developers

#### CI/CD Pipeline
- Automated test execution on every commit
- Unit tests, property tests, integration tests
- Code coverage reporting
- Security scanning (SAST, dependency vulnerabilities)

#### Staging Environment
- AWS environment mirroring production
- End-to-end testing with realistic data volumes
- Performance testing and load testing
- Security testing (DAST, penetration testing)

### Performance Testing

#### Load Testing
- **Tool**: Apache JMeter or Locust
- **Scenarios**:
  - 100 concurrent users submitting borrower profiles
  - 50 concurrent risk score generation requests
  - 20 concurrent simulation requests
- **Success Criteria**:
  - 95% of requests complete within SLA (3 seconds for risk scoring)
  - Error rate < 1%
  - System remains stable under load

#### Stress Testing
- Gradually increase load until system breaks
- Identify bottlenecks and capacity limits
- Verify graceful degradation under extreme load

#### Soak Testing
- Run at 70% capacity for 24 hours
- Detect memory leaks and resource exhaustion
- Verify system stability over extended periods

## Deployment Architecture

### AWS Infrastructure

#### Compute
- **Lambda Functions**: Serverless compute for data ingestion, API handlers, data processing
  - Memory: 512MB - 3GB depending on function
  - Timeout: 30 seconds for API handlers, 15 minutes for batch processing
  - Concurrency: Reserved concurrency for critical functions
- **SageMaker**: Managed ML platform for model training and inference
  - Training: ml.m5.xlarge instances for model training
  - Inference: ml.t2.medium instances with auto-scaling (1-10 instances)
- **EC2** (Optional): For web application hosting if not using S3+CloudFront
  - Instance Type: t3.medium (2 vCPU, 4GB RAM)
  - Auto Scaling Group: Min 2, Max 10 instances

#### Storage
- **RDS PostgreSQL**: Primary database for structured data
  - Instance: db.t3.medium (2 vCPU, 4GB RAM)
  - Multi-AZ: Enabled for high availability
  - Storage: 100GB SSD with auto-scaling to 500GB
  - Backups: Automated daily backups, 7-day retention
- **S3**: Object storage for unstructured data
  - Buckets: raw-data, feature-store, model-artifacts, reports
  - Lifecycle Policies: Transition to Glacier after 90 days, delete after 7 years
  - Versioning: Enabled for model-artifacts and feature-store
- **ElastiCache Redis** (Optional): Caching layer for frequently accessed data
  - Node Type: cache.t3.micro
  - Cluster Mode: Disabled for simplicity

#### Networking
- **VPC**: Isolated network for all resources
  - CIDR: 10.0.0.0/16
  - Subnets: Public (API Gateway, NAT), Private (Lambda, RDS, SageMaker)
- **API Gateway**: RESTful API with custom domain
  - Throttling: 100 requests/second per client
  - Caching: Enabled for GET requests (5-minute TTL)
- **CloudFront**: CDN for web application
  - Origin: S3 bucket with static files
  - SSL/TLS: Certificate from ACM
  - Caching: Aggressive caching for static assets

#### Security
- **IAM**: Least-privilege roles for all services
- **Cognito**: User authentication and authorization
  - User Pool: For loan officers, credit managers, administrators
  - Identity Pool: For federated access from web app
- **KMS**: Encryption key management
  - Customer-managed keys for sensitive data
  - Automatic key rotation enabled
- **Secrets Manager**: Secure storage for API keys and credentials
- **WAF**: Web Application Firewall for API Gateway
  - Rate limiting rules
  - SQL injection and XSS protection
- **Security Groups**: Restrictive inbound/outbound rules
  - RDS: Only accessible from Lambda security group
  - Lambda: Only outbound to RDS, S3, SageMaker

#### Monitoring and Logging
- **CloudWatch**: Centralized logging and monitoring
  - Log Groups: Separate groups for Lambda, API Gateway, RDS
  - Metrics: Custom metrics for business KPIs
  - Alarms: Automated alerts for errors and performance issues
- **CloudTrail**: Audit trail of all AWS API calls
- **X-Ray**: Distributed tracing for debugging
- **SNS**: Notification service for alerts
  - Topics: critical-alerts, warning-alerts, info-alerts
  - Subscriptions: Email, SMS for on-call engineers

### Deployment Pipeline

#### CI/CD Workflow
1. **Code Commit**: Developer pushes code to GitHub
2. **Build**: GitHub Actions triggers build
   - Install dependencies
   - Run linters and formatters
   - Run unit tests and property tests
   - Generate code coverage report
3. **Security Scan**: SAST and dependency vulnerability scanning
4. **Package**: Create deployment packages
   - Lambda functions: ZIP files
   - SageMaker models: Model artifacts
   - Web app: Static files
5. **Deploy to Staging**: Automated deployment to staging environment
   - Infrastructure as Code (Terraform or CloudFormation)
   - Database migrations
   - Lambda function updates
   - SageMaker endpoint updates
6. **Integration Tests**: Run end-to-end tests in staging
7. **Manual Approval**: Product owner approves production deployment
8. **Deploy to Production**: Blue-green deployment
   - Deploy new version alongside old version
   - Gradually shift traffic to new version
   - Monitor error rates and performance
   - Rollback if issues detected
9. **Post-Deployment**: Smoke tests and monitoring

#### Infrastructure as Code
- **Tool**: Terraform or AWS CloudFormation
- **Structure**:
  - `modules/`: Reusable infrastructure modules (VPC, RDS, Lambda)
  - `environments/`: Environment-specific configurations (dev, staging, prod)
  - `main.tf`: Root module orchestrating all resources
- **State Management**: Terraform state stored in S3 with DynamoDB locking
- **Version Control**: All infrastructure code in Git

#### Database Migrations
- **Tool**: Flyway or Liquibase
- **Process**:
  - Migrations versioned and stored in Git
  - Automated execution during deployment
  - Rollback scripts for each migration
  - Validation before and after migration

### Disaster Recovery

#### Backup Strategy
- **RDS**: Automated daily backups, 7-day retention, manual snapshots before major changes
- **S3**: Versioning enabled, cross-region replication for critical buckets
- **Configuration**: Infrastructure code in Git, secrets in Secrets Manager

#### Recovery Procedures
- **RDS Failure**: Automatic failover to standby instance in Multi-AZ setup (< 2 minutes)
- **Region Failure**: Manual failover to backup region using cross-region replicas (< 1 hour)
- **Data Corruption**: Restore from latest backup snapshot (< 30 minutes)

#### Recovery Time Objective (RTO) and Recovery Point Objective (RPO)
- **RTO**: 1 hour (maximum acceptable downtime)
- **RPO**: 24 hours (maximum acceptable data loss)

## Scalability Strategy

### Horizontal Scaling

#### Lambda Functions
- **Auto-Scaling**: Automatic based on request volume
- **Concurrency**: Reserved concurrency for critical functions to prevent throttling
- **Optimization**: Keep functions warm with scheduled pings to reduce cold starts

#### SageMaker Endpoints
- **Auto-Scaling**: Based on invocations per instance metric
- **Target**: 70% utilization per instance
- **Scale-Out**: Add instances when utilization > 70% for 3 minutes
- **Scale-In**: Remove instances when utilization < 30% for 15 minutes

#### RDS
- **Read Replicas**: Add read replicas for analytics queries
- **Connection Pooling**: RDS Proxy to manage connection pooling
- **Vertical Scaling**: Upgrade instance type during maintenance window if needed

### Vertical Scaling

#### Lambda Memory
- Increase memory allocation for compute-intensive functions
- Memory increase also increases CPU allocation

#### RDS Instance Type
- Upgrade to larger instance types (db.t3.large, db.m5.xlarge) as data grows
- Scheduled during maintenance window to minimize downtime

### Data Partitioning

#### S3 Partitioning
- Partition by date: `s3://bucket/year=2024/month=01/day=15/`
- Partition by entity: `s3://bucket/borrower_id=123/`
- Enables efficient querying and lifecycle management

#### Database Sharding (Future)
- Shard by geographic region (state or district)
- Each shard handles borrowers from specific regions
- Reduces query load on individual databases

### Caching Strategy

#### Application-Level Caching
- **Redis/ElastiCache**: Cache frequently accessed data
  - Mandi prices (TTL: 1 day)
  - Climate data (TTL: 1 day)
  - Risk scores (TTL: 1 hour)
- **Cache Invalidation**: Invalidate on data updates

#### API Gateway Caching
- Cache GET requests for risk scores and explainability reports
- TTL: 5 minutes
- Cache key: Request path + query parameters

#### CloudFront Caching
- Aggressive caching for static web app assets
- TTL: 24 hours for JS/CSS, 1 hour for HTML

### Performance Optimization

#### Database Optimization
- **Indexing**: Create indexes on frequently queried columns (borrower_id, created_at, risk_band)
- **Query Optimization**: Use EXPLAIN to analyze and optimize slow queries
- **Connection Pooling**: Reuse database connections to reduce overhead

#### Model Optimization
- **Model Compression**: Use quantization to reduce model size
- **Batch Inference**: Process multiple predictions in a single SageMaker call
- **Model Caching**: Cache model in memory to avoid repeated loading

#### Lambda Optimization
- **Code Optimization**: Minimize cold start time by reducing package size
- **Provisioned Concurrency**: Keep Lambda functions warm for critical paths
- **Async Processing**: Use SQS for non-critical async tasks

## Future Extensibility

### Phase 2 Enhancements

#### Mobile Application
- **Platform**: React Native for cross-platform (Android/iOS)
- **Features**: Offline data collection, photo capture for land verification, voice input
- **Sync**: Background synchronization when connectivity restored

#### Advanced Data Sources
- **Satellite Imagery**: Integrate with Planet Labs or Sentinel for crop health assessment
- **Social Data**: Community reputation scores (with consent)
- **Digital Payments**: UPI transaction history for borrowers with digital footprints
- **Government Schemes**: Integration with PM-KISAN, MGNREGA databases

#### Enhanced AI Models
- **Deep Learning**: Explore neural networks for improved accuracy
- **Ensemble Models**: Combine multiple models (XGBoost, Random Forest, Neural Network)
- **Transfer Learning**: Adapt models trained in one region to new regions
- **Continuous Learning**: Online learning to update models with new loan outcomes

### Architectural Extensibility

#### Microservices Migration
- Current: Monolithic Lambda functions
- Future: Decompose into microservices (borrower service, scoring service, explainability service)
- Benefits: Independent scaling, technology diversity, fault isolation

#### Event-Driven Architecture
- Current: Synchronous API calls
- Future: Event-driven with EventBridge or Kafka
- Benefits: Loose coupling, async processing, event replay

#### Multi-Tenancy
- Current: Single-tenant deployment
- Future: Multi-tenant SaaS platform
- Isolation: Separate databases per tenant or row-level security
- Benefits: Reduced operational overhead, faster onboarding

### Integration Extensibility

#### Webhook Support
- Allow external systems to register webhooks for events (risk score generated, loan approved)
- Secure webhook delivery with HMAC signatures

#### GraphQL API
- Complement REST API with GraphQL for flexible querying
- Benefits: Reduce over-fetching, client-driven queries

#### Batch Processing API
- Support bulk operations (upload 1000 borrowers, generate 1000 risk scores)
- Async processing with status polling or webhooks

### Data Science Extensibility

#### Feature Store
- Centralized feature repository for reuse across models
- Versioned features for reproducibility
- Online and offline feature serving

#### Model Registry
- Centralized registry for all model versions
- Metadata: Training data, hyperparameters, performance metrics
- Lifecycle management: Development, staging, production, archived

#### Experiment Tracking
- Track all model training experiments (MLflow or SageMaker Experiments)
- Compare experiments to identify best models
- Reproducibility: Store code, data, and environment for each experiment

## Tech Stack Summary

### Backend
- **Language**: Python 3.11 (Lambda functions, data processing, model training)
- **ML Framework**: XGBoost, scikit-learn, SHAP
- **API Framework**: FastAPI or Flask (for Lambda handlers)
- **Database**: PostgreSQL 14
- **ORM**: SQLAlchemy
- **Caching**: Redis (ElastiCache)

### Frontend
- **Framework**: React 18 with TypeScript
- **State Management**: Redux Toolkit or Zustand
- **UI Library**: Material-UI or Ant Design
- **Charts**: Recharts or Chart.js
- **Maps**: Leaflet or Mapbox GL JS
- **Build Tool**: Vite

### Infrastructure
- **Cloud Provider**: AWS
- **Compute**: Lambda, SageMaker, EC2 (optional)
- **Storage**: S3, RDS PostgreSQL
- **Networking**: VPC, API Gateway, CloudFront
- **Security**: IAM, Cognito, KMS, Secrets Manager, WAF
- **Monitoring**: CloudWatch, CloudTrail, X-Ray
- **IaC**: Terraform or CloudFormation
- **CI/CD**: GitHub Actions

### Development Tools
- **Version Control**: Git, GitHub
- **Code Quality**: Black (formatter), Flake8 (linter), mypy (type checker)
- **Testing**: pytest, hypothesis (property testing), pytest-cov (coverage)
- **Documentation**: Sphinx (Python docs), Swagger/OpenAPI (API docs)
- **Local Development**: Docker, Docker Compose

### External Services
- **Weather Data**: India Meteorological Department (IMD) API or OpenWeather API
- **Mandi Prices**: Agmarknet API (Government of India)
- **Email**: Amazon SES
- **SMS**: Amazon SNS or Twilio

## Risk Mitigation Strategy

### Technical Risks

#### Risk: External API Unavailability
- **Mitigation**: Implement caching, graceful degradation, circuit breakers
- **Fallback**: Use regional averages when external data unavailable
- **Monitoring**: Alert on consecutive API failures

#### Risk: Model Performance Degradation
- **Mitigation**: Continuous monitoring, automated retraining triggers, A/B testing
- **Fallback**: Rollback to previous model version if performance drops
- **Monitoring**: Track accuracy, precision, recall, AUC-ROC

#### Risk: Data Quality Issues
- **Mitigation**: Comprehensive validation, outlier detection, data quality scoring
- **Fallback**: Reject low-quality data, use confidence penalties
- **Monitoring**: Data quality dashboards, alerts on quality drops

#### Risk: Scalability Bottlenecks
- **Mitigation**: Auto-scaling, load testing, performance optimization
- **Fallback**: Rate limiting, queue-based processing
- **Monitoring**: Track latency, throughput, error rates

### Security Risks

#### Risk: Data Breach
- **Mitigation**: Encryption at rest and in transit, least-privilege access, security audits
- **Detection**: Intrusion detection, anomaly detection in access patterns
- **Response**: Incident response plan, data breach notification procedures

#### Risk: Unauthorized Access
- **Mitigation**: Strong authentication (MFA), RBAC, session management
- **Detection**: Log all access attempts, alert on suspicious patterns
- **Response**: Automatic account lockout, security team notification

#### Risk: API Abuse
- **Mitigation**: Rate limiting, API key management, WAF rules
- **Detection**: Monitor for unusual request patterns
- **Response**: Throttle or block abusive clients

### Operational Risks

#### Risk: Deployment Failures
- **Mitigation**: Blue-green deployments, automated rollback, comprehensive testing
- **Detection**: Smoke tests, error rate monitoring
- **Response**: Automatic rollback on failure detection

#### Risk: Data Loss
- **Mitigation**: Automated backups, cross-region replication, versioning
- **Detection**: Data integrity checks, backup validation
- **Response**: Restore from backups, disaster recovery procedures

#### Risk: Vendor Lock-In
- **Mitigation**: Use open standards (REST, SQL), abstract cloud-specific services
- **Portability**: Infrastructure as Code enables migration to other clouds
- **Response**: Multi-cloud strategy for critical components

### Regulatory Risks

#### Risk: Non-Compliance with RBI Guidelines
- **Mitigation**: Regular compliance audits, legal review, explainability features
- **Detection**: Compliance monitoring, fairness metrics
- **Response**: Remediation plan, regulatory reporting

#### Risk: Data Privacy Violations
- **Mitigation**: Consent management, data minimization, anonymization
- **Detection**: Privacy impact assessments, audit logs
- **Response**: Data deletion procedures, privacy breach notification

#### Risk: Algorithmic Bias
- **Mitigation**: Fairness metrics, diverse training data, bias testing
- **Detection**: Demographic parity analysis, equal opportunity metrics
- **Response**: Model retraining, bias mitigation techniques

---

This design document provides a comprehensive blueprint for implementing GramScore AI. The architecture is cloud-native, scalable, and secure, with clear component boundaries and well-defined interfaces. The correctness properties ensure that the system behaves correctly across all scenarios, and the testing strategy provides confidence in the implementation. The deployment architecture leverages AWS managed services for reliability and reduced operational overhead, while the extensibility considerations ensure the platform can evolve to meet future needs.
