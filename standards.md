# Standards & API Reference

> Project: Inventory Optimization · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 28000:2022 — Security and Resilience: Security Management Systems Requirements**
- URL: https://www.iso.org/standard/79612.html
- Relevance: Specifies requirements for a security management system for the supply chain, covering physical threats, cyber risks, and operational risks across inventory, warehousing, and logistics. Updated in March 2022, it now applies to all organisation types and sizes. Relevant when an inventory optimisation platform intersects with risk management, supplier resilience, or logistics security for enterprise customers.

**ISO 55000 — Asset Management**
- URL: https://www.iso.org/standard/55088.html
- Relevance: Provides principles and terminology for asset management, applicable to organisations managing physical inventory as a corporate asset. Relevant for inventory optimisation tools targeting manufacturers and infrastructure-heavy industries where inventory is treated as a managed asset portfolio.

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Relevance: The foundation of third-party API authentication used by ERP connectors (SAP, Oracle, Microsoft Dynamics), e-commerce platforms (Shopify, BigCommerce), and cloud supply chain tools. An inventory optimisation platform must implement OAuth 2.0 to obtain and refresh credentials when pulling sales, order, and inventory data from connected systems on behalf of users.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Relevance: Defines how bearer tokens are transmitted in HTTP requests. Required for compliance with all ERP and e-commerce platform authentication patterns used in inventory integrations.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Relevance: Defines HTTP method semantics and status codes underpinning all REST API integrations. Critical for correct error handling in inventory synchronisation, replenishment feed ingestion, and demand sensing API calls.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Relevance: Standardises structured error response payloads from REST APIs. Adopting this for the inventory optimisation platform's own API layer makes error handling consistent for integration partners connecting ERP, WMS, or marketplace systems.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html ; https://swagger.io/specification/
- Relevance: The industry standard for documenting REST APIs. SAP IBP, Oracle Fusion Cloud SCM, Cin7, Unleashed, and ToolsGroup SO99+ all expose APIs documented in OpenAPI/Swagger format. An inventory optimisation platform should both consume OpenAPI-defined APIs from ERP/WMS partners and publish its own OpenAPI 3.1 spec to enable SDK generation and contract testing.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification
- Relevance: Standard for validating JSON payloads. Used to validate product catalogue records, demand signal feeds, purchase order payloads, and replenishment recommendation outputs. Critical for ensuring data quality in high-SKU environments where data inconsistencies compound forecasting errors.

**GS1 EDI — Electronic Data Interchange Standards**
- URL: https://www.gs1.org/standards/edi/implementation
- Relevance: GS1 EDI automates the exchange of purchase orders (850), invoices (810), advance shipping notices (856), and inventory inquiry/advice (846) between trading partners. Inventory optimisation platforms targeting enterprise retailers or Walmart/Target supplier programmes must support GS1 EDI over AS2 or SFTP alongside REST APIs.

**GS1 GDSN — Global Data Synchronisation Network**
- URL: https://www.gs1.org/services/gdsn
- Relevance: A network of interoperable data pools for synchronising product master data (GTINs, attributes, classification) between trading partners. Relevant for inventory optimisation platforms that ingest product master data from CPG manufacturers and retailers.

**GS1 EPCIS 2.0 — Electronic Product Code Information Services**
- URL: https://www.gs1.org/standards/epcis ; https://openepcis.io/docs/epcis/
- Relevance: A GS1 standard for capturing and sharing event data about physical movement of goods (what, where, when, why, how). EPCIS 2.0 provides near-real-time inventory visibility when paired with RFID or barcode scanning. Inventory optimisation platforms that consume warehouse event streams or IoT data should support EPCIS 2.0 as a source of real-time inventory position data.

**ANSI ASC X12 EDI — Transaction Sets for Inventory and Procurement**
- URL: https://x12.org/products/transaction-sets
- Relevance: US retail standard EDI transaction sets including 850 (Purchase Order), 856 (ASN), 810 (Invoice), and 846 (Inventory Inquiry/Advice). Required for enterprise integration with Walmart Marketplace suppliers and major CPG wholesale channels.

