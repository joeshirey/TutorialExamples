# Chapter 2: Frontend Service

Welcome back! In [Chapter 1: Microservices Architecture](01_microservices_architecture_.md), we learned the basics of microservices – breaking down a big application like our Online Boutique into smaller, independent services.

Now, let's zoom in on one of the most important services: the **Frontend Service**.

## What Problem Does the Frontend Service Solve?

Imagine walking into a physical store. The first thing you see is the storefront, the displays, and maybe a customer service desk. This is where you browse items, ask questions, and decide what you want to buy.

In our online shop, the **Frontend Service** plays this exact role. It's the main web interface that you, the user, interact with through your web browser. Without it, customers wouldn't be able to see the products, add items to their cart, or check out. It's the "face" of our online store.

**Use Case: Visiting the Home Page**

When you type the website address for the Online Boutique into your browser and hit Enter, you expect to see the home page with products, maybe some ads, and your shopping cart status. How does this happen? The Frontend Service is responsible for putting that page together for you.

## Key Concepts

1.  **Web Server:** At its core, the Frontend Service is a web server. Its main job is to receive requests from your browser (like "show me the home page") and send back the necessary files (HTML for structure, CSS for style, JavaScript for interactivity) so your browser can display the website.
2.  **User Interface (UI):** It serves the visual part of the application – the buttons, images, text, and layout that you see and click on.
3.  **Orchestrator:** The Frontend doesn't know everything itself. It doesn't store the product list or manage the shopping cart directly. Instead, when you request a page, it acts like a coordinator or "orchestrator". It asks other specialized backend microservices for the specific pieces of information it needs (like getting the product list from the `productcatalogservice` or the cart contents from the `cartservice`).
4.  **Primary Entry Point:** For a typical user browsing the website, the Frontend is the *only* service their browser directly communicates with. All other microservices work behind the scenes, coordinated by the Frontend.

## How the Frontend Works: A Quick Tour

Let's revisit our example: loading the home page.

1.  **You:** Open your browser and navigate to the Online Boutique's address.
2.  **Browser:** Sends an HTTP request to the Frontend Service.
3.  **Frontend Service:** Receives the request. It knows it needs several pieces of information to build the home page:
    *   List of products.
    *   Supported currencies.
    *   Items currently in your shopping cart (identified by a session cookie).
    *   Maybe a promotional ad.
