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

## Other Resources

- [Clean Architecutre Concepts and Creation Presentation](https://www.youtube.com/watch?v=2dKZ-dWaCiU)
- [Clean Architecture Dependancy Rule Blog Post](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