**Apache Kafka / CloudEvents — Event Streaming for Inventory**
- URL: https://kafka.apache.org ; https://cloudevents.io
- Relevance: Apache Kafka is the dominant event streaming platform for real-time inventory management at scale (used by Walmart processing 11 billion events/day, Albertsons across 2,200 stores). CloudEvents (CNCF) provides a standard schema for describing event data. Inventory optimisation platforms requiring continuous demand sensing should support Kafka-based integration and CloudEvents payloads for inventory change events.

### Frameworks & Methodologies (De Facto Standards)

**SCOR Digital Standard (SCOR DS) — Supply Chain Operations Reference**
- URL: https://www.ascm.org/corporate-solutions/standards-tools/scor-ds/
- Relevance: Managed by ASCM (formerly APICS/Supply Chain Council), SCOR DS is the cross-industry process reference model for supply chain management. It defines 150+ hierarchically structured KPIs (fill rate, OTIF, cash-to-cash cycle time, days of supply, inventory days of supply) that provide the measurement framework for inventory optimisation outcomes. The SCOR DS is open-access and now includes sustainability standards and supply chain orchestration enablers.

**Multi-Echelon Inventory Optimisation (MEIO) — Analytical Framework**
- URL: https://umbrex.com/resources/frameworks/supply-chain-frameworks/multi-echelon-inventory-optimization-meio/ ; https://stockpyl.readthedocs.io/en/latest/tutorial/tutorial_meio.html
- Relevance: MEIO is the established academic and industry framework for optimising inventory across distribution networks (serial, tree, or arbitrary topology systems). Based on Clark–Scarf (1960) and extended by Chen–Zheng (1994), MEIO is now implemented by major commercial platforms (ToolsGroup, Logility, SAP IBP) and can be approached via stochastic-service model (SSM) or guaranteed-service model (GSM) formulations. The open-source `stockpyl` Python library (MIT licence) provides a reference implementation of MEIO algorithms.

**ABC/XYZ Analysis — Inventory Segmentation Standard**
- URL: https://www.ascm.org/ (published in ASCM supply chain management dictionary)
- Relevance: The de facto industry methodology for tiering inventory optimisation strategy. ABC classifies by value contribution (A: top 80% of value; B: next 15%; C: bottom 5%). XYZ classifies by demand variability (X: stable; Y: variable; Z: irregular/lumpy). Combined ABC/XYZ segmentation determines which replenishment model and safety stock policy applies to each SKU class.

**Vendor-Managed Inventory (VMI) — Industry Practice**
- URL: (no single standards body; described in GS1 and ASCM publications)
- Relevance: VMI is a supply chain practice in which the supplier manages replenishment at the customer's location using shared demand and inventory data. Inventory optimisation platforms targeting manufacturer or CPG clients are expected to support VMI data-sharing workflows, including EDI 846 inventory inquiry feeds and supplier portal access.

### Security & Compliance Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Relevance: Defines the ten most critical API security risks, including Broken Object-Level Authorization (critical for multi-tenant SaaS platforms with multiple inventory locations), Broken Authentication (OAuth token hygiene for ERP integrations), and Unrestricted Resource Consumption (demand signal ingestion rate limiting). Essential baseline for enterprise customer security reviews.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Relevance: Applies to any inventory optimisation platform processing EU business data, including supplier details, customer demand signals, and order data that may contain personal identifiers. Data Processing Agreements are required with ERP and e-commerce platform providers. Data minimisation and retention policies must be implemented for demand history containing buyer-level data.

### MCP Server Specifications

**Model Context Protocol (MCP) — Agentic Supply Chain Integration**
- URL: https://modelcontextprotocol.io/ ; https://www.arcweb.com/blog/ai-supply-chain-part-3-mcp-model-context-protocol-shared-reasoning-across-agents
- Relevance: MCP is emerging as the standard for connecting AI agents to supply chain tools and data sources. In inventory management, MCP enables agents to simultaneously query inventory databases (stock levels, reorder status), ERP systems (purchase orders, supplier lead times), and external data sources (weather, logistics disruptions) within a single reasoning workflow — automating replenishment decisions that previously required human orchestration. By 2026, MCP has 97M+ monthly SDK downloads and governance under the Agentic AI Foundation. Inventory optimisation platforms should plan for MCP server exposure alongside their REST API to support AI agent-driven replenishment workflows.

