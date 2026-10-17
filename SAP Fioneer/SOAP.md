# SOAP – Interview Notes (Java Backend)

> Target role: Java Full Stack / Backend Developer (Spring Boot, REST, SOAP, OData, Microservices, Banking/Finance domain)

---

## 1. What is SOAP? (30-second answer)

**SOAP (Simple Object Access Protocol)** is an XML-based messaging protocol for exchanging structured information between applications over a network (usually HTTP, but also SMTP, JMS, TCP).

- It is a **protocol** with strict rules (REST is an architectural style).
- Messages are always **XML**, wrapped in a standard **envelope**.
- The service contract is described by a **WSDL** (Web Services Description Language) file.
- Strongly typed, contract-first, with built-in standards for security (WS-Security), transactions (WS-AtomicTransaction), reliability (WS-ReliableMessaging).
- Still widely used in **banking, insurance, payments, telecom, and legacy enterprise (SAP)** integrations.

**Say in interview:** *"SOAP is a strict, contract-driven, XML-based protocol. I'd choose it when I need formal contracts, WS-* standards like WS-Security, or when integrating with legacy/enterprise systems, which is common in financial services."*

---

## 2. SOAP Message Structure

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <!-- Optional: auth tokens, WS-Security, correlation IDs, transaction info -->
  </soap:Header>
  <soap:Body>
    <!-- Mandatory: request/response payload OR Fault -->
    <ns:GetAccountBalanceRequest xmlns:ns="http://bank.com/accounts">
      <ns:accountId>123456</ns:accountId>
    </ns:GetAccountBalanceRequest>
  </soap:Body>
</soap:Envelope>
```

| Element | Required? | Purpose |
|---|---|---|
| **Envelope** | Yes | Root element; defines the XML as a SOAP message |
| **Header** | No | Metadata: security, routing, transaction, session |
| **Body** | Yes | Actual payload or a Fault |
| **Fault** | No (only in Body on error) | Error reporting |

### SOAP Fault structure
**SOAP 1.1:** `faultcode`, `faultstring`, `faultactor`, `detail`
**SOAP 1.2:** `Code`, `Reason`, `Node`, `Role`, `Detail`

```xml
<soap:Fault>
  <faultcode>soap:Server</faultcode>
  <faultstring>Account not found</faultstring>
  <detail>
    <ns:AccountNotFound>
      <ns:accountId>123456</ns:accountId>
    </ns:AccountNotFound>
  </detail>
</soap:Fault>
```

- `Client` fault = caller's mistake (bad input). `Server` fault = service-side problem.
- SOAP **always** returns HTTP 200 for application faults in 1.1 historically; **500** is typically used for Fault responses (SOAP 1.1 spec mandates 500).

---

## 3. WSDL – The Contract

WSDL is an XML document describing **what** the service does, **how** to call it, and **where** it lives.

```
<definitions>
  <types>        → XML Schema (XSD) data types for messages
  <message>      → Input/output message definitions
  <portType>     → Abstract operations (like a Java interface)
  <binding>      → Protocol & message format (SOAP/HTTP, document/rpc, literal/encoded)
  <service>      → Concrete endpoint URL(s) (port + address)
