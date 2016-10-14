# Gauth

Djangae includes two applications to aid authentication and user management with
App Engine. Each provides both an abstract class, to extend if you're defining your own custom User model, or a concrete version to use in place of `'django.contrib.auth.models.User'`.  Also provided are custom authentication backends which delegate to the App Engine users API and a middleware to handle the link between the Django's user object and App Engine's (amongst other things).


## Using the Datastore

Allows the use of Django's permissions system on the Datastore, despite it usually requiring many-to-many relationships, which are not supported on the Datastore.

### Setup

1. Add `'djangae.contrib.gauth.datastore'` to `INSTALLED_APPS` probably
after `'django.contrib.auth'`.
2. Replace `'django.contrib.auth.middleware.AuthenticationMiddleware'` with
`'djangae.contrib.gauth.middleware.AuthenticationMiddleware'`.
3. Set `AUTH_USER_MODEL = 'djangae.GaeDatastoreUser'` in your settings file to use the supplied user model, or create your own by subclassing `djangae.contrib.gauth.datastore.models.GaeAbstractDatastoreUser`.
4. Add the backend to `AUTHENTICATION_BACKENDS` in your settings file eg:

```python
AUTHENTICATION_BACKENDS = (
	'djangae.contrib.gauth.datastore.backends.AppEngineUserAPIBackend',
	 ...
)
```

### Permissions

The Datastore-based user models have a `user_permissions` list field, which takes the place of the usual many-to-many relationship to a `Permission` model.  For groups, Djangae provides `djangae.contrib.gauth.Group`, which again has a list field for storing the permissions.  This `Group` model is registered with the Django admin automatically for you.


## Using a relational database (CloudSQL)


### Setup

1. Add `'djangae.contrib.gauth.sql'` to `INSTALLED_APPS` probably
after `'django.contrib.auth'`.
2. Replace `'django.contrib.auth.middleware.AuthenticationMiddleware'` with
`'djangae.contrib.gauth.middleware.AuthenticationMiddleware'`.
3. Set `AUTH_USER_MODEL = 'sql.GaeUser'` in your settings file to use the supplied user model or create your own by subclassing `djangae.contrib.gauth.sql.models.GaeAbstractUser`.
4. Add the backend to `AUTHENTICATION_BACKENDS` in your settings file eg:

```python
AUTHENTICATION_BACKENDS = (
	'djangae.contrib.gauth.sql.backends.AppEngineUserAPIBackend',
	 ...
)
```


## Using your own permissions system

If you want to write your own permissions system, but you still want to take advantage of the authentication provided by the Google Users API, then you may want to subclass `djangae.contrib.gauth.common.models.GaeAbstractBaseUser`.


## Authentication for unknown users

By default Djangae will deny access for unknown users (unless the user is an administrator for the App Engine application).

Add `DJANGAE_CREATE_UNKNOWN_USER=True` to your settings and Djangae will always grant access (for authenticated Google Accounts users), creating a Django user if one does not exist.

If there is a Django user with a matching email address and username set to `None` then Djangae will update the Django user, setting the username to the Google user ID. If there is a user with a matching email address and username set to another user ID then Djangae will set the existing user's email address to `None` and create a new Django user.

App Engine administrators are always granted access, and a Django user will be created if one does not exist.

## Customizing user data syncing

By default `djangae.contrib.gauth.middleware.AuthenticationMiddleware` syncs email and superuser status. In case you need to customize this behaviour (for example sync first and last names as well) you could inherit from `AuthenticationMiddleware` and override `sync_user_data` method.

For example if we would like to sync only an email address, not a superuser status, we could do the following:

```python
class MyAuthenticationMiddleware(AuthenticationMiddleware):

    def sync_user_data(self, django_user, google_user):
        if django_user.email != google_user.email():
            django_user.email = google_user.email()
            django_user.save()
```

and replace `'djangae.contrib.gauth.middleware.AuthenticationMiddleware'` with your middleware.

## Pre-creating Users

You can add users to the database before they have logged in.  If you've set `DJANGAE_CREATE_UNKNOWN_USER` to `False` then **only** users who already exist in the database can log in.

Users are keyed by their Google User ID, which is stored in the `username` field.  However, it is impossible to know what a user's Google User ID will be until they have logged in.  Therefore, pre-created users who have not yet logged in are keyed by their email address (case insensitively).  To create a user who has not yet logged in you can either:

* Create the user via the Django admin, leaving the `username` field (labelled _"User ID"_) blank.  Or...
* Create the user via the remote shell with `get_user_model().objects.pre_create_google_user("user@example.com")`.

## `get_or_create` with pre-created Users

When using pre-created Users you should be careful using `get_or_create`. The line:

```python
User.objects.get_or_create(email=email)
```

will result in error, if the pre-created user already exists with the email that is case-sensitive-different, but case-insensitive-equal to the provided value.

For instance, if you have pre-created user with email: `JOHN@gmail.com` and you have `get_or_create` with email `John@gmail.com`, you will end up trying to create a new user and failing because both versions (`JOHN@gmail.com` and `John@gmail.com`) have the same case-insensitive value.

To avoid the problem, when using `get_or_create`, you should use `email_lower` instead like this:

```python
User.objects.get_or_create(email_lower=email.lower(), defaults={"email": email})
```


## Username/password authentication

As well as using Djangae's Google Accounts-based authentication, you can also use the standard authentication backend from django.contrib.auth.  They can work alongside each other.  Simply include both, like this:

```python
AUTHENTICATION_BACKENDS = (
    'djangae.contrib.gauth.datastore.backends.AppEngineUserAPIBackend',
    'django.contrib.auth.backends.ModelBackend',
)

MIDDLEWARE_CLASSES = (
    'djangae.contrib.gauth.middleware.AuthenticationMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
)
```

## Switching accounts

There is a `switch_accounts` view which allows a user to change which of their Google accounts they're logged in with.

The URL for the user to be sent to afterwards should be provided in `request.GET['next']``.

Learn more about [Google multiple sign-in on App Engine here](https://p.ota.to/blog/2014/2/google-multiple-sign-in-on-app-engine/).

### Example usage:

Include GAuth urls in your main urls.py file.

```python
url(r'^gauth/', include(djangae.contrib.gauth.urls))
```

Use this URL to add "Switch account" functionality for user:

```html
<a href="{% url 'djangae_switch_accounts' %}">Switch account</a>
```
