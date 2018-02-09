Laravel GraphQL
===============

Use Facebook GraphQL with Laravel 5.2 >=. It is based on the PHP
implementation [here](https://github.com/webonyx/graphql-php). You can find more
information about GraphQL in the [GraphQL Introduction](http://facebook.github.io/react/blog/2015/05/01/graphql-introduction.html)
on the [React](http://facebook.github.io/react) blog or you can read the
[GraphQL specifications](https://facebook.github.io/graphql/).

[![Latest Stable Version](https://poser.pugx.org/studio-net/laravel-graphql/v/stable)](https://packagist.org/packages/studio-net/laravel-graphql)
[![Latest Unstable Version](https://poser.pugx.org/studio-net/laravel-graphql/v/unstable)](https://packagist.org/packages/studio-net/laravel-graphql)
[![Total Downloads](https://poser.pugx.org/studio-net/laravel-graphql/downloads)](https://packagist.org/packages/studio-net/laravel-graphql)
[![Monthly Downloads](https://poser.pugx.org/studio-net/laravel-graphql/d/monthly)](https://packagist.org/packages/studio-net/laravel-graphql)
[![Daily Downloads](https://poser.pugx.org/studio-net/laravel-graphql/d/daily)](https://packagist.org/packages/studio-net/laravel-graphql)
[![License](https://poser.pugx.org/studio-net/laravel-graphql/license)](https://packagist.org/packages/studio-net/laravel-graphql)
[![Build Status](https://api.travis-ci.org/studio-net/laravel-graphql.svg?branch=master)](https://travis-ci.org/studio-net/laravel-graphql)

## Installation

```bash
composer require studio-net/laravel-graphql @dev
```

If you're not using Laravel 5.5>=, don't forget to append facade and service
provider to you `config/app.php` file. Next, you have to publish vendor.

```bash
php artisan vendor:publish --provider="StudioNet\GraphQL\ServiceProvider"
```

## Usage

- [Definition](#definition)
- [Query](#query)
- [Mutation](#mutation)
- [Self documentation](#self-documentation)
- [Examples](#examples)

### Definition

Each source of data must have a corresponding definition in order to retrieve
fetchable and mutable fields.

```php
# app/GraphQL/Definition/UserDefinition.php

namespace App\GraphQL\Definition;

use StudioNet\GraphQL\Definition\Type;
use StudioNet\GraphQL\Support\Definition\EloquentDefinition;
use App\User;

/**
 * Specify user GraphQL definition
 *
 * @see EloquentDefinition
 */
class UserDefinition extends EloquentDefinition {
	/**
	 * Set a name to the definition. The name will be lowercase in order to
	 * retrieve it with `\GraphQL::type` or `\GraphQL::listOf` methods
	 *
	 * @return string
	 */
	public function getName() {
		return 'User';
	}

	/**
	 * Set a description to the definition
	 *
	 * @return string
	 */
	public function getDescription() {
		return 'Represents a User';
	}

	/**
	 * Represents the source of the data. Here, Eloquent model
	 *
	 * @return string
	 */
	public function getSource() {
		return User::class;
	}

	/**
	 * Which fields are queryable ?
	 *
	 * @return array
	 */
	public function getFetchable() {
		return [
			'id'          => Type::id(),
			'name'        => Type::string(),
			'last_login'  => Type::datetime(),
			'is_admin'    => Type::bool(),
			'permissions' => Type::json(),

			// Relationship between user and posts
			'posts'       => \GraphQL::listOf('post')
		];
	}

	/**
	 * Resolve field `permissions`
	 *
	 * @param  User $user
	 * @return array
	 */
	public function resolvePermissionsField(User $user) {
		return $user->getPermissions();
	}

	/**
	 * Which fields are mutable ?
	 *
	 * @return array
	 */
	public function getMutable() {
		return [
			'id'          => Type::id(),
			'name'        => Type::string(),
			'is_admin'    => Type::bool(),
			'permissions' => Type::array(),
			'password'    => Type::string()
		];
	}
}

# config/graphql.php

return [
	// ...
	'definitions' => [
		\App\GraphQL\Definition\UserDefinition::class,
		\App\GraphQL\Definition\PostDefinition::class
	],
	// ...
]
```

The definition is an essential part in the process. It defines queryable and
mutable fields. Also, it allows you to apply transformers for only some data
with the `getTransformers` methods. There's 5 kind of transformers to apply on :

* `list`  : create a query to fetch many objects (`User => users`)
* `view`  : create a query to retrieve one object (`User => user`)
* `drop`  : create a mutation to delete an object (`User => deleteUser`)
* `store` : create a mutation to update an object (`User => user`)
* `batch` : create a mutation to update many object at once (`User => users`)
* `restore` : create a mutation to restore an object (`User => restoreUser`)

By the default, the definition abstract class handles Eloquent model
transformation.

A definition is composed from types. Our custom class extend the default
`GraphQL\Type\Definition\Type` class in order to implement `json` and `datetime`
availabled types.

### Query

If you want create a query by hand, it's possible.

```php
# app/GraphQL/Query/Viewer.php

namespace App\GraphQL\Query;

use StudioNet\GraphQL\Support\Definition\Query;
use Illuminate\Support\Facades\Auth;

class Viewer extends Query {
	/**
	 * {@inheritDoc}
	 */
	public function getRelatedType() {
		return \GraphQL::type('user');
	}

	/**
	 * Return logged user
	 *
	 * @return \App\User|null
	 */
	public function getResolver() {
		return Auth::user();
	}
}

# config/graphql.php

return [
	'schema' => [
		'definitions' => [
			'default' => [
				'query' => [
					'viewer' => \App\GraphQL\Query\Viewer::class
				]
			]
		]
	],

	'definitions' => [
		\App\GraphQL\Definition\UserDefinition::class
	]
];
```

### Mutation

Mutation are used to update or create data.

```php
# app/GraphQL/Mutation/Profile.php

namespace App\GraphQL\Mutation;

use StudioNet\GraphQL\Support\Definition\Mutation;
use StudioNet\GraphQL\Definition\Type;
use App\User;

class Profile extends Mutation {
	/**
	 * {@inheritDoc}
	 *
	 * @return ObjectType
	 */
	public function getRelatedType() {
		return \GraphQL::type('user');
	}

	/**
	 * {@inheritDoc}
	 */
	public function getArguments() {
		return [
			'id'      => ['type' => Type::nonNull(Type::id())],
			'blocked' => ['type' => Type::string()]
		];
	};

	/**
	 * Update user
	 *
	 * @param  mixed $root
	 * @param  array $args
	 *
	 * @return User
	 * @SuppressWarnings(PHPMD.UnusedFormalParameter)
	 */
	public function getResolver($root, array $args) {
		$user = User::findOrFail($args['id']);
		$user->update($args);

		return $user;
	}
}

# config/graphql.php

return [
	'schema' => [
		'definitions' => [
			'default' => [
				'query' => [
					'viewer' => \App\GraphQL\Query\Viewer::class
				],
				'mutation' => [
					'viewer' => \App\GraphQL\Mutation\Profile::class
				]
			]
		]
	],

	'definitions' => [
		\App\GraphQL\Definition\UserDefinition::class
	]
];
```

### Self documentation

A documentation generator is implemented with the package. By default, you can access it by navigate to `/doc/graphql`. You can change this behavior within the configuration file. The built-in documentation is implemented from [this repository](https://github.com/mhallin/graphql-docs).

### Examples

```graphql
query {
	viewer {
		name
		email

		posts {
			title
			content
		}
	}
}

# is equivalent to (if user id exists)

query {
	user (id: 1) {
		name
		email

		posts {
			title
			content
		}
	}
}
```

#### Using filters

In order to use operator, you can refer to example below. Otherwise, this is the
default syntax to use : `(operator) text`, excepting for `%` and `=` that
doesn't need to handle an operator case, just the string within like `studio-net`
will produce `= studio-net` and `studio-net%` will produce `ilike 'studio-net%'`.

| Operator | PostgreSQL | MySQL                          |
| -------- | ---------- | ------------------------------ |
| lt       | `<`        | `<`                            |
| lte      | `<=`       | `<=`                           |
| gt       | `>`        | `>`                            |
| gte      | `>=`       | `>=`                           |
| %        | `ilike`    | `like` (but process a `lower`) |
| =        | `=`        | `=`                            |

By default, the `AND` operator will be used until find `or` array key. All
elements in `or` key will be considered as `OR`.

```
[
	'field' => [
		'or' => [
			'toto',
			'tata',

			'and' => [
				'%lolo',
				'lala%'
			]
		]
	]
]

'field' = ('toto' OR 'tata' OR ('%lolo' AND 'lala%'))
```

```graphql
query {
	users (take: 2, filter: {"first_name": ["%Targaryen"], "id": {"or" : ["(gt) 5", "1"]}}) {
		id
		first_name
		last_name

		posts (take: 5) {
			id
			title
			content
		}
	}
}
```

#### Mutation

```graphql
mutation {
	# Delete object
	delete : deleteUser(id: 5) {
		first_name
		last_name
	},

	# Update object
	update : user(id: 5, with : { first_name : "toto" }) {
		id
		first_name
		last_name
	},

	# Create object
	create : user(with : { first_name : "toto", last_name : "blabla" }) {
		id
		first_name
		last_name
	},

	# Update or create many objects at once
	batch  : users(objects: [{with: {first_name: 'studio'}}, {with: {first_name: 'net'}}]) {
		id
		first_name
	}
}
```

## Contribution

If you want participate to the project, thank you ! In order to work properly,
you should install all dev dependencies and run the following commands before
pushing in order to prevent bad PR :

```bash
$> ./vendor/bin/phpmd src text phpmd.xml
$> ./vendor/bin/phpmd tests text phpmd.xml
$> ./vendor/bin/phpstan analyse --autoload-file=_ide_helper.php --level 1 src
$> ./vendor/bin/php-cs-fixer fix
```
