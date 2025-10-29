# API Gateway

## 🎯 Purpose

The API Gateway serves as the **single entry point** for all client requests to the microservices ecosystem. It handles routing, load balancing, and service discovery, allowing clients to interact with multiple microservices through a unified interface without needing to know individual service locations.

## 🔌 Port

- **Default Port:** `8080`
- **Gateway URL:** http://localhost:8080

## 🛠️ Tech Stack

- **Framework:** Spring Boot 3.5.7
- **Gateway:** Spring Cloud Gateway (WebFlux - Reactive)
- **Service Discovery:** Spring Cloud Netflix Eureka Client
- **Load Balancing:** Spring Cloud LoadBalancer (client-side)
- **Monitoring:** Spring Boot Actuator
- **Containerization:** Docker
- **Build Tool:** Maven

## 📦 Dependencies

**Required Services:**
1. **Service Registry** (Eureka) - Must be running on port 8761
2. **Config Server** (Optional) - For externalized configuration

**Downstream Services:**
- Inventory Service (port 8081)
- Order Service (port 8082) - Planned

## ⚙️ Configuration

### Key Configuration (`application.yml`):

```yaml
server:
  port: 8080

spring:
  application:
    name: ecommerce-api-gateway
  cloud:
    gateway:
      server:
        webflux:  # CRITICAL: Must use WebFlux, not MVC
          discovery:
            locator:
              enabled: true
              lower-case-service-id: true

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    fetch-registry: true
    register-with-eureka: true
```

### Important Notes:

- **WebFlux vs MVC:** This gateway uses the **reactive WebFlux** implementation, NOT the servlet-based MVC variant. The MVC variant has limited functionality and should not be used.
- **Automatic Service Discovery:** With `discovery.locator.enabled: true`, the gateway automatically creates routes for all services registered in Eureka
- **URL Pattern:** Services are accessible via `http://localhost:8080/{service-name}/{endpoint}`
- **Case Sensitivity:** `lower-case-service-id: true` ensures service names are lowercase in URLs
- **Load Balancing:** Automatically distributes requests across multiple instances of the same service (client-side load balancing)

---

## 🚀 How to Run

### Prerequisites

- Java 21 or higher
- Maven 3.8+
- Service Registry running on port 8761
- At least one business service (e.g., Inventory Service) registered in Eureka
- Docker (for containerized deployment)

---

## 🐳 Option 1: Running with Docker (Recommended)

### Quick Start

```bash
# Ensure Service Registry is running first
docker run -d \
  --name service-registry \
  --network ecommerce-network \
  -p 8761:8761 \
  ecommerce-service-registry:latest

# Run API Gateway
docker run -d \
  --name api-gateway \
  --network ecommerce-network \
  -p 8080:8080 \
  -e EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://service-registry:8761/eureka/ \
  ecommerce-api-gateway:latest
```

### Building the Docker Image

```bash
# Clone the repository
git clone https://github.com/DanLearnings/ecommerce-api-gateway.git
cd ecommerce-api-gateway

# Build the Docker image
docker build -t ecommerce-api-gateway:latest .

# Run the container
docker run -d \
  --name api-gateway \
  --network ecommerce-network \
  -p 8080:8080 \
  ecommerce-api-gateway:latest
```

### Docker Environment Variables

```bash
# Run with custom configurations
docker run -d \
  --name api-gateway \
  --network ecommerce-network \
  -p 8080:8080 \
  -e JAVA_OPTS="-Xmx1g -Xms512m" \
  -e EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://service-registry:8761/eureka/ \
  ecommerce-api-gateway:latest
```

### Viewing Logs

```bash
# View logs
docker logs api-gateway

# Follow logs in real-time
docker logs -f api-gateway
```

### Stopping and Removing

```bash
# Stop the container
docker stop api-gateway

# Remove the container
docker rm api-gateway

# Stop and remove in one command
docker rm -f api-gateway
```

---

## 💻 Option 2: Running with Maven (Development)

### Running Locally

```bash
# Clone the repository
git clone https://github.com/DanLearnings/ecommerce-api-gateway.git
cd ecommerce-api-gateway

# Ensure Service Registry is running first
# Then run API Gateway
mvn spring-boot:run

# Or build and run the JAR
mvn clean package
java -jar target/ecommerce-api-gateway-1.0.0.jar
```

