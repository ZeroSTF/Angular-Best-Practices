# Angular Best Practices

## 1. Authentication Service

Create an authentication service to handle login, logout, and token management.

```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { BehaviorSubject, Observable } from "rxjs";
import { map } from "rxjs/operators";

@Injectable({
	providedIn: "root",
})
export class AuthService {
	private currentUserSubject: BehaviorSubject<any>;
	public currentUser: Observable<any>;

	constructor(private http: HttpClient) {
		this.currentUserSubject = new BehaviorSubject<any>(
			JSON.parse(localStorage.getItem("currentUser"))
		);
		this.currentUser = this.currentUserSubject.asObservable();
	}

	public get currentUserValue() {
		return this.currentUserSubject.value;
	}

	login(username: string, password: string) {
		return this.http.post<any>("/api/auth/login", { username, password }).pipe(
			map((user) => {
				localStorage.setItem("currentUser", JSON.stringify(user));
				this.currentUserSubject.next(user);
				return user;
			})
		);
	}

	logout() {
		return this.http.post<any>("/api/auth/logout", {}).pipe(
			map(() => {
				localStorage.removeItem("currentUser");
				this.currentUserSubject.next(null);
			})
		);
	}

	refreshToken() {
		const currentUser = this.currentUserValue;
		if (currentUser && currentUser.refreshToken) {
			return this.http
				.post<any>("/api/auth/refresh-token", {
					refreshToken: currentUser.refreshToken,
				})
				.pipe(
					map((user) => {
						localStorage.setItem("currentUser", JSON.stringify(user));
						this.currentUserSubject.next(user);
						return user;
					})
				);
		}
		return new Observable((subscriber) =>
			subscriber.error("No refresh token available")
		);
	}
}
```

## 2. HTTP Interceptor

Create an interceptor to add the JWT token to all outgoing requests and handle token refresh.

```typescript
import { Injectable } from "@angular/core";
import {
	HttpRequest,
	HttpHandler,
	HttpEvent,
	HttpInterceptor,
	HttpErrorResponse,
} from "@angular/common/http";
import { Observable, throwError, BehaviorSubject } from "rxjs";
import { catchError, filter, take, switchMap } from "rxjs/operators";
import { AuthService } from "./auth.service";

@Injectable()
export class JwtInterceptor implements HttpInterceptor {
	private isRefreshing = false;
	private refreshTokenSubject: BehaviorSubject<any> = new BehaviorSubject<any>(
		null
	);

	constructor(private authService: AuthService) {}

	intercept(
		request: HttpRequest<any>,
		next: HttpHandler
	): Observable<HttpEvent<any>> {
		let currentUser = this.authService.currentUserValue;
		if (currentUser && currentUser.accessToken) {
			request = this.addToken(request, currentUser.accessToken);
		}

		return next.handle(request).pipe(
			catchError((error) => {
				if (error instanceof HttpErrorResponse && error.status === 401) {
					return this.handle401Error(request, next);
				} else {
					return throwError(error);
				}
			})
		);
	}

	private addToken(request: HttpRequest<any>, token: string) {
		return request.clone({
			setHeaders: {
				Authorization: `Bearer ${token}`,
			},
		});
	}

	private handle401Error(request: HttpRequest<any>, next: HttpHandler) {
		if (!this.isRefreshing) {
			this.isRefreshing = true;
			this.refreshTokenSubject.next(null);

			return this.authService.refreshToken().pipe(
				switchMap((token: any) => {
					this.isRefreshing = false;
					this.refreshTokenSubject.next(token.accessToken);
					return next.handle(this.addToken(request, token.accessToken));
				}),
				catchError((err) => {
					this.isRefreshing = false;
					this.authService.logout();
					return throwError(err);
				})
			);
		} else {
			return this.refreshTokenSubject.pipe(
				filter((token) => token != null),
				take(1),
				switchMap((jwt) => {
					return next.handle(this.addToken(request, jwt));
				})
			);
		}
	}
}
```

## 3. Route Guards

Implement route guards to protect routes that require authentication.

```typescript
import { inject } from "@angular/core";
import { Router } from "@angular/router";
import { AuthService } from "./auth.service";

export const authGuard = () => {
	const authService = inject(AuthService);
	const router = inject(Router);

	if (authService.isLoggedIn()) {
		return true;
	}

	return router.parseUrl("/login");
};
```

## 4. Error Handling

Create a global error handler to manage API errors consistently.

