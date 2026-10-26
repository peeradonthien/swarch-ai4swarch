# Online Boutique — recovered container architecture (run B)

## Agent

Codex (GPT-6 family; exact model identifier unavailable).

## Files read

- `prompts/1-recover-architecture.md`
- `system/online-boutique/protos/demo.proto`
- `system/online-boutique/kubernetes-manifests/adservice.yaml`
- `system/online-boutique/kubernetes-manifests/cartservice.yaml`
- `system/online-boutique/kubernetes-manifests/checkoutservice.yaml`
- `system/online-boutique/kubernetes-manifests/currencyservice.yaml`
- `system/online-boutique/kubernetes-manifests/emailservice.yaml`
- `system/online-boutique/kubernetes-manifests/frontend.yaml`
- `system/online-boutique/kubernetes-manifests/kustomization.yaml`
- `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`
- `system/online-boutique/kubernetes-manifests/paymentservice.yaml`
- `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml`
- `system/online-boutique/kubernetes-manifests/README.md`
- `system/online-boutique/kubernetes-manifests/recommendationservice.yaml`
- `system/online-boutique/kubernetes-manifests/shippingservice.yaml`

## Containers table

Each row represents one C4 container, not a Kubernetes Service or ServiceAccount. Responsibilities below are interface-level capabilities unless marked otherwise; implementation behavior is not established by RPC declarations. The supplied configuration is evidence of deployment intent, not an observed running deployment.

