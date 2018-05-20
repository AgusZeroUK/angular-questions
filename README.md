# Questions

#### 1. What would you use for angular application to communicate with backend services over the HTTP protocol.

I would use the **HttpClient** API introduced in the version 4.3 replacing and simplifying the existing Angular Http API.

Example of http request:

_app.module.ts_
```typescript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';

import { AppComponent } from './app.component';

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

**HttpClientModule** is the module we import in our application module.

_app.component.ts_
```typescript
import { Component, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
})
export class AppComponent implements OnInit {
  public data: string;

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.loadData();
  }

  public loadData() {
    this.http.get<string>('http://example-api-url/data')
      .subscribe((response: string) => {
        this.data = response;
      },
      (err) => {
        console.log(err);
      });
  }
}
  
```

We inject **HttpClient** service in our component to handle http requests.

#### 2. You want to redirect a user to a login page when any API request returns 401 status code. What would you use to handle such case globally?

I would create a *interceptor* which implements **HttpInterceptor** interface to handle globally unauthorized responses. 

_auth.interceptor.ts_
```typescript
export class AuthInterceptor implements HttpInterceptor {
  constructor(private router: Router) {}
  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    
    return next.handle(request).do((event: HttpEvent<any>) => {
      if (event instanceof HttpResponse) {
        // In case we want to handle responses
      }
    }, (err: any) => {
      if (err instanceof HttpErrorResponse) {
        if (err.status === 401) {
          this.router.navigate(['login']);
        }
      }
    });
  }
}

```

As side-effect, the interceptor needs to provided to the `HTTP_INTERCEPTORS` array for our application on `AppModule`.

_app.module.ts_
```typescript
import { HTTP_INTERCEPTORS, HttpClientModule,  } from '@angular/common/http';
import { AuthInterceptor } from './auth.interceptor.ts';
 
@NgModule({
  bootstrap: [AppComponent],
  imports: [
    HttpClientModule
  ],
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: AuthInterceptor,
      multi: true
    }
  ]
})

export class AppModule {}

```

#### 3. What for are the route guards in Angular?.

Router Guard is another way of handle authentication. It prevent users from accessing areas that they’re not allowed.

Here is one example resolving the question 2 preventing unauthenticated users and redirecting to login page:

_auth.service.ts_
```typescript
import { Injectable } from '@angular/core';
import { helperTokenService } from 'some-authentication-helper';
@Injectable()
export class AuthService {
  constructor(public helper: helperTokenService) {}
  // ...
  public isAuthenticated(): boolean {
    const token = localStorage.getItem('token');
    return !this.helper.isTokenExpired(token);
  }
}
```

`AuthService` provides the method to check if user is authenticated through `isAuthenticated()`. It checks if the authentication token is not expired.

_auth-guard.service.ts_
```typescript
import { Injectable } from '@angular/core';
import { Router, CanActivate } from '@angular/router';
import { AuthService } from './auth.service';
@Injectable()
export class AuthGuardService implements CanActivate {
  constructor(public auth: AuthService, public router: Router) {}
  canActivate(): boolean {
    if (!this.auth.isAuthenticated()) {
      this.router.navigate(['login']);
      return false;
    }
    return true;
  }
}
```

There are several types of guards, but the one I use is the `canActivate` type which is run before you navigate to a route.

Router guard tell the router whether or not it should allow navigation to a requested route returning `true` or `false` value.

#### 4. Angular component has subscribed to several observers. Why, when and how do you unsubscribe from those subscriptions.

__Why__ 

A subscription is created when we `subscribe()` to an observable. we need to unsubscribe from any subscription to avoid memory leaks.

__When__ 

In the most cases we unsubscribe when the component is destroyed with the `ngOnDestroy()` lifecycle method.


__How__

* 3 ways of unsubscribe:
   1. Store the subscription and call `unsubscribe()` method when the component is destroyed.
   
      ```typescript
          import { Component, OnInit, OnDestroy } from '@angular/core';
          import { Subscription } from 'rxjs/Subscription';
          
          @Component({
            selector: 'app-root',
          })
          export class AppComponent implements OnInit, OnDestroy {
            public data: any;
            public data2: any;
            private dataSubscription: Subscription;
            private data2Subscription: Subscription;
            
            ngOnInit() {   
              this.dataSubscription = someServiceMethodReturningObservable()
                .subscribe(response => this.data = response);
        
              this.data2Subscription = someServiceMethodReturningObservable()
                .subscribe(response => this.data2 = response);
            }
      
            ngOnDestroy() {
              this.dataSubscription.unsubscribe();
              this.data2Subscription.unsubscribe();
            }
          }
      ```
   2. We can use the **Async Pipe** that subscribes to an Observable and returns the latest value it has emmited. it also unsubscribes automatically when the component is destroyed.
      ```typescript
          import { Component, OnInit } from '@angular/core';
          import { Observable } from 'rxjs/Observable';
          
          @Component({
            selector: 'app-root',
            template: `
            <p>{{data | async}}</p>
            <p>{{data2 | async}}</p>
          `
          })
          export class AppComponent implements OnInit {
            public data: Observable<any>;
            public data2: Observable<any>;
            
            ngOnInit() {   
              this.data = someServiceMethodReturningObservable();
              this.data2 = someServiceMethodReturningObservable();
            }
          }
      ```
   3. We can use the `takeWhile()` operator which mirrors the source Observable until such time as some condition becomes false, at which point TakeWhile stops mirroring the source Observable and terminates its own Observable.
      ```typescript
          import { Component, OnInit, OnDestroy } from '@angular/core';
          import "rxjs/add/operator/takeWhile";

          @Component({
            selector: 'app-root',
          })
          export class AppComponent implements OnInit, OnDestroy {
            public data: any;
            public data2: any;
            private isActive: boolean = true;
            private isActive2: boolean = true;
            
            ngOnInit() {   
              someServiceMethodReturningObservable()
                .takeWhile(() => this.isActive)
                .subscribe(response => this.data = response);
        
              someServiceMethodReturningObservable()
                .takeWhile(() => this.isActive2)
                .subscribe(response => this.data2 = response);
            }
      
            ngOnDestroy() { 
              this.isActive = false;
              this.isActive2 = false;
            }
          }
      ```
