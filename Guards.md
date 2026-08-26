# NestJS Guards

## Guard কী?

Guard হলো `@Injectable()` decorator দিয়ে annotate করা একটা class, যেটা `CanActivate` interface implement করে।

Guard-এর কাজ একটাই: **নির্দিষ্ট condition (permission, role, ACL ইত্যাদি) অনুযায়ী runtime-এ ঠিক করা একটা request route handler পর্যন্ত পৌঁছাবে কিনা।** একেই বলে **Authorization**।

### Middleware vs Guard — পার্থক্য কোথায়?

Traditional Express app-এ authentication/authorization সাধারণত middleware দিয়ে করা হয়। Middleware authentication-এর জন্য ভালো, কারণ token validate করা বা request object-এ property attach করা — এসব কাজ কোনো নির্দিষ্ট route context-এর সাথে শক্তভাবে যুক্ত না।

কিন্তু middleware-এর একটা সমস্যা আছে — **middleware "বোকা" (dumb)।** `next()` call করার পর কোন handler চলবে, middleware সেটা জানে না।

অন্যদিকে, **Guard-এর কাছে `ExecutionContext` instance থাকে**, তাই এটা ঠিক জানে পরে কী execute হতে যাচ্ছে। Exception filter, pipe, interceptor-এর মতোই, Guard-ও request/response cycle-এর ঠিক সঠিক জায়গায় processing logic বসানোর জন্য বানানো — এবং সেটা declarative ভাবে।

> **Hint:** Execution order মনে রাখা জরুরি — **সব middleware চলার পর Guard চলে, কিন্তু কোনো interceptor বা pipe চলার আগে।**

---

## Authorization Guard বানানো

যেহেতু নির্দিষ্ট route শুধু নির্দিষ্ট permission থাকা user-এর জন্যই available হওয়া উচিত, Authorization-এর জন্য Guard একদম উপযুক্ত।

একটা basic `AuthGuard` — যেটা ধরে নেয় user authenticated (মানে request header-এ token attached আছে), token extract করে validate করবে, আর সেই তথ্য দিয়ে ঠিক করবে request আগাবে কিনা:

```ts
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

`validateRequest()` function-এর ভিতরের logic যত simple বা যত complex দরকার তত হতে পারে — এই example-এর মূল বিষয় হলো Guard কীভাবে request/response cycle-এ ফিট হয় সেটা দেখানো।

### `canActivate()` — Guard-এর মূল method

প্রতিটা Guard-কেই একটা `canActivate()` function implement করতে হয়। এটা একটা `boolean` return করে (sync বা async — Promise/Observable দিয়েও হতে পারে), যেটা দিয়ে বোঝা যায় বর্তমান request allow করা হবে কিনা:

- **`true`** return করলে → request process হবে
- **`false`** return করলে → Nest request deny করে দেবে

---

## Execution Context

`canActivate()` একটাই argument নেয় — `ExecutionContext` instance। এটা `ArgumentsHost`-কে inherit করে (Exception Filters chapter-এ যেটা আগে দেখানো হয়েছিল)। উপরের উদাহরণে `Request` object পেতে সেই একই helper method ব্যবহার করা হয়েছে।

`ArgumentsHost`-কে extend করার কারণে `ExecutionContext`-এ আরও কিছু নতুন helper method আছে, যেগুলো current execution process সম্পর্কে extra তথ্য দেয় — এগুলো দিয়ে অনেক controller/method/context জুড়ে কাজ করা যায় এমন generic Guard বানানো সহজ হয়।

---

## Role-based Authentication

এবার একটা বেশি কার্যকর Guard বানানো যাক, যেটা শুধু নির্দিষ্ট role-এর user-কেই access দেবে। শুরুতে একটা basic template — যেটা এখনও সব request-ই allow করে:

```ts
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

## Guard কোথায় Apply করা যায় (Binding Guards)

Pipe আর Exception filter-এর মতোই, Guard-ও তিন স্তরে scope করা যায়: **controller-scoped, method-scoped, বা global-scoped।**

### Controller-scoped

