# Chapter 1: Microservices Architecture

Welcome to the `microservices-demo` project tutorial! This project, called "Online Boutique," is an online store built using a modern approach called **Microservices Architecture**.

Imagine you want to build a big, complex application like an online shop. If you put all the code for every feature (browsing products, shopping cart, payment, user accounts, etc.) into one single giant program, it becomes very hard to manage. This old way is often called a "monolith".

*   **Problem:** Making a small change in one part of a monolith (like updating the payment system) could accidentally break something else (like the product display). Testing becomes slow, and updating the entire application takes a long time. Different teams can't easily work on different features simultaneously without stepping on each other's toes.

*   **Solution:** Microservices! Instead of one big program, we break the application down into many small, independent programs called "services". Each service does one specific job.

## What are Microservices?

In a microservices architecture, the application is split into smaller, independent services. For our Online Boutique shop, we have services like:

*   `frontend`: Shows the website to the user.
*   `productcatalogservice`: Keeps track of all the products available.
*   `cartservice`: Manages the items in a user's shopping cart.
*   `checkoutservice`: Handles the process of buying the items.
*   `paymentservice`: Processes credit card payments (don't worry, it's fake in this demo!).
*   `shippingservice`: Calculates shipping costs.
*   ...and several others!

**Analogy: Company Departments**

Think of it like a company. Instead of one person trying to do *everything*, a company has different departments:

*   Sales Department (like `frontend` - interacts with the customer)
*   Warehouse Department (like `productcatalogservice` - knows the inventory)
*   Billing Department (like `paymentservice` - handles money)
*   Shipping Department (like `shippingservice` - handles delivery)

Each department is specialized and focuses on its own tasks. They need to communicate and collaborate to fulfill a customer's order, but they can mostly work independently.

**Key Ideas:**

1.  **Small and Focused:** Each microservice does one thing well (e.g., `cartservice` *only* deals with the shopping cart).
2.  **Independent:** Services can be developed, tested, updated, and deployed separately. If the team working on `recommendationservice` wants to release a new feature, they can do it without waiting for the `paymentservice` team.
3.  **Network Communication:** Services don't directly share code like in a monolith. Instead, they talk to each other over the network, usually using lightweight methods like APIs (Application Programming Interfaces). In this project, we primarily use **gRPC** for this communication (we'll learn more about this in [Chapter 4: gRPC Service Definitions (Protobufs)](04_grpc_service_definitions__protobufs__.md)).
4.  **Technology Choice:** Different services can be written in different programming languages! The team best suited for the job can use the tools they know best. For example:
    *   `frontend` is written in Go.
    *   `cartservice` is written in C#.
    *   `emailservice` is written in Python.
    *   `currencyservice` is written in Node.js.

## How `microservices-demo` Uses Microservices

Our Online Boutique application is composed of 11 microservices. You can see them all listed in the project's main `README.md` file. Here's a visual overview:

[![Architecture of microservices](/docs/img/architecture-diagram.png)](/docs/img/architecture-diagram.png)

Let's trace a simple user action: **Viewing a product page**.

1.  You open your web browser and go to the Online Boutique website. Your browser talks to the `frontend` service.
2.  The `frontend` service needs to display the product's details (name, picture, price). It asks the `productcatalogservice` for this information.
3.  The `frontend` service might also want to show related products. It asks the `recommendationservice` for suggestions based on the current product.
4.  The `recommendationservice` might itself ask the `productcatalogservice` for details about the recommended products.
5.  Once the `frontend` service gathers all the necessary information from the other services, it constructs the web page and sends it back to your browser.

This interaction happens over the network, with each service handling its specific part of the request.

Here's a diagram showing that flow:

