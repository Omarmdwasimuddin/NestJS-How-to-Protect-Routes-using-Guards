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


### `product.controller.ts`
> Note: product path e AuthGuard setup
```bash
import { Controller, Param, Get, UseGuards } from '@nestjs/common';
import { ProductService } from './product.service';
import { AuthGuard } from 'src/guards/auth/auth.guard';

@Controller('product')
export class ProductController {
    constructor(private readonly productService: ProductService){}
        @Get()
        @UseGuards(AuthGuard)
        getProducts(){
            return this.productService.getAllProducts();
        }
        @Get(':id')
        getProduct(@Param('id') id:string){
            return this.productService.getProductById(Number(id))
        }
    
}
```
---


> Note: Headers e token set na kore dile value show hobe na
> 
> <img width="518" height="124" alt="image" src="https://github.com/user-attachments/assets/18b289e5-e5db-4ea2-bd23-39d59186f176" />

##

> Note: Headers e token set kore dite hobe tai Headers e Authorization e Bearer my-secret-token diye dibo
>
> <img width="898" height="641" alt="image" src="https://github.com/user-attachments/assets/71d7bd7f-5a74-497e-9a41-213872ad7d28" />
