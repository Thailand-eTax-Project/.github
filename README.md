# Thailand e-Tax Invoice Platform

A microservices-based platform for processing, signing, and submitting Thai e-Tax documents to the Revenue Department via ebXML (ebMS 2.0).

## Overview

| Component | Description | Port |
|-----------|-------------|------|
| **teda** | Thai e-Tax Invoice library with JAXB classes, XSD schemas, and database-backed code lists | — |
| **eidasremotesigning** | Remote Signing Service via CSC API v2.0 (XAdES/PAdES signatures) | 9000 |
| **invoice-microservices** | 16 Spring Boot services for document processing pipeline | 8081–8101 |

## Project Structure

```
etax/
├── teda/                           # Thai e-Tax Invoice library (JAR)
├── eidasremotesigning/             # eIDAS Remote Signing Service
└── invoice-microservices/
    └── services/
        ├── document-intake-service/                     # 8081 - Document intake & validation
        ├── invoice-processing-service/                 # 8082 - Invoice processing
        ├── taxinvoice-processing-service/              # 8083 - Tax Invoice processing
        ├── receipt-processing-service/                 # 8084 - Receipt processing
        ├── cancellationnote-processing-service/        # 8085 - Cancellation Note processing
        ├── debitcreditnote-processing-service/         # 8086 - Debit/Credit Note processing
        ├── abbreviatedtaxinvoice-processing-service/   # 8087 - Abbreviated Tax Invoice processing
        ├── xml-signing-service/                         # 8088 - XML digital signatures (XAdES)
        ├── pdf-signing-service/                         # 8089 - PDF digital signatures (PAdES)
        ├── invoice-pdf-generation-service/             # 8092 - Invoice PDF/A-3 generation
        ├── taxinvoice-pdf-generation-service/          # 8093 - Tax Invoice PDF/A-3 generation
        ├── receipt-pdf-generation-service/             # 8094 - Receipt PDF/A-3 generation
        ├── cancellationnote-pdf-generation-service/    # 8095 - Cancellation Note PDF/A-3 generation
        ├── debitcreditnote-pdf-generation-service/     # 8096 - Debit/Credit Note PDF/A-3 generation
        ├── abbreviatedtaxinvoice-pdf-generation-service/ # 8097 - Abbreviated Tax Invoice PDF/A-3
        ├── document-storage-service/                    # 8098 - Document archive (MongoDB/S3)
        ├── notification-service/                        # 8099 - Email/webhook notifications
        ├── orchestrator-service/                        # 8100 - Saga orchestrator
        └── ebms-sending-service/                        # 8101 - ebXML submission to Revenue Dept
```

## Technology Stack

| Layer | Technology |
|-------|------------|
| Language | Java 17 (teda, eidasremotesigning), Java 21 (microservices) |
| Framework | Spring Boot 3.2.5, Spring Cloud 2023.0.1 |
| Message Routing | Apache Camel 4.14.4 |
| Database | PostgreSQL 16, MongoDB |
| Messaging | Apache Kafka |
| PDF Generation | Apache FOP 2.x, Apache PDFBox 3.x |
| Digital Signatures | XAdES (XML), PAdES (PDF) via CSC API v2.0 |
| Service Discovery | Netflix Eureka |
| Database Migration | Flyway |

## Supported Document Types

1. **Tax Invoice** (ใบเสร็จรับเงิน/ใบกำกับภาษี)
2. **Invoice** (ใบแจ้งหนี้)
3. **Receipt** (ใบเสร็จรับเงิน)
4. **Debit Note** (ใบเดบิต)
5. **Credit Note** (ใบเครดิต)
6. **Cancellation Note** (ใบยกเลิก)
7. **Abbreviated Tax Invoice** (ใบกำกับภาษีอย่างย่อ)

## Quick Start

### Prerequisites

- Java 17+ / 21+
- Maven 3.6+
- PostgreSQL 16+
- MongoDB
- Kafka on `localhost:9092`
- Eureka (optional)

### Build

```bash
# 1. Build teda library (required first)
cd teda && mvn clean install && cd ..

# 2. Build saga-commons (required for microservices)
cd saga-commons && mvn clean install && cd ..

# 3. Build eidasremotesigning (for signing services)
cd eidasremotesigning && mvn clean package && cd ..

# 4. Build all microservices
cd invoice-microservices/services
for dir in */; do (cd "$dir" && mvn clean package -DskipTests); done
```

### Run

```bash
# Start eidasremotesigning (signing backend)
cd eidasremotesigning && mvn spring-boot:run

# Start microservices
cd invoice-microservices/services/document-intake-service && mvn spring-boot:run
```

## Architecture

### Saga Orchestration Pipeline

```
[Document Intake] → Validated XML
                          ↓
              [Orchestrator] → saga.command.{type}
                          ↓
              [Processing Service] (parse, calculate, persist)
                          ↓
              [XML Signing] (XAdES via CSC API)
                          ↓
              [PDF Generation] (PDF/A-3 with embedded XML)
                          ↓
              [PDF Signing] (PAdES via CSC API)
                          ↓
              [Document Storage] (MongoDB + S3)
                          ↓
              [ebMS Sending] → Thailand Revenue Department
                          ↓
              [Notification] (email/webhook)
```

### Transactional Outbox Pattern

All services use the **Transactional Outbox** pattern with Debezium CDC for reliable event delivery:

1. Domain state + outbox event saved atomically in same transaction
2. Debezium reads `outbox_events` table via PostgreSQL logical replication
3. Events routed to Kafka topics based on `topic` field

## Key Patterns

- **DDD (Domain-Driven Design)**: Hexagonal/Ports-and-Adapters architecture
- **Saga Orchestration**: Orchestrator service coordinates distributed transactions
- **Transactional Outbox**: Atomic persistence + event publishing via Debezium CDC
- **Strategy Pattern**: Document-type-specific validation and PDF generation


## License

MIT License
