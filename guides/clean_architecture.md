# Clean Architecture

This document is intended to give a brief overview of Clean Architecture, and also discuss some practical implementation considerations from past experience.

## What Is Clean Architecture?

[Clean Architecture] is an architectural framework for software applications. It is built on top of other patterns such as Hexagonal and Onion Architecture with an emphasis on clearly defined boundaries for code interactions. The 'Clean' part is there to emphasise that the core business logic of the system should be free from implementation details such as frameworks or ORM annotations.

One of the driving motivations behind Clean Architecture is that it aims to make it immediately apparent *what the application does* rather than *how it does it*. To give an example, at the first time looking at the codebase, an engineer should be able to say "this is a banking application" rather than "this is a Spring/Hibernate application".

Finally, by having a clearly defined layout for projects, split into 4 layers, it allows for faster code navigation even if you are unfamiliar with the specific codebase. However, within the layers themselves there is a lot of freedom to structure the application as you wish, avoiding the structure getting in the way of how you like to write code.

[Clean Architecture]: https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html

## Why Do We Use It?

The decision to use Clean Architecture came from experiences with previous projects where the codebase became very large and unwieldy due to an 'ad hoc' approach to project structure. Whilst it initially allowed us to quite quickly stand up a project, the different packages soon became quite tightly coupled, even after going through a few refactors. The blast radius of even small changes became quite large and meant that new developers took a long time to understand the codebase, ultimately impacting on delivery of new features.

Additionally, due to the complexity of the codebase, knowledge and expertise in certain areas became quite siloed. As a team we wanted to adopt an approach which would lend itself to pair (and even mob) programming in order to facilitate knowledge sharing and help to get new developers onboarded quickly. 

Clean Architecture was selected as an approach to try as it provided a fairly unambiguous core structure whilst still being very flexible within each layer. This allows different teams to still use a shared set of terminology whilst still giving them freedom to design components to suit their needs.

In addition, Clean Architecture lends itself well to larger projects, not just in terms of lines of code, but also the number of engineers working on an application and the amount of external touch points it has. When working in a team of engineers, Clean Architecture can be very beneficial as it allows for a natural breakdown of work needed to complete a vertically sliced feature. In a micro-service architecture there also tend to be a lot of calls to other APIs and queues, which Clean Architecture helps to organise very neatly. Finally, by having a common structure to applications, it facilitates the changing of code ownership.

Of course, as in all aspects of software engineering, Clean Architecture is not a 'one size fits all' silver bullet. As discussed, it lends itself very well to applications being developed as part of a team or ones that sit in a rich ecosystem of other applications. For small personal projects, or quick and dirty PoC's Clean Architecture might be too much overhead for the benefit it brings. Additionally, smaller 'middleware-esque' services that are tightly targeted, or severless applications might also not benefit from a larger application architecture.

## Prequisites

There are a few concepts that are intrinsic to Clean Architecture that are good to be familiar with:

- [Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control)
- Dependency Injection
- Repository Pattern

## Core Concepts

Fundamentally, there are three main driving concepts behind Clean Architecture. All three are quite interleaved, and work towards providing a consistent approach to a project.

### What Not How

As mentioned above, the first concept is that a Clean Architecture project should make clear the application's capabilities rather than implementation. This is achieved by the concept of the 'usecase' (also referred to as 'interactors'). This is the idea that each specific function of the application should be neatly defined, independent of the other operations.

Ideally, each usecase is placed in its own file, with the filename reflecting the usecase that is contained. This allows someone to quickly get a sense of the application without having to even open any code files. Referring to the usecases layer in the example application gives a good overview of the capabilities it has, as well as providing domain context that this application is working with insurance policies.

### Code Type

The second concept is that at its heart, the structure of Clean Architecture is founded in the idea that there are two distinct types of code in an application:

- Business Dependant - The code that is required to model the business need that the application is fulfilling
- Application Dependant - The code that is required for the application itself to work

By combining the matrix of these two types of code, we have four distinct layers in the application:

