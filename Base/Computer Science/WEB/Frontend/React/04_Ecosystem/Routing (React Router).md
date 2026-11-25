---
tags:
  - web/react
  - web/api
---

## Routing в [[SPA]] решает две задачи:

1. синхронизировать [[UI]] с [[URL]],    
2. загрузить нужные данные и код только для активного экрана.

Современный стек для роутинга в [[React]] — это **React Router 6.22+** (и v7‑ready API), который даёт декларативные маршруты, вложенный роутинг, ленивую загрузку и «data API» (loaders/actions).

---
## Базовая настройка роутера

В новых версиях удобнее использовать «data router» через `createBrowserRouter` и `RouterProvider` вместо старого `<BrowserRouter><Routes>…`.

### Пример минимальной настройки:

``` tsx
// main.tsx / index.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import {
  createBrowserRouter,
  RouterProvider,
} from "react-router-dom";
import { AppLayout } from "./AppLayout";
import { HomePage } from "./pages/HomePage";
import { AboutPage } from "./pages/AboutPage";

const router = createBrowserRouter([
  {
    path: "/",
    element: <AppLayout />, // общий layout
    children: [
      { index: true, element: <HomePage /> },
      { path: "about", element: <AboutPage /> },
    ],
  },
]);

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>
);
```

### Ключевые моменты:

- `createBrowserRouter([...])` — описываешь дерево маршрутов как обычный массив объектов.​
    
- `element` — React‑компонент, который рендерится на этом маршруте.
    
- `index: true` — «индексный» маршрут для родителя (`/` вместо `/something`).​
    
- `<RouterProvider router={router} />` — один компонент, который подключает роутер к приложению.​
    
---
## Навигация и ссылки

Для кликов по ссылкам нужно использовать компоненты и хуки React Router, чтобы не было перезагрузки страницы.​

### Основные инструменты:

- `<Link to="/about">О нас</Link>` — обычная ссылка в SPA.​
    
- `<NavLink>` — как `Link`, но умеет подсвечивать активный маршрут (для меню).​
    
- `useNavigate()` — программная навигация из кода (например, после успешного логина).​
    
### Пример с `useNavigate`:

```tsx
import { useNavigate } from "react-router-dom";

function LoginForm() {
  const navigate = useNavigate();

  const handleSuccessLogin = () => {
    navigate("/dashboard");
  };

  // ...
}
```

Важно: не использовать `<a href="/...">` для внутренних переходов — это ломает идею [[SPA]] и перезагружает страницу.​

---

## Вложенные роуты и layout‑компоненты

Вложенный роутинг позволяет описывать многоуровневые интерфейсы (layout + вложенные страницы) без дублирования разметки.

### Пример:

```tsx
// router
const router = createBrowserRouter([
  {
    path: "/",
    element: <AppLayout />, // header, footer, общий стиль
    children: [
      {
        path: "dashboard",
        element: <DashboardLayout />, // левое меню, панель
        children: [
          { index: true, element: <OverviewPage /> },
          { path: "reports", element: <ReportsPage /> },
          { path: "settings", element: <SettingsPage /> },
        ],
      },
    ],
  },
]);
```

Layout‑компонент должен рендерить `<Outlet />`, куда React Router подставит активный дочерний маршрут:​

```tsx
import { Outlet, NavLink } from "react-router-dom";

export function DashboardLayout() {
  return (
    <div className="dashboard">
      <aside>
        <NavLink to=".">Обзор</NavLink>
        <NavLink to="reports">Отчёты</NavLink>
        <NavLink to="settings">Настройки</NavLink>
      </aside>
      <main>
        <Outlet /> {/* сюда приходит конкретная страница */}
      </main>
    </div>
  );
}

```

Идея: layouts — это компоненты‑обёртки, которые определяют общий UI для группы страниц (dashboard, admin, public и т.д.).​

---
## Параметры маршрута и URL‑state

Параметры маршрута (`/users/:id`) и строка запроса (`?page=2`) позволяют передавать [[Состояние (State)|состояние]] через [[URL]].

Пример динамического маршрута:

```tsx
const router = createBrowserRouter([
  {
    path: "/users/:userId",
    element: <UserPage />,
  },
]);
```

Внутри компонента:

```tsx
import { useParams, useSearchParams } from "react-router-dom";

function UserPage() {
  const { userId } = useParams<{ userId: string }>();
  const [searchParams] = useSearchParams();
  const tab = searchParams.get("tab") ?? "profile";

  // userId берём из URL, tab — из строки запроса
}
```

Рекомендация: всё, что важно для «ссылкопригодного» состояния (фильтры, сортировка, текущий объект) старайся хранить в URL, а не только в React‑state.​

---

## Data API: loaders и actions

Начиная с React Router 6.4+ появились **loaders** и **actions** — [[API]] для загрузки и изменения данных на уровне маршрутов, а не компонентов.

Идея:

