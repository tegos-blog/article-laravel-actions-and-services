Laravel gives you a lot of freedom about where business logic goes. You can drop it in a controller, a job, an event listener, a model. Nobody stops you. And that is exactly the problem: after a year of "nobody stops you," the order flow lives in four places and you are afraid to touch any of them.

**TL;DR:** Direction decides the layer. Outside code triggers the whole operation -> **Action** (owns the transaction, has a `handle()`, may dispatch jobs). Other code calls in for one piece of work -> **Service** (calculate, validate, wrap one API; IO is fine). The one hard rule: **a Service never calls an Action and never dispatches a job or event.** Wiring is always `controller -> Action -> Services`.

In my projects I keep this under control with two layers: **Actions** and **Services**. This is an opinionated guideline based on how my team works, not a universal rule. Actions borrow the spirit of the [Command pattern](https://refactoring.guru/design-patterns/command), without the undo or queue machinery. And no, this is not DDD. It is just a practical way to organize business logic in Laravel.

Nuno Maduro, who popularized the Action pattern in Laravel, frames Actions as use-case objects, and that is roughly the spirit here. The benefit that keeps me using this: an Action runs the same whether it is triggered from HTTP, an Artisan command, or a queued job. The entry point does not leak into the logic.

## The one rule that decides Action or Service

Forget purity, forget reusability for a second. The split comes down to direction.

- An **Action** is an entry point. It is the unit that a controller, console command, job, or event handler invokes to run one complete operation. Its method is always `handle()`.
- A **Service** is a collaborator. Actions (or other Services) call it to do one focused piece of work: calculate, validate, transform, or wrap a single external call.

So:

- Outside code triggers the whole operation, you are looking at an Action.
- Other code reaches in mid-operation to get one thing done, that is a Service.

The normal wiring is `controller -> Action -> Services`. Read it in that order and the layers fall out on their own.

There is exactly one hard invariant here, and it is the one most "Actions vs Services" posts skip: **a Service never calls an Action, and never dispatches a job or event.** Services do work; they do not start workflows. In my main codebase that holds across every Service, no exceptions. If you find yourself wanting to dispatch a job from a Service, that work belongs in an Action.

![Call direction: every entry point calls into an Action; the Action calls down into Services and Repositories, and is the only layer that dispatches](https://raw.githubusercontent.com/tegos/laravel-action-and-service-guideline/refs/heads/main/assets/call-direction.png?v=2)

*Calls run one way: any entry point into an Action, the Action down into Services and Repositories. Only the Action dispatches. Nothing calls back up. The Action usually owns the transaction, though a self-contained Service invoked top-level may own its own.*

The reason is not testability - Laravel's `Queue::fake()` and `Event::fake()` make dispatching code easy to test. It is that orchestration stays legible when it lives in one layer: read an Action and you see the whole operation, including what it fires, while Services stay predictable because they only ever compute and return. So return a value from the Service and let the Action decide what to dispatch next.

This covers domain events and jobs that kick off downstream workflows. Framework-level lifecycle events - model observers, Eloquent events fired inside a model - are a separate concern and may live wherever they make sense for the app.

## Quick check

Still unsure where a class goes? Run it through these:

1. Is this a complete operation that something outside triggers (checkout, registration, import)? Action. Example: `OrderCreateAction`.
2. Is it a calculation or transformation other code asks for? Service. Example: `DeliveryScheduleService`.
3. Does it dispatch a job or event, send notifications, or own the database transaction? Action. Example: `PasswordRequestPhoneAction`.
4. Does it wrap a single external call (one API, one reader)? Service. Example: `SupplierApi`.
5. Does it only read, assemble, or transform data - no transaction, no dispatch, no notifications? That is a Service collaborator, not an Action. Put it in `app/Services/` and name it after what it builds or does: `CartItemDtoFactory` rather than `OrderCartItemDTOsFetchAction`.

Notice what is not on the list: "is it reusable." Reusability is a nice property, not a deciding factor - sub-Actions get reused too. And "does it have side effects" is intentionally absent - Services do IO all the time, and the next section explains why that is fine.

## The purity myth

A lot of guides tell you Services are pure, stateless, and side-effect free. I used to write it that way too. Then I counted the Services in my own project, and the number that are actually pure is maybe a fifth of them.

The rest hit the database, call HTTP APIs, read the cache, touch the filesystem, write logs, or call an external AI. `SupplierApi` makes HTTP requests and caches responses. `PartNameNormalizerAI` calls OpenAI. The image services write files and rows. Validators like `OrderCourierDateValidator` run queries. These are all Services, and they all do IO.

So the honest version of the rule:

- Prefer stateless Services. A pure calculator that takes input and returns output is the easiest thing in the world to test, so reach for that shape when you can.
- IO-heavy Services are completely fine. A class that wraps one external API or one cache region is still a Service.
- What makes something an Action is not "it has a side effect." It is owning the transaction, dispatching a job or event, or sending a notification.

That last point is where most "is this an Action" confusion comes from. A Service reading and writing the database is normal. A Service that dispatches `OrderShippedNotification` is in the wrong layer.

State is the same story. Most Services hold no instance state, and that is the default I aim for. But a few legitimately do: `SupplierApi` carries a `$cacheEnabled` flag you can toggle, and a log buffer accumulates entries before flushing them to the database. Prefer stateless. Do not pretend the exceptions do not exist.

## Actions

An Action wraps one complete operation with a clear start and end. Create an order. Cancel an order item. Track a search query. Each is one Action.

A few conventions I stick to:

- One operation per Action. If it is doing three unrelated things, it is three Actions.
- Name it `[Domain][Object][Verb]Action`: `OrderCreateAction`, `OrderItemCancelAction`, `SearchQueryLogAction`. Nabil Hassen makes the case for consistent naming in [_Action Pattern in Laravel_](https://nabilhassen.com/action-pattern-in-laravel-concept-benefits-best-practices). Note: the community majority uses verb-first (`CreateOrderAction`). Domain-first has one practical upside - in a directory with 30+ Action files, all order-related classes sort together alphabetically. Pick one convention and commit to it.
- Put it in `app/Actions/[Domain]`, for example `app/Actions/Order`.
- Tag it with the `Actionable` marker interface. It declares no methods. It just labels the class as an Action and documents the `handle()` entry point for the IDE. Nothing enforces the signature at compile time, the convention does.

The interface is just a marker:

```php
namespace App\Actions;

/** @method handle() */
interface Actionable
{
}
```

And an Action, simplified to three collaborators (the production one carries more):

```php
namespace App\Actions\Order;

use App\Actions\Actionable;
use App\DataTransferObjects\Order\OrderCreateDTO;
use App\Models\Order;
use App\Repositories\Order\OrderItemNotificationRepository;
use App\Repositories\OrderRepository;
use App\Services\Order\OrderCourierDateValidator;
use Illuminate\Support\Facades\DB;

final readonly class OrderCreateAction implements Actionable
{
    public function __construct(
        private OrderRepository $orderRepository,
        private OrderCourierDateValidator $courierDateValidator,
        private OrderItemNotificationRepository $notificationRepository,
    ) {}

    public function handle(OrderCreateDTO $dto, int $userId): Order
    {
        $this->courierDateValidator->validate($dto->courierDate, $dto->addressId);

        return DB::transaction(function () use ($dto, $userId) {
            $order = $this->orderRepository->create($dto, $userId);
            $this->notificationRepository->insert($order->items->toArray());

            return $order;
        });
    }
}
```

Notice `final readonly class ... implements Actionable`. The `readonly` keyword fits because the Action holds only injected dependencies and no mutable state. That is the default shape for both Actions and stateless Services. For Services that hold mutable state - a cache toggle, a log buffer - use plain `final class` instead.

What I do inside an Action:

- Inject everything through the constructor: repositories, Services, and sub-Actions.
- Own the transaction here. Wrap the write path in `DB::transaction()` so a half-finished order never lands. This short walks through the same idea: [_Action Design Pattern Explained_](https://youtube.com/shorts/wD1DAeRQ778).
- Use `handle()` as the entry point. Spatie's convention uses `execute()` instead, specifically to avoid a double-injection edge case when an Action is injected into a job's own `handle()` method. Either works; `handle()` is our team default.
- Let business exceptions bubble up. The Action throws; a global handler turns it into a clean response (more on that below).
- Return the affected resource, or nothing.

The composition is the nice part. `OrderCreateAction` leans on `CartItemDtoFactory` to assemble its cart item DTOs, and a Repository to persist. Small, testable units instead of one giant method. A class that only assembles or transforms data belongs in `app/Services/`, not `app/Actions/` - that boundary prevents Actions from quietly becoming data-access wrappers.

Here is a step I got wrong at first. I had an `OrderItemNotificationAction` doing that notification write. It only wrote rows, and nothing triggered it on its own, so it was never really an operation. It was data access mislabeled as an Action. I moved it into `OrderItemNotificationRepository`. The actual sending of those notifications lives somewhere else: a scheduled command, `OrderItemNotificationSendCommand`, which is its own entry point. Writing the rows is a Repository. Sending them is triggered by a command. One Action had quietly merged two different jobs, and the direction rule is what pulled them back apart.

## Services

A Service does one focused job that other code needs: a calculation, a validation, a single external call. If you are copy-pasting the same logic across two Actions, that is the signal to pull it into a Service.

Conventions:

- Name it `[Domain][Purpose]Service`, or end it with `Validator` or `Api` when that reads better: `OrderConditionService`, `CartItemValidator`, `SupplierApi`.
- Put it in `app/Services/[Domain]`, for example `app/Services/Cart`.
- Keep methods focused. Prefer stateless and prefer returning new values over mutating what you were handed.

A calculation Service in its cleanest form takes input and returns output:

```php
namespace App\Services\Order;

final readonly class OrderVatService
{
    public function calculate(int $netAmount, float $vatRate): int
    {
        return (int) round($netAmount * $vatRate);
    }
}
```

No instance state, no input mutation, trivially unit-testable. That is the shape to aim for when the work is pure math. Most Services will not be this clean, and that is fine.

One trap worth calling out: if a Service takes a DTO, do not mutate it in place. My own `DeliveryScheduleService` is guilty of exactly this. It sets `shippingDate` on the DTO it receives instead of returning a fresh value, so every call site has to remember that side effect happened. Return new values, and treat DTOs as read-only inside Services.

Validators are Services too, and they show the range. `CartItemValidator` is pure: it checks input against business rules and throws on failure. But `OrderCourierDateValidator` queries the database to validate. Both are Services. A validator that reads the database is still a validator.

About dependency count: I do not have a magic number. The more a class pulls in, the harder it is to follow, so more than a handful is usually a sign to split. But my own `OrderCreateAction` orchestration leans on a fair number of collaborators, and that is fine because each does one thing. Treat the count as a smell to investigate, not a hard cap to enforce.

## Repositories, DTOs, and exceptions

Three supporting pieces show up in the code above and deserve a line each.

**Repositories** own database reads and writes. Actions call them for persistence (`OrderRepository->create()`) so the transaction body stays short and the Action reads like a story instead of a query builder.

**DTOs** are the typed, immutable boundary. The controller builds one from the validated request and hands it down. Pass DTOs, not loose arrays, so the data flow is typed end to end.

**Business exceptions** carry their own rendering. Instead of try/catch in the controller, an Action throws something like `UserCheckException`, which extends a base exception that knows how to `render()` itself into a clean JSON error. A marker interface (`BusinessExceptionShouldntReport`) keeps expected, user-facing failures out of your error tracker. The controller stays clean and the framework does the rest.

## Composing Actions

Actions can call sub-Actions and Services to build a workflow. The direction is strictly one way:

- Actions may call sub-Actions and Services.
- Services may call other Services only.
- Services never call Actions.

But be strict about what earns the "sub-Action" label. A class is an Action only if it is an entry point in its own right (a controller, command, job, or event could call it), or it dispatches a job/event/notification, or it owns a transaction. A class that only filters, groups, transforms, validates, or reads - and is called only from inside another Action - is a Service mislabeled as an Action. We learned this on a search pipeline that carried `SearchBrandGroupingAction` and a stack of filter "Actions" that never dispatched, never owned a transaction, and were never called from anywhere but the orchestrating `SearchAction`. They were Services. Renaming them to `SearchBrandGroupingService`, `SearchAllowedSuppliersFilterService`, and friends made the search Action honest: it injects Services now, not fake sub-Actions.

For example, `OrderCreateAction` pulls in `CartItemDtoFactory` to assemble its cart item DTOs and `OrderConditionService` to price them, then writes notification rows through `OrderItemNotificationRepository`. One transaction per Action, no deep chains.

When an Action starts doing too much, split by operation. Instead of an overloaded `OrderProcessAction`, you get `OrderCreateAction`, `OrderPaymentProcessAction`, and `OrderInventoryUpdateAction`, each with one job. Do not go the other way and spawn an Action for every trivial task; use them for operations that mean something.

## Controllers

Keep controllers thin. Grab what you need, build the DTO, hand it to an Action, return the response.

```php
namespace App\Http\Controllers\Customer\Order;

use App\Actions\Customer\Order\OrderCreateAction;
use App\DataTransferObjects\Customer\Order\OrderCreateDTO;
use App\Http\Controllers\Controller;
use App\Http\Requests\Customer\Order\OrderStoreRequest;
use App\Http\Resources\Customer\Order\OrderResource;
use App\Repositories\User\CurrentAuthUserRepository;

final class OrderController extends Controller
{
    public function store(
        OrderStoreRequest $request,
        CurrentAuthUserRepository $currentAuthUserRepository,
        OrderCreateAction $orderCreateAction,
    ): OrderResource {
        $user = $currentAuthUserRepository->getApiCustomerUser();
        $validatedInput = $request->safe();

        $orderCreateDTO = new OrderCreateDTO(
            ip: $request->ip(),
            userAgent: $request->userAgent(),
            items: $validatedInput->input('items'),
            directionTypeId: $validatedInput->input('direction_type_id'),
        );

        $order = $orderCreateAction->handle($orderCreateDTO, $user->id);

        return OrderResource::make($order);
    }
}
```

`controller -> Action` is the default. A controller calling a Service directly is allowed only for a stateless read or validation with no orchestration. The moment more than one step is involved, that belongs in an Action.

And because the Action is context-free, the same `handle()` runs from a command:

```php
namespace App\Console\Commands;

use App\Actions\Order\OrderCreateAction;
use App\DataTransferObjects\Order\OrderCreateDTO;
use Illuminate\Console\Command;

final class OrderCreateCommand extends Command
{
    protected $signature = 'order:create {user}';

    public function handle(OrderCreateAction $orderCreateAction): int
    {
        $dto = new OrderCreateDTO(/* ... */);
        $orderCreateAction->handle($dto, (int) $this->argument('user'));

        return self::SUCCESS;
    }
}
```

Same Action, different entry point. That is the whole payoff.

## When not to bother

This is not a tax you pay on every line. For a simple read or a minimal CRUD write, an Action is overkill; a repository call or a few lines in the controller is fine. If a Service would have exactly one caller and no reuse, keep the logic in the Action instead of adding an abstraction nobody else uses. Add the layer when there is real logic or real reuse to justify it.

## Why this holds up

The payoff is consistency. Every operation looks the same: a controller hands a DTO to an Action's `handle()`, the Action orchestrates Services and sub-Actions inside one transaction, exceptions bubble to a global handler. New team members learn the shape once and then read the whole codebase. Future-you, three months out, does not have to reverse-engineer where the order logic lives.

It is opinionated and it is not DDD. It is a pragmatic setup that has kept my Laravel projects readable as they grew. Laravel, after the happy path.

{% details TL;DR %}

- **Direction decides it.** Outside code triggers the whole operation: Action. Other code calls in for one piece of work: Service. Wiring is `controller -> Action -> Services`.
- **Actions** are entry points with a `handle()` method. They own the transaction, may dispatch jobs and events, and orchestrate Services and sub-Actions. Shape: `final readonly class X implements Actionable`.
- **Services** do one focused job: calculate, validate, transform, or wrap one external call. Prefer stateless, but IO-heavy Services (API clients, validators that query) are fine.
- **The hard rule:** a Service never calls an Action and never dispatches a job or event.
- **Naming:** `[Domain][Object][Verb]Action`, `[Domain][Purpose]Service`. Keep controllers thin; pass DTOs, not arrays.
- Full version on [GitHub](https://github.com/tegos/laravel-action-and-service-guideline/blob/main/action-and-service-guidelines.md).

{% enddetails %}

## Your turn

Two things I know are contentious:

1. I name Actions **domain-first** (`OrderCreateAction`); the community majority goes **verb-first** (`CreateOrderAction`). Which do you use, and why?
2. My hard rule is that **a Service never dispatches a job or event.** Do you hold that line, or do you let Services fire events? Where does it break for you?

Drop your setup in the comments - especially if you disagree.

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).
