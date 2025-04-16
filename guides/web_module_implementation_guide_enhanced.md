---
status: draft
last updated: 2025-04-16
version: "0.2"
---

# Implementation Guide: Web Module

## Purpose and Responsibilities

A Web Module is a UI/Presentation layer component that serves as the entry point for user or system interaction with your application. It has the following responsibilities:

- Define and expose HTTP endpoints (routes) for a specific bounded context
- Orchestrate the flow of data between the client and the application services
- Transform domain data into appropriate response formats (HTML, XML, JSON)
- Handle HTTP-specific concerns (status codes, headers, content negotiation)
- Manage error handling and response formatting for API consumers

Web Modules handle two main types of content:
1. **UI Content** - HTML responses for browser consumption using ScalaTags
2. **API Content** - Structured data (XML, JSON) for system integration

## Architecture Decision: Tapir-based Web Modules

For consistency and maintainability, we use Tapir to define all web modules, regardless of whether they serve HTML UI or API data. This gives us:

1. **Unified Approach** - One consistent pattern for all endpoints
2. **Type Safety** - Type-safe endpoint definitions for all routes
3. **Self-documenting API** - Automatic documentation
4. **Improved Testing** - Consistent testing patterns
5. **Clear Separation** - Explicit distinction between endpoint definition and implementation

## Structure and Patterns

### Common Module Structure

All Web Modules should follow this basic structure:

```
web/
├── [FeatureName]Module.scala       # Main module class with endpoint definitions 
├── [FeatureName]Service.scala      # Business logic (optional)
├── [FeatureName]ViewModel.scala    # View models and DTOs (optional)
└── views/                          # HTML or XML renderers (optional)
    └── [FeatureName]View.scala
```

For simpler modules, all components can be combined in a single file with nested objects.

### Tapir Module Components

A typical Tapir Web Module consists of:

1. **Endpoint Definitions** - Declarative API specification (input/output types, paths, methods)
2. **Business Logic Methods** - Implementation of the actual operations
3. **Server Endpoint Connections** - Linking endpoints to implementation logic
4. **Module Definition** - Exposing endpoints and server endpoints

## Implementation Steps

### 1. Define Module Class and Dependencies

```scala
class CustomerModule extends TapirWebModule[CustomerService]:
  // Module contents with dependency on CustomerService

// Or with multiple dependencies
class OrderModule extends TapirWebModule[OrderService & CustomerService & EmailService]:
  // Module contents with multiple service dependencies
```

### 2. Define Base Endpoints 

Define all endpoints using Tapir's declarative endpoint builders:

```scala
// For JSON API responses
private val apiBaseEndpoint = endpoint
  .in("api" / "customers")
  .errorOut(jsonBody[ErrorResponse])

// For HTML UI responses
private val htmlBaseEndpoint = endpoint
  .in("customers")
  .errorOut(stringBody.mapTo[String]) // Error messages for UI
  .out(stringBody.mapTo[String])      // HTML output
```

### 3. Define Specific Endpoints

```scala
// API endpoint
val getCustomerApiEndpoint = apiBaseEndpoint
  .name("Get Customer API")
  .description("Retrieve customer data in JSON format")
  .get
  .in(path[Long]("id"))
  .out(jsonBody[CustomerDTO])

// HTML UI endpoint
val getCustomerPageEndpoint = htmlBaseEndpoint
  .name("Get Customer Page")
  .description("Render customer profile page")
  .get
  .in(path[Long]("id"))
```

### 4. Implement Business Logic

```scala
// For API endpoint
private def getCustomerJson(id: Long): ZIO[CustomerService, ErrorResponse, CustomerDTO] =
  CustomerService.getCustomerById(id)
    .mapError(e => ErrorResponse("NOT_FOUND", e.getMessage))
    .map(customer => CustomerDTO.from(customer))

// For HTML UI endpoint
private def getCustomerPage(id: Long): ZIO[CustomerService, String, String] =
  CustomerService.getCustomerById(id)
    .mapError(e => s"Customer not found: ${e.getMessage}")
    .flatMap { customer =>
      val viewModel = CustomerViewModel.from(customer)
      val html = appShell.wrap(
        s"Customer: ${customer.name}", 
        customerView.render(viewModel)
      )
      ZIO.succeed(html)
    }
```

