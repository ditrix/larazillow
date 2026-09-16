# Установка LaraZillow

Инструкция для локального запуска проекта на Laravel 11, Inertia и Vue 3 через Laravel Sail.

## Требования

- Docker Desktop
- Git
- Проект LaraZillow

Все команды ниже выполняются из корня проекта.

## Установка проекта

Если проект был клонирован из Git, установите PHP-зависимости внутри контейнера:

```bash
./vendor/bin/sail composer install
```

Для запуска Sail в проекте должны быть доступны `vendor/bin/sail` и Docker Desktop.

Создайте файл окружения и сгенерируйте ключ приложения:

```bash
cp .env.example .env
./vendor/bin/sail artisan key:generate
```

## Запуск Sail

Запустите контейнеры в фоновом режиме:

```bash
./vendor/bin/sail up -d
```

Проверьте состояние контейнеров:

```bash
./vendor/bin/sail ps
```

## База данных

Параметры базы данных в `.env` должны соответствовать сервису из `compose.yaml`. Для стандартной конфигурации Sail обычно используются:

```dotenv
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password
```

Выполните миграции:

```bash
./vendor/bin/sail artisan migrate
```

Для пересоздания базы данных с тестовыми данными:

```bash
./vendor/bin/sail artisan migrate:fresh --seed
```

## Inertia и Vue 3

Установите frontend-зависимости:

```bash
./vendor/bin/sail npm install
```

Если зависимости Inertia и Vue ещё не добавлены в `package.json`, установите их:

```bash
./vendor/bin/sail npm install vue @vue/compiler-sfc
./vendor/bin/sail npm install @inertiajs/vue3
./vendor/bin/sail npm install --save-dev @inertiajs/vite
```

### `resources/js/app.js`

Это главный JavaScript-файл приложения. Он подключает Bootstrap, запускает Inertia, находит Vue-страницу по имени и монтирует приложение в HTML-элемент, созданный директивой `@inertia`.

```js
import "./bootstrap";
import { createInertiaApp } from "@inertiajs/vue3";
import { resolvePageComponent } from "laravel-vite-plugin/inertia-helpers";
import { createApp, h } from "vue";

createInertiaApp({
	title: (title) => `${title} - Laravel`,
	resolve: (name) =>
		resolvePageComponent(
			`./Pages/${name}.vue`,
			import.meta.glob("./Pages/**/*.vue"),
		),
	setup({ el, App, props, plugin }) {
		createApp({ render: () => h(App, props) })
			.use(plugin)
			.mount(el);
	},
});
```

Основные части файла:

- `createInertiaApp` создаёт клиентское Inertia-приложение;
- `resolvePageComponent` ищет страницы в `resources/js/Pages`;
- `import.meta.glob` автоматически регистрирует все Vue-файлы в этой папке;
- `createApp(...).use(plugin).mount(el)` подключает Inertia к Vue и монтирует его в страницу.

### `vite.config.js`

Это конфигурация Vite. Laravel Vite Plugin обрабатывает CSS и JavaScript, а Vue Plugin позволяет Vite компилировать файлы `.vue`.

```js
import { defineConfig } from "vite";
import laravel from "laravel-vite-plugin";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
	plugins: [
		laravel({
			input: ["resources/css/app.css", "resources/js/app.js"],
			refresh: true,
		}),
		vue(),
	],
});
```

`input` указывает входные файлы сборки. Параметр `refresh: true` перезагружает страницу после изменения серверных файлов Laravel.

### `app/Http/Middleware/HandleInertiaRequests.php`

Middleware добавляет Inertia-обработку к web-маршрутам Laravel. Свойство `$rootView` указывает Blade-шаблон, в который Inertia вставляет Vue-приложение.

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Request;
use Inertia\Middleware;

class HandleInertiaRequests extends Middleware
{
	protected $rootView = 'app';

	public function version(Request $request): ?string
	{
		return parent::version($request);
	}

	public function share(Request $request): array
	{
		return [
			...parent::share($request),
		];
	}
}
```

Метод `share` используется для передачи общих данных во все Vue-страницы. Например, туда можно добавить текущего пользователя:

```php
return [
	...parent::share($request),
	'auth' => [
		'user' => $request->user(),
	],
];
```

### `resources/views/app.blade.php`

Это единственный Blade-шаблон-оболочка Inertia. Laravel отдаёт его при первом открытии страницы, после чего Vue и Inertia управляют навигацией без полной перезагрузки.

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
	<meta charset="utf-8" />
	<meta name="viewport" content="width=device-width, initial-scale=1">
	@vite(['resources/css/app.css', 'resources/js/app.js'])
	@inertiaHead
</head>
<body>
	@inertia
</body>
</html>
```

- `@vite` подключает CSS и JavaScript, собранные Vite;
- `@inertiaHead` выводит заголовок и другие head-элементы из Vue-компонента;
- `@inertia` создаёт DOM-элемент, в который монтируется Vue-приложение.

### `resources/js/Pages/Home.vue`

Это Vue-страница, которую Laravel открывает через `Inertia::render('Home')`. Имя `Home` соответствует файлу `Home.vue` в папке `resources/js/Pages`.

```vue
<script setup>
import { Head } from "@inertiajs/vue3";
</script>

<template>
	<Head title="Home" />

	<main class="min-h-screen bg-slate-950 px-6 py-16 text-white">
		<div class="mx-auto max-w-5xl">
			<p class="text-sm font-semibold uppercase tracking-[0.3em] text-emerald-400">
				LaraZillow
			</p>
			<h1 class="mt-6 max-w-3xl text-5xl font-bold tracking-tight sm:text-7xl">
				Find a place that feels like home.
			</h1>
			<p class="mt-6 max-w-xl text-lg leading-8 text-slate-300">
				Your Laravel 11, Inertia and Vue 3 application is ready.
			</p>
		</div>
	</main>
</template>
```

`<script setup>` содержит логику компонента, а `<template>` его HTML-разметку. Компонент `Head` меняет заголовок вкладки браузера через Inertia.

### `routes/web.php`

Маршрут связывает URL Laravel с Vue-страницей:

```php
<?php

use Inertia\Inertia;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
	return Inertia::render('Home');
});
```

При запросе `GET /` Laravel не возвращает отдельный Blade-файл. `Inertia::render('Home')` передаёт Inertia имя страницы, после чего `app.js` загружает `resources/js/Pages/Home.vue`.

## Запуск frontend

Для разработки запустите Vite:

```bash
./vendor/bin/sail npm run dev
```

В отдельном терминале запустите Laravel, если он не входит в общий dev-скрипт:

```bash
./vendor/bin/sail artisan serve --host=0.0.0.0
```

Откройте приложение по адресу [http://localhost](http://localhost).

Для production-сборки используйте:

```bash
./vendor/bin/sail npm run build
```

## Полезные команды

```bash
./vendor/bin/sail artisan route:list
./vendor/bin/sail artisan view:clear
./vendor/bin/sail artisan config:clear
./vendor/bin/sail artisan test
./vendor/bin/sail down
```

Если контейнеры нужно пересобрать после изменения Docker-конфигурации:

```bash
./vendor/bin/sail build --no-cache
./vendor/bin/sail up -d
```