```mermaid
sequenceDiagram
    participant User
    participant FE as Frontend Service
    participant PCS as Product Catalog Service
    participant RS as Recommendation Service

    User->>FE: Request Product Page (e.g., for "Camera")
    Note over FE,PCS: Frontend needs product details
    FE->>PCS: GetProduct(ProductID="Camera_123")
    PCS-->>FE: Product Details (Name, Price, Image...)
    Note over FE,RS: Frontend also needs recommendations
    FE->>RS: GetRecommendations(ProductID="Camera_123")
    RS-->>FE: Recommended ProductIDs ("Lens_456", "Tripod_789")
    Note over FE,PCS: Frontend needs details for recommendations
    FE->>PCS: GetProduct(ProductID="Lens_456")
    PCS-->>FE: Lens Details
    FE->>PCS: GetProduct(ProductID="Tripod_789")
    PCS-->>FE: Tripod Details
    Note over FE,User: Combine all info and build the page
    FE-->>User: Display Product Page with Recommendations
end
```

## How Services Talk & Run

*   **APIs (gRPC):** Services define "contracts" (APIs) for how others can talk to them. Our project uses gRPC, a modern way for services to communicate efficiently. Think of it as a very specific phone number and language that services use to call each other. We define these contracts in `.proto` files, covered in [Chapter 4: gRPC Service Definitions (Protobufs)](04_grpc_service_definitions__protobufs__.md).
*   **Running Independently (Containers & Kubernetes):** Each service runs as its own process. To make them easy to manage and deploy, we package them into containers (using Docker). Then, we use a tool called Kubernetes to manage all these containers – starting them, stopping them, making sure they can find each other, and scaling them up if needed. You'll learn more about how we define these deployments in [Chapter 5: Kubernetes Manifests (Base Deployment)](05_kubernetes_manifests__base_deployment__.md) and package them using [Chapter 7: Helm Chart (Packaging and Deployment)](07_helm_chart__packaging_and_deployment__.md).

You can find the actual code for each service inside the `src/` directory in the project:

```
microservices-demo/
├── src/
│   ├── adservice/              # Code for Ad Service (Java)
│   ├── cartservice/            # Code for Cart Service (C#)
│   ├── checkoutservice/        # Code for Checkout Service (Go)
│   ├── currencyservice/        # Code for Currency Service (Node.js)
│   ├── emailservice/           # Code for Email Service (Python)
│   ├── frontend/               # Code for Frontend Service (Go)
│   ├── loadgenerator/          # Code for Load Generator (Python)
│   ├── paymentservice/         # Code for Payment Service (Node.js)
│   ├── productcatalogservice/  # Code for Product Catalog Service (Go)
│   ├── recommendationservice/  # Code for Recommendation Service (Python)
│   └── shippingservice/        # Code for Shipping Service (Go)
├── ... (other configuration files)
```

Inside the code for one service (like `frontend`), you'll find parts that make calls to other services. Here's a *very simplified* conceptual example of what Go code in the `frontend` might look like when it needs product details:

```go
// --- Simplified Go example in frontend ---

// Assume 'productCatalogClient' is setup to talk to the Product Catalog Service

// Ask the Product Catalog Service for details about a specific product ID
productInfo, err := productCatalogClient.GetProduct(ctx, &pb.GetProductRequest{Id: "OLJCESKE"})
// (Real code would handle errors here)

// Now we can use the 'productInfo' received from the other service
fmt.Println("Got product:", productInfo.Name, "Price:", productInfo.PriceUsd)

// --- End of simplified example ---
```

This small piece of code triggers a network request from the `frontend` service to the `productcatalogservice`. The `productcatalogservice` processes the request, finds the product, and sends the details back.

## Conclusion

You've now learned the basic idea behind Microservices Architecture! It's about breaking a large application into smaller, independent services that communicate over the network. This makes the application easier to build, update, and scale.

Our Online Boutique project uses this architecture, with different services written in various languages, all working together.

In the next chapter, we'll take a closer look at one specific service: the [Frontend Service](02_frontend_service_.md), which is the part of the application that users directly interact with in their web browser.

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)