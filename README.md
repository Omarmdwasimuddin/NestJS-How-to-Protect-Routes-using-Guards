## How to Protect Routes using Guards


### Create guard
```bash
nest g guard [name]
```
### Create guard with folder
```bash
nest g guard guards/auth
```
<img width="233" height="68" alt="image" src="https://github.com/user-attachments/assets/6b8463e5-9e42-4756-ad1e-571c557a25ee" />

---


### `auth.guard.ts`
```bash
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    
    const request = context.switchToHttp().getRequest();
    const authHeader = request.headers.authorization;

    return authHeader === 'Bearer my-secret-token'
  }
}
```
---


### ``
```bash

```
---