```ts
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> **Hint:** `@UseGuards()` decorator আসে `@nestjs/common` package থেকে।

এখানে class পাঠানো হয়েছে (instance না) — instantiation-এর দায়িত্ব framework-এর, আর এতে DI-ও কাজ করে। চাইলে সরাসরি instance-ও দেওয়া যায়:

```ts
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

এভাবে controller-এর ভিতরের **সবগুলো** handler-এই Guard apply হয়ে যায়।

### Method-scoped

শুধু একটা নির্দিষ্ট method-এ apply করতে চাইলে, `@UseGuards()` সেই method-এর level-এই বসাতে হবে।

### Global-scoped

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> **Notice:** Hybrid app-এর ক্ষেত্রে `useGlobalGuards()` by default gateway আর microservice-এ Guard setup করে না। তবে "standard" (non-hybrid) microservice app-এ এটা ঠিকই global ভাবে mount হয়।

**সমস্যা:** `useGlobalGuards()`-এর মাধ্যমে (কোনো module-এর বাইরে থেকে) register করা global Guard-এ DI দিয়ে dependency inject করা যায় না।

**সমাধান:** `APP_GUARD` token ব্যবহার করে module-এর ভিতর থেকে global Guard register করা:

```ts
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

> **Hint:** এভাবে register করলে Guard যেই module-এই লেখা হোক, সেটা আসলে global-ই থাকে। যে module-এ Guard define করা আছে (এখানে `RolesGuard`), সেখানেই এটা করা ভালো।

---

## প্রতিটা Handler-এ আলাদা Role সেট করা

`RolesGuard` কাজ করছে ঠিকই, কিন্তু এখনও "স্মার্ট" না — এটা এখনও জানে না কোন route-এ কোন role লাগবে। যেমন, `CatsController`-এর বিভিন্ন route-এ বিভিন্ন permission scheme থাকতে পারে — কিছু route শুধু admin-এর জন্য, কিছু সবার জন্য open।

এই কাজে আসে **custom metadata**। Nest দুইভাবে handler-এ custom metadata attach করতে দেয়:
1. `Reflector.createDecorator` static method দিয়ে বানানো decorator
2. built-in `@SetMetadata()` decorator

### `Reflector.createDecorator` দিয়ে `@Roles()` decorator বানানো

```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

`Roles` decorator এখানে একটা function, যেটা `string[]` type-এর একটা argument নেয়।

এখন handler-এ এটা ব্যবহার করা যায়:

```ts
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

এখানে `create()` method-এ `Roles` metadata attach করা হলো — মানে শুধু `admin` role-এর user-ই এই route access করতে পারবে।

> `Reflector.createDecorator`-এর বদলে built-in `@SetMetadata()` decorator দিয়েও একই কাজ করা যায়।

---

## সবকিছু একসাথে জোড়া লাগানো

এখন `RolesGuard`-কে আসল কাজে লাগানো যাক। এখন পর্যন্ত এটা সবসময় `true` return করে। আমরা চাই — current user-এর role আর route-এ required role compare করে সিদ্ধান্ত নেওয়া হোক। এর জন্য আবার `Reflector` helper class ব্যবহার করা হবে:

```ts
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

> **Hint:** Node.js-এর জগতে authenticated user-কে `request` object-এ attach করে রাখাটা common practice। তাই ধরে নেওয়া হয়েছে `request.user`-এ user instance আর তার allowed role আছে। এই association সাধারণত তোমার নিজের authentication Guard বা middleware-এই করা হয়।

> **Warning:** `matchRoles()` function-এর ভিতরের logic যত simple বা sophisticated দরকার হয় তত হতে পারে। মূল বিষয় হলো Guard কীভাবে request/response cycle-এ ফিট হয় সেটা বোঝানো।

---

## Guard False দিলে কী হয়?

কোনো user-এর permission না থাকলে Nest automatic এই response পাঠায়:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

ভেতরের ঘটনা হলো — Guard `false` return করলে framework নিজে থেকেই একটা `ForbiddenException` throw করে।

যদি ভিন্ন কোনো error response দিতে চাও, নিজে থেকেই নির্দিষ্ট exception throw করতে হবে:

```ts
throw new UnauthorizedException();
```

Guard থেকে throw করা যেকোনো exception exceptions layer-ই handle করবে (global exception filter এবং current context-এ apply করা যেকোনো exception filter)।

---