</definitions>
```

**Memory trick:** **T-M-P-B-S** → *Types, Messages, PortType, Binding, Service*
(What data → What messages → What operations → How to talk → Where to call)

*WSDL 2.0* renamed: `portType` → `interface`, `port` → `endpoint`, removed `message` element. Most real world is still WSDL 1.1.

---

## 4. Binding Styles (Frequent Interview Question)

| Style / Use | Description |
|---|---|
| **Document/Literal** | Body contains an XML doc validated against XSD. Simple, interoperable |
| **Document/Literal Wrapped** | Document/literal + wrapper element named after the operation. **Most common & WS-I compliant** |
| **RPC/Literal** | Body contains operation name + parameters as elements |
| **RPC/Encoded** | Uses SOAP encoding rules. **Deprecated / not WS-I compliant** |

**Recommended answer:** *"Document/Literal wrapped is the standard best practice because it's WS-I Basic Profile compliant, schema-validatable, and interoperable."*

---

## 5. SOAP 1.1 vs SOAP 1.2

| Aspect | 1.1 | 1.2 |
|---|---|---|
| Namespace | `schemas.xmlsoap.org/soap/envelope/` | `www.w3.org/2003/05/soap-envelope` |
| HTTP Content-Type | `text/xml` + `SOAPAction` header | `application/soap+xml` (action in content-type param) |
| Fault format | faultcode/faultstring | Code/Reason (more structured) |
| Standard | W3C Note | W3C Recommendation |
| Binding | HTTP only (formally) | Transport-independent, better defined |

---

## 6. SOAP vs REST (Must Know!)

| Feature | SOAP | REST |
|---|---|---|
| Type | Protocol | Architectural style |
| Format | XML only | JSON, XML, text, etc. |
| Contract | WSDL (strict, formal) | OpenAPI/Swagger (optional) |
| Transport | HTTP, SMTP, JMS, TCP | HTTP mostly |
| HTTP methods | Mostly POST | GET, POST, PUT, DELETE, PATCH |
| State | Can be stateful (WS-*) | Stateless |
| Caching | Not cacheable (POST) | Cacheable (GET) |
| Security | WS-Security (message-level), plus TLS | TLS, OAuth2, JWT (transport-level) |
| Performance | Heavier (verbose XML) | Lightweight |
| Error handling | SOAP Fault | HTTP status codes |
| Transactions | WS-AtomicTransaction | None built-in |
| Reliability | WS-ReliableMessaging | Client retry logic |
| Tooling | Auto-generated stubs (wsimport) | Hand-written / OpenAPI-generated |
| Best for | Enterprise, banking, legacy, formal contracts | Web/mobile APIs, microservices |

**When to choose SOAP:**
- Formal contract and strong typing needed
- Message-level security (WS-Security) / non-repudiation / digital signatures
- Distributed transactions or reliable messaging
- Integrating with legacy/ERP/core banking systems
- Transport other than HTTP (JMS)

---

## 7. WS-* Standards (Know the names)

| Standard | Purpose |
|---|---|
| **WS-Security** | Message-level authentication, integrity (XML Signature), confidentiality (XML Encryption), tokens (UsernameToken, SAML, X.509) |
| **WS-Addressing** | Routing info in header (To, ReplyTo, MessageID) for async |
| **WS-ReliableMessaging** | Guaranteed delivery, ordering |
| **WS-AtomicTransaction** | Distributed transactions (2PC) |
| **WS-Policy** | Express requirements/capabilities of a service |
| **WS-I Basic Profile** | Interoperability guidelines |
| **MTOM/XOP** | Efficient binary attachments (alternative to base64) |

---

## 8. SOAP in Java

### 8.1 Java APIs / Frameworks

| Technology | Notes |
|---|---|
| **JAX-WS** (Jakarta XML Web Services) | Standard API for SOAP; annotation based |
| **JAXB** (Jakarta XML Binding) | Maps Java ⇄ XML (marshalling/unmarshalling) |
| **SAAJ** | Low-level API to build/read SOAP messages & attachments |
| **Apache CXF** | Popular full-featured JAX-WS implementation; Spring integration |
| **Apache Axis2** | Older implementation |
| **Metro (GlassFish)** | Reference implementation |
| **Spring Web Services (Spring-WS)** | Contract-first SOAP framework from Spring |

> Note: Since Java 11, JAX-WS/JAXB were **removed from the JDK** and must be added as dependencies. Also package moved `javax.xml.ws` → `jakarta.xml.ws` (Jakarta EE 9+; Spring Boot 3).

### 8.2 Code-First vs Contract-First

| | Code-First | Contract-First |
|---|---|---|
| Approach | Write Java → generate WSDL | Write WSDL/XSD → generate Java |
| Pros | Fast, easy for internal services | Stable contract, language-neutral, better versioning |
| Cons | Contract tied to implementation, can break unintentionally | More upfront effort |
| Recommended for | Prototypes / internal | **Production / enterprise / external** |

Spring-WS **strongly advocates contract-first**.

### 8.3 JAX-WS Example (Code-First)

```java
@WebService(serviceName = "AccountService")
public class AccountServiceImpl {

    @WebMethod
    public BigDecimal getBalance(@WebParam(name = "accountId") String accountId) {
        return accountRepository.findBalance(accountId);
    }
}
```

Key annotations: `@WebService`, `@WebMethod`, `@WebParam`, `@WebResult`, `@SOAPBinding(style=DOCUMENT, use=LITERAL)`, `@HandlerChain`.

### 8.4 Spring Boot + Spring-WS Example (Contract-First)

**Dependency**
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web-services</artifactId>
</dependency>
```

