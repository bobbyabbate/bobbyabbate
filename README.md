# Treehouse Newsletter Signup — Angular Solution

A single-page Angular application that lists, adds, and removes users from a
newsletter subscription list via a provided REST API.

## Challenge recap

- List page: shows all current submissions, **newest first**, with delete support.
- Add page: form with **required** `name` and `email` fields.
- Client-side routing between the two pages.
- Built on a current **LTS** version of the framework (Angular 17).
- Talks to a REST API for `GET /newsletter`, `POST /newsletter`, and
  `DELETE /newsletter/{id}`, authenticated via an `Authorization` header.

## Tech stack

- **Angular 17** (standalone components, no NgModules)
- **Angular Router** for navigation between the list and add pages
- **Reactive Forms** for validation on the add-user form
- **HttpClient** for talking to the REST API

## Project structure

```
newsletter-app/
├── src/
│   ├── app/
│   │   ├── models/
│   │   │   └── subscriber.model.ts
│   │   ├── services/
│   │   │   └── newsletter.service.ts
│   │   ├── newsletter-list/
│   │   │   ├── newsletter-list.component.ts
│   │   │   ├── newsletter-list.component.html
│   │   │   └── newsletter-list.component.css
│   │   ├── newsletter-add/
│   │   │   ├── newsletter-add.component.ts
│   │   │   ├── newsletter-add.component.html
│   │   │   └── newsletter-add.component.css
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   └── app.routes.ts
│   ├── environments/
│   │   ├── environment.template.ts   (committed — copy this)
│   │   ├── environment.ts            (gitignored — real API key goes here)
│   │   └── environment.prod.ts       (gitignored)
│   ├── index.html
│   ├── main.ts
│   └── styles.css
├── angular.json
├── package.json
├── tsconfig.json
└── tsconfig.app.json
```

## Setup

1. Clone this repo and `cd` into the `newsletter-app` folder.
2. Install dependencies:
   ```bash
   npm install
   ```
3. Create your local environment files from the template:
   ```bash
   cp src/environments/environment.template.ts src/environments/environment.ts
   cp src/environments/environment.template.ts src/environments/environment.prod.ts
   ```
4. Open both new files and replace `'YOUR_API_KEY'` with your actual API key
   for `https://treehousechallenge.contractornation.com`.

   > **Never commit your real API key.** `environment.ts` and
   > `environment.prod.ts` are already listed in `.gitignore` for this reason.

5. Run the dev server:
   ```bash
   npm start
   ```
6. Visit `http://localhost:4200`.

## How it works

- **`NewsletterService`** (`src/app/services/newsletter.service.ts`) is the
  single place that knows about the API — base URL, auth header, and the
  three HTTP calls (`getSubscribers`, `addSubscriber`, `deleteSubscriber`).
- **`NewsletterListComponent`** loads subscribers on init, sorts them newest
  first, and lets you delete a row (with an in-flight "Removing…" state per
  row).
- **`NewsletterAddComponent`** uses a `FormGroup` with `Validators.required`
  on `name` and `Validators.required` + `Validators.email` on `email`. The
  submit button is disabled until the form is valid, and on success it
  navigates back to the list.
- **`app.routes.ts`** wires `'' → list`, `'add' → add form`, and a wildcard
  redirect back to the list.

## API reference used

| Method | Endpoint             | Purpose                  |
|--------|-----------------------|---------------------------|
| GET    | `/newsletter`         | Fetch all subscribers     |
| POST   | `/newsletter`         | Create a subscriber       |
| DELETE | `/newsletter/{id}`    | Remove a subscriber       |

Base URL: `https://treehousechallenge.contractornation.com`
Auth: `Authorization` header with the provided key (set in `environment.ts`).

## Full source

<details>
<summary><code>src/app/models/subscriber.model.ts</code></summary>

```typescript
export interface Subscriber {
  id: string;
  name: string;
  email: string;
  createdAt?: string;
}

export interface NewSubscriber {
  name: string;
  email: string;
}
```
</details>

<details>
<summary><code>src/app/services/newsletter.service.ts</code></summary>

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { NewSubscriber, Subscriber } from '../models/subscriber.model';

@Injectable({ providedIn: 'root' })
export class NewsletterService {
  private readonly baseUrl = `${environment.apiBaseUrl}/newsletter`;

  private readonly headers = new HttpHeaders({
    Authorization: environment.apiKey,
    'Content-Type': 'application/json',
  });

  constructor(private http: HttpClient) {}

  getSubscribers(): Observable<Subscriber[]> {
    return this.http.get<Subscriber[]>(this.baseUrl, { headers: this.headers });
  }

  addSubscriber(subscriber: NewSubscriber): Observable<Subscriber> {
    return this.http.post<Subscriber>(this.baseUrl, subscriber, { headers: this.headers });
  }

