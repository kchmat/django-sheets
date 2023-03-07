Use the email field as username in Django
March 24, 2021  ‐ 5 min read

Django comes with its own user model which plays a part in the authentication system of Django. By default these Django users require a unique username for authentication. If you prefer to use the email instead of the username to login you can do so by implementing a custom user model.

Adding this custom user model is something you preferably do before you create migrations for the first time.

1. Create a custom user model
The first step is creating a custom user model. I often prefer to create a users app in my Django projects when I need a custom user model. But feel free to pick another name for this app, something like auth makes sense too. Anyway, you can create an app with the following command, I will stick to an app named users in this post:

$ python manage.py startapp users
Next, we need to add the newly created users app to our INSTALLED_APPS in settings.py. You can add the app as both users or 'users.apps.UsersConfig', there is a slight functional difference though. But that's not relevant for now.

# settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Local apps
    'users.apps.UsersConfig',
]
With our new users app added to the INSTALLED_APPS we can create the a custom user model. In users/models.py we will make a new class called User which is a subclass of django.contrib.auth.models.AbstractUser. In our User class we will set the username property to None, so that we don't get it from AbstractUser. Next we add the email property as a unique EmailField to the user model.

By setting the USERNAME_FIELD you let Django know which field to use to uniquely identify users. The REQUIRED_FIELDS are the required fields which you will be prompted for when you create a user via the createsuperuser command, but you exclude the USERNAME_FIELD ans password from this list. We set because for AbstractUser the field email is part of this list, which is now our USERNAME_FIELD :).

from django.db import models
from django.contrib.auth.models import AbstractUser
from django.utils.translation import gettext_lazy as _


class User(AbstractUser):
    username = None
    email = models.EmailField(_('email address'), unique=True)

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []
Before we can make our migrations we need to go back to settings.py and set AUTH_USER_MODEL to the new user model class we just made. If you picked different names, make sure to use the following format: <APP_NAME>.<CUSTOM_USER_MODEL_CLASS>.

# settings.py
AUTH_USER_MODEL = 'users.User'
With that we should be good to go to make migrations.

$ python manage.py makemigrations
Migrations for 'users':
  users/migrations/0001_initial.py
    - Create model User
...and run them:

$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions, users
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0001_initial... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying users.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying sessions.0001_initial... OK
2. Adjust the user manager
We created a custom user model, but need to make a change to the UserManager as well. If you would run the createsuperuser command now it would exit with the error:TypeError: create_superuser() missing 1 required positional argument: 'username'.

You might want to create a separate file for this in your app, but for convenience I am adding the manager to models.py. The Django docs have a fine example on this too.

For our custom user manager we will first extend the BaseUserManager class from Django. Secondly, we add the line objects = UserManager() to our custom user model.

from django.db import models
from django.contrib.auth.models import AbstractUser
from django.utils.translation import gettext_lazy as _
from django.contrib.auth.base_user import BaseUserManager


class UserManager(BaseUserManager):
    use_in_migrations = True

    def _create_user(self, email, password, **extra_fields):
        if not email:
            raise ValueError('Users require an email field')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_user(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', False)
        extra_fields.setdefault('is_superuser', False)
        return self._create_user(email, password, **extra_fields)

    def create_superuser(self, email, password, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')

        return self._create_user(email, password, **extra_fields)



class User(AbstractUser):
    username = None
    email = models.EmailField(_('email address'), unique=True)

    objects = UserManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []
With that you should be able to run python manage.py createsuperuser again.

3. User admin
Now, to confirm that the email field is used as a username field we can open-up the Django admin. If you see the field "Email address" instead of "Username" you should be fine.

Django admin with email as username

But once you log in however, the user admin isn't available since we are using a custom user model now. To fix this we'll make changes to users/admin.py.

from django.contrib import admin
from users.models import User

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    pass
