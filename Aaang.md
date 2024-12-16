In Angular 18+, which uses standalone components, you don’t have an app.module.ts. Instead, the application relies on component-based bootstrapping. Here’s how to update the structure and code to align with Angular 18+:


---

Updated Folder Structure

src/
├── app/
│   ├── guards/
│   │   ├── auth.guard.ts
│   │   ├── admin.guard.ts
│   ├── interceptors/
│   │   ├── auth.interceptor.ts
│   ├── services/
│   │   ├── auth.service.ts
│   ├── components/
│   │   ├── login/
│   │   │   ├── login.component.ts
│   │   │   ├── login.component.html
│   │   │   ├── login.component.css
│   │   ├── signup/
│   │   │   ├── signup.component.ts
│   │   │   ├── signup.component.html
│   │   │   ├── signup.component.css
│   │   ├── dashboard/
│   │   │   ├── dashboard.component.ts
│   │   │   ├── dashboard.component.html
│   │   │   ├── dashboard.component.css
│   │   ├── admin-dashboard/
│   │   │   ├── admin-dashboard.component.ts
│   │   │   ├── admin-dashboard.component.html
│   │   │   ├── admin-dashboard.component.css
│   ├── app.routes.ts
│   ├── main.ts


---

Key Code Updates

1. App Routes (app.routes.ts)

Defines the routes directly with standalone components.

import { provideRoutes } from '@angular/router';
import { LoginComponent } from './components/login/login.component';
import { SignupComponent } from './components/signup/signup.component';
import { DashboardComponent } from './components/dashboard/dashboard.component';
import { AdminDashboardComponent } from './components/admin-dashboard/admin-dashboard.component';
import { AuthGuard } from './guards/auth.guard';
import { AdminGuard } from './guards/admin.guard';

export const APP_ROUTES = [
  { path: 'login', component: LoginComponent },
  { path: 'signup', component: SignupComponent },
  { path: 'dashboard', component: DashboardComponent, canActivate: [AuthGuard] },
  { path: 'admin-dashboard', component: AdminDashboardComponent, canActivate: [AdminGuard] },
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: '**', redirectTo: 'login' },
];

export const ROUTES_PROVIDERS = provideRoutes(APP_ROUTES);


---

2. Main Entry (main.ts)

Bootstraps the root component and provides routing.

import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './components/login/login.component'; // Root component
import { ROUTES_PROVIDERS } from './app.routes';
import { AuthInterceptor } from './interceptors/auth.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    ROUTES_PROVIDERS,
    provideHttpClient(withInterceptors([AuthInterceptor])),
  ],
});


---

3. Login Component (login.component.ts)

Updated for standalone use.

import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { HttpClient } from '@angular/common/http';

@Component({
  standalone: true,
  selector: 'app-login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css'],
})
export class LoginComponent {
  email: string = '';
  password: string = '';

  constructor(private http: HttpClient, private router: Router) {}

  login() {
    this.http.post('http://your-backend-api-url/login', { email: this.email, password: this.password })
      .subscribe({
        next: () => this.router.navigate(['/dashboard']),
        error: () => alert('Invalid credentials'),
      });
  }
}


---

4. Signup Component (signup.component.ts)

For new user registration.

import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { HttpClient } from '@angular/common/http';

@Component({
  standalone: true,
  selector: 'app-signup',
  templateUrl: './signup.component.html',
  styleUrls: ['./signup.component.css'],
})
export class SignupComponent {
  email: string = '';
  password: string = '';

  constructor(private http: HttpClient, private router: Router) {}

  signup() {
    this.http.post('http://your-backend-api-url/signup', { email: this.email, password: this.password })
      .subscribe({
        next: () => {
          alert('Signup successful!');
          this.router.navigate(['/login']);
        },
        error: () => alert('Signup failed'),
      });
  }
}


---

5. Login Page HTML (login.component.html)

Simple login form.

<div class="login-container">
  <h2>Login</h2>
  <form (submit)="login()">
    <label for="email">Email</label>
    <input id="email" type="email" [(ngModel)]="email" name="email" required />

    <label for="password">Password</label>
    <input id="password" type="password" [(ngModel)]="password" name="password" required />

    <button type="submit">Login</button>
  </form>
  <p>Don't have an account? <a routerLink="/signup">Signup here</a></p>