  deleteSubscriber(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`, { headers: this.headers });
  }
}
```
</details>

<details>
<summary><code>src/app/newsletter-list/newsletter-list.component.ts</code></summary>

```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

import { NewsletterService } from '../services/newsletter.service';
import { Subscriber } from '../models/subscriber.model';

@Component({
  selector: 'app-newsletter-list',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './newsletter-list.component.html',
  styleUrl: './newsletter-list.component.css',
})
export class NewsletterListComponent implements OnInit {
  subscribers: Subscriber[] = [];
  loading = true;
  error: string | null = null;
  deletingId: string | null = null;

  constructor(private newsletterService: NewsletterService) {}

  ngOnInit(): void {
    this.fetchSubscribers();
  }

  fetchSubscribers(): void {
    this.loading = true;
    this.error = null;

    this.newsletterService.getSubscribers().subscribe({
      next: (subscribers) => {
        this.subscribers = [...subscribers].sort((a, b) => {
          if (a.createdAt && b.createdAt) {
            return new Date(b.createdAt).getTime() - new Date(a.createdAt).getTime();
          }
          return b.id.localeCompare(a.id);
        });
        this.loading = false;
      },
      error: () => {
        this.error = 'Unable to load subscribers. Please try again later.';
        this.loading = false;
      },
    });
  }

  deleteSubscriber(id: string): void {
    this.deletingId = id;
    this.newsletterService.deleteSubscriber(id).subscribe({
      next: () => {
        this.subscribers = this.subscribers.filter((s) => s.id !== id);
        this.deletingId = null;
      },
      error: () => {
        this.error = 'Unable to delete subscriber. Please try again.';
        this.deletingId = null;
      },
    });
  }
}
```
</details>

<details>
<summary><code>src/app/newsletter-list/newsletter-list.component.html</code></summary>

```html
<div class="page">
  <div class="page-header">
    <h1>Newsletter Subscribers</h1>
    <a class="btn btn-primary" routerLink="/add">Add Subscriber</a>
  </div>

  <p *ngIf="error" class="error">{{ error }}</p>

  <p *ngIf="loading">Loading subscribers…</p>

  <table *ngIf="!loading && subscribers.length" class="subscriber-table">
    <thead>
      <tr>
        <th>Name</th>
        <th>Email</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      <tr *ngFor="let subscriber of subscribers">
        <td>{{ subscriber.name }}</td>
        <td>{{ subscriber.email }}</td>
        <td class="actions">
          <button
            class="btn btn-danger"
            [disabled]="deletingId === subscriber.id"
            (click)="deleteSubscriber(subscriber.id)"
          >
            {{ deletingId === subscriber.id ? 'Removing…' : 'Delete' }}
          </button>
        </td>
      </tr>
    </tbody>
  </table>

  <p *ngIf="!loading && !subscribers.length && !error">No subscribers yet.</p>
</div>
```
</details>

<details>
<summary><code>src/app/newsletter-add/newsletter-add.component.ts</code></summary>

```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { Router, RouterLink } from '@angular/router';

import { NewsletterService } from '../services/newsletter.service';

@Component({
  selector: 'app-newsletter-add',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  templateUrl: './newsletter-add.component.html',
  styleUrl: './newsletter-add.component.css',
})
export class NewsletterAddComponent {
  submitting = false;
  error: string | null = null;

  form = this.fb.group({
    name: ['', [Validators.required, Validators.minLength(1)]],
    email: ['', [Validators.required, Validators.email]],
  });

  constructor(
    private fb: FormBuilder,
    private newsletterService: NewsletterService,
    private router: Router,
  ) {}

  onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitting = true;
    this.error = null;

    const { name, email } = this.form.getRawValue();

    this.newsletterService.addSubscriber({ name: name!, email: email! }).subscribe({
      next: () => {
        this.submitting = false;
        this.router.navigate(['/']);
      },
      error: () => {
        this.submitting = false;
        this.error = 'Unable to add subscriber. Please try again.';
      },
    });
  }
}
```
</details>

<details>
<summary><code>src/app/newsletter-add/newsletter-add.component.html</code></summary>

```html
<div class="page">
  <div class="page-header">
    <h1>Add Subscriber</h1>
    <a class="btn btn-secondary" routerLink="/">Back to list</a>
  </div>

  <form [formGroup]="form" (ngSubmit)="onSubmit()" class="subscriber-form">
    <label for="name">Name</label>
    <input id="name" type="text" formControlName="name" placeholder="Jane Doe" />
    <p class="field-error" *ngIf="form.controls.name.invalid && form.controls.name.touched">
      Name is required.
    </p>

    <label for="email">Email</label>
    <input id="email" type="email" formControlName="email" placeholder="jane@example.com" />
    <p class="field-error" *ngIf="form.controls.email.invalid && form.controls.email.touched">
      A valid email is required.
    </p>

    <p *ngIf="error" class="error">{{ error }}</p>

    <button type="submit" class="btn btn-primary" [disabled]="form.invalid || submitting">
      {{ submitting ? 'Adding…' : 'Add Subscriber' }}
    </button>
  </form>
</div>
```
</details>

<details>
<summary><code>src/app/app.routes.ts</code></summary>

```typescript
import { Routes } from '@angular/router';

import { NewsletterListComponent } from './newsletter-list/newsletter-list.component';
import { NewsletterAddComponent } from './newsletter-add/newsletter-add.component';

export const routes: Routes = [
  { path: '', component: NewsletterListComponent },
  { path: 'add', component: NewsletterAddComponent },
  { path: '**', redirectTo: '' },
];
```
</details>

<details>
<summary><code>src/main.ts</code></summary>

```typescript
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';

import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes), provideHttpClient()],
}).catch((err) => console.error(err));
```
</details>

---

*Note: `environment.ts` / `environment.prod.ts` (which hold the real API key)
are intentionally excluded from this repo via `.gitignore`. Use
`environment.template.ts` to create your own local copies.*