### 5. Connect Endpoints to Logic

```scala
// Connect API endpoint to implementation
val getCustomerApiRoute = getCustomerApiEndpoint
  .zServerLogic(id => getCustomerJson(id))

// Connect UI endpoint to implementation
val getCustomerPageRoute = getCustomerPageEndpoint
  .zServerLogic(id => getCustomerPage(id))
```

### 6. Expose Module Components

```scala
// Expose all endpoints for documentation
override def endpoints: List[Endpoint[?, ?, ?, ?, ?]] = List(
  getCustomerApiEndpoint,
  getCustomerPageEndpoint,
  // Other endpoints...
)

// Expose server endpoints for route generation
override def serverEndpoints: List[ZServerEndpoint[CustomerService, Any]] = List(
  getCustomerApiRoute.widen,
  getCustomerPageRoute.widen,
  // Other server endpoints...
)
```

## Advanced Tapir Concepts

### Input/Output Handling

Tapir allows composing multiple inputs and outputs in a type-safe manner:

```scala
// Combining multiple inputs with different sources
val createUserEndpoint = apiBaseEndpoint
  .post
  .in("users")
  .in(jsonBody[UserCreateRequest])                // Request body
  .in(header[String]("X-Correlation-ID"))         // Header
  .in(cookie[Option[String]]("session").optional) // Optional cookie
  .out(jsonBody[UserResponse])                    // Success response
  .out(header[String]("X-Request-ID"))            // Response header

// Implementing logic with multiple inputs
private def createUser(
  req: UserCreateRequest, 
  correlationId: String,
  sessionId: Option[String]
): ZIO[UserService, ErrorResponse, (UserResponse, String)] =
  for
    user <- UserService.createUser(req, correlationId)
    requestId = UUID.randomUUID().toString
  yield (UserResponse.from(user), requestId)

// Connect with server logic that handles multiple inputs and outputs
val createUserRoute = createUserEndpoint
  .zServerLogic { case (req, correlationId, sessionId) => 
    createUser(req, correlationId, sessionId)
  }
```

### Error Handling with OneOf

For more sophisticated error handling, use `oneOf` to define multiple error outputs:

```scala
// Define error variants
sealed trait ApiError
case class NotFoundError(id: String) extends ApiError
case class ValidationError(errors: List[String]) extends ApiError
case class ServerError(message: String) extends ApiError

// Map to status codes and bodies
val errorMapping = oneOf[ApiError](
  oneOfVariant(StatusCode.NotFound, jsonBody[NotFoundError]),
  oneOfVariant(StatusCode.BadRequest, jsonBody[ValidationError]),
  oneOfVariant(StatusCode.InternalServerError, jsonBody[ServerError])
)

// Use in endpoint definition
val userEndpoint = endpoint
  .get
  .in("users" / path[String]("id"))
  .errorOut(errorMapping)
  .out(jsonBody[UserResponse])

// Implementation with specific errors
private def getUser(id: String): ZIO[UserService, ApiError, UserResponse] =
  UserService.findById(id).mapError {
    case UserService.UserNotFound => NotFoundError(id)
    case UserService.ValidationFailed(errors) => ValidationError(errors)
    case e: Throwable => ServerError(e.getMessage)
  }
```

### Authentication and Security

Tapir provides built-in support for common security schemes:

