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