**Endpoint**
```java
@Endpoint
public class AccountEndpoint {

    private static final String NAMESPACE = "http://bank.com/accounts";

    private final AccountService accountService;

    public AccountEndpoint(AccountService accountService) {
        this.accountService = accountService;
    }

    @PayloadRoot(namespace = NAMESPACE, localPart = "GetAccountBalanceRequest")
    @ResponsePayload
    public GetAccountBalanceResponse getBalance(@RequestPayload GetAccountBalanceRequest request) {
        GetAccountBalanceResponse response = new GetAccountBalanceResponse();
        response.setBalance(accountService.getBalance(request.getAccountId()));
        return response;
    }
}
```

**Configuration**
```java
@EnableWs
@Configuration
public class WebServiceConfig extends WsConfigurerAdapter {

    @Bean
    public ServletRegistrationBean<MessageDispatcherServlet> messageDispatcherServlet(ApplicationContext ctx) {
        MessageDispatcherServlet servlet = new MessageDispatcherServlet();
        servlet.setApplicationContext(ctx);
        servlet.setTransformWsdlLocations(true);
        return new ServletRegistrationBean<>(servlet, "/ws/*");
    }

    @Bean(name = "accounts")
    public DefaultWsdl11Definition defaultWsdl11Definition(XsdSchema accountsSchema) {
        DefaultWsdl11Definition wsdl = new DefaultWsdl11Definition();
        wsdl.setPortTypeName("AccountsPort");
        wsdl.setLocationUri("/ws");
        wsdl.setTargetNamespace("http://bank.com/accounts");
        wsdl.setSchema(accountsSchema);
        return wsdl;
    }

    @Bean
    public XsdSchema accountsSchema() {
        return new SimpleXsdSchema(new ClassPathResource("accounts.xsd"));
    }
}
```

**Flow:** `MessageDispatcherServlet` → `EndpointMapping` (by `@PayloadRoot`) → `@Endpoint` method → JAXB marshals response. WSDL exposed at `/ws/accounts.wsdl`.

**Generating classes from XSD:** `jaxb2-maven-plugin` or `maven-jaxb2-plugin`; from WSDL: `wsimport` / `cxf-codegen-plugin`.

### 8.5 Consuming a SOAP Service

```java
@Configuration
public class SoapClientConfig {
    @Bean
    public Jaxb2Marshaller marshaller() {
        Jaxb2Marshaller m = new Jaxb2Marshaller();
        m.setContextPath("com.bank.generated");
        return m;
    }

    @Bean
    public WebServiceTemplate webServiceTemplate(Jaxb2Marshaller marshaller) {
        WebServiceTemplate t = new WebServiceTemplate();
        t.setMarshaller(marshaller);
        t.setUnmarshaller(marshaller);
        t.setDefaultUri("http://host/ws");
        return t;
    }
}

// usage
GetAccountBalanceResponse resp =
    (GetAccountBalanceResponse) webServiceTemplate.marshalSendAndReceive(request);
```

Alternatives: generated JAX-WS client stubs via `wsimport`, or Apache CXF client (`JaxWsProxyFactoryBean`).

### 8.6 Error Handling in Spring-WS

```java
@SoapFault(faultCode = FaultCode.CLIENT)
public class AccountNotFoundException extends RuntimeException { ... }
```
or use `SoapFaultMappingExceptionResolver` / `@SoapFault` annotation with `SoapFaultAnnotationExceptionResolver`.

### 8.7 Interceptors / Handlers

- **Spring-WS:** `EndpointInterceptor` (logging, validation, security) / `ClientInterceptor`.
- **JAX-WS:** `SOAPHandler` / `LogicalHandler` in a handler chain.
- **Validation:** `PayloadValidatingInterceptor` validates against XSD.
- **Security:** `Wss4jSecurityInterceptor` (WSS4J) for UsernameToken, signing, encryption.

---

## 9. Security in SOAP

1. **Transport-level:** HTTPS / TLS (+ mutual TLS) – protects whole channel point-to-point.
2. **Message-level (WS-Security):** protects the message itself, works end-to-end across intermediaries.
    - **UsernameToken** (plain / digest)
    - **XML Signature** → integrity & non-repudiation
    - **XML Encryption** → confidentiality of selected parts
    - **SAML / X.509 / Kerberos tokens**