</div>


---

6. Signup Page HTML (signup.component.html)

Simple signup form.

<div class="signup-container">
  <h2>Signup</h2>
  <form (submit)="signup()">
    <label for="email">Email</label>
    <input id="email" type="email" [(ngModel)]="email" name="email" required />

    <label for="password">Password</label>
    <input id="password" type="password" [(ngModel)]="password" name="password" required />

    <button type="submit">Signup</button>
  </form>
  <p>Already have an account? <a routerLink="/login">Login here</a></p>
</div>


---

CSS Styles (Optional)

For both pages (login.component.css and signup.component.css):

.login-container, .signup-container {
  max-width: 400px;
  margin: 50px auto;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

h2 {
  text-align: center;
}

form {
  display: flex;
  flex-direction: column;
}

label {
  margin-bottom: 5px;
  font-weight: bold;
}

input {
  margin-bottom: 15px;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
}

button {
  padding: 10px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #0056b3;
}

p {
  text-align: center;
}

a {
  color: #007bff;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}


---

Key Changes for Angular 18+

1. Standalone components for all pages.


2. Routes defined via provideRoutes.


3. Application bootstrapped in main.ts.


4. Simplified structure with no AppModule.



Let me know if you need further assistance!











82>÷<>=^=*×>[2;÷[÷<÷8>4^484&4>=&>=<=73>4&==&=&



Here's a full implementation to set up protected routes in Angular 18. Since your cookies are already stored directly in the cookie storage by the backend, we will use those cookies for validation, role-based access control, and logout functionality.


---

Folder Structure

app/
|-- core/
|   |-- auth/
|   |   |-- login/
|   |   |   |-- interfaces/
|   |   |   |   |-- user.interface.ts
|   |   |   |-- pages/
|   |   |   |   |-- login-page.component.ts
|   |   |   |   |-- login-page.component.html
|   |   |   |-- login.guard.ts
|   |   |   |-- admin.guard.ts
|   |   |   |-- login.service.ts
|   |-- dashboard/
|   |   |-- admin-dashboard.component.ts
|   |   |-- user-dashboard.component.ts
app.routes.ts
main.ts


---

Code

1. Interfaces

user.interface.ts:

export interface User {
  email: string;
  password: string;
}

export interface ApiResponse {
  success: boolean;
  message: string;
  role?: string; // 'admin' or 'user'
}


---

2. Guards

login.guard.ts:

import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { LoginService } from './login.service';

@Injectable({
  providedIn: 'root',
})
export class LoginGuard implements CanActivate {
  constructor(private loginService: LoginService, private router: Router) {}

  canActivate(): boolean {
    if (this.loginService.isLoggedIn()) {
      return true;
    }
    this.router.navigate(['/login']); // Redirect to login page if not logged in
    return false;
  }
}

admin.guard.ts:

import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { LoginService } from './login.service';

@Injectable({
  providedIn: 'root',
})
export class AdminGuard implements CanActivate {
  constructor(private loginService: LoginService, private router: Router) {}

  canActivate(): boolean {
    const role = this.loginService.getRole();
    if (role === 'admin') {
      return true;
    }
    this.router.navigate(['/dashboard']); // Redirect non-admin users to the dashboard
    return false;
  }
}


---

3. Login Service

login.service.ts:

import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class LoginService {
  isLoggedIn(): boolean {
    const token = this.getCookie('jwt');
    return !!token; // Return true if JWT exists
  }

  getRole(): string | null {
    const role = this.getCookie('role');
    return role; // Return the role (admin or user)
  }

  logout(): void {
    document.cookie = 'jwt=; Max-Age=0;'; // Clear JWT cookie
    document.cookie = 'role=; Max-Age=0;'; // Clear role cookie
  }

  private getCookie(name: string): string | null {
    const matches = document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
    return matches ? decodeURIComponent(matches[1]) : null;
  }
}


---

4. Login Page

login-page.component.ts:

import { Component } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-login-page',
  templateUrl: './login-page.component.html',
})
export class LoginPageComponent {
  email = '';
  password = '';

  constructor(private router: Router) {}

  login() {
    // Assume cookie is set by backend after successful login
    if (this.email === 'admin@example.com' && this.password === 'admin') {
      document.cookie = 'jwt=token; path=/;';
      document.cookie = 'role=admin; path=/;';
      this.router.navigate(['/admin']); // Redirect to admin dashboard
    } else if (this.email === 'user@example.com' && this.password === 'user') {
      document.cookie = 'jwt=token; path=/;';
      document.cookie = 'role=user; path=/;';
      this.router.navigate(['/dashboard']); // Redirect to user dashboard
    } else {
      alert('Invalid credentials');
    }
  }
}

login-page.component.html:

<div>
  <h2>Login</h2>
  <form (ngSubmit)="login()">
    <input type="email" [(ngModel)]="email" name="email" required placeholder="Email">
    <input type="password" [(ngModel)]="password" name="password" required placeholder="Password">
    <button type="submit">Login</button>
  </form>
</div>


---

5. Dashboard Components

admin-dashboard.component.ts:

import { Component } from '@angular/core';
import { LoginService } from '../auth/login/login.service';

@Component({
  selector: 'app-admin-dashboard',
  template: `<h2>Admin Dashboard</h2> <button (click)="logout()">Logout</button>`,
})
export class AdminDashboardComponent {
  constructor(private loginService: LoginService) {}

  logout() {
    this.loginService.logout();
    location.reload(); // Reload to redirect to login
  }
}

user-dashboard.component.ts:

import { Component } from '@angular/core';
import { LoginService } from '../auth/login/login.service';

@Component({
  selector: 'app-user-dashboard',
  template: `<h2>User Dashboard</h2> <button (click)="logout()">Logout</button>`,
})
export class UserDashboardComponent {
  constructor(private loginService: LoginService) {}

  logout() {
    this.loginService.logout();
    location.reload(); // Reload to redirect to login
  }
}


---

6. Routes

app.routes.ts:

import { LoginPageComponent } from './core/auth/login/pages/login-page.component';
import { UserDashboardComponent } from './core/dashboard/user-dashboard.component';
import { AdminDashboardComponent } from './core/dashboard/admin-dashboard.component';
import { LoginGuard } from './core/auth/login/login.guard';
import { AdminGuard } from './core/auth/login/admin.guard';

export const APP_ROUTES = [
  { path: 'login', component: LoginPageComponent },
  { path: 'dashboard', component: UserDashboardComponent, canActivate: [LoginGuard] },
  { path: 'admin', component: AdminDashboardComponent, canActivate: [LoginGuard, AdminGuard] },
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: '**', redirectTo: 'login' },
];


---

7. Main Bootstrap

main.ts:

import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { APP_ROUTES } from './app.routes';

bootstrapApplication(LoginPageComponent, {
  providers: [provideHttpClient(), provideRouter(APP_ROUTES)],
});


---

How It Works

1. Login Logic:

The backend sets JWT and role cookies on successful login.



2. Guards:

LoginGuard: Ensures only logged-in users access protected routes.

AdminGuard: Ensures only admin users access admin routes.



3. Routing:

/login: Login page (open to everyone).

/dashboard: Protected route for regular users.

/admin: Protected route for admins only.



4. Logout:

Clears cookies and reloads the page to redirect to login.




Let me know if you need further clarification!




irhdufuf8ri4h3ieuheyey548382929iwvejsbgdue99202 ou g bri go 4
If your JWT contains both the user ID and role information, we can decode the JWT directly in the Angular application using a library like jwt-decode. Here’s how you can adapt the auth service to decode the JWT and extract the role.


---

Update Auth Service

Install jwt-decode:

Run the following command to install the jwt-decode library:

npm install jwt-decode


---

Updated login.service.ts:

import { Injectable } from '@angular/core';
import jwt_decode from 'jwt-decode';

@Injectable({
  providedIn: 'root',
})
export class LoginService {
  private jwtCookieName = 'jwt';

  isLoggedIn(): boolean {
    const token = this.getCookie(this.jwtCookieName);
    return !!token; // Return true if JWT exists
  }

  getRole(): string | null {
    const token = this.getCookie(this.jwtCookieName);
    if (token) {
      try {
        const decodedToken = jwt_decode<{ role: string }>(token);
        return decodedToken.role; // Extract role from decoded token
      } catch (error) {
        console.error('Error decoding JWT:', error);
        return null;
      }
    }
    return null;
  }

  logout(): void {
    document.cookie = `${this.jwtCookieName}=; Max-Age=0;`; // Clear JWT cookie
  }

  private getCookie(name: string): string | null {
    const matches = document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`));
    return matches ? decodeURIComponent(matches[1]) : null;
  }
}


---

How It Works

1. Decoding the JWT:

The jwt-decode library decodes the JWT payload (base64).

In this case, the payload should include a role field.



2. Role Retrieval:

The getRole() method decodes the JWT stored in the cookies and extracts the role field (e.g., admin, user).



3. Logout:

The logout() method clears the JWT cookie, effectively logging the user out.



4. Error Handling:

If the JWT cannot be decoded (e.g., it is malformed), the service logs the error and returns null.





---

Update Guards

**`login

Here’s how you can update the guards to use the decoded JWT role information from the updated LoginService:


---

Updated Guards

login.guard.ts (Protects routes for logged-in users):

import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { LoginService } from './login.service';

@Injectable({
  providedIn: 'root',
})
export class LoginGuard implements CanActivate {
  constructor(private loginService: LoginService, private router: Router) {}

  canActivate(): boolean {
    if (this.loginService.isLoggedIn()) {
      return true;
    }
    this.router.navigate(['/login']); // Redirect to login if not authenticated
    return false;
  }
}


---

admin.guard.ts (Restricts access to admin-only routes):

import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { LoginService } from './login.service';

@Injectable({
  providedIn: 'root',
})
export class AdminGuard implements CanActivate {
  constructor(private loginService: LoginService, private router: Router) {}

  canActivate(): boolean {
    const role = this.loginService.getRole();
    if (role === 'admin') {
      return true;
    }
    this.router.navigate(['/dashboard']); // Redirect non-admin users to their dashboard
    return false;
  }
}


---

Update Routes

With the updated guards in place, the app.routes.ts file will ensure proper access control.

app.routes.ts:

import { LoginPageComponent } from './core/auth/login/pages/login-page.component';
import { UserDashboardComponent } from './core/dashboard/user-dashboard.component';
import { AdminDashboardComponent } from './core/dashboard/admin-dashboard.component';
import { LoginGuard } from './core/auth/login/login.guard';
import { AdminGuard } from './core/auth/login/admin.guard';

export const APP_ROUTES = [
  { path: 'login', component: LoginPageComponent },
  { path: 'dashboard', component: UserDashboardComponent, canActivate: [LoginGuard] },
  { path: 'admin', component: AdminDashboardComponent, canActivate: [LoginGuard, AdminGuard] },
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: '**', redirectTo: 'login' },
];


---

How It Works

1. LoginGuard:

Ensures that the user is logged in by checking the presence of the JWT in cookies.

If not logged in, redirects to the /login route.



2. AdminGuard:

Uses the decoded JWT to check if the user has the admin role.

If the role is not admin, redirects to the /dashboard.



3. Routing:

/login: Open to everyone.

/dashboard: Protected for logged-in users.

/admin: Accessible only to users with the admin role.





---

Final Notes

JWT Payload Format: Ensure that your backend includes the role field in the JWT payload, e.g.:

{
  "sub": "userId",
  "role": "admin",
  "exp": 1718928000
}

Token Expiry Handling: If your JWT includes an expiration (exp) field, you can enhance the isLoggedIn() method to check whether the token is expired:

const decoded = jwt_decode<{ exp: number }>(token);
const isExpired = decoded.exp * 1000 < Date.now(); // JWT expiration is in seconds
return !isExpired;


Let me know if you need additional functionality or further clarification!

4678y u hhcfhjooo9887766554wwwdxcvbbnnmkugyui99987765433s c hhjiii


Here’s how to modify the setup based on your token structure and requirements:


---

Steps to Retrieve Roles and Implement Guards

1. Update LoginService to Decode Roles

Since your token has a roles array with objects like { authority: "ADMIN" }, we will update the LoginService to decode the token and extract the role.

Updated login.service.ts:

import { Injectable } from '@angular/core';
import jwt_decode from 'jwt-decode';

@Injectable({
  providedIn: 'root',
})
export class LoginService {
  private tokenName = 'bearertoken'; // Your token name

  isLoggedIn(): boolean {
    return !!this.getToken(); // Return true if token exists
  }

  getToken(): string | null {
    const matches = document.cookie.match(new RegExp(`(?:^|; )${this.tokenName}=([^;]*)`));
    return matches ? decodeURIComponent(matches[1]) : null;
  }

  getRole(): string | null {
    const token = this.getToken();
    if (token) {
      try {
        const decoded = jwt_decode<{ roles: { authority: string }[] }>(token);
        if (decoded.roles && decoded.roles.length > 0) {
          return decoded.roles[0].authority; // Return the first role's authority
        }
      } catch (error) {
        console.error('Error decoding JWT:', error);
      }
    }
    return null; // Return null if no role found or token is invalid
  }

  logout(): void {
    document.cookie = `${this.tokenName}=; Max-Age=0;`; // Clear token
  }
}


---

2. Update the Auth Guard

Now, let’s implement the logic to handle redirection based on:

Whether the token exists.

The user’s role.


Updated login.guard.ts:

import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { LoginService } from './core/auth/login/login.service';

@Injectable({
  providedIn: 'root',
})
export class AuthGuard implements CanActivate {
  constructor(private loginService: LoginService, private router: Router) {}

  canActivate(): boolean {
    const token = this.loginService.getToken();

    if (!token) {
      this.router.navigate(['/login']); // Redirect to login if no token
      return false;
    }

    const role = this.loginService.getRole();

    if (role === 'ADMIN') {
      this.router.navigate(['/admin/dashboard']); // Redirect to admin dashboard
    } else if (role === 'USER') {
      this.router.navigate(['/user/dashboard']); // Redirect to user dashboard
    } else {
      this.router.navigate(['/login']); // Redirect to login if role is invalid
    }

    return false; // Prevent navigation to the requested route
  }
}


---

3. Update Routes

We will now define proper paths for /admin/dashboard and /user/dashboard.

Updated app.routes.ts:

import { LoginPageComponent } from './core/auth/login/pages/login-page.component';
import { UserDashboardComponent } from './core/dashboard/user-dashboard.component';
import { AdminDashboardComponent } from './core/dashboard/admin-dashboard.component';
import { AuthGuard } from './core/auth/login/login.guard';

export const APP_ROUTES = [
  { path: 'login', component: LoginPageComponent },
  { path: 'admin/dashboard', component: AdminDashboardComponent, canActivate: [AuthGuard] },
  { path: 'user/dashboard', component: UserDashboardComponent, canActivate: [AuthGuard] },
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: '**', redirectTo: 'login' },
];


---

Interceptor for Adding Token

To automatically include the token in outgoing HTTP requests, you can use an interceptor.

Updated auth.interceptor.ts:

import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { LoginService } from './core/auth/login/login.service';
import { Router } from '@angular/router';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private loginService: LoginService, private router: Router) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.loginService.getToken();

    // Clone request and add Authorization header if token exists
    const authReq = token
      ? req.clone({ headers: req.headers.set('Authorization', `Bearer ${token}`) })
      : req;

    return next.handle(authReq).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Unauthorized access: redirect to login
          this.loginService.logout();
          this.router.navigate(['/login']);
        }
        return throwError(() => error);
      })
    );
  }
}


