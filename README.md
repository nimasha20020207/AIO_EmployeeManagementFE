# AIO Employee Management

Vue 3, TypeScript, and Vite frontend for the Laravel employee payroll API.

## Run locally

Start Laravel from `AIO_EmployeeManagementBE` after configuring MySQL and running `php artisan migrate --seed`:

```powershell
php artisan serve
```

Then start the frontend from this project:

```powershell
npm install
npm run dev
```

Vite proxies `/api` requests to Laravel at `http://127.0.0.1:8000`. Set `VITE_API_BASE_URL` to override the API base URL.

## Features

- Add employee details and select Intern, Associate, Senior, or Manager designation.
- Display salary package, calculated monthly tax, yearly bonus, monthly net salary, and yearly net salary.
- Edit employee phone and monthly salary; derived payroll values are recalculated by Laravel.
- Delete employees and search by phone.
- Export payroll details as CSV.

## Scripts

- `npm run dev` — start the development server
- `npm run build` — type-check and create a production build
- `npm run preview` — preview the production build
