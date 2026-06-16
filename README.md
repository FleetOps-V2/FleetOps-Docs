# 📖 FleetOps Documentation Hub

Welcome to the FleetOps technical documentation directory. Please select one of the core modules below to explore:

---

## 🗂️ Documentation Index

1.  **[Core Architecture & System Audit](./ARCHITECTURE.md)**
    *   System topology and container routing diagrams.
    *   Microservice catalog breakdown (Gateway, Auth, Vehicle, Maintenance, Request Services).
    *   Stateless Security Filter context & JWT role claims structure.
    *   Redis caching topology, entry TTLs, and resilience fallbacks.
    *   Service Request state machine transition rules.
    *   Isolated database schema allocations.
    *   Full Integration Audit results (33/33 tests passed).

2.  **[AWS Production Provisioning & Deployment Guide](./DEPLOYMENT_AWS.md)**
    *   Network infrastructure (Custom VPC, Public/Private subnets, NAT/Internet Gateways).
    *   RDS PostgreSQL database clustering and isolated db registration.
    *   Managed ElastiCache Redis setup and connectivity rules.
    *   Secure secret injection using AWS Secrets Manager.
    *   Tagging and pushing container images to AWS Elastic Container Registry (ECR).
    *   Deploying a Docker Compose stack on an EC2 instance.
    *   Production deployment architecture on AWS ECS Fargate.
    *   Path-based listeners on AWS Application Load Balancers (ALB).
    *   Serverless static asset hosting on S3 + CloudFront.

3.  **[Amazon Bedrock AI Co-Pilot Blueprint](./AI_INTEGRATION.md)**
    *   Diagnostics request-response data flow diagram.
    *   AWS Bedrock model provisioning and IAM security policies.
    *   Spring Boot Integration dependencies and bean setups.
    *   Contextual AI prompt structuring utilizing historical vehicle records, mileage, and task queues.
    *   Complete Java Service and REST Controller classes.
    *   React UI Chat Sidebar Component and CSS styles.

---
*For local environment setup and access credentials, please refer to the [Root README](../README.md).*
