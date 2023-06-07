[⬅️ Back to Documentation overview](/nova-calendar)

---

# Event filters

Event filters allow the end user to view a pre-defined subset of events in a calendar. When you add event filters to your calendar data provider, a filter icon is shown above the calendar view on the right, allowing the user to pick one, or show all events.

Event filters do *not* combine; the user either applies 1 event filter, or none.

## Table of Contents
- [Adding event filters to the calendar](#adding-event-filters-to-the-calendar)
- [Available filter types](#available-filter-types)
- [Customization options](#customization-options)
  - [Customizing the 'Show all' label](#customizing-the-show-all-label)
  - [Setting a default event filter](#setting-a-default-event-filter)

## Adding event filters to the calendar
Add the following method to the calendar data provider of the calendar that you want to add filters to:

```php
    public function filters() : array
    {
        return [
        
        ];
    }
```

Then, add one or more instances of the following filters to the array in the order that you want them to be shown to the end user.

* [`NovaResourceFilter`](#novaresourcefilter) to show only Nova resources of one or more specific classes
* [`ExcludeNovaResourceFilter`](#excludenovaresourcefilter) to exclude Nova resources of one or more specific classes
* [`CustomFilter`](#excludenovaresourcefilter) to define your own filtering logic using a callback method
* A [custom event filter](#custom-event-filters) subclass that you implement yourself for better code organization and filter reusability across calendar data providers

The first argument to a filter constructor is always the filter label, the rest of the arguments define how the filter works.

## Available filter types

### `NovaResourceFilter`
Show only calendar events for Nova resources of one or more specific classes.

```php
use Wdelfuego\NovaCalendar\EventFilter\NovaResourceFilter;

use App\Nova\User as NovaUser;
use App\Nova\Flight as NovaFlight;

public function filters() : array
{
    return [
        // Show only resources of one specific class
        new NovaResourceFilter(__('Only users'), NovaUser::class),

        // To show only resources that have one of multiple classes
        new NovaResourceFilter(__('Only users and flights'), [NovaUser::class, NovaFlight::class]),
    ];
}
```

Optionally, you can supply a callback function as a third argument to make the filter stricter.


```php
// Show only users that have a gmail address
new NovaResourceFilter(__('Only gmail users'), NovaUser::class, function($event) { return str_ends_with($event->model()->email, 'gmail.com'); })),
```

The callback method will receive the calendar event `$event`; use the `resource()` method to get the related Nova resource or the `model()` method for the underlying Eloquent model. In the case of a `NovaResourceFilter`, the event is guaranteed to contain both a Nova resource and an Eloquent model before the callback method gets called, so you don't need to do any checks.


### `ExcludeNovaResourceFilter`
Exclude calendar events for Nova resources of one or more specific classes.

```php
use Wdelfuego\NovaCalendar\EventFilter\ExcludeNovaResourceFilter;

use App\Nova\User as NovaUser;
use App\Nova\Flight as NovaFlight;

public function filters() : array
{
    return [
        // Hide resources of one specific class
        new ExcludeNovaResourceFilter(__('Hide users'), NovaUser::class),

        // Hide resources that have one of multiple classes
        new ExcludeNovaResourceFilter(__('Hide users and flights'), [NovaUser::class, NovaFlight::class]),
    ];
}
```

Just like with a normal `NovaResourceFilter`, you can supply a callback function to the `ExcludeNovaResourceFilter` to make the filter stricter. 

```php
// Hide only users that have a gmail address
new ExcludeNovaResourceFilter(__('Hide gmail users'), NovaUser::class, function($event) { return str_ends_with($event->model()->email, 'gmail.com'); })),
```

As with a `NovaResourceFilter`, the callback method receives the calendar event `$event`, so you can use the `resource()` method to get the related Nova resource or the `model()` method for the underlying Eloquent model, and the event is guaranteed to contain both a Nova resource and an Eloquent model before the callback method gets called, so you don't need to do any checks.



### `CustomFilter`
Define your own custom filtering logic using just a callback method.

```php
use Wdelfuego\NovaCalendar\EventFilter\CustomFilter;

public function filters() : array
{
    return [
        // Only show events that have an underlying Eloquent model that has an even id
        new CustomFilter(__('Eloquent models with an even id'), function($event) { return $event->model() && $event->model()->id % 2 == 0; }),
    ];
}
```

Since this filter is applied to any event on your calendar, the event is *not* strictly guaranteed to contain a Nova resource and an Eloquent model before the callback method gets called, so you need to do checks before using the `model()` and `resource()` methods.

In practice, this is only required if your `nonNovaEvents` method returns calendar events, because otherwise all events on your calendar have underlying Nova resources and Eloquent models anyway, but not doing those checks could cause problems in the future if you start using the `nonNovaEvents` method without checking your filter implementations.

### Custom event filters
Writing complex filtering logic into a `CustomFilter` callback method gets ugly quickly, so it's best to define custom filtering logic in your own filter classes. That has the added advantage of being able to reuse the filter across different calendar data providers and it's better for testing.

Implement a subclass of `Wdelfuego\NovaCalendar\EventFilter\AbstractEventFilter`. The only method that you have to implement is `showEvent(Event $event): bool`. Implement it however you see fit and you can add an instance of the custom filter class to the `filters()` method in your CalendarDataProvider.

The inline `CustomFilter` example above, that only shows Eloquent models with an even `id`, would be implemented as a custom filter class as follows:

```php
use Wdelfuego\NovaCalendar\EventFilter\AbstractEventFilter;
use Wdelfuego\NovaCalendar\Event;

class ModelHasEvenIdFilter extends AbstractEventFilter
{
    public function showEvent(Event $event): bool
    {
        return $event->model() && $event->model()->id % 2 == 0;
    }
}

```

Then in your CalendarDataProvider's `filters()` method:
```php
public function filters() : array
{
    return [
        // Only show events that have an underlying Eloquent model that has an even id
        new ModelHasEvenIdFilter(__('Eloquent models with an even id')),
    ];
}
```

## Customization options

### Customizing the 'Show all' label
When your calendar offers event filters and the end user has an active filter, the filter menu shows an option to reset the filter and show all events. The default label used for that option is 'Show all' (localized).

To show a custom label here, implement the `resetFiltersLabel()` method in your calendar data provider:

```php
public function resetFiltersLabel() : string
{
    return __('My custom show all label');
}
```

### Setting a default event filter
When a user opens the calendar view for the first time, the default behavior is to show all events. If you want the calendar to open with an active filter by default, add a call to `useAsDefaultFilter()` to one of the filters returned by `filters()` in your calendar data provider:

```php
// Show only resources of one specific class
(new NovaResourceFilter(__('Only users'), NovaUser::class))->useAsDefaultFilter(),
```

The `useAsDefaultFilter` method has an optional boolean argument that when it's `false` will not set the filter as the default, so you can easily make the behavior dynamic if you want.

Note that as of version 2.0 of this package, the calendar stores it's state in localStorage and resumes it when you open it again. That means the default filter will not be applied if you've visited the calendar before and changed the filter settings.

---

[⬅️ Back to Documentation overview](/nova-calendar)