```scala
import sttp.tapir.server.ziohttp.ZioHttpInterpreter
import sttp.tapir.server.ziohttp.ZioHttpServerOptions
import sttp.model.StatusCode
import sttp.tapir.ztapir._
import sttp.tapir.server.interceptor.auth.AuthInterceptor

// Define security scheme
case class User(id: String, roles: List[String])

val securityLogic: String => ZIO[UserService, Unauthorized, User] = token =>
  UserService.validateToken(token)
    .mapError(_ => Unauthorized())

// Use the security logic in an auth interceptor
val authInterceptor = AuthInterceptor(securityLogic, _ => ZIO.succeed(StatusCode.Unauthorized))

// Set up server options with the auth interceptor
val serverOptions = ZioHttpServerOptions
  .customiseInterceptors
  .prependInterceptor(authInterceptor)
  .options

// Define a secured endpoint
val securedEndpoint = endpoint
  .securityIn(auth.bearer[String]())
  .errorOut(statusCode(StatusCode.Unauthorized))

// Base endpoint for secured APIs
val securedApiEndpoint = securedEndpoint
  .in("api" / "secured")
  .errorOut(jsonBody[ErrorResponse])

// A specific secured endpoint
val getSecuredResource = securedApiEndpoint
  .get
  .in("resources" / path[String]("id"))
  .out(jsonBody[Resource])

// Server logic that uses the authenticated user
val getSecuredResourceLogic: (String, String) => ZIO[ResourceService & UserService, ErrorResponse, Resource] = 
  (token, resourceId) =>
    for
      user <- ZIO.serviceWithZIO[UserService](_.validateToken(token))
              .mapError(_ => ErrorResponse("UNAUTHORIZED", "Invalid token"))
      resource <- ResourceService.getResourceForUser(resourceId, user)
                  .mapError(e => ErrorResponse("NOT_FOUND", e.getMessage))
    yield resource

// Connect endpoint to server logic
val getSecuredResourceRoute = getSecuredResource
  .zServerLogic { case (token, resourceId) => 
    getSecuredResourceLogic(token, resourceId)
  }
```

### Content Negotiation

Tapir supports multiple response formats for the same endpoint:

```scala
// Define endpoint that can return both JSON and XML
val getProductEndpoint = endpoint
  .get
  .in("products" / path[String]("id"))
  .errorOut(oneOf(
    oneOfVariant(StatusCode.NotFound, jsonBody[ErrorResponse]),
    oneOfVariant(StatusCode.NotFound, xmlBody[ErrorResponse])
  ))
  .out(
    oneOf(
      oneOfVariant(jsonBody[ProductDTO]),
      oneOfVariant(xmlBody[ProductDTO])
    )
  )

// Implement logic to handle content negotiation
private def getProductWithFormat(id: String): ZIO[ProductService, ErrorResponse, ProductDTO] =
  ProductService.getProductById(id)
    .mapError(e => ErrorResponse("NOT_FOUND", e.getMessage))
    .map(product => ProductDTO.from(product))
    
// The content type of the response will be determined by the client's Accept header
```

## Examples

### XML API Module Example

```scala
class CSNOLApiModule extends TapirWebModule[NormaService & CiselnikService]:
  // Error response definition
  case class ErrorResponse(code: String, message: String) derives JsonCodec, Schema

  // API Endpoints
  private val baseEndpoint = endpoint
    .out(stringBodyUtf8AnyFormat(Codec.string.format(CodecFormat.Xml())))

  // Get Norma endpoint
  val getNormaEndpoint = baseEndpoint
    .name("Get Norma")
    .description("Retrieve a standard by ID")
    .get
    .in("csnol" / "normy" / path[Int]("id"))
    .errorOut(
      oneOf[ErrorResponse](
        oneOfVariant(StatusCode.NotFound, jsonBody[ErrorResponse]),
        oneOfVariant(StatusCode.InternalServerError, jsonBody[ErrorResponse])
      )
    )

  // Implement business logic
  private def getNorma(id: Int): ZIO[NormaService, ErrorResponse, String] =
    NormaService.getNormaById(id)
      .mapError(e => ErrorResponse("SERVER_ERROR", e.getMessage))
      .flatMap {
        case Some(norma) => 
          val xml = <export>
            <norma>
              {NormaXmlMapper.toXml(norma)}
            </norma>
          </export>
          ZIO.succeed(xml.toString)
        case None => 
          ZIO.fail(ErrorResponse("NOT_FOUND", s"Norma $id not found"))
      }

  // Connect endpoint to logic
  val getNormaRoute = getNormaEndpoint
    .zServerLogic(id => getNorma(id))

  // Expose endpoints
  override def endpoints = List(getNormaEndpoint)
  override def serverEndpoints = List(getNormaRoute.widen)
```

### HTML UI Module Example with Tapir