- **loader** — асинхронная функция, которая запускается перед рендерингом маршрута и загружает данные.[](https://www.linkedin.com/pulse/understanding-loaders-actions-react-router-fetching-writing-alam-nhzpc)​
    
- **action** — функция для обработки POST/PUT/DELETE и др. (например, отправки формы).[](https://www.linkedin.com/pulse/understanding-loaders-actions-react-router-fetching-writing-alam-nhzpc)​
    
- `useLoaderData()` — читать данные внутри компонента.[](https://www.linkedin.com/pulse/understanding-loaders-actions-react-router-fetching-writing-alam-nhzpc)​
    
- `Form` — компонент, который отправляет данные в `action` без ручного `fetch`.
    

Пример маршрута с loader и action:

```tsx 
import {
  createBrowserRouter,
} from "react-router-dom";
import { UsersPage, usersLoader, usersAction } from "./pages/UsersPage";

export const router = createBrowserRouter([
  {
    path: "/users",
    element: <UsersPage />,
    loader: usersLoader,
    action: usersAction,
  },
]);
```

```tsx 
// pages/UsersPage.tsx
import { useLoaderData, Form, redirect } from "react-router-dom";

type User = { id: string; name: string };

export async function usersLoader() {
  const res = await fetch("/api/users");
  const users: User[] = await res.json();
  return { users };
}

export async function usersAction({ request }: { request: Request }) {
  const formData = await request.formData();
  const name = formData.get("name");

  await fetch("/api/users", {
    method: "POST",
    body: JSON.stringify({ name }),
  });

  return redirect("/users");
}

export function UsersPage() {
  const { users } = useLoaderData() as { users: User[] };

  return (
    <div>
      <ul>
        {users.map((u) => (
          <li key={u.id}>{u.name}</li>
        ))}
      </ul>

      <Form method="post">
        <input name="name" placeholder="New user" />
        <button type="submit">Create</button>
      </Form>
    </div>
  );
}
```

Почему это удобно:

- данные грузятся до рендера, можно показывать skeleton/loader, пока выполняется loader;
    
- логика работы с данными вынесена из компонентов, проще тестировать и переиспользовать;[](https://www.linkedin.com/pulse/understanding-loaders-actions-react-router-fetching-writing-alam-nhzpc)​
    
- лучше интегрируется с серверным рендерингом и React Server Components.[](https://reactrouter.com/start/framework/data-loading)​
    

---

## Lazy loading маршрутов

[[Lazy loading]] уменьшает стартовый бандл: код для отдельных страниц подгружается только при первом переходе.

На уровне React:

```tsx
import React, { Suspense, lazy } from "react";
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const HomePage = lazy(() => import("./pages/HomePage"));
const AboutPage = lazy(() => import("./pages/AboutPage"));

const router = createBrowserRouter([
  {
    path: "/",
    element: (
      <Suspense fallback={<div>Загрузка…</div>}>
        <HomePage />
      </Suspense>
    ),
  },
  {
    path: "/about",
    element: (
      <Suspense fallback={<div>Загрузка…</div>}>
        <AboutPage />
      </Suspense>
    ),
  },
]);
```

В новых версиях React Router есть специальное поле `lazy` для роутов (поддерживает data‑router и loaders).[](https://github.com/remix-run/react-router/discussions/10322)​

Пример идеи:

```tsx 
{
  path: "login",
  lazy: () =>
    import("./routes/auth").then(({ LoginPage }) => ({
      Component: LoginPage,
    })),
}
```

Практический совет:  
группируй ленивые роуты логически (auth‑блок, dashboard‑блок, marketing‑страницы), чтобы не создавать слишком много мелких чанков.[](https://github.com/remix-run/react-router/discussions/10322)​

---

## Ошибки и NotFound‑страницы

Data‑router позволяет настраивать error‑UI на уровне маршрута.[](https://reactrouter.com/)​

- `errorElement` — компонент, который покажется, если loader/action бросили ошибку.[](https://reactrouter.com/)​
    
- `useRouteError()` — хук для извлечения ошибки внутри error‑компонента.[](https://reactrouter.com/)​
    
- отдельный маршрут `"*"` — классическая 404‑страница.[](https://www.geeksforgeeks.org/reactjs/how-to-setup-react-router-v6/)​
    

Пример:

```tsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <AppLayout />,
    errorElement: <RootErrorBoundary />,
    children: [
      { index: true, element: <HomePage /> },
      { path: "*", element: <NotFoundPage /> },
    ],
  },
]);
```

---

## Миграция и версия React Router

На сегодня в продакшене актуальна линейка **React Router 6.22+** с постепенным переходом к v7 через `future`‑флаги.

Если ты раньше видел код с `<Switch>`, `component`/`render` и `exact`, это был v5 — в v6/v7 используется `Routes`, `element` и иная модель матчей; при чтении статей всегда смотри, под какую версию написан пример.[](https://dev.to/vishwark/react-router-v6-the-complete-guide-to-whats-new-with-v5-vs-v6-code-examples-1955)​