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