```scala
class CustomerUIModule(
  appShell: AppShell,
  customerListView: CustomerListView,
  customerFormView: CustomerFormView
) extends TapirWebModule[CustomerService]:

  // View model
  case class CustomerViewModel(id: Long, name: String, email: String)
  object CustomerViewModel:
    def from(customer: Customer): CustomerViewModel =
      CustomerViewModel(customer.id, customer.name, customer.email)
  
  // HTML endpoint definitions
  private val baseEndpoint = endpoint
    .errorOut(stringBody.mapTo[String])  // Use simple string for UI errors
    .out(stringBody.mapTo[String])       // HTML output
    .out(header[String]("Content-Type").description("text/html; charset=UTF-8"))

  // List customers endpoint
  val listCustomersEndpoint = baseEndpoint
    .name("Customer List Page")
    .description("Render page with all customers")
    .get
    .in("customers")

  // View single customer endpoint
  val viewCustomerEndpoint = baseEndpoint
    .name("Customer Details Page")
    .description("Render page with customer details")
    .get
    .in("customers" / path[Long]("id"))
  
  // Business logic methods
  private def getCustomerListPage: ZIO[CustomerService, String, (String, String)] =
    CustomerService.getAllCustomers
      .mapError(e => s"Failed to load customers: ${e.getMessage}")
      .map { customers =>
        val viewModels = customers.map(CustomerViewModel.from)
        val html = appShell.wrap("Customer List", customerListView.render(viewModels))
        (html, "text/html; charset=UTF-8")
      }
      
  private def getCustomerDetailPage(id: Long): ZIO[CustomerService, String, (String, String)] =
    CustomerService.getCustomerById(id)
      .mapError(e => s"Customer not found: ${e.getMessage}")
      .map { customer =>
        val viewModel = CustomerViewModel.from(customer)
        val html = appShell.wrap(s"Customer: ${customer.name}", customerFormView.render(viewModel))
        (html, "text/html; charset=UTF-8")
      }
  
  // Connect endpoints to logic
  val listCustomersRoute = listCustomersEndpoint
    .zServerLogic(_ => getCustomerListPage)
    
  val viewCustomerRoute = viewCustomerEndpoint
    .zServerLogic(id => getCustomerDetailPage(id))
  
  // Expose endpoints
  override def endpoints = List(
    listCustomersEndpoint,
    viewCustomerEndpoint
  )
  
  override def serverEndpoints = List(
    listCustomersRoute.widen,
    viewCustomerRoute.widen
  )
```

### Combined UI and API Module Example

```scala
class ProductModule(
  appShell: AppShell,
  productListView: ProductListView,
  productDetailView: ProductDetailView
) extends TapirWebModule[ProductService]:

  // Error and data models
  case class ErrorResponse(code: String, message: String) derives JsonCodec, Schema
  case class ProductDTO(id: Long, name: String, price: BigDecimal) derives JsonCodec, Schema
  
  // Base endpoints for different output types
  private val htmlEndpoint = endpoint
    .errorOut(stringBody.mapTo[String])
    .out(stringBody.mapTo[String])
    .out(header[String]("Content-Type").description("text/html; charset=UTF-8"))
    
  private val apiEndpoint = endpoint
    .in("api")
    .errorOut(jsonBody[ErrorResponse])
  
  // UI endpoints
  val productListPageEndpoint = htmlEndpoint
    .name("Product List Page")
    .get
    .in("products")
    
  val productDetailPageEndpoint = htmlEndpoint
    .name("Product Detail Page") 
    .get
    .in("products" / path[Long]("id"))
  
  // API endpoints
  val getAllProductsApiEndpoint = apiEndpoint
    .name("Get All Products API")
    .get
    .in("products")
    .out(jsonBody[List[ProductDTO]])
    
  val getProductApiEndpoint = apiEndpoint
    .name("Get Product API")
    .get
    .in("products" / path[Long]("id"))
    .out(jsonBody[ProductDTO])
  
  // Business logic for UI
  private def getProductListPage: ZIO[ProductService, String, (String, String)] =
    ProductService.getAllProducts
      .mapError(e => s"Failed to load products: ${e.getMessage}")
      .map { products => 
        val html = appShell.wrap("Products", productListView.render(products))
        (html, "text/html; charset=UTF-8")
      }
      
  private def getProductDetailPage(id: Long): ZIO[ProductService, String, (String, String)] =
    ProductService.getProductById(id)
      .mapError(e => s"Product not found: ${e.getMessage}")
      .map { product => 
        val html = appShell.wrap(product.name, productDetailView.render(product))
        (html, "text/html; charset=UTF-8") 
      }
  
  // Business logic for API
  private def getAllProducts: ZIO[ProductService, ErrorResponse, List[ProductDTO]] =
    ProductService.getAllProducts
      .mapError(e => ErrorResponse("SERVER_ERROR", e.getMessage))
      .map(_.map(p => ProductDTO(p.id, p.name, p.price)))
      
  private def getProduct(id: Long): ZIO[ProductService, ErrorResponse, ProductDTO] =
    ProductService.getProductById(id)
      .mapError {
        case _: ProductNotFoundError => ErrorResponse("NOT_FOUND", s"Product $id not found")
        case e => ErrorResponse("SERVER_ERROR", e.getMessage)
      }
      .map(p => ProductDTO(p.id, p.name, p.price))
  
  // Server routes
  val productListPageRoute = productListPageEndpoint
    .zServerLogic(_ => getProductListPage)
    
  val productDetailPageRoute = productDetailPageEndpoint
    .zServerLogic(id => getProductDetailPage(id))
    
  val getAllProductsApiRoute = getAllProductsApiEndpoint
    .zServerLogic(_ => getAllProducts)
    
  val getProductApiRoute = getProductApiEndpoint
    .zServerLogic(id => getProduct(id))
    
  // Module definition
  override def endpoints = List(
    productListPageEndpoint,
    productDetailPageEndpoint,
    getAllProductsApiEndpoint,
    getProductApiEndpoint
  )
  
  override def serverEndpoints = List(
    productListPageRoute.widen,
    productDetailPageRoute.widen,
    getAllProductsApiRoute.widen,
    getProductApiRoute.widen
  )
```