---

## Similar Products — Developer Documentation & APIs

### SAP Integrated Business Planning (SAP IBP)

- **Description:** SAP's cloud-based supply chain planning solution covering demand planning, inventory optimisation, supply planning, and S&OP, powered by SAP HANA in-memory technology. One of the dominant enterprise platforms with deep integration into SAP ERP and S/4HANA.
- **API Documentation:** https://api.sap.com/products/SAPIntegratedBusinessPlanningforSupplyChain/apis/packages
- **Developer Guide:** https://help.sap.com/docs/SAP_INTEGRATED_BUSINESS_PLANNING
- **Integration Method:** OData APIs (communication scenario SAP_COM_0720 for external planning data integration); SAP Cloud Integration for on-premise connectors
- **Standards:** OData v4, REST/JSON, OpenAPI specs via SAP API Business Hub
- **Authentication:** OAuth 2.0 (SAP BTP Identity Authentication)

### Oracle Fusion Cloud Supply Chain Management (SCM)

- **Description:** Oracle's cloud ERP and SCM suite covering inventory management, demand planning, supply planning, and procurement. Provides REST APIs for inventory management enabling integration with 3PL, WMS, and third-party planning systems.
- **API Documentation:** https://docs.oracle.com/en/cloud/saas/supply-chain-and-manufacturing/25c/fasrp/
- **Inventory REST APIs:** https://docs.oracle.com/en/cloud/saas/supply-chain-and-manufacturing/25c/faips/inv-rest-apis-inbound.html
- **Standards:** REST/JSON; TMF/OGC-aligned schemas; OpenAPI documentation
- **Authentication:** Oracle Identity Cloud Service (IDCS); OAuth 2.0

### ToolsGroup SO99+

- **Description:** ToolsGroup's flagship probabilistic demand forecasting and inventory optimisation platform. Covers multi-location replenishment, service-level optimisation, and supply disruption modelling. Runs on Microsoft Azure with an exposed OpenAPI 3.0 web API.
- **API Documentation:** OpenAPI 3.0 spec and API viewer on ToolsGroup demo/test instances (contact vendor for production access)
- **Integration Points:** SAP ERP certified integration; Microsoft Dynamics 365 (SO99+ for Dynamics); custom API for third-party connectors
- **Standards:** OpenAPI 3.0; REST/JSON; Azure-hosted microservices
- **Authentication:** OAuth 2.0 and API Key (both supported)

### Cin7 (Core and Omni) API

- **Description:** Cin7 is a multi-location inventory management platform for SMB and mid-market businesses with AI-powered demand forecasting and e-commerce/marketplace integrations. Exposes a REST API for custom integrations covering products, orders, inventory, and suppliers.
- **API Documentation (Cin7 Core):** https://dearinventory.docs.apiary.io/
- **API Documentation (Cin7 Omni):** https://api.cin7.com/
- **Getting Started:** https://help.core.cin7.com/hc/en-us/articles/9982480315407-Connecting-to-the-Cin7-Core-API
- **Standards:** REST/JSON; Apiary/OpenAPI documentation
- **Authentication:** Account ID + API Application Key (generated in Cin7 application settings)

### Unleashed Software API

- **Description:** Unleashed is a cloud inventory management platform for manufacturers and distributors, with real-time demand sensing, multi-location support, and production planning. Provides a REST API covering products, sales orders, purchase orders, inventory movements, and stock levels.
- **API Documentation:** https://apidocs.unleashedsoftware.com/
- **API Access Guide:** https://help.unleashedsoftware.com/accessing-unleashed-api
- **Standards:** REST; both XML and JSON supported (Content-Type/Accept headers); HMAC-SHA256 signature for request authentication
- **Authentication:** API ID + HMAC-SHA256 signature per request; no OAuth

### Netstock Predictor (Inventory Advisor)

- **Description:** Netstock is a cloud-based predictive inventory planning platform that integrates via API connectors with 60+ ERPs (NetSuite, SAP, Dynamics, Acumatica) to provide demand forecasting, replenishment recommendations, and exception-based management.
- **API Documentation:** Available via Netstock's integration partner programme (not publicly published; contact vendor)
- **Integration Hubs:** Acumatica Marketplace, Sage Marketplace, NetSuite SuiteApp
- **Standards:** REST/JSON (ERP-side integration via native ERP APIs)
- **Authentication:** ERP-side OAuth / API key; Netstock-side credentials per installation

