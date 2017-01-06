Updating Plugins for Craft 3
============================

The general concept of plugins and plugin-supplied component types remains mostly the same in Craft 3, but all the implementation details have changed.

Before reading through this guide, make sure you’ve read through [Upgrading from Craft 2](upgrade.md) and [Intro to Plugin Dev](plugin-intro.md).

- [High Level Notes](#high-level-notes)
- [Service Names](#service-names)
- [Translations](#translations)
- [DB Queries](#db-queries)
  - [Table Names](#table-names)
  - [Select Queries](#select-queries)
  - [Operational Queries](#operational-queries)
- [Files](#files)
- [Plugin Hooks](#plugin-hooks)
  - [General Hooks](#general-hooks)
  - [Routing Hooks](#routing-hooks)
  - [Element Hooks](#element-hooks)
  
## High Level Notes

- Plugins must now have a `composer.json` file that defines some basic info about the plugin.
- Plugins now get their own root namespace, rather than sharing a `Craft\` namespace with all of Craft and other plugins, and all Craft and plugin code must follow the [PSR-4](http://www.php-fig.org/psr/psr-4/) specification.
- Plugins are now an extension of [Yii modules](http://www.yiiframework.com/doc-2.0/guide-structure-modules.html).
- The main application instance is available via `Craft::$app` now, rather than `craft()`.

## Service Names

The following core service names have changed:

Old             | New
--------------- | ----------------
`assetSources`  | `volumes`
`email`         | `mailer`
`templateCache` | `templateCaches`
`templates`     | `view`
`userSession`   | `user`

## Translations

`Craft::t()` requires a `$category` argument now, which should be set to one of these translation categories:

- `yii` for Yii translation messages
- `app` for Craft translation messages
- `site` for front-end translation messages
- a plugin handle for plugin-specific translation messages

```php
Craft::t('app', 'Entries')
```

In addition to front-end translation messages, the `site` category should be used for admin-defined labels in the Control Panel:

```php
Craft::t('app', 'Post a new {section} entry', [
    'section' => Craft::t('site', $section->name)
])
```

To keep front-end Twig code looking clean, the `|t` and `|translate` filters don’t require that you specify the category, and will default to `site`. So these two tags will give you the same output:

```twig
{{ "News"|t }}
{{ "News"|t('site') }}
```

## DB Queries

### Table Names

Craft no longer auto-prepends the DB table prefix to table names, so you must write table names in Yii’s `{{%tablename}}` syntax. 

### Select Queries

Select queries are defined by `craft\db\Query` classes now.

```php
$results = (new Query())
    ->select(['column1', 'column2'])
    ->from(['{{%tablename}}'])
    ->where(['foo' => 'bar'])
    ->all();
```

### Operational Queries

Operational queries can be built from the helper methods on `craft\db\Command` (accessed via `Craft::$app->db->createCommand()`), much like the `Craft\DbCommand` class in Craft 2.

One notable difference is that the helper methods no longer automatically execute the query, so you must chain a call to `execute()`.

```php
$result = Craft::$app->db->createCommand()
    ->insert('{{%tablename}}', $rowData)
    ->execute();
```

## Files

- `Craft\IOHelper` has been replaced with `craft\helpers\FileHelper`, which extends Yii’s `yii\helpers\BaseFileHelper`.
- Directory paths returned by `craft\helpers\FileHelper` and `craft\services\Path` methods no longer include a trailing slash.
- File system paths in Craft now use the `DIRECTORY_SEPARATOR` PHP constant (which is set to either `/` or `\` depending on the environment) rather than hard-coded forward slashes (`/`).

## Plugin Hooks

The concept of “plugin hooks” has been removed in Craft 3. Here’s a list of the previously-supported hooks and how you should accomplish the same things in Craft 3: 

### General Hooks

#### `addRichTextLinkOptions`

```php
// Old:
public function addRichTextLinkOptions()
{
    return [
        [
            'optionTitle' => Craft::t('Link to a product'),
            'elementType' => 'Commerce_Product',
        ],
    ];
}

// New:
yii\base\Event::on(
    craft\fields\RichText::class,
    craft\fields\RichText::EVENT_REGISTER_LINK_OPTIONS,
    function(craft\events\RegisterRichTextLinkOptionsEvent $event) {
        $event->linkOptions[] = [
            'optionTitle' => Craft::t('commerce', 'Link to a product'),
            'elementType' => Product::class,
        ];
    }
);
```

#### `addTwigExtension`

```php
// Old:
public function addTwigExtension()
{
    Craft::import('plugins.cocktailrecipes.twigextensions.MyExtension');
    return new MyExtension();
}

// New:
Craft::$app->view->twig->addExtension(new MyExtension);
```

#### `addUserAdministrationOptions`

```php
// Old:
public function addUserAdministrationOptions(UserModel $user)
{
    if (!$user->isCurrent()) {
        return [
            [
                'label'  => Craft::t('Send Bacon'),
                'action' => 'baconater/sendBacon'
            ],
        ];
    }
}

// New:
yii\base\Event::on(
    craft\controllers\UsersController::class,
    craft\controllers\UsersController::EVENT_REGISTER_USER_ACTIONS,
    function(craft\events\RegisterUserActionsEvent $event) {
        if ($event->user->isCurrent) {
            $event->miscActions[] = [
                'label' => Craft::t('baconater', 'Send Bacon'),
                'action' => 'baconater/send-bacon'
            ];
        }
    }
);
```

#### `getResourcePath`

```php
// Old:
public function getResourcePath($path)
{
    if (strpos($path, 'myplugin/') === 0) {
        return craft()->path->getStoragePath().'myplugin/'.substr($path, 9);
    }
}

// New:
yii\base\Event::on(
    craft\services\Resources::class,
    craft\services\Resources::EVENT_RESOLVE_RESOURCE_PATH,
    function(craft\events\ResolveResourcePathEvent $event) {
        if (strpos($event->uri, 'myplugin/') === 0) {
            $event->path = Craft::$app->path->getStoragePath().'/myplugin/'.substr($event->uri, 9);
            
            // Prevent other event listeners from getting invoked
            $event->handled = true;
        }
    }
);
```

#### `modifyCpNav`

```php
// Old:
public function modifyCpNav(&$nav)
{
    if (craft()->userSession->isAdmin()) {
        $nav['utils'] = ['label' => 'Utils', 'url' => 'utils'];
    }
}

// New:
yii\base\Event::on(
    craft\web\twig\variables\Cp::class,
    craft\web\twig\variables\Cp::EVENT_REGISTER_CP_NAV_ITEMS,
    function(craft\events\RegisterCpNavItemsEvent $event) {
        if (Craft::$app->user->identity->admin) {
            $event->navItems['utils'] = ['label' => 'Utils', 'url' => 'utils'];
        }
    }
);
```

#### `registerCachePaths`

```php
// Old:
public function registerCachePaths()
{
    return [
        craft()->path->getStoragePath().'drinks/' => Craft::t('Drink images'),
    ];
}

// New:
yii\base\Event::on(
    craft\tools\ClearCaches::class,
    craft\tools\ClearCaches::EVENT_REGISTER_CACHE_OPTIONS,
    function(craft\events\RegisterCacheOptionsEvent $event) {
        $event->options[] = [
            'key' => 'drink-images',
            'label' => Craft::t('drinks', 'Drink images'),
            'action' => Craft::$app->path->getStoragePath().'/drinks'
        ];
    }
);
```

#### `registerEmailMessages`

```php
// Old:
public function registerEmailMessages()
{
    return ['custom_message_key'];
}

// New:
yii\base\Event::on(
    craft\services\EmailMessages::class,
    craft\services\EmailMessages::EVENT_REGISTER_MESSAGES,
    function(craft\events\RegisterEmailMessagesEvent $event) {
        $event->messages[] = [
           'key' => 'custom_message_key',
           'category' => 'myplugin',
           'sourceLanguage' => 'en-US'
       ];
    }
);
```

#### `registerUserPermissions`

```php
// Old:
public function registerUserPermissions()
{
    return [
        'drinkAlcohol' => ['label' => Craft::t('Drink alcohol')],
        'stayUpLate' => ['label' => Craft::t('Stay up late')],
    ];
}

// New:
yii\base\Event::on(
    craft\services\UserPermissions::class,
    craft\services\UserPermissions::EVENT_REGISTER_PERMISSIONS,
    function(craft\events\RegisterUserPermissionsEvent $event) {
        $event->permissions[Craft::t('vices', 'Vices')] = [
            'drinkAlcohol' => ['label' => Craft::t('vices', 'Drink alcohol')],
            'stayUpLate' => ['label' => Craft::t('vices', 'Stay up late')],
        ];
    }
);
```

#### `getCpAlerts`

```php
// Old:
public function getCpAlerts($path, $fetch)
{
    if (craft()->config->devMode) {
        return [Craft::t('Dev Mode is enabled!')];
    }
}

// New:
yii\base\Event::on(
    craft\helpers\Cp::class,
    craft\helpers\Cp::EVENT_REGISTER_ALERTS,
    function(craft\events\RegisterCpAlertsEvent $event) {
        if (Craft::$app->config->get('devMode')) {
            $event->alerts[] = Craft::t('myplugin', 'Dev Mode is enabled!');
        }
    }
);
```

#### `modifyAssetFilename`

```php
// Old:
public function modifyAssetFilename($filename)
{
    return 'KittensRule-'.$filename;
}

// New:
yii\base\Event::on(
    craft\helpers\Assets::class,
    craft\helpers\Assets::EVENT_SET_FILENAME,
    function(craft\events\SetElementTableAttributeHtmlEvent $event) {
        $event->filename = 'KittensRule-'.$event->filename;
            
        // Prevent other event listeners from getting invoked
        $event->handled = true;
    }
);
```

### Routing Hooks

#### `registerCpRoutes`

```php
// Old:
public function registerCpRoutes()
{
    return [
        'cocktails/new' => 'cocktails/_edit',
        'cocktails/(?P<widgetId>\d+)' => ['action' => 'cocktails/editCocktail'],
    ];
}

// New:
yii\base\Event::on(
    craft\web\UrlManager::class,
    craft\web\UrlManager::EVENT_REGISTER_CP_URL_RULES,
    function(craft\events\RegisterUrlRulesEvent $event) {
        $event->rules['cocktails/new'] = ['template' => 'cocktails/_edit'];
        $event->rules['cocktails/<widgetId:\d+>'] = 'cocktails/edit-cocktail';
    }
);
```

#### `registerSiteRoutes`

```php
// Old:
public function registerSiteRoutes()
{
    return [
        'cocktails/new' => 'cocktails/_edit',
        'cocktails/(?P<widgetId>\d+)' => ['action' => 'cocktails/editCocktail'],
    ];
}

// New:
yii\base\Event::on(
    craft\web\UrlManager::class,
    craft\web\UrlManager::EVENT_REGISTER_SITE_URL_RULES,
    function(craft\events\RegisterUrlRulesEvent $event) {
        $event->rules['cocktails/new'] = ['template' => 'cocktails/_edit'];
        $event->rules['cocktails/<widgetId:\d+>'] = 'cocktails/edit-cocktail';
    }
);
```

#### `getElementRoute`

```php
// Old:
public function getElementRoute(BaseElementModel $element)
{
    if (
        $element->getElementType() === ElementType::Entry &&
        $element->getSection()->handle === 'products'
    )
    {
        return ['action' => 'products/viewEntry'];
    }
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_SET_ROUTE,
    function(craft\elements\SetElementRouteEvent $event) {
        /** @var craft\elements\Entry $entry */
        $entry = $event->sender;

        if ($entry->section->handle === 'products') {
            $event->route = 'products/view-entry';
            
            // Prevent other event listeners from getting invoked
            $event->handled = true;
        }
    }
);
```

### Element Hooks

The following sets of hooks have been combined into single events that are shared across all element types.

For each of these, you could either pass `craft\base\Element::class` to the first argument of `yii\base\Event::on()` (registering the event listener for *all* element types), or a specific element type class (registering the event listener for just that one element type). 

#### `addEntryActions`, `addCategoryActions`, `addAssetActions`, & `addUserActions`

```php
// Old:
public function addEntryActions($source)
{
    return [new MyElementAction()];
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_REGISTER_ACTIONS,
    function(craft\events\RegisterElementActionsEvent $event) {
        $event->actions[] = new MyElementAction();
    }
);
```

#### `modifyEntrySortableAttributes`, `modifyCategorySortableAttributes`, `modifyAssetSortableAttributes`, & `modifyUserSortableAttributes`

```php
// Old:
public function modifyEntrySortableAttributes(&$attributes)
{
    $attributes['id'] = Craft::t('ID');
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_REGISTER_SORTABLE_ATTRIBUTES,
    function(craft\events\RegisterElementSortableAttributesEvent $event) {
        $event->sortableAttributes['id'] = Craft::t('app', 'ID');
    }
);
```

#### `modifyEntrySources`, `modifyCategorySources`, `modifyAssetSources`, & `modifyUserSources`

```php
// Old:
public function modifyEntrySources(&$sources, $context)
{
    if ($context == 'index') {
        $sources[] = [
            'heading' => Craft::t('Statuses'),
        ];

        $statuses = craft()->elements->getElementType(ElementType::Entry)
            ->getStatuses();
        foreach ($statuses as $status => $label) {
            $sources['status:'.$status] = [
                'label' => $label,
                'criteria' => ['status' => $status]
            ];
        }
    }
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_REGISTER_SOURCES,
    function(craft\events\RegisterElementSourcesEvent $event) {
        if ($event->context === 'index') {
            $sources[] = [
                'heading' => Craft::t('myplugin', 'Statuses'),
            ];

            $statuses = craft\elements\Entry::statuses();
            foreach ($statuses as $status => $label) {
                $sources[] = [
                    'key' => 'status:'.$status,
                    'label' => $label,
                    'criteria' => ['status' => $status]
                ];
            }
        }
    }
);
```

#### `defineAdditionalEntryTableAttributes`, `defineAdditionalCategoryTableAttributes`, `defineAdditionalAssetTableAttributes`, & `defineAdditionalUserTableAttributes`

```php
// Old:
public function defineAdditionalEntryTableAttributes()
{
    return [
        'foo' => Craft::t('Foo'),
        'bar' => Craft::t('Bar'),
    ];
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_REGISTER_TABLE_ATTRIBUTES,
    function(craft\events\RegisterElementTableAttributesEvent $event) {
        $event->tableAttributes['foo'] = ['label' => Craft::t('myplugin', 'Foo')];
        $event->tableAttributes['bar'] = ['label' => Craft::t('myplugin', 'Bar')];
    }
);
```

#### `getEntryTableAttributeHtml`, `getCategoryTableAttributeHtml`, `getAssetTableAttributeHtml`, & `getUserTableAttributeHtml`

```php
// Old:
public function getEntryTableAttributeHtml(EntryModel $entry, $attribute)
{
    if ($attribute === 'price') {
        return '$'.$entry->price;
    }
}

// New:
yii\base\Event::on(
    craft\elements\Entry::class,
    craft\base\Element::EVENT_SET_TABLE_ATTRIBUTE_HTML,
    function(craft\events\SetElementTableAttributeHtmlEvent $event) {
        if ($event->attribute === 'price') {
            /** @var craft\elements\Entry $entry */
            $entry = $event->sender;

            $event->html = '$'.$entry->price;

            // Prevent other event listeners from getting invoked
            $event->handled = true;
        }
    }
);
```

#### `getTableAttributesForSource`

```php
// Old:
public function getTableAttributesForSource($elementType, $sourceKey)
{
    if ($sourceKey == 'foo') {
        return craft()->elementIndexes->getTableAttributes($elementType, 'bar');
    }
}
```

> {note} There is no direct Craft 3 equivalent for this hook, which allowed plugins to completely change the table attributes for an element type right before the element index view was rendered. The closest thing in Craft 3 is the `craft\base\Element::EVENT_REGISTER_TABLE_ATTRIBUTES` event, which can be used to change the available table attributes for an element type when an admin is customizing the element index sources.