---

Updates to main.ts

Include the interceptor and updated guards.

main.ts:

import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { APP_ROUTES } from './app.routes';
import { AuthInterceptor } from './auth.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(withInterceptors([AuthInterceptor])),
    provideRouter(APP_ROUTES),
  ],
});


---

Final Folder Structure

Here’s your updated folder structure:

app/
├── core/
│   ├── auth/
│   │   ├── login/
│   │   │   ├── login.guard.ts
│   │   │   ├── login.service.ts
│   │   │   ├── pages/
│   │   │   │   └── login-page.component.ts
│   │   │   ├── interfaces/
│   │   │   ├── types/
│   │   │   └── login.module.ts
│   │   ├── auth.interceptor.ts
│   ├── dashboard/
│       ├── user-dashboard.component.ts
│       └── admin-dashboard.component.ts
├── app.routes.ts
├── app.component.ts
├── main.ts


---

Flow Summary

1. Interceptor:

Automatically adds the JWT token to requests.

Redirects to /login on 401 errors.



2. 


8484838398393939ejjrhfurjjekejsbdbdhdujrbehrhrurue8ei3jejrr



To connect to a WebSocket in Angular and manage the connection lifecycle (onInit to establish the connection and onDestroy to terminate it), follow these steps:

Steps to Implement:

