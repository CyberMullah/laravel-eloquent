# 🔥 🚀 Laravel Eloquent Tips

This is a shortlist of the amazing hidden Laravel eloquent 30 tips that make the code go on smoothly.

## 1 – Invisible Database Columns

The invisible column is a new concept in MySQL 8. What it does: when you run a `select *` query it won't retrieve any invisible column. If you need an invisible column's value you have to specify it explicitly in the `select` statement.

And now, Laravel supports these columns:

```php
Schema::table('users', function (Blueprint $table){
  $table->string('password')->invisble();
});

$user = User::first();
$user->secret == null;
```

---

## 2 – saveQuietly

If you ever need to save a model but you don't want to trigger any model events, you can use this method:

```php
$user = User::first();
$user->name = "Hamid Afghan";

$user->saveQuietly();
```

---

## 3 – Default Attribute Values

In Laravel, you can define default values for columns in two places: Migrations and models.

```php
Schema::create('orders', function(Blueprint $table){
  $table->bigIncrements('id');
  $table->string('status', 20)
    ->nullable(false)
    ->default(App\Enums\OrderStatuses::DRAFT);
});
```

This is a well-known feature. The status column will have a default draft value.

But what about this?

```php
$order = new Order();
$order->status = null;
```

In this case, the status will be null, because it's not persisted yet. And sometimes it causes annoying null value bugs. But fortunately, you can specify default attribute values in the Model as well:

```php
class Order extends Model
{
  protected $attributes = [
    'status' => App\Enums\OrderStatuses::DRAFT,
  ];
}
```

And now the status will be draft for a New Order:

```php
$order = new Order();
$order->status  === 'draft';
```

You can use these two approaches together and you'll never have a null value bug again.

---

## 4 – Attribute Cast

Before Laravel 8. x we wrote attribute accessors and mutators like these:

```php
class User extends Model{
  public function getNameAttribute(string $value): string
  {
      return Str::upper($value);
  }

  public function setNameAttribute(string $value): string
  {
      $this->attributes['name'] = Str::lower($value);
  }
}
```

It's not bad at all, but as Taylor says in the pull request:

> This aspect of the framework has always felt a bit "dated" to me. To be honest, I think it's one of the least elegant parts of the framework that currently exists. First, it requires two methods. Second, the framework does not typically prefix methods that retrieve or set data on an object with get and set

So he recreated this feature this way:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model {
  protected function name(): Attribute {
    return new Attribute(
       get: fn (string $value) => Str::upper($value),
       set: fn (string $value) => Str::lower($value)
    );
  }
}
```

The main differences:

- You have to write only one method
- It returns an Attribute instead of a scalar value
- The Attribute itself takes a getter and a setter function

In this example, I used PHP 8 named arguments (the get and set before the functions).

---

## 5 – find

Everyone knows about the find method, but did you know that it accepts an array of IDs? So instead of this:

```php
$users = User::whereIn('id', $ids)->get();
```

You can use this:

```php
$users = User::find($ids);
```

---

## 6 – Get Dirty

In Eloquent you can check if a model is "dirty" or not. Dirty means it has some changes that are not persisted yet:

```php
$user = User::first();
$user->name = 'Hamid Afghan';
$user->isDirty() === true;
$user->getDirty === ['name' => 'Guest User'];
```

The `isDirty` simply returns a bool while the `getDirty ` returns every dirty attribute.

---

## 7 – push

Sometimes you need to save a model and its relationship as well. In this case, you can use the push method:

```php
$employee = Employee::first();
$employee->name = 'New Name';
$employee->address->city = 'New York';

