# NestJS Guards

## Guard কী?

Guard একটা class, যেটা `@Injectable()` decorator দিয়ে annotate করা থাকে এবং `CanActivate` interface implement করে। Guard-এর একটাই দায়িত্ব — একটা নির্দিষ্ট request route handler পর্যন্ত পৌঁছাবে কিনা, সেটা ঠিক করা। এই সিদ্ধান্ত নেয়া হয় runtime-এ থাকা কিছু condition-এর ভিত্তিতে — যেমন permission, role, ACL ইত্যাদি। এটাকেই সাধারণত বলা হয় **authorization**।

Authorization (এবং তার সাথে সম্পর্কিত authentication) সাধারণত ঐতিহ্যবাহী Express অ্যাপ্লিকেশনে **middleware** দিয়ে হ্যান্ডেল করা হতো। Middleware authentication-এর জন্য ভালো, কারণ token validation বা request object-এ property attach করার মতো কাজগুলো কোনো নির্দিষ্ট route context-এর সাথে শক্তভাবে সম্পর্কিত না।

## Middleware বনাম Guard

Middleware স্বভাবতই "বোকা" — এটা জানে না `next()` call করার পর কোন handler execute হবে। অন্যদিকে, Guard-এর কাছে থাকে **ExecutionContext** instance, তাই Guard ঠিক জানে এরপর কী execute হতে যাচ্ছে। Exception filter, pipe, interceptor-এর মতোই Guard-ও design করা হয়েছে যাতে request/response cycle-এর ঠিক সঠিক জায়গায় processing logic বসানো যায়, declaratively। এতে code DRY এবং declarative থাকে।

## উদাহরণ

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user; // যেমন ধরো JWT থেকে এসেছে

    return !!user; // true হলে handler-এ যাবে, false হলে 403 Forbidden
  }
}
```

ব্যবহার:

```typescript
@UseGuards(AuthGuard)
@Get('profile')
getProfile() {
  return 'শুধু লগইন করা ইউজাররাই এটা দেখতে পাবে';
}
```

## Authorization Guard

আগেই বলা হয়েছে, authorization একটা দারুণ use case Guards-এর জন্য, কারণ নির্দিষ্ট route শুধু তখনই available হওয়া উচিত যখন caller-এর (সাধারণত একজন authenticated user) পর্যাপ্ত permission থাকে। এখন আমরা যে `AuthGuard` বানাবো, সেটা ধরে নেয় user আগে থেকেই authenticated (মানে request headers-এ একটা token attach করা আছে)। এটা token extract ও validate করবে, এবং extract করা তথ্য দিয়ে ঠিক করবে request এগোতে পারবে কিনা।

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