1. Create a WebSocket service for reusable and maintainable WebSocket connection logic.


2. Use ngOnInit in your component to subscribe to WebSocket events.


3. Use ngOnDestroy to close the WebSocket connection.



Example Implementation:

1. WebSocket Service

import { Injectable } from '@angular/core';
import { Observable, Subject } from 'rxjs';

@Injectable({
  providedIn: 'root',
})
export class WebSocketService {
  private socket!: WebSocket;
  private messages$ = new Subject<string>();

  connect(url: string): void {
    this.socket = new WebSocket(url);

    // Listen for incoming messages
    this.socket.onmessage = (event) => {
      this.messages$.next(event.data);
    };

    // Handle socket close
    this.socket.onclose = () => {
      console.log('WebSocket closed');
    };

    // Handle socket errors
    this.socket.onerror = (error) => {
      console.error('WebSocket error', error);
    };
  }

  sendMessage(message: string): void {
    if (this.socket && this.socket.readyState === WebSocket.OPEN) {
      this.socket.send(message);
    } else {
      console.error('WebSocket is not open');
    }
  }

  getMessages(): Observable<string> {
    return this.messages$.asObservable();
  }

  disconnect(): void {
    if (this.socket) {
      this.socket.close();
    }
  }
}

2. Component

import { Component, OnInit, OnDestroy } from '@angular/core';
import { Subscription } from 'rxjs';
import { WebSocketService } from './web-socket.service';

