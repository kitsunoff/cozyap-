# Platform Vision — Product Owner Constraints

## Status

Product owner input, 2026-09-07. A durable design input alongside
`5.14-developer-platform.md`. Where requirement 5.14 says *what the customer
buys*, this document says *what the platform must be*.

## The eight constraints

### V1 — An application is a set of bricks

An application is not a monolithic template. It is a composition of building
blocks: a git repository, a build, an S3 bucket, a deployment into a given
Kubernetes cluster. Each brick is independently defined, versioned and
replaceable.

### V2 — Bricks are delivered as packages

Bricks reach an installation through `Package` resources handled by
`cozystack-operator`, not through a bespoke distribution channel.

### V3 — A package may carry anything a brick needs

Operators, Crossplane providers, Crossplane compositions, and UI plugins — the
latter potentially as a custom resource pointing at an image that serves a
Module Federation bundle.

### V4 — The model must express very different applications

A Java service, a WordPress install, and a CS2 game server must all be
expressible. The model may not be tuned to twelve-factor web applications
alone.

### V5 — The presentation layer is a set of capabilities

An application in the UI is an aggregate: resources joined by reference, label
selector or annotation, plus capabilities contributed by bricks — a link to
Grafana, a link to documentation, deep SCM integration, resources performing
one-time or periodic work, and pluggable modules such as a documentation
generator, an OpenAPI spec fetcher, or CI/CD generation so that the platform
later observes the produced image and its updates (GitLab CI, GitHub Actions,
or eventually a built-in CI engine).

### V6 — Data planes stay clean

Clusters where applications run must carry minimal platform machinery by
default. Anything additionally required is installed by the platform, on
demand, and removed when no longer needed.

### V7 — Cozystack is one provider, not the platform

Cozystack is one way to deploy and to obtain resources. The platform must not
assume it.

### V8 — Cozystack is the first delivery vehicle

Initially, within Cozystack, a user orders a platform instance and adds their
child tenants as users in separate namespaces.

## Tension with the recorded ADRs

**V1 versus ADR-0019.** ADR-0019 draws the boundary "core is an own
controller, dependencies are Crossplane compositions". V1 says *everything* is
a brick, build and deployment included. Taken literally that would put the
core into compositions and re-import their weakness on status and revisions.

The resolution is recorded in `docs/design/brick-model.md`: the uniform thing
is the **contract** — typed ports, readiness, status, billing metadata — not
the runtime. A brick declares an implementation kind, and platform-native
controllers remain one of those kinds. The boundary of ADR-0019 survives as an
implementation detail of specific brick types rather than as a split in the
model.

**V2/V3 versus ADR-0020.** ADR-0020 specifies an own OCI bundle format because
a Crossplane Configuration package cannot carry UI, dashboards, RBAC or billing
metadata. V2 says the vehicle is the Cozystack `Package`. These are compatible
and ADR-0020 needs a correction: the **content model** stays ours, the
**distribution mechanism** is the existing Cozystack `Package` /
`PackageSource` pair rather than a newly invented one. V7 then requires that
the content model itself carry no Cozystack assumptions, so that a non-Cozystack
installer can consume the same artefact.

**V3 versus ADR-0020 §4.** ADR-0020 rejected Module Federation frontend plugins
(level 2) in favour of declarative view descriptors (level 1). V3 asks for
level 2 explicitly. Revised position in `docs/design/brick-model.md`: level 1
remains the default and covers most bricks; level 2 is available as a declared
escape hatch, with its operational cost accepted knowingly.

## Appendix — original text

> 1) Приложение является набором кирпичиков (то есть приложение в этом случае это гит репо, сборка, s3 bucket, деплоймент в этот куб кластер)
> 2) мы эти кирпичики будем поставлять через packages в cozystack-operator
> 3) в package могут поставляться операторы, провайдеры кроссплейна, композиции, плагины к ui мб ввиде кастом ресурса с image который сервит module federation
> 4) можно было гибко описать различные приложения, как и java app, так и условно wordpress или вообще поднимать cs2 в кластерах
> 5) приложение в слое отображение будет это набор capabilities или набор ресурсов с рефом или лейбл селектором или аннотацией что это относится к приложению, например ссылка на графану или на документацию, или например — интеграция с SCMкой глубокая, например ресурсы которые выполняют one time job или переодическую работу или подключаемые модули — генератор документации, фетч openapi spec, генерацию ci/cd чтобы платформа в будущем увидела образ и его обновления (так и gitlab ci, gh actions или может быть даже встроенный ci движок)
> 6) датаплейны где будут крутиться приложения по умолчанию я бы хотел чтобы они имели минимальную загаженность, если надо — это все доустанавливает платформа
> 7) не обязательно платформа козистек онли, это один из провайдеров как деплоить или ресурсы получать
> 8) в начале в рамках козистека пользователь сможет заказать платформу инстанс, и добавить свой child tenant как юзеров в отдельных неймспейсах
