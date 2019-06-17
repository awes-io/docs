# Creating first section

After [installing Awes-io](https://github.com/awes-io/awes-io#installation) you can create your first `section`.

Sections are main building blocks for a project based on Awes-io platform. Let's create one and inspect its directory structure.

To generate new section we can use `awes-io/generator` package and its Artisan command:

```bash
php artisan make:section Leads
```

This command will place new controller, model, repository and resource classes within your `app/Sections/Leads` directory. As well as index blade template in `resources/views/sections/leads`.

Let's also define basic routes to controller actions like so:

```php
// routes/web.php
Route::namespace('\App\Sections\Leads\Controllers')->prefix('leads')->group(function () {
    Route::get('/', 'LeadController@index')->name('leads.index');
    Route::get('scope', 'LeadController@scope')->name('leads.scope');
});
```

And migration for new leads table:

```php
// database/migrations/XXXX_XX_XX_XXXXXX_create_leads_table.php
...
Schema::create('leads', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->string('name');
    $table->timestamps();
});
...
```

After migrating, we can access new section on `localhost/leads` and see basic UI:

<img src="https://static.awes.io/docs/guide/01_basic_ui.png" alt="Awes.io">

New localization file will also be created within the `resources/lang` directory, thanks to `awes-io/localization-helper` package, which we'll discuss later.

## Controller

Let's check out the newly generated controller. It includes several methods, firstly `constructor`, in which we're injecting repository (more on this later):

```php
public function __construct(LeadRepository $leads)
{
    $this->leads = $leads;
}
```

`Index` method, where we return auto-generated blade template, as well as page title and all leads:

```php
public function index(Request $request)
{
    return view('sections.leads.index', [
        'h1' => _p('pages.leads.h1', 'leads'),
        'leads' => $this->scope($request)->response()->getData()
    ]);
}
```

`_p($file_key, $default, $placeholders)` is a helper function from our localization package, [more info is available in documentation](https://github.com/awes-io/localization-helper).

Scope method, which is used to retrieve, filter and sort lead records (more on this later):

```php
public function scope(Request $request)
{
    return Lead::collection(
        $this->leads->scope($request)->smartPaginate()
    );
}
```

We're also paginating results and transforming response data via auto-generated API resource class.

## Model

`Lead` model class is pretty simple, all we have is `$orderable` property, where we add all attributes we want to sort lead records by. We will discuss sorting later, but more info can be found in [awes-io/repository package documentation](https://github.com/awes-io/repository#scope-filter-and-order)

```php
public $orderable = [];
```

## Repository

`App\Sections\Leads\Repositories\LeadRepository` class includes `$searchable` property and `entity()` method, which returns full repository model class name.

```php
protected $searchable = [];

public function entity()
{
    return Lead::class;
}
```

We will discuss repositories and their filtering and ordering features later in detail. All documentation on this topic is [available here](https://github.com/awes-io/repository)

## API resource

`App\Sections\Leads\Resources\Lead` is just a basic [Eloquent API Resource](https://laravel.com/docs/master/eloquent-resources)

## View

Auto-generated `index` template is located in `resources/views/sections/leads`.

It extends main layout of `awes-io/indigo-layout` [package which contains useful styles and blade components](https://github.com/awes-io/indigo-layout) for fast building of responsive UI.

```php
@extends('indigo-layout::main')
```

Next, there are meta title and description sections:

```php
@section('meta_title', _p('pages.leads.meta_title', 'meta_title') . ' // ' . config('app.name'))
@section('meta_description', _p('pages.leads.meta_description', 'meta_description'))
```

Here we can see how `_p()` helper localization function is used. First argument is the file.key string, if we open newly created `resources/lang/en/pages.php` file we'll see different translation strings:

```php
<?php

return [
    "leads" => [
        "h1" => "leads",
        "meta_title" => "meta_title",
        "meta_description" => "meta_description",
        ...
    ]
];
```

The second argument is a default value, which will be added to the localization file if key doesn't exist. Thus, by the time the interface build process is completed, you will have a complete base language file, ready for translation.

Further in the template we can see `create_button` section, which determines how "floating" action button looks like:

```html
@section('create_button')
    <button class="frame__header-add" @click="AWES.emit('modal::leads.open')"><i class="icon icon-plus"></i></button>
@endsection
```

Next goes `group filter` component:

```php
@filtergroup(['filter' => ['' => 'All', '1' => 'Public', '0' => 'Private'], 'variable' => 'is_public'])
```

<img src="https://static.awes.io/docs/guide/02_filter_group_component.png" alt="Awes.io">

If we click on one of the filters, we'll see that it modifies `is_public` URL's parameter to respective value. Our component will track these changes and send server requests to get filtered data. We will return to this topic when we'll create additional filters ourselves.

After filters, we can see one more very useful component - context menu, which in this case, controls ordering our leads by their names. 

```html
<context-menu button-class="filter__slink" right>
    <template slot="toggler">
        <span>{{  _p('pages.filter.sort_by', 'Sort by') }}</span>
    </template>
    <cm-query :param="{orderBy: 'name'}">{{ _p('pages.leads.filter.name', 'Name') }} &darr;</cm-query>
    <cm-query :param="{orderBy: 'name_desc'}">{{ _p('pages.leads.filter.name', 'Name') }} &uarr;</cm-query>
</context-menu>
```

Sorting is another topic we'll analyze in the future, when we'll build our custom filters based on repositories.

Next is the button to open main filters panel, for now, it only displays one parameter, but we'll add more, later:

```html
<button class="filter__slink" @click="$refs.filter.toggle()">
    <i class="icon icon-filter" v-if="">
        <span class="icn-dot" v-if="$awesFilters.state.active['leads']"></span>
    </i>
    {{  _p('pages.filter.title', 'Filter') }}
</button>
...
<slide-up-down ref="filter">
    <filter-wrapper name="leads">
        <div class="grid grid-gap-x grid_forms">
            <div class="cell">
                <fb-input name="name" label="{{ _p('pages.leads.filter.name', 'Name') }}"></fb-input>
            </div>
        </div>
    </filter-wrapper>
</slide-up-down>
```
<img src="https://static.awes.io/docs/guide/03_main_filters.png" alt="Awes.io">

Next, we see the usage of one of our most powerful component - Table Builder, which powers `@table` blade component for easy interactive tables set up:

```php
@table([
    'name' => 'leads',
    'row_url' => route('leads.index') . '/{id}',
    'scope_api_url' => route('leads.scope'),
    'scope_api_params' => ['orderBy', 'is_public', 'name'],
    'default_data' => $leads
])
...
```

And the last one is the modal window with leads creation form:

```html
<modal-window name="leads" class="modal_formbuilder" title="Create">
    <form-builder url="" :disabled-dialog="true">
        <fb-input name="name" label="{{ _p('pages.leads.modal_add.name', 'Name') }}"></fb-input>
    </form-builder>
</modal-window>
```

We'll inspect Modal Window and Form Builder components in greater detail later when we'll update our current project.

# Let's build something new

We got a closer look at the general structure of a generated section. It's time to build something new and get into the platform's details.

## Improving existing filters

Let's go back to our group filter and update it to display leads with different statuses.

Firstly we need to create new migration and add the `status` column to our `leads` table:

```php
// database/migrations/XXXX_XX_XX_XXXXXX_add_status_to_leads_table.php
...
Schema::table('leads', function (Blueprint $table) {
    $table->string('status');
});
...
```
After migrating, let's add directly to database table couple records with statuses `new` and `closed`:

<img src="https://static.awes.io/docs/guide/04_leads_with_statuses.png" alt="Awes.io">

Now if we refresh `/leads` page, we'll see that we need to add a new column to the table in order to see which status lead currently has:

<img src="https://static.awes.io/docs/guide/05_leads_without_statuses_table.png" alt="Awes.io">

It's time to slightly dive into `table-builder` package functionality. Firstly let's add new `status` column:

```html
<tb-column name="name" label="{{ _p('pages.leads.table.col.name', 'Name') }}"></tb-column>
<tb-column name="status" label="{{ _p('pages.leads.table.col.status', 'Status') }}"></tb-column>
```

As you can see, we added a new `tb-column` tag. `Name` parameter is a key in the data object, which is just our `status` column name. And we're passing new language string to a label property, as we discussed earlier.

Now if we refresh lead's page, we'll see their statuses:

<img src="https://static.awes.io/docs/guide/06_leads_with_statuses_table.png" alt="Awes.io">

Let's go back to the group filter and slightly edit it to retrieve records by status:

```php
// @filtergroup(['filter' => ['' => 'All', '1' => 'Public', '0' => 'Private'], 'variable' => 'is_public'])
@filtergroup(['filter' => ['' => 'All', 'new' => 'New', 'closed' => 'Closed'], 'variable' => 'status'])
```

Normally we'd have to implement some filtering logic, but thanks to `awes-io/repository` package, all we need to do is to add `status` parameter to `searchable` property in `App\Sections\Leads\Repositories\LeadRepository`:

```php
protected $searchable = ['status'];
```

and add `status` to `@table`'s `scope_api_params` property (this will allow the component to track any changes in parameter value and handle them respectively):

```php
// 'scope_api_params' => ['orderBy', 'is_public', 'name'],
'scope_api_params' => ['orderBy', 'is_public', 'name', 'status'],
```

now if we click on filter option, request with `status` parameter will be sent to the server and `repository` package will filter data and return it for `table-builder` to render:

<img src="https://static.awes.io/docs/guide/07_group_filter_by_status.png" alt="Awes.io">

It's that easy, more info on filtering can be found in [awes-io/repository documentation](https://github.com/awes-io/repository).

## Implementing new filters

We've updated the existing filter, but what if we want to build a custom filter of some kind for our lead management UI?

## New sorting

## Customizing table

## Creating and updating leads