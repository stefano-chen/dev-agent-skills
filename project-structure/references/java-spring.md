# Java / Spring Boot

## Baseline layout

Maven and Gradle share the standard layout, and both the build and the IDE assume it:

```
my-service/
├── pom.xml | build.gradle(.kts)
├── src/
│   ├── main/
│   │   ├── java/com/acme/shop/
│   │   │   ├── ShopApplication.java        # @SpringBootApplication, in the root package
│   │   │   ├── order/
│   │   │   ├── catalog/
│   │   │   └── customer/
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-<profile>.yml
│   │       └── db/migration/               # Flyway (or db/changelog for Liquibase)
│   └── test/
│       ├── java/com/acme/shop/             # mirrors main, same packages
│       └── resources/
└── README.md
```

- The application class sits in the root package. Component scanning, entity scanning and repository scanning all start from its package, so anything outside that subtree is silently not picked up unless scanning is configured by hand.
- The root package is `<reversed domain>.<application>`, all lowercase, no underscores. It is expensive to change later.
- Start from Spring Initializr (start.spring.io or the IDE integration) for new projects.

## Package by feature

```
com.acme.shop.order/
├── OrderController.java        # public only if something outside needs it
├── OrderService.java
├── OrderRepository.java
├── Order.java
├── OrderStatus.java
├── CreateOrderRequest.java     # DTOs live with the feature that owns them
└── OrderNotFoundException.java
```

Java's package-private visibility is the reason this works well: with one feature per package, everything except the feature's intended API can drop the `public` modifier, and the compiler enforces the boundary. A layered layout (`controller/`, `service/`, `repository/`, `model/`) forces nearly every class to be public, so nothing stops any class from calling any other.

When a feature grows past roughly ten to fifteen classes, split it by sub-feature first. If it needs internal layers instead, use sub-packages (`order.web`, `order.domain`, `order.persistence`), accepting that classes crossing those sub-packages must become public.

Many existing Spring codebases are layered. In place mode, follow the layering that is there; don't start a feature package in the middle of a layered project.

## Files

- One top-level type per file, named for the type. The compiler requires this for public types; apply it to package-private ones too, so that a class can be found by its file name.
- Small types that exist only for one class (a builder, a private record, the permitted records of a sealed interface) are nested in that class's file. Once another class uses the type, it gets its own file.
- Class names carry the feature and the role (`OrderController`, `OrderService`, `CreateOrderRequest`), matching the suffixes the project already uses. A class that needs `Manager`, `Helper` or `Util` in its name usually has no single responsibility yet.
- A service that keeps growing is split by use case or sub-feature (`OrderCheckoutService`, `OrderRefundService`), not into `OrderService2` or a static helper class.
- `package-info.java` holds package-level documentation and annotations (nullness defaults, Spring Modulith's `@ApplicationModule`). Add one where there is something to say, not in every package.
- Generated sources (MapStruct, OpenAPI Generator, jOOQ, QueryDSL) are written under `target/generated-sources` or `build/generated` and are not committed or edited. If the project commits generated code, it stays in its own package or module.

## Cross-cutting code

Give it named packages rather than a `common` or `util` package:

- `config/` for `@Configuration` classes that belong to no single feature (security, Jackson, OpenAPI). Feature-specific configuration stays in the feature.
- `web/` or `error/` for the global `@RestControllerAdvice` and shared error responses.
- `security/` for authentication and authorization wiring.

Bind configuration with `@ConfigurationProperties` records placed beside the code that uses them, in preference to `@Value` scattered through the codebase.

## Dependency direction

- Controllers depend on services, services on repositories and domain types. Domain types don't depend on web or persistence DTOs.
- Features call each other through a service or a small facade, not through each other's repositories or entities. Where features only need to react to each other, application events avoid a direct dependency.
- Enforcement options, lightest first: package-private visibility; ArchUnit tests (layer rules, no cycles); Spring Modulith, which treats each top-level package as a module, verifies that modules only use each other's exposed API, and can generate module documentation.

## Tests

- `src/test/java` mirrors the main package structure, so tests can see package-private classes.
- Name unit tests `*Test`, run by Surefire. Name integration tests `*IT`, run by Failsafe in the `verify` phase, or give them their own source set in Gradle. Follow whichever split the project already uses.
- Prefer slice tests (`@WebMvcTest`, `@DataJpaTest`) placed in the feature's package over full `@SpringBootTest` contexts for everything.
- Shared fixtures and test configuration go under `src/test/java` in a clearly named package, and test data under `src/test/resources`.

## Resources and configuration

- `application.yml` holds defaults; `application-<profile>.yml` holds per-environment overrides. Secrets come from the environment or a secret store and are never committed.
- `static/` and `templates/` only if the service renders server-side pages.
- Database migrations are versioned files under `db/migration`, never edited after release.

## Multi-module builds

Stay single-module until there is a concrete reason to split: several deployables sharing code, a library published separately, or a boundary the team wants the build to enforce. Modules add build time and configuration, and packages plus ArchUnit or Modulith give most of the isolation at far lower cost.

When a split is justified, a common shape is:

```
shop/
├── pom.xml | settings.gradle(.kts)     # parent / root
├── shop-app/                           # the Spring Boot application, wiring only
├── shop-order/
├── shop-catalog/
└── shop-shared-kernel/                 # kept deliberately small
```

Prefer modules cut by feature or bounded context. Modules cut by layer (`-web`, `-service`, `-persistence`) reproduce the layered-package problem at build level.

## Common problems to look for in an audit

- Application class not in the root package, compensated with explicit `scanBasePackages`.
- Layered packages where every class is public and services call each other's repositories freely.
- A `util` or `common` package holding unrelated helpers, or domain logic in static utility classes.
- Entities used directly as request and response bodies across features.
- Cycles between packages (ArchUnit's `slices().should().beFreeOfCycles()` finds them).
- One `@Configuration` class holding the whole application's bean definitions.
- Service classes of a thousand lines covering several use cases, or several top-level types in one file.
- Generated sources checked into `src/main/java` beside hand-written code, or edited by hand.
- Multi-module builds where each module has one or two classes.
