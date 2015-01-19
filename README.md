# Carrooi/Security

[![Build Status](https://travis-ci.org/Carrooi/Nette-Security.svg?branch=master)](https://travis-ci.org/Carrooi/Nette-Security)
[![Donate](http://b.repl.ca/v1/donate-PayPal-brightgreen.png)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=FQUQ9LVAKADK8)

Extensible authorization built on top of [nette/security](https://github.com/nette/security).

This package came in handy if you want to create modular website and keep all pieces decoupled with "custom" checking 
for privileges.

Now you can really easily check if eg. given user is author of some book and so on..

This idea comes from [nette/addons](https://github.com/nette/web-addons.nette.org/blob/master/app/model/Authorizator.php) 
website.

## Installation

```
$ composer require carrooi/security
$ composer update
```

Then just enable nette extension in your config.neon:

```neon
extensions:
	security: Carrooi\Security\DI\SecurityExtension
```

## Configuration

```neon
extendsions:
	security: Carrooi\Security\DI\SecurityExtension

security:
	default: true

	resources:
		book:
			default: false
			
			actions:
				view: true
				add:
					loggedIn: true
				edit:
					roles: [admin]
				delete:
					roles: [admin]
```

Well, there is nothing modular.... Yet.... We just say that resource `book` has `view` action which is accessible to 
everyone, `add` to logged users and `edit` with `delete` actions to users with `admin` role.

There are also two `default` options. With the first one we say that each `->isAllowed()` call on unknown action will 
automatically return `true`. But the second `default` will overwrite this option for all `book` actions to `false`.

That means that eg. `->isAllowed('book', 'detail')` will return `false`, but `->isAllowed('user', 'detail')` `true`.

## Other resources and actions

If `default` option is not enough, you can create default resource or default action with asterisk.

```neon
security:
	
	resources:
		favorites:
			actions:
				*:
					loggedIn: true
```

## Custom resource authorizator

Now lets create the same authorization for books by hand.

```neon
services:

	- App\Model\Books

security:
	resources:
		book: App\Model\Books
```

**`App\Model\Books` must be registered service.**

```php
<?php

namespace App\Model;

use Carrooi\Security\Authorization\IResourceAuthorizator;
use Carrooi\Security\User\User;

/**
 * @author David Kudera
 */
class Books implements IResourceAuthorizator
{


	/**
	 * @param \Carrooi\Security\User\User $user
	 * @param string $action
	 * @param mixed $data
	 * @return bool
	 */
	public function isAllowed(User $user, $action, $data = null)
	{
		if ($action === 'view') {
			return true;
		}
		
		if ($action === 'add' && $user->isLoggedIn()) {
			return true;
		}

		if (in_array($action, ['edit', 'delete']) && $user->isInRole('admin')) {
			return true;
		}

		return false;
	}

}
```

## Use objects as resources

In previous code you may noticed unused argument `$data` in `isAllowed` method. Imagine that you want to allow all users 
to update or delete their own books. First thing you need to do, is register some kind of "translator" from objects to 
resource names (lets say mappers).

```neon
security:
	targetResources:
		App\Model\Book: book
```

Now every time you pass `App\Model\Book` object as resource, it will be automatically translated to `book` resource, 
which will be then processed with your `App\Model\Books` service registered in previous example.

```php
<?php

namespace App\Presenters;

use Nette\Application\BadRequestException;
use Nette\Application\ForbiddenRequestException;

/**
 * @author David Kudera
 */
class BooksPresenter extends BasePresenter
{

	// ...

	/**
	 * @param int $id
	 * @throws \Nette\Application\BadRequestException
	 * @throws \Nette\Application\ForbiddenRequestException
	 */
	public function actionEdit($id)
	{
		$this->book = $this->books->findOneById($id);
		if (!$this->book) {
			throw new BadRequestException;
		}
		if (!$this->getUser()->isAllowed($this->book, 'edit')) {
			throw new ForbiddenRequestException;
		}
	}

}
```

```php

// ...
class Books implements IResourceAuthorizator
{

	// ...
	public function isAllowed(User $user, $action, $data = null)
	{
		// ...

		if (
			in_array($action, ['edit', 'delete']) &&
			$data instanceof Book && 
			(
				$user->isInRole('admin') ||
				$data->getAuthor()->getId() === $user->getId()
			)
		) {
			return true;
		}

		return false;
	}

}
```

## Compiler extension

Your own DI compiler extensions can implement interface `Carrooi\Security\DI\ITargetResourcesProvider` for resource 
mappers.

```php
<?php

namespace App\DI;

use Carrooi\Security\DI\ITargetResourcesProvider;
use Nette\DI\CompilerExtension;

/**
 * @author David Kudera
 */
class AppExtension extends CompilerExtension implements ITargetResourcesProvider
{


	/**
	 * @return array
	 */
	public function getTargetResources()
	{
		return [
			'App\Model\Book' => 'book',
		];
	}

}
```

## Extending User class

Be carefull if you want to extend `Nette\Security\User` class, because `carrooi\security` already extends that class 
for it's own needs.

## Changelog

* 1.0.0
	+ Initial commit