@Component({
  selector: 'app-websocket',
  template: `<div *ngFor="let message of messages">{{ message }}</div>`,
})
export class WebsocketComponent implements OnInit, OnDestroy {
  private subscription!: Subscription;
  messages: string[] = [];

  constructor(private webSocketService: WebSocketService) {}

  ngOnInit(): void {
    // Connect to the WebSocket
    this.webSocketService.connect('wss://example.com/socket');

    // Subscribe to WebSocket messages
    this.subscription = this.webSocketService.getMessages().subscribe((message) => {
      this.messages.push(message);
    });
  }

  ngOnDestroy(): void {
    // Unsubscribe and disconnect
    if (this.subscription) {
      this.subscription.unsubscribe();
    }
    this.webSocketService.disconnect();
  }
}

Explanation:

1. Service Responsibilities:

Establish and manage the WebSocket connection.

Provide an observable (getMessages()) to allow components to subscribe to incoming messages.

Handle sending messages and clean up the connection (disconnect()).



2. Component Responsibilities:

Subscribe to WebSocket messages using the service during ngOnInit.

Unsubscribe from the observable and terminate the WebSocket connection during ngOnDestroy.




Notes:

Use dependency injection to share the WebSocket service across components if needed.

Handle reconnect logic in the service if the WebSocket connection might drop unexpectedly.