$employee->push();
```

In this case, the, save would only save the name column in the employee's table but not the city column in the addresses table. The push method will save both.

---

## 8 – Boot Eloquent Traits

We all write traits that are being used by Eloquent models. If you need to initialize something in your trait when an event happened in the model, you can boot your trait.

For example, if you have models with slug, you don't want to rewrite the slug creation logic in every model. Instead, you can define a trait, and use the creating event in the boot method:

```php
trait HasSlug {
  public static function bootHasSlug() {
      static::creating(function (Model $model) {
        $model->slug = Str::slug($model->title);
      });
  }
}
```

So you need to define a bootTraitName method, and Eloquent will automatically call this when it's booting a model.

---

## 9 – updateOrCreate

Creating and updating a model often use the same logic. Fortunately Eloquent provides a very convenient method called updateOrCreate:

```php
$flight = Flight::updateOrCreate(
  ['id' => $id],
  ['price' => 99, 'discounted' => 1],
);
```

It takes two arrays:

- The first one is used to determine if the model exists or not. In this example, I use the id.
- The second one is the attributes that you want to insert or update.

And the way it works:

- If a Flight is found based on the given id it will be updated with the second array.
- If there's no Flight with the given id it will be inserted with the second array.

I want to show you a real-world example of how I handle creating and updating models

The Controller:

```php
public function store(UpsertDepartmentRequest $request): JsonResponse {
    return DepartmentResource::make($this->upsert($request, new Department()))
        ->response()
        ->setStatusCode(Response::HTTP_CREATED);
}


public function update( UpsertDepartmentRequest $request, Department $department): HttpResponse {
    $this->upsert($request, $department);

    return response()->noContent();
}

private function upsert(UpsertDepartmentRequest $request, Department $department): Department {

    $departmentData = new DepartmentData(...$request->validated());

    return $this->upsertDepartment->execute($department, $departmentData);
}
```

As you can see I often extract a method called upsert . This method accepts a Department . In the store method I use an empty Department instance because in this case, I don't have a real one. But in the

update I pass the currently updated instance.

The $this->upsertDepartment refers to an Action:

```php
class UpsertDepartmentAction {

public function execute( Department $department, DepartmentData $departmentData): Department {

  return Department->updateOrCreate(
      ['id' => $department->id],$departmentData->toArray()
    );
 }
}
```

It takes a Department which is the model (an empty one, or the updated one), and a DTO (a simple object that holds data). In the first array, I use the $department->id which is:

- null if it's a new model.
- A valid ID if it's an updated model.

And the second argument is the DTO as an array, so the attributes of the Department.

---

## 10 – upsert

Just for confusion Laravel uses the word upsert for multiple update or create operations. This is how it looks:

```php
Flight::upsert(
  [
    ['departure' => 'Oakland', 'destination' => 'San Diego', 'price' =>99],
    ['departure' => 'Chicago', 'destination' => 'New York', 'price' => 150]
  ],
  ['departure', 'destination'],
  ['price']
);
```

It's a little bit more complicated:

- First array: the values to insert or update
- Second: unique identifier columns used in the select statement
- Third: columns that you want to update if the record exists

So this example will:

- Insert or update a flight from Oakland to San Diego with the price of 99
- Insert or update a flight from Chicago to New York with the price of 150

---

## 11 – Order by Mutator
Imagine you have this:

```PHP
function getFullNameAttribute()
{
  return $this->attributes['first_name'] . ' ' . $this->attributes['last_name'];
}
```

Now, you want to order by that full_name? This won’t work:

```PHP
$clients = Client::orderBy('full_name')->get(); // doesn't work
```

The solution is quite simple. We need to order the results after we get them.

```PHP
$clients = Client::get()->sortBy('full_name'); // works!
```

Notice that the function name is different – it’s not orderBy, it’s sortBy.

**Note:** it is important to keep in your mind, if your query `Client::get()` returns a huge rows, `->sortBy()` function would require memory usege. Make sure the server don't go out of memorey. 


--- 

## 12 – Raw query methods
Eloquent statements may need the addition of raw queries. There are functions for it, however.

```PHP
// WhereRaw
$order = DB::table('orders')
  ->whereRaw('price < IF(state = "TX", ?, 100 )', [200])
  ->get();
  
// havingRaw
Product::groupBy('categrory_id')
  ->havingRaw('COUNT(*) > 1')
  ->get();
  
// orderbyRaw
User::query()
  ->where('created_at', '>', $request->date)
  ->orderByRaw('(updated_at - created_at) DESC')
  ->get();

```

## 13 – whereColumn method
The whereColumn method in Laravel's allows us to compare two columns from the same table. 

In this example, we are using it to retrieve all products where the price is less than or equal to the cost.

```PHP
// WhereColumn
$products = Product::whereColumn('price', '<=', 'cost')->get();
```