# 🔥 🚀 Laravel Eloquent Tips
This is a shortlist of the amazing hidden Laravel eloquent  30 tips that make the code go on smoothly.

## 1 – Invisible Database Columns
The invisible column is a new concept in MySQL 8. What it does: when you run a `select *` query it won't retrieve any invisible column. If you need an invisible column's value you have to specify it explicitly in the `select` statement.

And now, Laravel supports these columns:

```
Schema::table('users', function (Blueprint $table){
  $table->string('password')->invisble();
});

$user = User::first();
$user->secret == null;
```
