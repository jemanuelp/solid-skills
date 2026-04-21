# SOLID in NestJS (Practical Guide)

## Why this document exists

Use this guide when implementing SOLID in real NestJS projects. The goal is to apply principles pragmatically, with low coupling and high cohesion, without overengineering.

## Cohesion and Coupling First

Before applying any principle, check:

- **High cohesion**: each class/module has a clear and focused purpose.
- **Low coupling**: changes in one part should not force changes in many unrelated parts.

If SOLID usage increases accidental complexity without clear payoff, prefer the simpler design.

## NestJS Mapping for SOLID

### SRP (Single Responsibility)

- Controllers: transport concerns only (HTTP mapping, status codes, DTO in/out).
- Use cases / application services: orchestration and business flow.
- Domain services/entities: business rules.
- Infrastructure adapters: DB/API/framework details.

```typescript
// users.controller.ts
@Controller('users')
export class UsersController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.createUser.execute(dto);
  }
}
```

### OCP (Open/Closed)

Prefer extension via strategy/polymorphism instead of `if/else` chains.

```typescript
export interface PaymentStrategy {
  supports(method: PaymentMethod): boolean;
  charge(command: ChargePaymentCommand): Promise<void>;
}

@Injectable()
export class PaymentService {
  constructor(private readonly strategies: PaymentStrategy[]) {}

  async charge(command: ChargePaymentCommand): Promise<void> {
    const strategy = this.strategies.find((s) => s.supports(command.method));
    if (!strategy) throw new Error('Unsupported payment method');
    await strategy.charge(command);
  }
}
```

### LSP (Liskov Substitution)

Enforce explicit contracts for ports and ensure implementations keep behavior consistent.

- Same error semantics for all implementations of a port.
- Same nullability/return contracts.
- No hidden side effects in one adapter that others do not have.

### ISP (Interface Segregation)

Use small, capability-based ports.

```typescript
export interface UserReader {
  findById(id: UserId): Promise<User | null>;
}

export interface UserWriter {
  save(user: User): Promise<void>;
}
```

This avoids forcing consumers to depend on methods they do not need.

### DIP (Dependency Inversion)

Depend on abstractions (ports), bind implementations in modules.

```typescript
// user.tokens.ts
export const USER_REPOSITORY = Symbol('USER_REPOSITORY');

export interface UserRepository {
  save(user: User): Promise<void>;
}

// users.module.ts
@Module({
  providers: [
    CreateUserUseCase,
    { provide: USER_REPOSITORY, useClass: PostgresUserRepository },
  ],
})
export class UsersModule {}
```

Use tokens/symbols for runtime-safe DI boundaries.

## Recommended Feature Structure

```text
src/
  users/
    application/
      use-cases/
      ports/
    domain/
      entities/
      value-objects/
      services/
    infrastructure/
      persistence/
      http/
    users.module.ts
    users.controller.ts
```

This keeps feature-level vertical slices while preserving clean boundaries.

## Testing Strategy in NestJS + SOLID

- Unit test use cases with fake/in-memory ports.
- Integration test adapters (e.g., repository + real DB test container).
- E2E test only critical user flows.
- Contract-test shared ports when multiple adapters exist.

## Pragmatic Rules

- SOLID is a heuristic, not dogma.
- Avoid creating interfaces for classes with only one stable implementation unless you have a real extension/testing need.
- Introduce abstractions when you detect volatility, multiple implementations, or difficult testing seams.
- Prefer explicitness over cleverness in DI setup and module wiring.
