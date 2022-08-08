# Configuring Wagtail App

[Wagtail](https://docs.wagtail.org/en/stable/index.html) is a content management system built on top of [Django](https://docs.djangoproject.com/en/4.1/).

We will use Docker to install and run all the software needed to serve our Wagtail app securely behind Nginx, popular web server software, which we'll use to force all requests to our Wagtail app to use https. This will ensure that all information that users of our Wagtail app send to and receive from our server will be encrypted while in transit over the internet. 

*The configuration described here assumes that [Docker and Docker Compose are installed](/docs/install-docker-and-compose.md) on our droplet*

## Required Environment Variables

### Django Settings
- **`DJANGO_ALLOWED_HOSTS`**: This value is used to for Django's [ALLOWED_HOSTS](https://docs.djangoproject.com/en/4.1/ref/settings/#allowed-hosts) setting
- **`DJANGO_SETTINGS_MODULE`**: This value [instructs Django which which settings module to use](https://docs.djangoproject.com/en/4.1/topics/settings/#designating-the-settings). It should be in Python path syntax. **If this value is not set, any cli commands using `manage.py` will default to using development settings.** For a Wagtail app with the directory structure below, this value should be `myproject.settings.production` when deploying to production.
  ```
  myproject
    ├── myproject
    │   ├── __init__.py
    │   ├── settings
    │   │   ├── __init__.py
    │   │   ├── base.py
    │   │   ├── dev.py
    │   │   └── production.py
    ├── manage.py
  ```
- **`DJANGO_SECRET_KEY`**: This value is used for Django's [SECRET_KEY](https://docs.djangoproject.com/en/4.0/ref/settings/#secret-key) setting.

### Django Storages
See [django-storages documentation](https://django-storages.readthedocs.io/en/latest/backends/digital-ocean-spaces.html) for more
- **`AWS_ACCESS_KEY_ID`**: Just what it sounds like.
- **`AWS_SECRET_ACCESS_KEY`**: Just what it sounds like. 
- **`AWS_STORAGE_BUCKET_NAME`**: The name of your S3 bucket.
- **`AWS_S3_REGION_NAME`**: The Digital Ocean region (such as nyc3 or sfo2) of your S3 bucket.
- **`AWS_S3_ENDPOINT_URL`**: The value of `https://${AWS_S3_REGION_NAME}.digitaloceanspaces.com`.
- **`AWS_S3_CUSTOM_DOMAIN`**: Set this to the proper domain for your bucket's CDN.

### Miscellaneous
- **`CLIENT_URL`**: This value instructs wagtail where to redirect to when viewing content previews.
- **`CLIENT_PREVIEW_SECRET`**: This value is shared with the frontend client to more securely fetch content when viewing content previews.
- **`CLIENT_REVALIDATE_SECRET`**: This value is shared with the frontend client and used during cache invalidation after publishing new content.
- **`HOST`**: This value is used by Nginx. It should be just the domain (no scheme "https://" or path "/some/path").
- **`DATABASE_URL`**: A PostgreSQL connection URI (see section *34.1.1.2 Connection URIs* of Postgres [Connection Strings](https://www.postgresql.org/docs/14/libpq-connect.html#LIBPQ-CONNSTRING) docs) 