Customize error handling based on your application's requirements.

*;₹>₹**#÷>74939030+772&÷^#;&#*,'bdjxkpdosheiei9eury484hrbrmkdkdksndbxhxksoosjdhfjrjdjjrirut749499392yehendksksjdidjdjjeiejrhr78393iejr

To implement this setup on the frontend with Angular, you can use the STOMP over WebSocket protocol. The @stomp/stompjs library is a great tool for this. Below is the step-by-step implementation:


---

1. Install the Required Library

Install the @stomp/stompjs library:

npm install @stomp/stompjs


---

2. Create a WebSocket Service

Create a reusable service to handle WebSocket connections, sending messages, and subscribing to topics.

import { Injectable } from '@angular/core';
import { Client, IMessage, StompConfig } from '@stomp/stompjs';

@Injectable({
  providedIn: 'root',
})
export class WebSocketService {
  private client!: Client;

  // Initialize the WebSocket client
  connect(endpoint: string): void {
    this.client = new Client({
      brokerURL: endpoint, // Your WebSocket endpoint
      reconnectDelay: 5000, // Auto-reconnect every 5 seconds
      debug: (msg: string) => console.log(msg),
    });

    // Activate the connection
    this.client.activate();
  }

  // Subscribe to a specific topic
  subscribe(topic: string, callback: (message: IMessage) => void): void {
    this.client.onConnect = () => {
      console.log(`Connected to WebSocket: ${topic}`);
      this.client.subscribe(topic, callback);
    };

    this.client.onDisconnect = () => {
      console.log('Disconnected from WebSocket');
    };
  }