| Layer                  | Business Dependant | Application Dependant | Function                    |
| ---------------------- | ------------------ | --------------------- | --------------------------- |
| Domain/Entities        | Yes                | No                    | Enterprise business rules   |
| Usecases/Interactors   | Yes                | Yes                   | Application business rules  |
| Adapters/Interfaces    | No                 | Yes                   | External interface adapters |
| Infrastructure/Drivers | No                 | No                    | Frameworks and drivers      |

Note that different resources use different names for the layers. The actual names themselves are not as important as agreeing on a set of terminology with the team to ensure understanding and consistency. Nonetheless, we recommend the following:

| Layer    | Reasoning                                                                                        |
| -------- | ------------------------------------------------------------------------------------------------ |
| Domain   | Entities tends to have connotations of just 'models', which we wanted to avoid                   |
| Usecases | Usecases is a much better description of the layer's purpose than Interactors                    |
| Adapters | We chose Adapters to avoid confusion with Interfaces in the actual code                          |
| Drivers  | Drivers chose as it was shorter word than 'infrastructure' and avoid confusion with the platform |

Following the matrix table above allows you to quickly identify where different bits of code should live, both for adding new code or for finding something that you're interested in. The separation of the layers also help to decouple the code, as it encourages self contained packages, using the layer boundaries as a clear logical interaction point.

It's important to remember that the definition of the layers can seem rigid, they are quite broad and there is a lot of freedom within the layers to implement logic as the team sees fit. However, in order to make the navigation of the project easier, we recommend you avoid deeply nested packages within the layers, rather preferring a flat structure.

### Layer Dependency Hierarchy

The final concept of Clean Architecture is the one that is most commonly associated with the approach (and contributes to where the name comes from). This is that not only are there clear layers to an application, but there is also a hierarchy between the different layers that determines the direction of dependencies. Starting with the domain layer as the center (and therefore not having any dependencies), each layer should only depend on itself and inner layers to the application. It is important to note that layers can be skipped, as long as the direction is respected (e.g. the adapters layer can directly reference the domain layer).

This results in the business logic being 'clean' in the sense that it is devoid of any frameworks or similar implementation details, and instead provides as 'clean' a representation of the business problem as possible. This is also where the Repository Pattern comes into play, by abstracting away interactions with external components from the business logic. The business logic only cares that a user is stored somewhere rather than it being specifically stored in a Postgres database, for example.

## Practical Implementation Tips

Below are a set of useful tips that we have built up whilst developing services with Clean Architecture. These are purely advisory based on our experience, and are not requirements of implementing Clean Architecture in an application.

### Use the Repository Pattern to Get Off the Ground Quickly

The use of the Repository pattern provides a natural way to get a service up and running quickly without having to worry about you data storage logic. Initial development of your service can make use of in memory databases whilst you develop the business logic and API code for the application. This has several benefits:

- Faster initial development facilitates early demos as well as shortening lead time to integration and testing
- Data models on a new service rarely survive contact with development, by not writing a bunch of data storage logic upfront you reduce the penalty of changing the data model.
- For demos and manual testing, a mock repository implementation that already contains some dummy data can be used, before any create/insert logic is even written.

### Treat Everything External to the Application as a Repository

Whilst repositories are generally thought of in the frame of data storage of application entities, any external interactions of the application can be modeled as a repository. Other services that are interacted with should also be hidden behind interfaces to facilitate testing.

Going even further, interaction with external items such as the operating system should also be considered a repository. For example, a logger writing out to stdout or code fetching time from the system clock can be considered repositories (they are writing or reading data from something outside of the application).

### The Drivers/Infrastructure Layer is Generally Absent

Due to the fact that our backend services stop at the API layer (be it REST, gRPC or sending/receiving events), most services will end up with either a very small or non-existent Drivers layer. In fact, most code that lands in the Driver layer is a good candidate for contributing to shared code libraries.

### Clearly Define Which Logic Owns Which Fields on Your Domain Models

