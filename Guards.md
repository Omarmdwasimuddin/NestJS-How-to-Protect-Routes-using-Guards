# NestJS Guards — সম্পূর্ণ গাইড

## Guard কী?

Guard একটা class, যেটা `@Injectable()` decorator দিয়ে annotate করা থাকে এবং `CanActivate` interface implement করে। Guard-এর একটাই দায়িত্ব — একটা নির্দিষ্ট request route handler পর্যন্ত পৌঁছাবে কিনা, সেটা ঠিক করা। এই সিদ্ধান্ত নেয়া হয় runtime-এ থাকা কিছু condition-এর ভিত্তিতে — যেমন permission, role, ACL ইত্যাদি। এটাকেই সাধারণত বলা হয় **authorization**।

Authorization (এবং তার সাথে সম্পর্কিত authentication) সাধারণত ঐতিহ্যবাহী Express অ্যাপ্লিকেশনে **middleware** দিয়ে হ্যান্ডেল করা হতো। Middleware authentication-এর জন্য ভালো, কারণ token validation বা request object-এ property attach করার মতো কাজগুলো কোনো নির্দিষ্ট route context-এর সাথে শক্তভাবে সম্পর্কিত না।

Middleware স্বভাবতই "বোকা" — এটা জানে না `next()` call করার পর কোন handler execute হবে। অন্যদিকে, Guard-এর কাছে থাকে **ExecutionContext** instance, তাই Guard ঠিক জানে এরপর কী execute হতে যাচ্ছে। Exception filter, pipe, interceptor-এর মতোই Guard-ও design করা হয়েছে যাতে request/response cycle-এর ঠিক সঠিক জায়গায় processing logic বসানো যায়, declaratively। এতে code DRY এবং declarative থাকে।

> **হিন্ট:** Guard সব middleware-এর পরে execute হয়, কিন্তু যেকোনো interceptor বা pipe-এর আগে।

---

## Authorization Guard

Authorization হলো Guard-এর জন্য দারুণ একটা use case, কারণ নির্দিষ্ট route শুধু তখনই available হওয়া উচিত যখন caller-এর (সাধারণত একজন authenticated user) পর্যাপ্ত permission থাকে। এখন আমরা যে `AuthGuard` বানাবো, সেটা ধরে নেয় user আগে থেকেই authenticated (মানে request headers-এ একটা token attach করা আছে)। এটা token extract ও validate করবে, এবং extract করা তথ্য দিয়ে ঠিক করবে request এগোতে পারবে কিনা।

**auth.guard.ts**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

