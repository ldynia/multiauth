# Laravel Multi Auth #

- **Laravel**: 5
- **Author**: Ramon Ackermann
- **Author Homepage**: https://github.com/sboo
- **Author**: Ollie Read
- **Author Homepage**: http://ollieread.com

For Laravel 4.2 version, see https://github.com/ollieread/multiauth

This package is not a replacement for laravels default Auth library, but instead something
that sits between your code and the library.

Think of it as a factory class for Auth. Now, instead of having a single table/model to
authenticate users against, you can now have multiple, and unlike the previous version of
this package, you have access to all functions, and can even use a different driver 
for each user type.

On top of that, you can use multiple authentication types, simultaneously, so you can be logged
in as a user, a master account and an admin, without conflicts!

## Custom Auth Drivers ##

At this current moment in time, custom Auth drivers written for the base Auth class will not work. I'm currently looking into this particular issue but for the meantime, you can work around this by changing your closure to return an instance of `Ollieread\Multiauth\Guard` instead of the default.

## Installation ##

Firstly you want to include this package in your composer.json file.

    "require": {
    		"sboo/multiauth" : "4.0.*"
    }
    
Now you'll want to update or install via composer.

    composer update

Next you open up app/config/app.php and replace the AuthServiceProvider with

    "Ollieread\Multiauth\MultiauthServiceProvider"

**NOTE** It is very important that you replace the default service providers. If you do not wish to use Reminders, then remove the original Reminder server provider as it will cause errors.

Configuration is pretty easy too, take config/auth.php with its default values:

    return [

		'driver' => 'eloquent',

		'model' => 'App\User',

		'table' => 'users',

		'password' => [
        		'email' => 'emails.password',
        		'table' => 'password_resets',
        		'expire' => 60,
        	],

	];

Now remove the first three options and replace as follows:

    return array(

		'multi'	=> [
			'account' => [
				'driver' => 'eloquent',
				'model'	=> 'Account'
			],
			'user' => [
				'driver' => 'database',
				'table' => 'users'
			]
		],

		'password' => [
        		'email' => 'emails.password',
        		'table' => 'password_resets',
        		'expire' => 60,
        	],

	);

## Reminders ##

If you wish to use reminders, you will need to replace Illuminate\Auth\Passwords\PasswordResetServiceProvider in you
config/app.php file with the following.

	Ollieread\Multiauth\Passwords\PasswordResetServiceProvider

To generate the reminders table you will need to run the following command.

	php artisan multiauth:resets-table

Likewise, if you want to clear all reminders, you have to run the following command.

	php artisan multiauth:clear-resets

The `reminders-controller` command has been removed, as it wouldn't work with the
way this package handles authentication. I do plan to look into this in the future.

The concept is the same as the default Auth reminders, except you access everything
the same way you do using the rest of this package, in that prefix methods with the
authentication type.

If you wish to use a different view per user type, then just add an email option to the config,
much the same way as it is inside `auth.reminder`.

To send a reminder you would do the following.

	Password::account()->remind(Input::only('email'), function($message) {
		$message->subject('Password reminder');
	});

And to reset a password you would do the following.

	Password::account()->reset($credentials, function($user, $password) {
		$user->password = Hash::make($password);
		$user->save();
	});

For simple identification of which token belongs to which user, as it's perfectly feasible
that we could have two different users, of different types, with the same token, I've modified my reminder
email to have a type attribute.

	To reset your password, complete this form: {{ URL::to('password/reset', array($type, $token)) }}.

This generates a URL like the following.

	http://laravel.ollieread.com/password/reset/account/27eb8fe5fe666b3b8d0521156bbf53266dbca572

Which matches the following route.

	Route::any('/password/reset/{type}/{token}', 'Controller@method');


## Usage ##

Everything is done the exact same way as the original library, the one exception being
that all method calls are prefixed with the key (account or user in the above examples)
as a method itself.

    Auth::account()->attempt(array(
    	'email'		=> $attributes['email'],
    	'password'	=> $attributes['password'],
    ));
    Auth::user()->attempt(array(
    	'email'		=> $attributes['email'],
    	'password'	=> $attributes['password'],
    ));
    Auth::account()->check();
    Auth::user()->check();

I found that have to call the user() method on a user type called user() looked messy, so
I have added in a nice get method to wrap around it.

	Auth::user()->get();

In the instance where you have a user type that can impersonate another user type, example being
an admin impersonating a user to recreate or check something, I added in an impersonate() method
which simply wraps loginUsingId() on the request user type.

	Auth::admin()->impersonate('user', 1, true);

The first argument is the user type, the second is the id of said user, and the third is
whether or not to remember the user, which will default to false, so can be left out
more often than not.

And so on and so forth.


Check this gist for the modified files in a default Laravel 5 installation:
https://gist.github.com/sboo/10943f39429b001dd9d0

There we go, done! Enjoy yourselves.


## Testing ##

Laravel integration/controller testing implements `$this->be($user)` to the base TestCase class. The implementation of #be() does not work correctly with Multiauth. To get around this, implement your own version of #be() as follows:

    public function authenticateAs($type, $user) {
      $this->app['auth']->$type()->setUser($user);
    }


### License

This package inherits the licensing of its parent framework, Laravel, and as such is open-sourced 
software licensed under the [MIT license](http://opensource.org/licenses/MIT)
