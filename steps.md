1-création venv de mon projet et activation
	 python -m venv test
	 python -m venv .env

   Script de venv ==> activate.bat

2-installation du django 
  Pip install django

3- pip freeze
    pip freeze > requirements.txt
4- pip install psycopg2

5-django-admin.exe startproject DjangoBlog

6-rename DjangoBlog src

7--python manage.py startapp users

8- add users app on installed application settings of the project

     # Local apps
    'users.apps.UsersConfig',

9- 

  # settings.py
AUTH_USER_MODEL = 'users.User'