### IBM Sterling Supply Chain Intelligence Suite

- **Description:** IBM Sterling powers supply chain management for major global retailers, including inventory optimisation, order management, and supply chain visibility. Exposes open APIs and pre-built connectors for ERP, CRM, e-commerce, and logistics systems.
- **API Documentation:** https://developer.ibm.com/components/sterling/apis/
- **Store Inventory Management APIs:** https://developer.ibm.com/apis/catalog/storeinventory-store-inventory-management/
- **GitHub:** https://github.com/IBM/supply-chain-insights
- **Standards:** REST/JSON; OpenAPI specifications; event-driven integration via IBM MQ / Kafka
- **Authentication:** IBM IAM (Identity and Access Management); OAuth 2.0

### Stockpyl — Open-Source Python Inventory Optimisation Library

- **Description:** Stockpyl is an MIT-licensed Python package for inventory optimisation and simulation implementing classical and modern algorithms: EOQ, newsvendor, Wagner-Whitin, MEIO under stochastic-service model (SSM) and guaranteed-service model (GSM), and multi-echelon simulation. Published via INFORMS TutORials in Operations Research (2023).
- **GitHub:** https://github.com/LarrySnyder/stockpyl
- **PyPI:** https://pypi.org/project/stockpyl/
- **Documentation:** https://stockpyl.readthedocs.io/en/latest/
- **Standards:** MIT Licence; Python 3.x; compatible with NumPy/SciPy scientific stack
- **Authentication:** Not applicable (open-source library)
- **Note:** Provides reference implementations of Clark–Scarf (1960) and Chen–Zheng (1994) MEIO algorithms; useful for building AI-native inventory optimisation core engines.

### Apache Kafka / Confluent Platform — Event Streaming for Inventory

- **Description:** Apache Kafka is the leading open-source distributed event streaming platform for real-time inventory management (used by Walmart, Albertsons, Michelin). Confluent Platform adds enterprise features (Schema Registry, Kafka Connect connectors). Key for inventory optimisation platforms needing continuous demand sensing from POS, warehouse, and marketplace event streams.
- **API Documentation:** https://kafka.apache.org/documentation/
- **Confluent Developer Docs:** https://developer.confluent.io/
- **CloudEvents Spec:** https://cloudevents.io (CNCF standard for event schema)
- **Standards:** Apache 2.0 (Kafka open source); REST Proxy API; Confluent Schema Registry (Avro, JSON Schema, Protobuf)
- **Authentication:** SASL/SCRAM, mTLS, OAuth 2.0 (Confluent Cloud)

---

## Notes

**SCOR DS as a KPI Reference:** Inventory optimisation platforms should map their analytics and reporting outputs to SCOR DS metrics (fill rate, OTIF, inventory days of supply, cash-to-cash cycle time) to facilitate enterprise customer benchmarking and provide a common language with supply chain practitioners.

**EPCIS 2.0 for Real-Time Inventory Events:** EPCIS 2.0 (released 2022) updated the standard to support JSON-LD and REST alongside legacy XML/SOAP, making it accessible for modern cloud inventory platforms. An open-source implementation is available at https://openepcis.io/. Inventory platforms integrating with warehouse management systems or RFID infrastructure should evaluate EPCIS 2.0 as an inbound event data standard.

**Stockpyl as an Open-Source Foundation:** Unlike commercial platforms, Stockpyl provides MIT-licensed reference implementations of classic and multi-echelon inventory algorithms. An AI-native inventory optimisation tool could use Stockpyl as a validated algorithmic baseline, layering ML models on top of its proven operations research foundations.

**MCP and Agentic Replenishment:** The ARC Advisory Group has identified MCP as the emerging standard for shared reasoning across supply chain AI agents. Platforms that expose MCP servers allowing AI agents to query stock levels, trigger purchase orders, and receive demand signals will be better positioned as agentic workflows become mainstream in supply chain planning by 2026–2027.