## Integration Points

Web Modules interact with:

1. **Application Services** - For orchestrating use cases
2. **Domain Services** - For direct domain operations when appropriate
3. **Other Web Modules** - For composing larger applications
4. **Frontend/Client Applications** - When returning data for client consumption

## Testing Approach

### Tapir Module Testing

Tapir provides excellent testing capabilities through its stub interpreter:

```scala
import sttp.tapir.server.stub.TapirStubInterpreter
import sttp.client4.testing.SttpBackendStub
import sttp.client4.Response
import sttp.client4.basicRequest
import sttp.model.StatusCode
import zio.test._
import zio.test.Assertion._

object ProductModuleSpec extends ZIOSpecDefault:
  val productService = TestProductService()
  val productModule = new ProductModule(productService)
  
  override def spec = suite("ProductModule")(
    test("GET /products/1 returns product HTML when it exists") {
      // Create a test client using the stub interpreter
      val backend = TapirStubInterpreter(SttpBackendStub.synchronous)
        .whenServerEndpointRunLogic(productModule.productDetailPageRoute)
        .backend()

      // Execute request
      val response = basicRequest
        .get(uri"http://test.example/products/1")
        .send(backend)

      // Assertions
      assert(response.code)(equalTo(StatusCode.Ok)) &&
      assert(response.body.toOption.get)(containsString("<h1>Product Name</h1>"))
    },
    
    test("GET /api/products/1 returns product JSON when it exists") {
      // Create a test client using the stub interpreter
      val backend = TapirStubInterpreter(SttpBackendStub.synchronous)
        .whenServerEndpointRunLogic(productModule.getProductApiRoute)
        .backend()

      // Execute request
      val response = basicRequest
        .get(uri"http://test.example/api/products/1")
        .send(backend)

      // Assertions
      assert(response.code)(equalTo(StatusCode.Ok)) &&
      assert(response.body.toOption.get)(containsString("\"name\":\"Product Name\""))
    },
    
    test("GET /api/products/999 returns 404 Not Found for non-existent product") {
      // Create a test client using the stub interpreter
      val backend = TapirStubInterpreter(SttpBackendStub.synchronous)
        .whenServerEndpointRunLogic(productModule.getProductApiRoute)
        .backend()

      // Execute request
      val response = basicRequest
        .get(uri"http://test.example/api/products/999")
        .send(backend)

      // Assertions
      assert(response.code)(equalTo(StatusCode.NotFound)) &&
      assert(response.body.toOption.get)(containsString("NOT_FOUND"))
    }
  )
```