One difficulty that you can encounter as an application matures and changes, is ambiguity over which parts of your data model in the domain are owned by your business logic and which parts are owned by adapter implementations. For example, if you have a model with the field 'LastModified', do you expect your business logic in the Usecase/Domain layers to set that field, or is it a field that is implicitly updated by the Adapter implementation of your repository?

There is no correct solution to this, beyond making sure that you are explicit and consistent. In our experiences, we've found that stating that the business logic owns all fields on the domain models (i.e. your usecase is responsible for updating the 'lastModified' field when it updates the model) works fairly well in simple cases. However, if the data is coming from another service for instance, this might not be a decision you can make. In this case we've found the best is to document the expected behaviour in comments.

### Make Usecases (Almost) Stateless

Having tried both stateful and stateless usecases, almost stateless usecases seem to be the clear winner. To define what we mean by 'almost stateless', these are usecases that all the state they need is static and can be configured at startup (usually via dependency injection). The benefits of having stateless usecases are:

- Reduce abstraction in the system by not requiring a factory or provider for your usecases
- Allow usecases themselves to be injected as dependencies into other component
- Remove boilerplate code required in using and testing usecases
- Removes any concurrency and race condition concerns

### Agree Terminology Beforehand

As with Design Patterns and other codified approaches to software design, the real power of the approach comes from having shared terminology within the team that allows quick discussions to take place about the project structure. As there are different names for each layer in Clean Architecture, the team should agree beforehand on what they are going to use.

It is also worthwhile defining other expected standards/approaches amongst the team, and documenting these in the README/DevGuide/Contributing doc for the application. For example, if the Usecases layer is to be considered an error boundry (errors returned by the usecases do not wrap or contain information about errors further down the domain stack), then that should be explicitly documented, as well as being worthwhile adding some test cases to ensure that.

### Clean Architecture Won't Make a Bad Project Good

On initial reading it may seem that Clean Architecture is a fairly prescriptive approach. However, the four layers described are quite broad, and there is a lot of leeway about how to break things down into packages. This means that the team should still carefully consider decisions about how to organise their code within the layers themselves, attempting to adhere to the principles of single responsibility and decoupling. Whilst Clean Architecture provides an application structure, the code itself should still make use of best practices such as design patterns and deliberate testing.

Working with Clean Architecture requires discipline from the whole team, not only when writing the code but also during reviews. The value of using the approach disappears if the principles are broken for the sake of faster delivery. We all know that once we have tech debt in the codebase (e.g. 'oh this logic shouldn't be in the Adapters layer, but I just put it there for the moment, I'll move it out into a usecase after this is merged') it usually lingers around for a long time.

It is also important to remember that whilst the four layers involved in Clean Architecture is the aspect that tends to be the focus of of discussions and blog posts due to being the most salient part, understanding the reasoning behind why those four layers exist is far more important. This will allow you to make more informed decisions about how to implement the pattern. Importantly, understanding the underlying concepts will help you to be able to decide what are good situations to explicitly deviate from the pattern, should you need to.

## Layers

### Domain

The Domain layer is the part of the application that models the **business dependent** but **application independent** code. It covers the representation in your application of the real world objects and processes that the application intends to model. This layer is also the centre of the application, meaning that it should not include any dependencies on the other three layers of the application.

This layer tends to be quite thin, and sometimes only contains some struct and repository definitions. However, representations of business processes also live in this layer.

When figuring out if code should go in the Domain layer, a good starting point is to ask yourself 'does this code model something that would exist in the business even if our application did not exist?'. If the answer is yes, then it most likely lives here.

### Packages

The below describes the different packages in the Domain layer for the example application. The breakdown into these packages is not a prescription of Clean Architecture, and you should feel free to have a different organisation in your application should you wish.

#### Entities

This package can be considered the application's 'models' package. It describes the business objects that the application intends to interact with, as well as enums and types.