| Name | Technology/language | Responsibility | Evidence |
|---|---|---|---|
| frontend | HTTP service; implementation language not evidenced | Exposes the shop HTTP entry point | `system/online-boutique/kubernetes-manifests/frontend.yaml` (Service frontend and frontend-external; httpGet /_healthz) |
| cartservice | gRPC service; implementation language not evidenced | Adds items, retrieves carts, and empties carts | `system/online-boutique/kubernetes-manifests/cartservice.yaml` (Deployment cartservice; REDIS_ADDR); `system/online-boutique/protos/demo.proto` (CartService: AddItem, GetCart, EmptyCart) |
| checkoutservice | gRPC service; implementation language not evidenced | Accepts order placement | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (Deployment checkoutservice); `system/online-boutique/protos/demo.proto` (CheckoutService: PlaceOrder) |
| currencyservice | gRPC service; implementation language not evidenced | Lists supported currencies and converts money | `system/online-boutique/kubernetes-manifests/currencyservice.yaml` (Deployment currencyservice); `system/online-boutique/protos/demo.proto` (CurrencyService: GetSupportedCurrencies, Convert) |
| emailservice | gRPC service; implementation language not evidenced | Exposes order-confirmation email operation | `system/online-boutique/kubernetes-manifests/emailservice.yaml` (Deployment emailservice); `system/online-boutique/protos/demo.proto` (EmailService: SendOrderConfirmation) |
| paymentservice | gRPC service; implementation language not evidenced | Exposes card charging operation | `system/online-boutique/kubernetes-manifests/paymentservice.yaml` (Deployment paymentservice); `system/online-boutique/protos/demo.proto` (PaymentService: Charge) |
| productcatalogservice | gRPC service; implementation language not evidenced | Lists, retrieves, and searches products | `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (Deployment productcatalogservice); `system/online-boutique/protos/demo.proto` (ProductCatalogService: ListProducts, GetProduct, SearchProducts) |
| recommendationservice | gRPC service; implementation language not evidenced | Returns recommended product IDs | `system/online-boutique/kubernetes-manifests/recommendationservice.yaml` (Deployment recommendationservice); `system/online-boutique/protos/demo.proto` (RecommendationService: ListRecommendations) |
| shippingservice | gRPC service; implementation language not evidenced | Returns shipping quotes and ships orders | `system/online-boutique/kubernetes-manifests/shippingservice.yaml` (Deployment shippingservice); `system/online-boutique/protos/demo.proto` (ShippingService: GetQuote, ShipOrder) |
| adservice | gRPC service; implementation language not evidenced | Returns advertisements for context keys | `system/online-boutique/kubernetes-manifests/adservice.yaml` (Deployment adservice); `system/online-boutique/protos/demo.proto` (AdService: GetAds) |
| redis-cart | Redis (redis:alpine) | Configured cart-service data store; /data uses emptyDir | `system/online-boutique/kubernetes-manifests/cartservice.yaml` (REDIS_ADDR; Deployment redis-cart; image redis:alpine; volumeMounts /data; volumes.redis-data.emptyDir) |
| loadgenerator | Container image loadgenerator; implementation language not evidenced | [Inferred] Generates shop traffic; checks frontend HTTP availability at startup | `system/online-boutique/kubernetes-manifests/loadgenerator.yaml` (Deployment loadgenerator; FRONTEND_ADDR, USERS, RATE; initContainers.frontend-check.command) |

The loadgenerator is included because its Deployment is present in `system/online-boutique/kubernetes-manifests/loadgenerator.yaml`; `system/online-boutique/kubernetes-manifests/kustomization.yaml` comments out its resource entry. It is therefore shown as a supplied optional workload, not as an enabled base resource. The frontend-check init container belongs to that workload and does not form a separate C4 application container.

## Dependencies table

The service-to-service rows recover configured destinations from caller environment variables. Their gRPC protocol is supported by the destination Service port named grpc and the matching RPC interface in demo.proto. These files do not establish which individual RPCs each caller executes. The required end-user actor and its use of the exposed endpoint are marked [Inferred]; the HTTP exposure itself is evidenced. No external-system identity is established.

| Caller → callee | Protocol | Relationship | Evidence |
|---|---|---|---|
| Customer → frontend | HTTP | Uses exposed shop endpoint [Inferred] | `system/online-boutique/kubernetes-manifests/frontend.yaml` (Service frontend-external: type LoadBalancer, port http 80 → targetPort 8080) |
| frontend → productcatalogservice | gRPC | Configured product catalog dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (Service productcatalogservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (ProductCatalogService) |
| frontend → currencyservice | gRPC | Configured currency dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (CURRENCY_SERVICE_ADDR=currencyservice:7000); `system/online-boutique/kubernetes-manifests/currencyservice.yaml` (Service currencyservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (CurrencyService) |
| frontend → cartservice | gRPC | Configured cart dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (CART_SERVICE_ADDR=cartservice:7070); `system/online-boutique/kubernetes-manifests/cartservice.yaml` (Service cartservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (CartService) |
| frontend → recommendationservice | gRPC | Configured recommendation dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080); `system/online-boutique/kubernetes-manifests/recommendationservice.yaml` (Service recommendationservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (RecommendationService) |
| frontend → shippingservice | gRPC | Configured shipping dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (SHIPPING_SERVICE_ADDR=shippingservice:50051); `system/online-boutique/kubernetes-manifests/shippingservice.yaml` (Service shippingservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (ShippingService) |
| frontend → checkoutservice | gRPC | Configured checkout dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (CHECKOUT_SERVICE_ADDR=checkoutservice:5050); `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (Service checkoutservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (CheckoutService) |
| frontend → adservice | gRPC | Configured ad dependency | `system/online-boutique/kubernetes-manifests/frontend.yaml` (AD_SERVICE_ADDR=adservice:9555); `system/online-boutique/kubernetes-manifests/adservice.yaml` (Service adservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (AdService) |
| checkoutservice → productcatalogservice | gRPC | Configured product catalog dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (Service productcatalogservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (ProductCatalogService) |
| checkoutservice → shippingservice | gRPC | Configured shipping dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (SHIPPING_SERVICE_ADDR=shippingservice:50051); `system/online-boutique/kubernetes-manifests/shippingservice.yaml` (Service shippingservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (ShippingService) |
| checkoutservice → paymentservice | gRPC | Configured payment dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (PAYMENT_SERVICE_ADDR=paymentservice:50051); `system/online-boutique/kubernetes-manifests/paymentservice.yaml` (Service paymentservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (PaymentService) |
| checkoutservice → emailservice | gRPC | Configured email dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (EMAIL_SERVICE_ADDR=emailservice:5000); `system/online-boutique/kubernetes-manifests/emailservice.yaml` (Service emailservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (EmailService) |
| checkoutservice → currencyservice | gRPC | Configured currency dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (CURRENCY_SERVICE_ADDR=currencyservice:7000); `system/online-boutique/kubernetes-manifests/currencyservice.yaml` (Service currencyservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (CurrencyService) |
| checkoutservice → cartservice | gRPC | Configured cart dependency | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml` (CART_SERVICE_ADDR=cartservice:7070); `system/online-boutique/kubernetes-manifests/cartservice.yaml` (Service cartservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (CartService) |
| recommendationservice → productcatalogservice | gRPC | Configured product catalog dependency | `system/online-boutique/kubernetes-manifests/recommendationservice.yaml` (PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml` (Service productcatalogservice, ports.name=grpc); `system/online-boutique/protos/demo.proto` (ProductCatalogService) |
| cartservice → redis-cart | Redis over TCP:6379 | Configured cart storage dependency | `system/online-boutique/kubernetes-manifests/cartservice.yaml` (REDIS_ADDR=redis-cart:6379; Deployment redis-cart image redis:alpine; Service redis-cart ports.tcp-redis) |
| loadgenerator → frontend | HTTP | Targets frontend; checks availability at startup | `system/online-boutique/kubernetes-manifests/loadgenerator.yaml` (FRONTEND_ADDR=frontend:80 (main and frontend-check); frontend-check.command uses http://${FRONTEND_ADDR}) |

For emailservice, `system/online-boutique/kubernetes-manifests/emailservice.yaml` maps Service port 5000 to targetPort 8080; the checkout address therefore agrees with the Service rather than the container port. Kubernetes health probes are deployment checks, not additional application dependency rows.

## Mermaid diagram

```mermaid
C4Container
  title Containers: Online Boutique (run B; includes optional loadgenerator)
  Person(customer, "Customer", "[Inferred] User of the exposed shop endpoint")
  System_Boundary(shop, "Online Boutique") {
    Container(frontend, "frontend", "HTTP service; implementation language not evidenced", "Exposes the shop HTTP entry point")
    Container(cartservice, "cartservice", "gRPC service; implementation language not evidenced", "Adds items, retrieves carts, and empties carts")
    Container(checkoutservice, "checkoutservice", "gRPC service; implementation language not evidenced", "Accepts order placement")
    Container(currencyservice, "currencyservice", "gRPC service; implementation language not evidenced", "Lists supported currencies and converts money")
    Container(emailservice, "emailservice", "gRPC service; implementation language not evidenced", "Exposes order-confirmation email operation")
    Container(paymentservice, "paymentservice", "gRPC service; implementation language not evidenced", "Exposes card charging operation")
    Container(productcatalogservice, "productcatalogservice", "gRPC service; implementation language not evidenced", "Lists, retrieves, and searches products")
    Container(recommendationservice, "recommendationservice", "gRPC service; implementation language not evidenced", "Returns recommended product IDs")
    Container(shippingservice, "shippingservice", "gRPC service; implementation language not evidenced", "Returns shipping quotes and ships orders")
    Container(adservice, "adservice", "gRPC service; implementation language not evidenced", "Returns advertisements for context keys")
    ContainerDb(redis_cart, "redis-cart", "Redis (redis:alpine)", "Configured cart-service data store; /data uses emptyDir")
    Container(loadgenerator, "loadgenerator", "Container image loadgenerator; implementation language not evidenced", "[Inferred] Generates shop traffic; checks frontend HTTP availability at startup")
  }
  Rel(customer, frontend, "Uses exposed shop endpoint [Inferred]", "HTTP")
  Rel(frontend, productcatalogservice, "Configured product catalog dependency", "gRPC")
  Rel(frontend, currencyservice, "Configured currency dependency", "gRPC")
  Rel(frontend, cartservice, "Configured cart dependency", "gRPC")
  Rel(frontend, recommendationservice, "Configured recommendation dependency", "gRPC")
  Rel(frontend, shippingservice, "Configured shipping dependency", "gRPC")
  Rel(frontend, checkoutservice, "Configured checkout dependency", "gRPC")
  Rel(frontend, adservice, "Configured ad dependency", "gRPC")
  Rel(checkoutservice, productcatalogservice, "Configured product catalog dependency", "gRPC")
  Rel(checkoutservice, shippingservice, "Configured shipping dependency", "gRPC")
  Rel(checkoutservice, paymentservice, "Configured payment dependency", "gRPC")
  Rel(checkoutservice, emailservice, "Configured email dependency", "gRPC")
  Rel(checkoutservice, currencyservice, "Configured currency dependency", "gRPC")
  Rel(checkoutservice, cartservice, "Configured cart dependency", "gRPC")
  Rel(recommendationservice, productcatalogservice, "Configured product catalog dependency", "gRPC")
  Rel(cartservice, redis_cart, "Configured cart storage dependency", "Redis over TCP:6379")
  Rel(loadgenerator, frontend, "Targets frontend; checks availability at startup", "HTTP")
  UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

## Gaps

Paths suggested below are candidates for a later evidence set; none were opened.

| Uncertainty | Evidence boundary | Extra file that would settle it |
|---|---|---|
| Application implementation languages and concrete image versions | Not evidenced in the provided files. App image values are bare names; `system/online-boutique/kubernetes-manifests/README.md` says Skaffold inserts image tags. The go_package option in `system/online-boutique/protos/demo.proto` does not establish every service language. | Each service build file, e.g. `system/online-boutique/src/cartservice/Dockerfile`; `system/online-boutique/skaffold.yaml` for image build configuration. |
| shoppingassistantservice deployment, protocol, and active frontend calls | `system/online-boutique/kubernetes-manifests/frontend.yaml` sets SHOPPING_ASSISTANT_SERVICE_ADDR=shoppingassistantservice:80, but ENABLE_ASSISTANT is commented out. No matching Deployment, Service, or RPC contract appears in the allowed files; `system/online-boutique/kubernetes-manifests/kustomization.yaml` comments out the shopping-assistant component. This is an unresolved configured reference, so no container or complete dependency is drawn. | `system/online-boutique/kustomize/components/shopping-assistant/kustomization.yaml` and its referenced resource file; `system/online-boutique/src/frontend/main.go` for activation and client protocol. |
| Exact RPC calls, call order, retries, timeouts, and failure behavior | Not evidenced in the provided files. `system/online-boutique/protos/demo.proto` defines operations and messages; *_SERVICE_ADDR variables establish configured targets, not execution traces. | Caller implementation files, e.g. `system/online-boutique/src/checkoutservice/main.go` and `system/online-boutique/src/frontend/main.go`. |
| Actual charging, email delivery, shipping integration, and external providers | Not evidenced in the provided files. Charge, SendOrderConfirmation, and ShipOrder in `system/online-boutique/protos/demo.proto` establish interfaces only. No provider or message broker is identified, so none is added. | Service implementation files, e.g. `system/online-boutique/src/paymentservice/server.js`, `system/online-boutique/src/emailservice/email_server.py`, and `system/online-boutique/src/shippingservice/main.go`. |
| Data sources, schemas, and durability beyond the Redis manifest | `system/online-boutique/kubernetes-manifests/cartservice.yaml` evidences REDIS_ADDR and Redis /data backed by emptyDir; durable storage, cart key structure, catalog storage, and exchange-rate sources are Not evidenced in the provided files. | `system/online-boutique/src/cartservice/src/cartstore/RedisCartStore.cs`; `system/online-boutique/src/productcatalogservice/main.go`; `system/online-boutique/src/currencyservice/server.js`. |
| Customer journeys, authentication, and TLS termination | Not evidenced in the provided files. `system/online-boutique/kubernetes-manifests/frontend.yaml` exposes HTTP port 80 and a health endpoint; it does not document customer journeys or an HTTPS ingress. Customer usage is [Inferred]. | `system/online-boutique/src/frontend/main.go` for routes and authentication; the deployment-specific ingress/gateway manifest (filename not evidenced) for TLS. |
| Which workloads actually run | Not evidenced in the provided files. `system/online-boutique/kubernetes-manifests/README.md` says these manifests need image substitution; `system/online-boutique/kubernetes-manifests/kustomization.yaml` excludes loadgenerator and optional components. | `system/online-boutique/skaffold.yaml` for development selection, plus the applied rendered manifest or cluster deployment export for actual runtime state. |