```typescript
import { Injectable } from "@angular/core";
import {
	HttpInterceptor,
	HttpRequest,
	HttpHandler,
	HttpEvent,
	HttpErrorResponse,
} from "@angular/common/http";
import { Observable, throwError } from "rxjs";
import { catchError } from "rxjs/operators";

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
	intercept(
		request: HttpRequest<any>,
		next: HttpHandler
	): Observable<HttpEvent<any>> {
		return next.handle(request).pipe(
			catchError((error: HttpErrorResponse) => {
				let errorMsg = "";
				if (error.error instanceof ErrorEvent) {
					// Client-side error
					errorMsg = `Error: ${error.error.message}`;
				} else {
					// Server-side error
					errorMsg = `Error Code: ${error.status},  Message: ${error.message}`;
				}
				console.error(errorMsg);
				return throwError(() => new Error(errorMsg));
			})
		);
	}
}
```

## 5. Environment Configuration

Use Angular's environment files to manage API URLs and other configuration settings.

```typescript
// environment.ts
export const environment = {
	production: false,
	apiUrl: "http://localhost:8080/api",
};

// environment.prod.ts
export const environment = {
	production: true,
	apiUrl: "https://your-production-api.com/api",
};
```

## 6. Reactive Forms

Use reactive forms for better form handling and validation.

```typescript
import { Component } from "@angular/core";
import { CommonModule } from "@angular/common";
import {
	ReactiveFormsModule,
	FormBuilder,
	FormGroup,
	Validators,
} from "@angular/forms";
import { AuthService } from "./auth.service";

@Component({
	selector: "app-login",
	standalone: true,
	imports: [CommonModule, ReactiveFormsModule],
	templateUrl: "./login.component.html",
})
export class LoginComponent {
	loginForm: FormGroup;

	constructor(
		private formBuilder: FormBuilder,
		private authService: AuthService
	) {
		this.loginForm = this.formBuilder.group({
			username: ["", Validators.required],
			password: ["", Validators.required],
		});
	}

	onSubmit() {
		if (this.loginForm.invalid) {
			return;
		}

		this.authService
			.login(
				this.loginForm.get("username")!.value,
				this.loginForm.get("password")!.value
			)
			.subscribe(
				(data) => {
					// Handle successful login
				},
				(error) => {
					// Handle login error
				}
			);
	}
}
```

## 7. Dependency Injection with inject function

Use the inject function for dependency injection in standalone components and services.

```typescript
import { Component, inject } from "@angular/core";
import { AuthService } from "./auth.service";

@Component({
	selector: "app-home",
	standalone: true,
	template: "<h1>Welcome, {{ username }}</h1>",
})
export class HomeComponent {
	private authService = inject(AuthService);
	username = this.authService.currentUserValue?.username;
}
```

## 8. Signals for State Management

Utilize Angular's Signals for reactive state management in components and services.

```typescript
import { Injectable, signal, computed } from "@angular/core";

@Injectable({ providedIn: "root" })
export class UserService {
	private userSignal = signal(null);

	user = computed(() => this.userSignal());

	isLoggedIn = computed(() => !!this.user());

	setUser(user: any) {
		this.userSignal.set(user);
	}
}
```

## 9. Lazy Loading

Implement lazy loading for better performance, especially in larger applications.

```typescript
import { Routes } from "@angular/router";

export const routes: Routes = [
	{
		path: "admin",
		loadComponent: () =>
			import("./admin/admin.component").then((m) => m.AdminComponent),
		canActivate: [authGuard],
	},
];
```

## 10. Environment Injection

Use environment injection for better type safety and easier mocking in tests.

```typescript
import { InjectionToken } from '@angular/core';

export interface Environment {
  production: boolean;
  apiUrl: string;
}

export const ENVIRONMENT = new InjectionToken<Environment>('environment');

// In your main.ts or app.config.ts
import { environment } from './environments/environment';

providers: [
  { provide: ENVIRONMENT, useValue: environment }
]

// Usage in a service
constructor(@Inject(ENVIRONMENT) private env: Environment) {}
```

## 11. Strong Typing

Use interfaces and types consistently for better type safety.

```typescript
interface User {
	id: number;
	username: string;
	email: string;
}

interface LoginResponse {
	user: User;
	accessToken: string;
	refreshToken: string;
}
```

## 12. Performance Optimization

Use OnPush change detection strategy for performance improvements in components that don't need frequent updates.

```typescript
import { Component, ChangeDetectionStrategy } from "@angular/core";

@Component({
	selector: "app-user-list",
	templateUrl: "./user-list.component.html",
	changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListComponent {
	// Component logic
}
```