It's important to note that the structs described here are intended to capture the information that **this** application needs to function, rather than just all information. For example, the business most likely holds a wealth of information on cars and trucks but for the purposes of our (extremely simple) policy calculation all we need to know is the VIN and how much it is worth.

#### Repositories

This package describes the interaction points with other parts of the business. It formalises how the transfer of information works in the problem that you are trying to model. 

Whilst it can be argued that these definitions could go in the `usecases` layer as it contains potentially application dependant concepts (for example the `Logger` stands out as a bit of an oddity here), these interaction points are deliberately implementation agnostic.

##### Business over Application

To provide a concrete example, at this layer we don't know if the Customer repository's  data is coming from a database, another service or even someone going over to the filing department and pulling a customer's records. All we know is that there is some process for fetching that information in the business that we interact with.

It's tempting to think of this package as the interaction point of the **application** rather than the interaction points of the **business**. However, the Domain layer is here to model business processes rather than how the application function. Therefore, we tend to group out repositories based on the **entities they interact with** rather than **how they may be implemented.**

Repositories should also aim to be **specific** and **targeted** interfaces, rather than just providing generic CRUD. Of course the Repositories will most likely contain CRUD-like functionality, but remember at this layer we are dealing with the **business process** rather than the application. Taking the `Policy` repository as an example, rather than having a generic `UpdatePolicy()` function we have specific operations to update the status, as that's what the business process intends to use the interaction point for.

##### External Interactions

As mentioned above the Logger and Clock repositories might initially seem out of place. This is because they don't directly describe a business application unless you squint at them. Instead, these are more driven by general good application design principles rather than explicitly Clean Architecture. We should be looking to abstract any external touch points of the application, including interaction with the OS (in this case getting system time or writing to a device) to make for easier testing and portability.

This is a good example of avoiding a completely rigid adherence to the architectural pattern to the detriment of your application. As long as you can justify the deviations, and you don't break a fundamental assumption such as the direction of dependencies, it is ok to deviate.

### Usecases

The Usecases layer is the part of the application that models the **business dependant** and **application dependant** code. It is the part of the application where the business processes and models that are defined in the domain layer are combined together to provide the application behaviour. This layer should still be agnostic of the input/output of the application, and still uses the domain representations or models and repositories.

The Usecases layer should be the first place that someone goes to look to get an understanding of what the application does, as it codifies how the application performs the functions for which it has been designed and built. Generally the Usecases layer is responsible for chaining together the repository and other business logic in the Domain layer into a single business process.

It is usually quite clear when something belongs in the usecase layer, as these are the usecases defined by the application requirements. **Due to the uniform nature of the usecases only the CalculateQuote usecase has been fully commented, as those comments apply to the other usecases as well.**

#### Usecases

The Usecases layer is the most important layer in Clean Architecture as it provides the summary of the application's capabilities. To facilitate this, each usecase is defined in a separate file, with the file name describing the usecase that is within. This allows someone to understand what the application does without even opening an of the code files.

In this example application there are only 4 usecases, and so they are all defined in the base 'usecases' dir. If the number of usecases in an application is quite large then it is fine to group them into sub-packages as long as the naming is kept clear for navigation.

Beyond having a separate object and file for each usecase, the actual structure and implementation of the usecase is very much up to the team's discretion. Below are some approaches that we have found to work quite well when implementing usecases, but the shouldn't be considered as hard requirements.

##### Static State/Stateless

The usecases defined here are all either stateless or hold static state (i.e. the state they hold is configured at startup and used throughout the lifetime of the application). This allows a single instance of each usecase to be instantiated at startup and then references to it injected into the rest of the application.

Any state that could possibly change (i.e. inputs or repository implementations) are provided at each invocation of the usecase. This also ensures that the usecases can be used concurrently without needing to consider race conditions.

As usual, it's perfectly fine to deviate from this for specific exceptions as long as it still fits with the core assumptions made. For example it would be acceptable to have something defined in the usecase that internally manages it's own runtime state, as long as its internal state management is encapsualted and concurrency safe.

##### Error Boundary