  // Send data to the server
  send(destination: string, body: any): void {
    if (this.client && this.client.connected) {
      this.client.publish({
        destination,
        body: JSON.stringify(body),
      });
    } else {
      console.error('WebSocket is not connected');
    }
  }

  // Disconnect the WebSocket
  disconnect(): void {
    if (this.client) {
      this.client.deactivate();
      console.log('WebSocket connection closed');
    }
  }
}


---

3. Use the WebSocket Service in a Component

Use this service in your component to connect to /topic/team/teamid and send data to /team/addteamtodo/userid.

Component:

import { Component, OnInit, OnDestroy } from '@angular/core';
import { WebSocketService } from './web-socket.service';

@Component({
  selector: 'app-team-todo',
  template: `
    <div>
      <h3>Messages for Team</h3>
      <ul>
        <li *ngFor="let message of messages">{{ message }}</li>
      </ul>

      <input [(ngModel)]="todo" placeholder="Enter a todo" />
      <button (click)="addTodo()">Add Todo</button>
    </div>
  `,
})
export class TeamTodoComponent implements OnInit, OnDestroy {
  messages: string[] = [];
  todo: string = '';
  private topic = '/topic/team/123'; // Replace 123 with dynamic team ID
  private destination = '/app/team/addteamtodo/456'; // Replace 456 with user ID

  constructor(private webSocketService: WebSocketService) {}

  ngOnInit(): void {
    // Connect to the WebSocket server
    this.webSocketService.connect('/ws-todo');

    // Subscribe to the topic for team updates
    this.webSocketService.subscribe(this.topic, (message) => {
      this.messages.push(message.body); // Append new messages to the list
    });
  }

  addTodo(): void {
    // Send a message to the backend
    const payload = { todo: this.todo, teamId: 123, userId: 456 }; // Adjust the payload structure as needed
    this.webSocketService.send(this.destination, payload);
    this.todo = ''; // Clear input
  }

  ngOnDestroy(): void {
    // Disconnect from the WebSocket server
    this.webSocketService.disconnect();
  }
}


---

4. Key Points to Understand

1. WebSocket Endpoint:

The WebSocket endpoint (/ws-todo) is used to establish the connection.



2. Topics:

The frontend subscribes to /topic/team/teamid to receive team-specific updates.



3. Sending Data:

Data is sent to /app/team/addteamtodo/userid. Since this has the application prefix (/app), the backend should handle this route as defined in your @MessageMapping method.



4. Dynamic Topics:

Replace hardcoded IDs (teamid, userid) with dynamic values based on your application's context.





---

Backend Mapping for Clarity

Ensure your Spring Boot backend maps the endpoints correctly:

@Controller
public class TeamController {
  
    @MessageMapping("/team/addteamtodo/{userid}")
    @SendTo("/topic/team/{teamid}")
    public TeamTodo handleAddTeamTodo(@DestinationVariable String userid, @Payload TeamTodo todo) {
        // Logic to handle the todo (e.g., save to database)
        return todo; // Broadcast to team
    }
}


---

Summary

Connect to /ws-todo using STOMP.

Subscribe to /topic/team/teamid for team-specific updates.

Send data to /app/team/addteamtodo/userid for processing on the backend.

Ensure the service handles WebSocket connection lifecycle to maintain robust communication.




