# GramScore AI
### AI-Powered Climate & Livelihood-Aware Credit Intelligence for Rural India

## Overview
GramScore AI is an alternative credit scoring platform designed for rural India. 
It leverages climate data, crop cycles, mandi prices, and livelihood intelligence to generate explainable AI-based credit risk scores.

## Problem
Traditional credit scoring models fail in rural contexts due to:
- Seasonal income variability
- Lack of formal credit history
- Climate risk exposure
- Informal cash flows

## Solution
GramScore AI provides:
- Rural-specific AI credit scoring
- Climate-adjusted repayment capacity estimation
- Explainable risk factors using SHAP
- Loan officer dashboard
- API-based integration

## Architecture
- AWS SageMaker for model hosting
- AWS Lambda for inference
- API Gateway for secure access
- RDS for structured storage
- S3 for data storage

## Tech Stack
- Python
- XGBoost
- SHAP
- AWS Cloud Services
- React (Dashboard)
- REST APIs

## Impact
- Faster loan approvals
- Reduced NPAs
- Improved rural financial inclusion

## Repository Structure
- requirements.md → Project specifications
- design.md → System architecture & technical design
- demo/ → Model and dataset

## Future Scope
- Integration with government schemes
- Expansion to MSME credit
- Mobile-first deployment for rural branches