4.  **Frontend Service (Orchestration):** It makes separate requests (using gRPC, which we'll cover in [Chapter 4: gRPC Service Definitions (Protobufs)](04_grpc_service_definitions__protobufs_.md)) to other microservices:
    *   Asks `productcatalogservice`: "Give me the list of products."
    *   Asks `currencyservice`: "What currencies do you support?"
    *   Asks `cartservice`: "What items are in this user's cart?"
    *   Asks `adservice`: "Give me a relevant ad."
5.  **Backend Services:** Each service processes its request and sends the data back to the Frontend.
    *   `productcatalogservice` -> sends product data.
    *   `currencyservice` -> sends currency codes (USD, EUR, etc.).
    *   `cartservice` -> sends cart items.
    *   `adservice` -> sends ad details.
6.  **Frontend Service:** Gathers all the responses. It uses HTML templates to combine this data into a complete web page.
7.  **Frontend Service:** Sends the final HTML page back to your browser.
8.  **Browser:** Renders the HTML, and you see the Online Boutique home page!

Here's a diagram illustrating this flow:

```mermaid
sequenceDiagram
    participant User Browser
    participant FE as Frontend Service
    participant PCS as Product Catalog Service
    participant CS as Cart Service
    participant ADS as Ad Service
    participant CurrencySvc as Currency Service

    User Browser->>FE: GET / (Request Home Page)
    FE->>PCS: ListProducts()
    PCS-->>FE: List of Products
    FE->>CS: GetCart(userID)
    CS-->>FE: Cart Items
    FE->>ADS: GetAds()
    ADS-->>FE: Ad Details
    FE->>CurrencySvc: GetSupportedCurrencies()
    CurrencySvc-->>FE: List of Currencies
    Note over FE: Combine data into HTML page
    FE-->>User Browser: HTML Page
end
```

## A Peek Inside the Code (Go)

The Frontend Service is written in the Go programming language. Let's look at simplified snippets to understand how it works.

**1. Handling the Home Page Request (`src/frontend/handlers.go`)**

When a request for the home page (`/`) comes in, a function like `homeHandler` takes over. It coordinates fetching data from other services.

```go
// Simplified version of homeHandler in src/frontend/handlers.go

func (fe *frontendServer) homeHandler(w http.ResponseWriter, r *http.Request) {
	log := /* get logger */
	log.Info("Request received for home page")

	// 1. Get products from ProductCatalogService
	products, err := fe.getProducts(r.Context())
	// (Real code handles errors)

	// 2. Get user's cart from CartService
	cart, err := fe.getCart(r.Context(), sessionID(r))
	// (Real code handles errors)

	// 3. Get currencies from CurrencyService
	currencies, err := fe.getCurrencies(r.Context())
	// (Real code handles errors)

	// 4. Choose an Ad from AdService
	ad := fe.chooseAd(r.Context(), []string{}, log)

	// 5. Prepare data and render the HTML template ("home.html")
	if err := templates.ExecuteTemplate(w, "home", map[string]interface{}{
		"products":   products,
		"cart_size":  cartSize(cart),
		"currencies": currencies,
		"ad":         ad,
		// ... other data for the page
	}); err != nil {
		log.Error(err)
	}
}
```

*Explanation:* This function shows the orchestration role. It calls other internal functions (`fe.getProducts`, `fe.getCart`, etc.) which are responsible for making the actual network calls to the backend microservices. Finally, it uses `templates.ExecuteTemplate` to merge the collected data with an HTML template (`home.html`) and sends the result back to the user's browser (`w`).

**2. Calling Another Service (`src/frontend/rpc.go`)**

How does `fe.getProducts` actually talk to the `productcatalogservice`? It uses gRPC.

```go
// Simplified version of getProducts in src/frontend/rpc.go

import pb "github.com/GoogleCloudPlatform/microservices-demo/src/frontend/genproto"

// getProducts makes a gRPC call to the ProductCatalogService
func (fe *frontendServer) getProducts(ctx context.Context) ([]*pb.Product, error) {
	// Get a gRPC client for ProductCatalogService (connection setup done earlier)
	client := pb.NewProductCatalogServiceClient(fe.productCatalogSvcConn)

	// Make the remote procedure call (RPC)
	resp, err := client.ListProducts(ctx, &pb.Empty{})
	if err != nil {
		// Handle error (e.g., network issue, service down)
		return nil, err
	}

	// Return the list of products received from the service
	return resp.GetProducts(), nil
}
```

*Explanation:* This function uses a pre-established gRPC connection (`fe.productCatalogSvcConn`) to create a `client`. It then calls the `ListProducts` function *on the remote service* just like calling a local function. The data (`resp`) is received over the network. The specific functions available (`ListProducts`, `GetProduct`, etc.) are defined using Protobufs, which we'll explore in [Chapter 4: gRPC Service Definitions (Protobufs)](04_grpc_service_definitions__protobufs_.md).

**3. Knowing Where Other Services Are (Configuration)**

How does the Frontend service know the network address of `productcatalogservice` or `cartservice`? This is configured when the service is deployed. Looking at the Kubernetes configuration ([Chapter 5: Kubernetes Manifests (Base Deployment)](05_kubernetes_manifests__base_deployment_.md)), we see environment variables telling the Frontend where to find its neighbors:

```yaml
# Simplified snippet from kubernetes-manifests/frontend.yaml
# Defines the Frontend Deployment in Kubernetes

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  template:
    spec:
      containers:
        - name: server
          image: frontend # The container image for this service
          env:
          # --- Environment variables telling Frontend where other services are ---
          - name: PORT
            value: "8080" # Port the frontend listens on
          - name: PRODUCT_CATALOG_SERVICE_ADDR
            value: "productcatalogservice:3550" # Address of Product Catalog Service
          - name: CURRENCY_SERVICE_ADDR
            value: "currencyservice:7000"       # Address of Currency Service
          - name: CART_SERVICE_ADDR
            value: "cartservice:7070"           # Address of Cart Service
          # ... other service addresses ...
```

*Explanation:* Kubernetes uses these environment variables (`PRODUCT_CATALOG_SERVICE_ADDR`, `CART_SERVICE_ADDR`, etc.) to inject the network addresses of the other microservices into the Frontend container when it starts up. The Frontend code (like in `src/frontend/main.go`) reads these variables to know how to connect to them via gRPC. Kubernetes handles the underlying networking to make sure `productcatalogservice:3550` resolves to the correct running container. We'll see how these deployments are managed in detail in later chapters covering Kubernetes and Helm.

## Conclusion

The Frontend Service is the user-facing gateway to our Online Boutique. It acts as a web server, serving the HTML UI, and orchestrates calls to various backend microservices via gRPC to gather the data needed to display pages like the product list, shopping cart, and checkout forms. It's written in Go and relies on configuration (like Kubernetes environment variables) to discover and communicate with the other services in our microservices architecture.

In the next chapter, we'll dive into another critical service that the Frontend talks to: the [Checkout Service](03_checkout_service_.md), which handles the final steps of placing an order.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)