### Testing Error Scenarios

```scala
test("Returns validation error when required fields are missing") {
  // Prepare invalid request payload
  val invalidJson = """{"name":""}"""
  
  // Create a test client using the stub interpreter
  val backend = TapirStubInterpreter(SttpBackendStub.synchronous)
    .whenServerEndpointRunLogic(module.createProductRoute)
    .backend()

  // Execute request with invalid payload
  val response = basicRequest
    .post(uri"http://test.example/api/products")
    .body(invalidJson)
    .contentType("application/json")
    .send(backend)

  // Assertions
  assert(response.code)(equalTo(StatusCode.BadRequest)) &&
  assert(response.body.toOption.get)(containsString("VALIDATION_ERROR"))
}
```

## OpenAPI Documentation

Tapir automatically generates OpenAPI documentation from your endpoint definitions:

```scala
import sttp.apispec.openapi.circe.yaml._
import sttp.tapir.docs.openapi.OpenAPIDocsInterpreter
import sttp.tapir.server.ziohttp.ZioHttpInterpreter
import zio.http.HttpApp

class ApiDocumentation(modules: List[TapirWebModule[Any]]):
  // Collect all endpoints from all modules
  val allEndpoints = modules.flatMap(_.endpoints)
  
  // Generate OpenAPI documentation
  val openApiDocs = OpenAPIDocsInterpreter().toOpenAPI(
    allEndpoints, 
    "My API", 
    "1.0"
  )
  
  // Create a route for serving the documentation
  val docsRoute: HttpApp[Any] = ZioHttpInterpreter().toHttp(
    endpoint.get
      .in("docs" / "openapi.yaml")
      .out(stringBody)
      .serverLogicSuccess(_ => ZIO.succeed(openApiDocs.toYaml))
  )
```

## Common Pitfalls

1. **Leaking Domain Logic** - Keep business logic in domain services
2. **Missing Error Handling** - Define appropriate error responses for all scenarios
3. **Inconsistent Response Formats** - Use consistent patterns for success/error responses
4. **Complex Data Transformations** - Keep transformations simple and focused
5. **Environment Type Mismatches** - Ensure environment types are correctly composed
6. **Mixing Concerns** - Separate endpoint definition from business logic implementation
7. **Content Type Issues** - Always specify correct content types, especially for HTML
8. **Not Setting Appropriate Status Codes** - Be precise with status codes for different scenarios

## Tapir Best Practices

### For All Modules

1. **Base Endpoints** - Create base endpoints for common configurations
2. **Error Handling** - Define clear error models with appropriate status codes
3. **Documentation** - Add names and descriptions to all endpoints
4. **Test Coverage** - Test all endpoints including error scenarios
5. **Environment Composition** - Properly compose ZIO environments for services
6. **Performance Considerations** - Use streaming for large responses when appropriate

### For HTML UI Modules

1. **HTML Output** - Use `stringBody.mapTo[String]` for HTML responses
2. **Content Type** - Always set "text/html; charset=UTF-8" for HTML responses
3. **Error Pages** - Return appropriate error pages for failures
4. **UI Consistency** - Use a common AppShell for consistent layouts

### For API Modules

1. **Schemas** - Define all input/output schemas with proper derivation
2. **Examples** - Add examples for better documentation
3. **Status Codes** - Use appropriate HTTP status codes for different scenarios
4. **Versioning** - Consider API versioning strategy if needed
5. **Content Negotiation** - Support multiple formats when appropriate

## Checklist

- [ ] Module extends TapirWebModule with appropriate environment type
- [ ] All dependencies are clearly defined and injected
- [ ] Endpoints are organized logically with proper naming
- [ ] Error handling is consistent and comprehensive
- [ ] Business logic is separated from endpoint definitions
- [ ] Data transformations are clean and focused
- [ ] Tests cover both success and error scenarios
- [ ] Documentation is provided for all endpoints
- [ ] Environment types are correctly composed
- [ ] HTML UI modules set appropriate content types
- [ ] Security is properly implemented for protected resources
- [ ] OpenAPI documentation is automatically generated