---

## 🔍 Routing Patterns

### Automatic Routing via Service Discovery

The gateway automatically routes requests based on service names registered in Eureka:

**Pattern:** `http://localhost:8080/{service-name}/{service-endpoint}`

**Examples:**

```bash
# Route to Inventory Service
curl http://localhost:8080/inventory-service/products

# Route to Order Service (when implemented)
curl http://localhost:8080/order-service/orders

# The service name in the URL matches the service name in Eureka
# inventory-service -> INVENTORY-SERVICE in Eureka (case insensitive)
```

### Manual Route Configuration (Optional)

You can also define explicit routes in `application.yml`:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: inventory-service-route
          uri: lb://inventory-service
          predicates:
            - Path=/api/inventory/**
          filters:
            - StripPrefix=2
```

*Note: The current implementation uses automatic discovery, not manual routes*

---

## 🔍 Endpoints

### Health Check

```bash
curl http://localhost:8080/actuator/health
```

**Expected Response:**
```json
{
  "status": "UP"
}
```

### Gateway Information (When Enabled)

```bash
curl http://localhost:8080/actuator/gateway/routes
```

### Example: Access Inventory Service via Gateway

```bash
# List all products
curl http://localhost:8080/inventory-service/products

# Get specific product
curl http://localhost:8080/inventory-service/products/1

# Create product
curl -X POST http://localhost:8080/inventory-service/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop",
    "description": "High-performance laptop",
    "price": 1200.00,
    "quantity": 50,
    "sku": "LAP-001"
  }'
```

---

## ✅ Health Check

Verify the gateway is working correctly:

```bash
# 1. Check gateway health
curl http://localhost:8080/actuator/health

# 2. Verify Eureka registration
# Visit http://localhost:8761 and check for API-GATEWAY

# 3. Test routing to a service
curl http://localhost:8080/inventory-service/products
```

---

## 🔧 How It Works

### Service Discovery Flow

1. **Gateway starts** and registers with Eureka
2. **Gateway fetches** the service registry from Eureka
3. **Client sends request** to `http://localhost:8080/inventory-service/products`
4. **Gateway extracts** service name (`inventory-service`) from URL
5. **Gateway queries** Eureka for instances of `inventory-service`
6. **Gateway forwards** the request to one of the available instances (load balancing)
7. **Service processes** the request and returns response
8. **Gateway returns** the response to the client

### Load Balancing

If multiple instances of a service are registered, the gateway automatically **load balances** requests across them using a round-robin strategy (client-side load balancing via Spring Cloud LoadBalancer).

**Example:**
```bash
# If you have 3 instances of inventory-service running:
# - inventory-service-1 (localhost:8081)
# - inventory-service-2 (localhost:8082)
# - inventory-service-3 (localhost:8083)

# The Gateway will distribute requests:
# Request 1 → inventory-service-1
# Request 2 → inventory-service-2
# Request 3 → inventory-service-3
# Request 4 → inventory-service-1 (round-robin)
```

---

## 🐛 Troubleshooting

### Gateway returns 404 for service requests

**Symptom:** `curl http://localhost:8080/inventory-service/products` returns 404

**Solutions:**
1. **Verify service is registered** in Eureka (http://localhost:8761)
2. **Check service name** matches Eureka registration (case-insensitive)
3. **Wait 30 seconds** after service startup for registration
4. **Verify WebFlux configuration** - Ensure using `spring.cloud.gateway.server.webflux`, not just `spring.cloud.gateway`
5. **Check gateway logs** for routing errors

### Service works directly but not through gateway

**Symptom:** `http://localhost:8081/products` works but `http://localhost:8080/inventory-service/products` doesn't

**Solutions:**
1. **Confirm gateway is using WebFlux** - Check dependencies in `pom.xml` for `spring-cloud-starter-gateway-server-webflux`
2. **Verify discovery locator** - Ensure `discovery.locator.enabled: true`
3. **Check Eureka fetch** - Confirm `eureka.client.fetch-registry: true`
4. **Review logs** for service discovery errors

### Gateway not registering with Eureka

**Symptom:** API-GATEWAY doesn't appear in Eureka dashboard

**Solutions:**
1. **Ensure Eureka is running** before starting the gateway
2. **Verify `eureka.client.register-with-eureka: true`** in configuration
3. **Check `defaultZone` URL** points to correct Eureka instance
4. **Wait 30 seconds** for initial registration
5. **Review logs** for connection errors

### WebFlux vs MVC Issues

**Symptom:** Routes don't work, strange errors, limited functionality

**Solution:**
- **Use WebFlux:** Ensure you're using `spring-cloud-starter-gateway-server-webflux`
- **NOT MVC:** Do NOT use `spring-cloud-starter-gateway-server-webmvc`
- **Configuration:** Use `spring.cloud.gateway.server.webflux` path in YAML

The MVC variant has significant limitations with service discovery and is not recommended.

### Docker: Cannot connect to Service Registry

**Symptom:** Gateway logs show Eureka connection errors

**Solution:**
```bash
# Verify both containers are in the same network
docker network inspect ecommerce-network

# Ensure Service Registry is running and healthy
docker ps | grep service-registry

# Test connectivity
docker exec api-gateway ping service-registry

# Ensure environment variable uses container name
-e EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://service-registry:8761/eureka/
```

### Docker: Gateway cannot route to downstream services

**Symptom:** Gateway returns 503 Service Unavailable

**Solution:**
```bash
# Verify downstream services are registered in Eureka
curl http://localhost:8761/eureka/apps

# Ensure all services are in the same Docker network
docker network inspect ecommerce-network

# Wait 60 seconds after starting all services for full registration
```

---

## 🎯 Key Learnings

### Why WebFlux?

During development, we initially attempted to use the MVC (servlet-based) variant of Spring Cloud Gateway but encountered significant issues:

1. **Limited Service Discovery:** The MVC variant has incomplete integration with Eureka
2. **Configuration Differences:** Different YAML paths and properties
3. **Missing Features:** Some gateway features only work with WebFlux
4. **Official Recommendation:** Spring strongly recommends WebFlux for Cloud Gateway

**Lesson:** Always use `spring-cloud-starter-gateway-server-webflux` for production systems.

### Configuration Hierarchy Matters

The correct configuration path for WebFlux is:
```yaml
spring.cloud.gateway.server.webflux.discovery.locator
```

NOT:
```yaml
spring.cloud.gateway.discovery.locator  # This doesn't work with WebFlux
```

This was discovered through extensive debugging and consulting real-world tutorials alongside official documentation.

---

## 🐳 Docker Image Details

### Multi-stage Build

The Dockerfile uses a multi-stage build:
- **Stage 1 (build):** Uses Maven image to compile the application
- **Stage 2 (runtime):** Uses lightweight JRE image to run the application

### Important Docker Features

- **Non-root user:** Runs as `spring:spring` for security
- **Health check:** Built-in health check with 60s start period
- **Environment variables:** Customizable Eureka location
- **Reactive runtime:** Optimized for WebFlux non-blocking operations

### Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1
```

---

## 🔮 Future Enhancements

- [ ] **Rate Limiting:** Prevent abuse with request rate limits
- [ ] **Authentication:** Add JWT token validation at the gateway level
- [ ] **Circuit Breaker:** Handle downstream service failures gracefully
- [ ] **Request/Response Transformation:** Modify headers, bodies, etc.
- [ ] **Logging & Monitoring:** Comprehensive request logging
- [ ] **Custom Filters:** Add business-specific request/response filters

---

## 📚 Additional Resources

- [Spring Cloud Gateway Documentation](https://spring.io/projects/spring-cloud-gateway)
- [Gateway WebFlux Reference](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
- [Service Discovery with Eureka](https://spring.io/guides/gs/service-registration-and-discovery/)
- [Docker Documentation](https://docs.docker.com/)

## 🔗 Related Services

- [Service Registry](https://github.com/DanLearnings/ecommerce-service-registry) - Discovers and routes to services (required)
- [Inventory Service](https://github.com/DanLearnings/ecommerce-inventory-service) - Example downstream service
- [Order Service](https://github.com/DanLearnings/ecommerce-order-service) - Planned downstream service

---

## 👨‍💻 Maintainer

**Danrley Brasil (Dan Learnings)**
- Personal GitHub: [@DanrleyBrasil](https://github.com/DanrleyBrasil)
- Organization: [DanLearnings](https://github.com/DanLearnings)

---

**Last Updated:** October 29, 2025