3. **Common attacks:** XXE (XML External Entity), XML bombs (Billion Laughs), SOAPAction spoofing, replay attacks, WSDL information disclosure.
    - Mitigation: disable DTDs/external entities, limit payload size, schema validation, timestamps/nonce, don't expose WSDL publicly in prod if not needed.

**Interview line:** *"TLS secures the pipe; WS-Security secures the message. In banking with multiple intermediaries, message-level security is often required."*

---

## 10. Attachments & Large Data

- **SwA** (SOAP with Attachments) – MIME multipart.
- **MTOM** (Message Transmission Optimization Mechanism) – XOP optimizes binary; avoids ~33% base64 overhead. Enable in JAX-WS via `@MTOM`; in Spring-WS via `Jaxb2Marshaller.setMtomEnabled(true)`.

---

## 11. Synchronous vs Asynchronous SOAP

- **Sync request-response:** default over HTTP.
- **One-way operations:** `@Oneway` – no response.
- **Async:** client-side async (callback/Future in JAX-WS), WS-Addressing with `ReplyTo`, or SOAP over JMS.

---

## 12. Versioning & Backward Compatibility

- Use **namespace versioning** (`http://bank.com/accounts/v2`) or **endpoint URL versioning** (`/ws/v2`).
- Prefer **additive, optional** changes (new optional elements) – non-breaking.
- Never remove/rename elements or change types in the same version.
- Keep old versions alive during deprecation window.
- Contract-first makes this manageable.

---

## 13. Testing SOAP Services

- **SoapUI / ReadyAPI**, **Postman** (supports raw SOAP XML), **curl**.
- **Spring-WS test:** `MockWebServiceClient` (server-side), `MockWebServiceServer` (client-side).
- Unit test endpoint logic by mocking service layer; integration tests with `@SpringBootTest`.
- **Contract testing:** validate against WSDL/XSD.

```bash
curl -X POST http://localhost:8080/ws \
  -H "Content-Type: text/xml; charset=utf-8" \
  -H "SOAPAction: \"\"" \
  -d @request.xml
```

---

## 14. Pros & Cons

**Pros**
- Strict, formal, language-neutral contract (WSDL) → auto-generated clients
- Built-in standards for security, transactions, reliability
- Works over multiple transports
- Strong typing and validation via XSD
- Mature tooling

**Cons**
- Verbose XML → bigger payloads, slower parsing
- Complex / steep learning curve
- Less flexible, harder to evolve
- Not cache-friendly, poor browser/mobile fit
- Heavier tooling

---

## 15. SOAP in Microservices / Modern Context

- Typically **REST/gRPC/events** for new microservices; **SOAP for integrating legacy/core systems**.
- Common pattern: build an **anti-corruption layer / adapter service** that exposes REST to the rest of the system and talks SOAP to the legacy backend.
- Resilience still applies: timeouts, retries (only for idempotent ops), circuit breaker (Resilience4j), connection pooling.
- Observability: correlation IDs in SOAP headers, log request/response (mask sensitive data like PAN, account numbers).

---

## 16. Banking / Financial Context (Relevant to this JD)

- Core banking, payment gateways, SWIFT-adjacent integrations, and insurance systems often expose **SOAP APIs**.
- WS-Security, digital signatures, and audit trails are common requirements.
- SAP systems have historically exposed **SOAP-based services** (e.g., SAP PI/PO, enterprise services), while newer SAP stack leans toward **OData/REST**.
- Emphasize: **idempotency, data integrity, auditability, and error handling**.

---

## 17. Top Interview Questions & Crisp Answers

**Q1. What is SOAP and how is it different from REST?**
SOAP is an XML-based protocol with a strict envelope and WSDL contract, supporting WS-* standards. REST is an architectural style using HTTP semantics and typically JSON, stateless and lightweight. SOAP suits enterprise/formal-contract scenarios; REST suits web/mobile and microservices.

**Q2. What is WSDL? What are its parts?**
XML contract describing a SOAP service: types, messages, portType, binding, service.

**Q3. What is the difference between code-first and contract-first?**
Code-first generates WSDL from Java; contract-first starts with WSDL/XSD and generates Java. Contract-first is preferred for stable, interoperable, versionable APIs.

**Q4. Explain the SOAP message structure.**
Envelope (root) → optional Header (metadata/security) → mandatory Body (payload or Fault).