> **হিন্ট:** বাস্তব দুনিয়ার authentication mechanism implement করার উদাহরণ দেখতে [Authentication](https://docs.nestjs.com/security/authentication) chapter দেখো। আর একটু জটিল authorization example-এর জন্য [Authorization](https://docs.nestjs.com/security/authorization) পেজ দেখো।

`validateRequest()` function-এর ভেতরের logic যত সহজ বা জটিল দরকার, তত-ই হতে পারে। এই উদাহরণের মূল উদ্দেশ্য হলো Guard কীভাবে request/response cycle-এ fit করে সেটা দেখানো।

প্রতিটা Guard-কে অবশ্যই `canActivate()` function implement করতে হয়। এই function একটা boolean return করে, যেটা বলে দেয় বর্তমান request allow করা হবে কিনা। এটা synchronously অথবা asynchronously (`Promise` বা `Observable`-এর মাধ্যমে) response return করতে পারে। Nest এই return value দিয়ে পরবর্তী action ঠিক করে:

- `true` return করলে → request process হবে।
- `false` return করলে → Nest request deny করবে।

---

## Execution Context

`canActivate()` function একটা মাত্র argument নেয় — `ExecutionContext` instance। `ExecutionContext`-এর inheritance আসে `ArgumentsHost` থেকে। `ArgumentsHost` আমরা আগে exception filters chapter-এ দেখেছি। উপরের উদাহরণে, আমরা `ArgumentsHost`-এ define করা একই helper method ব্যবহার করছি (যেগুলো আগেও ব্যবহার করেছিলাম) `Request` object-এর reference পাওয়ার জন্য। এই বিষয়ে আরও জানতে [exception filters](https://docs.nestjs.com/exception-filters#arguments-host) chapter-এর Arguments host section দেখতে পারো।

`ArgumentsHost`-কে extend করার মাধ্যমে, `ExecutionContext` আরও কিছু নতুন helper method যোগ করে, যেগুলো current execution process সম্পর্কে অতিরিক্ত তথ্য দেয়। এই তথ্যগুলো এমন generic Guard বানাতে সাহায্য করে যা বিভিন্ন controller, method এবং execution context জুড়ে কাজ করতে পারে। `ExecutionContext` সম্পর্কে আরও জানতে [এখানে](https://docs.nestjs.com/fundamentals/execution-context) দেখো।

---

## Role-based Authentication

চলো এমন একটা বেশি functional Guard বানাই, যেটা শুধু নির্দিষ্ট role থাকা user-দের access permit করে। আমরা শুরু করবো একটা basic Guard template দিয়ে, এবং পরের section-গুলোতে এটার উপর ভিত্তি করে আরও build করবো। আপাতত, এটা সব request-কে proceed করার অনুমতি দেয়:

**roles.guard.ts**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

---

## Binding Guards

Pipe এবং exception filter-এর মতোই, Guard-ও controller-scoped, method-scoped, অথবা global-scoped হতে পারে। নিচে, আমরা `@UseGuards()` decorator ব্যবহার করে একটা controller-scoped Guard সেট আপ করছি। এই decorator একটা single argument, বা comma দিয়ে আলাদা করা argument-এর list নিতে পারে। এতে এক declaration দিয়েই যথাযথ Guard-এর সেট সহজে apply করা যায়।

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> **হিন্ট:** `@UseGuards()` decorator টা `@nestjs/common` package থেকে import করা হয়।

উপরে, আমরা `RolesGuard` class-টা পাস করেছি (instance না), যাতে instantiation-এর দায়িত্ব framework-এর কাছে থাকে এবং dependency injection সম্ভব হয়। Pipe আর exception filter-এর মতোই, আমরা চাইলে একটা in-place instance-ও পাস করতে পারি:

```typescript
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

উপরের এই গঠনটা এই controller-এ declare করা প্রতিটা handler-এর সাথে Guard-টা attach করে দেয়। যদি আমরা চাই Guard শুধু একটা নির্দিষ্ট method-এর জন্যই apply হোক, তাহলে `@UseGuards()` decorator method level-এ apply করতে হবে।

একটা global guard সেট আপ করার জন্য, Nest application instance-এর `useGlobalGuards()` method ব্যবহার করো:

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> **নোটিস:** Hybrid app-এর ক্ষেত্রে `useGlobalGuards()` method by default gateway এবং microservice-এর জন্য Guard সেট আপ করে না (এই behavior পরিবর্তন করার তথ্যের জন্য Hybrid application দেখো)। "স্ট্যান্ডার্ড" (non-hybrid) microservice app-এর ক্ষেত্রে, `useGlobalGuards()` Guard-গুলো globally mount করে।

Global guard পুরো অ্যাপ্লিকেশন জুড়ে ব্যবহৃত হয় — প্রতিটা controller এবং প্রতিটা route handler-এর জন্য। Dependency injection-এর দিক থেকে, কোনো module-এর বাইরে থেকে register করা global guard (উপরের উদাহরণের মতো `useGlobalGuards()` দিয়ে) dependency inject করতে পারে না, কারণ এটা কোনো module-এর context-এর বাইরে করা হয়। এই সমস্যা সমাধান করতে, তুমি নিচের গঠন ব্যবহার করে যেকোনো module থেকে সরাসরি একটা Guard সেট আপ করতে পারো:

**app.module.ts**

```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

> **হিন্ট:** Guard-এর জন্য dependency injection করতে এই approach ব্যবহার করার সময় খেয়াল রেখো, এই গঠনটা যে module-এই ব্যবহার করা হোক না কেন, Guard-টা আসলে global-ই থাকে। এটা কোথায় করা উচিত? সেই module বেছে নাও যেখানে Guard-টা (উপরের উদাহরণে `RolesGuard`) define করা আছে। আর, custom provider registration handle করার একমাত্র উপায় `useClass` না। আরও জানতে [এখানে](https://docs.nestjs.com/fundamentals/custom-providers) দেখো।

---

## Setting Roles Per Handler

আমাদের `RolesGuard` কাজ করছে, কিন্তু এটা এখনো খুব smart না। আমরা এখনো Guard-এর সবচেয়ে গুরুত্বপূর্ণ feature-এর সুবিধা নিচ্ছি না — execution context। এটা এখনো role সম্পর্কে জানে না, বা কোন handler-এর জন্য কোন role allowed। উদাহরণস্বরূপ, `CatsController`-এর বিভিন্ন route-এর জন্য বিভিন্ন permission scheme থাকতে পারে। কিছু হয়তো শুধু admin user-এর জন্যই available, আবার কিছু সবার জন্য open হতে পারে। কীভাবে আমরা flexible এবং reusable উপায়ে role-কে route-এর সাথে match করবো?

এখানেই কাজে লাগে **custom metadata** (আরও জানতে [এখানে](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) দেখো)। Nest, route handler-এ custom metadata attach করার সুবিধা দেয়, হয় `Reflector.createDecorator` static method দিয়ে তৈরি decorator-এর মাধ্যমে, নয়তো built-in `@SetMetadata()` decorator দিয়ে।

উদাহরণস্বরূপ, চলো `Reflector.createDecorator` method ব্যবহার করে একটা `@Roles()` decorator বানাই, যেটা handler-এ metadata attach করবে। `Reflector` framework থেকেই out of the box পাওয়া যায় এবং `@nestjs/core` package থেকে expose করা হয়।

**roles.decorator.ts**

```typescript
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

এখানে `Roles` decorator একটা function, যেটা `string[]` টাইপের একটা মাত্র argument নেয়।

এখন, এই decorator ব্যবহার করতে, আমরা শুধু handler-টাকে এটা দিয়ে annotate করবো:

**cats.controller.ts**

```typescript
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

এখানে আমরা `Roles` decorator metadata `create()` method-এর সাথে attach করেছি, যেটা নির্দেশ করে যে শুধু `admin` role-এর user-রাই এই route access করতে পারবে।

বিকল্প হিসেবে, `Reflector.createDecorator` method ব্যবহার না করে, আমরা built-in `@SetMetadata()` decorator ব্যবহার করতে পারতাম। আরও জানতে [এখানে](https://docs.nestjs.com/fundamentals/execution-context#low-level-approach) দেখো।

---

## Putting It All Together

এবার চলো ফিরে গিয়ে এটাকে আমাদের `RolesGuard`-এর সাথে জুড়ে দিই। এই মুহূর্তে, এটা সবসময় `true` return করে, ফলে প্রতিটা request proceed করার অনুমতি পায়। আমরা চাই return value যেন conditional হয় — বর্তমান user-কে assign করা role-গুলোর সাথে বর্তমান route-এর জন্য প্রয়োজনীয় actual role-গুলো compare করে। Route-এর role(s) (custom metadata) access করার জন্য, আমরা আবার `Reflector` helper class ব্যবহার করবো, এভাবে:

**roles.guard.ts**

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

> **হিন্ট:** node.js জগতে, authorized user-কে request object-এর সাথে attach করা একটা common practice। তাই, উপরের sample code-এ, আমরা ধরে নিচ্ছি `request.user`-এ user instance এবং allowed role-গুলো থাকে। তোমার app-এ, তুমি সম্ভবত এই association তোমার custom authentication guard (বা middleware)-এ তৈরি করবে। এই বিষয়ে আরও তথ্যের জন্য [এই chapter](https://docs.nestjs.com/security/authentication) দেখো।

> **সতর্কতা:** `matchRoles()` function-এর ভেতরের logic যত সহজ বা জটিল দরকার, তত-ই হতে পারে। এই উদাহরণের মূল উদ্দেশ্য হলো Guard কীভাবে request/response cycle-এ fit করে সেটা দেখানো।

`Reflector` কে context-sensitive উপায়ে ব্যবহার করার আরও বিস্তারিত জানতে Execution context chapter-এর Reflection and metadata section দেখো।

যখন কোনো user-এর যথেষ্ট privilege না থাকে এবং সে কোনো endpoint request করে, Nest automatically নিচের response return করে:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

খেয়াল রাখো, পর্দার আড়ালে, যখন কোনো Guard `false` return করে, তখন framework একটা `ForbiddenException` throw করে। যদি তুমি ভিন্ন কোনো error response return করতে চাও, তাহলে তোমার নিজের specific exception throw করা উচিত। উদাহরণস্বরূপ:

```typescript
throw new UnauthorizedException();
```

Guard-এর থ্রো করা যেকোনো exception, exceptions layer (global exceptions filter এবং current context-এ apply করা যেকোনো exceptions filter) দিয়ে handle হবে।

---

## সংক্ষেপে (সব একসাথে)

- **Guard** হলো এমন class যা `CanActivate` implement করে এবং `canActivate()` method-এ `true`/`false` return করে request allow/deny করার জন্য — এটা মূলত **authorization**-এর জন্য ব্যবহৃত হয়।
- **AuthGuard** — token validate করে user authenticated কিনা চেক করে।
- **ExecutionContext** — `ArgumentsHost`-এর extended version, current execution সম্পর্কে বাড়তি তথ্য দেয়, generic Guard বানাতে সাহায্য করে।
- **RolesGuard** — প্রথমে সবসময় `true` return করে, পরে role-based logic যোগ করা হয়।
- **`@UseGuards()`** দিয়ে Guard controller-level বা method-level-এ bind করা যায়; `useGlobalGuards()` বা `APP_GUARD` provider দিয়ে global-ভাবে bind করা যায়।
- **Custom metadata** (`Reflector.createDecorator` বা `@SetMetadata()`) দিয়ে route handler-এ role-এর মতো তথ্য attach করা যায়, যেটা পরে Guard `Reflector` দিয়ে পড়ে নিয়ে decision নেয়।
- Access না থাকলে Nest by default `403 Forbidden` (`ForbiddenException`) return করে; চাইলে custom exception (যেমন `UnauthorizedException`) throw করে ভিন্ন response দেওয়া যায়।

### কাজ করার প্র্যাকটিক্যাল ফ্লো

1. একটা `AuthGuard` বানাও যেটা token validate করে user authenticated কিনা নিশ্চিত করে।
2. একটা `@Roles()` custom decorator বানাও, যেটা দিয়ে handler-এ প্রয়োজনীয় role attach করবে।
3. একটা `RolesGuard` বানাও, যেটা `Reflector` দিয়ে সেই metadata পড়বে এবং `request.user.roles`-এর সাথে compare করবে।
4. `@UseGuards()` দিয়ে controller বা method-এ Guard bind করো, অথবা পুরো app-এর জন্য `APP_GUARD` provider ব্যবহার করো।
5. Access না মিললে Nest নিজে থেকেই `403 Forbidden` return করবে — দরকার হলে নিজের exception throw করে response customize করো।