We have found that the Usecases layer is a natural error boundary for the application. This means that errors that occur internally in the usecase are logged but not bubbled up the the usecase caller. Instead a clean, new error is defined that does not wrap the other error.

This approach allows the callers of the usecase to be able to tell what error types they should be handling without having to dig through the rest of the codebase (they can instead just look at the usecase code itself). It helps to reinforce the idea that the Usecases layer is the entry and exit point of the business logic in the application.

This approach also simplifies some of the complexity with using `errors.Is` and `errors.As` as it means we don't need to consider wrapped error types, and instead can just focus on the returned error's type.

##### Input Encapsulation and Avoiding Mutation

The signatures for the usecases in the example code all follow the same pattern. This pattern is to have two structs that contain the input to the usecase, the first being the actual input values for the usecase and the second being the repository implementations that are to be used.

This approach has several benefits. The first is that some usecases have a large number of inputs, and this prevents the usecase's signature from getting unweildy. Whilst this might be considered overkill for usecases that only have one or two parameters, following a consistent approach across the usecases helps to unify the codebase and make it easier to read. 

The second benefit is that it makes refactoring of code easier as you don't have to change function signatures (and therefore potentially break interfaces that have been defined) if the inputs to the usecase change over time. Having the inputs encapsulated in a struct also allows for the centralisation of validation logic, as it can be defined as a function of the struct, rather than risk it sprawling across the usecase code itself.

Finally, with the input encapsulated into a struct it serves as a data transfer object. The fact that it is passed in by value signals to the caller that it will not be mutated by the usecase. Adopting this standard pattern across the usecases means that the code calling the usecases can be simplified as they are allowed to assume that input into a usecase will not be changed.

##### Consumer Defined Interfaces

It is common to see interfaces defined for the usecases in the Usecase layer. However, we can make use of Go's implicit interfaces and instead define the interfaces at the point of consumption rather than definition. This approach, of not specifying interfaces for the usecases in this layer, means that we are not making any assumptions about how our usecases will be organised or invoked. Avoiding this assumption allows for greater decoupling of the layers in the application.

Another example of this approach can be seen in the CalculateQuote usecase, where whilst the factory for creating the calculator is defined in the domain, the interface for it is defined in the next layer up (i.e. the Usecases layer).

### Adapters

The Adapters layer is responsible for providing the **application specific** logic. This is where all the code that would only exist because you are building this application (REST APIs, database logic etc.) lives. It can be thought of as the layer that is responsible for taking data in external formats (e.g a gRPC model or a database schema), transforming it into the internal representation, executing the usecases and then converting the usecase output from the internal representation to the external one.

Due to the fact that we work with a micro-service architecture, there is a lot of inter-service communication in DSCC applications and therefore the Adapters layer tends to be the largest. It is important to note that whilst a lot of the packages in the Adapter layer will correspond to a repository or an application entry point, this is not a requirement of the layer and so can contain any other **application dependant** but **business independent** code.

#### Packages

The below describes the different packages in the Adapters layer for the example application. The breakdown into these packages is not a prescription of Clean Architecture, and you should feel free to have a different organisation in your application should you wish.

##### Common

This package is an example of not all packages in the Adapters layer needing to be an implementation of a repository (which is sometimes an assumption that is made). This package serves to reduce the duplication of code across the other Adapters packages.

The usecase facade in this package is not a feature of Clean Architecture, and is rather just a way to avoid duplication of transaction logic when invoking usecases used by multiple interfaces. It also reduces boilerplate in unit tests for code using the facade as they can avoid mocking transaction logic. It has been included in the example project to demonstrate a way to cleanly handle usecases that are not transaction aware, and can be ignored if the desire is to reduce indirection in the project.

It may seem that the REST client defined in this package should be in the Drivers layer instead, as it does not contain any application specific code. However, it is serving the purpose of reducing code duplication in application specific code. The logic that is implemented in the REST client would be implemented in an application specific manner were the REST client code implemented in each of the adapters instead.