**Q5. How do you handle errors in SOAP?**
Via SOAP Fault (code, reason/string, detail). In Spring-WS, use `@SoapFault` or exception resolvers; map business exceptions to Client faults and technical ones to Server faults.

**Q6. How do you secure a SOAP service?**
TLS for transport; WS-Security (UsernameToken, signature, encryption, SAML) for message-level; plus XXE protection, schema validation, size limits.

**Q7. What is JAXB?**
Java API that marshals Java objects ⇄ XML based on XSD/annotations; used by JAX-WS and Spring-WS for binding payloads.

**Q8. JAX-WS vs Spring-WS vs CXF?**
JAX-WS is the standard spec (annotations). CXF is an implementation with rich WS-* support and Spring integration. Spring-WS is a contract-first, document-driven framework with payload-root/XPath-based routing.

**Q9. What is SOAPAction header?**
HTTP header in SOAP 1.1 hinting the intent/operation for routing; in SOAP 1.2 moved into the `action` parameter of Content-Type.

**Q10. What is MTOM?**
Optimized transmission of binary data in SOAP using XOP, avoiding base64 bloat.

**Q11. How do you version a SOAP API?**
Namespace/URL versioning, additive non-breaking changes, maintain old versions during deprecation.

**Q12. Is SOAP stateless?**
By default each call is independent (stateless over HTTP), but WS-* standards enable stateful/session-like behavior and transactions.

**Q13. How would you integrate a SOAP service in a Spring Boot microservice?**
Generate client classes from WSDL (`wsimport`/CXF plugin or JAXB plugin), configure `WebServiceTemplate` (or CXF client), add timeouts, interceptors (logging/security), resilience (circuit breaker/retry), and wrap in an adapter layer exposing a clean domain interface.

**Q14. How do you test SOAP services?**
SoapUI/Postman manually, Spring-WS `MockWebServiceClient/Server` for automated tests, XSD validation, and contract tests.

**Q15. Why is SOAP considered heavy?**
XML verbosity, namespace/envelope overhead, parsing cost, and WS-* processing.

**Q16. What is the difference between `javax.xml.ws` and `jakarta.xml.ws`?**
Same API; package renamed after Java EE moved to the Eclipse Foundation (Jakarta EE 9+). Spring Boot 3 requires `jakarta.*`.

**Q17. How does Spring-WS route requests to endpoints?**
`MessageDispatcherServlet` uses `EndpointMapping` implementations: `PayloadRootAnnotationMethodEndpointMapping` (by root element QName), `SoapActionAnnotationMethodEndpointMapping`, `XPathPayloadEndpointMapping`, etc.

---

## 18. Quick Revision Cheat Sheet

- **SOAP = Protocol + XML + Envelope + WSDL**
- Envelope → Header (optional) + Body (required) → Fault on error
- WSDL: **T-M-P-B-S** (Types, Messages, PortType, Binding, Service)
- Best style: **Document/Literal Wrapped**
- Java: **JAX-WS + JAXB**, **CXF**, **Spring-WS (contract-first)**
- Security: **TLS + WS-Security (message-level)**
- Large data: **MTOM**
- Use SOAP for: **enterprise, legacy, formal contracts, banking**
- Use REST for: **lightweight, web/mobile, microservices**
- Wrap legacy SOAP behind an **adapter / anti-corruption layer**

---

## 19. Be Ready for Follow-ups

1. "Have you worked with SOAP? Describe a project." → Prepare one real example: what system, contract-first or code-first, how you handled security, faults, testing, versioning.
2. "How would you migrate a SOAP service to REST?" → Strangler pattern; expose REST façade over the same service layer; run both in parallel; deprecate gradually; map faults → HTTP status codes; WSDL/XSD → OpenAPI/JSON Schema.
3. "How do you make SOAP calls resilient?" → Timeouts, connection pooling, retry with backoff for idempotent ops only, circuit breaker, bulkhead, fallback, monitoring.
4. "Performance tuning?" → Reuse JAXBContext (thread-safe, expensive), connection pooling, GZIP compression, MTOM, streaming (StAX) for large payloads, avoid logging full payloads.

---

*Next topics suggestion: REST APIs, Spring Boot internals, Hibernate/JPA, OData, Microservices, PostgreSQL, CI/CD (GitHub Actions), Design Patterns, Java 8+ features, etc.*