##### PolicyPostgres/PolicyMemDB

These two packages provide different implementations of the same repository, demonstrating the flexibility that the Repository Pattern provides. Whilst the most obvious utility of the repository pattern is the ability to swap out components should they need to (e.g. a REST interface that gets migrated to the gRPC interface), it can also help to accommodate developer's needs during different stages of the project.

In this example, the in-memory implementation would provide a lot of utility at the beginning of the project, as it requires less work to set up (both in terms of database schema but also running the project locally) and is more flexible to data model changes. Once the application has matured past the early stages and persistence is required, it can be swapped out for the SQL implementation as the data model will be much more concrete.

##### WorkflowNATS

This package provides two complimentary pieces of functionality that are concerned with an entry/exit point to the application. The first is an implementation of the Workflow repository for asynchronous messaging, either to request approval for a policy from the underwriters or to notify other services in the system that a policy has been approved/rejected (for example to send a notification email or schedule a follow up call).

The second piece of functionality is to listen for messages that come in after the underwriters have made a decision on an application to update the proposed policy. Both of these functions fulfil the requirement of converting between the application independent internal representations and the application dependant (in this case a NATS Jetstream queue) representations.

##### PolicyRESTServer

Some engineers might find the PolicyRESTServer package slightly confusing, in that it doesn't actually contain a REST server. Instead, it contains the application specific logic used in a REST server (i.e. models, handlers and middleware). This is following the idea of Inversion of Control, where rather than our application specific logic calling into generic libraries, instead we provide our logic to generic libraries to execute. In this case, the actual REST server logic itself lives in the Drivers layer (in a production application this server wouild most likely be a third party library/framework rather than be part of the codebase).

This package also provides the textbook demonstration of the main responsibility of the Adapters layer, specifically the conversion of the external representation of the application (in this case the REST API of the application) and the internal representation of the application (the Domain) and then back again after having executed some operation in the usecases. An example of this is the conversion from the REST layer's product focused policy term of either 1 or 3 years to the internal, flexibility focused policy term that is handled in months.

##### VehicleRESTClient/CustomerRESTClient

These two packages provide REST implementations for the Customer and Vehicle repositories. The packages themselves are pretty thin, which can be quite common for some adapters, especially when dealing with basic CRUD requires to share data across micro-services. Whilst it is reasonable to imagine that the actual Vehicle and Customer services would have much larger APIs, in our service we are only concerned with the endpoints required for our own functionality. As discussed in the code comments, in a real world example these packages would most likely just shim out to a generated REST client, and the only code would be the model conversion.

### Drivers

The Drivers (AKA 'Infrastructure'/'Frameworks and Drivers') layer is the home of code that is **neither application or business specific**. This is were the 'generic' code resides, and most of the time it can be used in other applications without modification. An example of this would be the UI rendering logic for a desktop application. Whilst the code to specify that there should be a dialog box of a specific size with the title 'Confirm Delete' with 'OK' and 'Cancel' would live in the Adapters layer, the code for actually rendering the dialog box on the screen (put these there pixels here, listen for mouse input on buttons) would live in the Drivers layer.

Due to our intention in DSCC to reuse code where possible, coupled with the fact that our backend applications' responsibility ends at the REST or gRPC API, it is quite common for our applications to have minimal or even no code in the Drivers layer.

#### Packages

##### RESTServer

Following the intention of this example using minimal outside dependencies, and for demonstration purposes, this package has been created to be used in the application rather than using a third party REST server as you would in a real application.

Note the use of Inversion of Control, where we are injecting handlers for the server to user, which allows this code to be reusable in other projects should we be so inclined.

##### Clock/Logging

These small helper packages are included in this example to reduce external dependencies. As with most Infrastructure/Drivers layer code, they are great candidates for promotion to a common code library.

## Other Resources

- [Clean Architecutre Concepts and Creation Presentation](https://www.youtube.com/watch?v=2dKZ-dWaCiU)
- [Clean Architecture Dependancy Rule Blog Post](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
