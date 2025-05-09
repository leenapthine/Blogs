# Creating Custom Modules for the ARCsoft Library

**Author**: [Lee Napthine](/team/lee-napthine)  
**Last updated:** 2025-02-01

**Originally published at:** [https://arcsoft.uvic.ca/log/2024-12-09-json-response-subclasses/](https://arcsoft.uvic.ca/log/2024-12-09-json-response-subclasses/)

At ARCsoft we have begun building a repository of custom Helpers libraries that will be included in future projects. First on the docket was looking at several giant UI test code blocks in Django. These often span dozens of modules and thousands of lines of code. Quite common throughout was the use of repetitive and non-descriptive `JsonResponse` calls. Let’s walk through how we compartmentalized these calls into a more cohesive set of custom `JsonResponse` classes and how we added them to our repository of custom methods that we will use here at ARCsoft. Along the way we will document the setup of the Helpers repository and the essential inclusions to a project for the package to be able to be used, imported, and published.

## Writing a Library

Let’s first demonstrate how our UI tests might have used previous `JsonResponse` calls:

```python
return JsonResponse({
        'error': "User is unauthorized to access this url"
    }, status=403)
```

Some projects had hundreds of these response calls with the only identifier being the status code
literal.

We endeavoured to come up with a cleaner, albeit simple, solution to this repetitive use of the `JsonResponse` class. Django already contains a set of `HttpResponse` subclasses specific to different responses. Here are the specific objects Django has chosen to provide responses for: [Response objects](https://docs.djangoproject.com/en/5.1/ref/request-response/). The list is not expansive and addresses only the most likely response scenarios.

Our model for the custom `JsonResponse` classes would follow the same structure and only focus on the specific responses we felt were useful for us at ARCsoft. The list is flexible and easy to append for developers that have an interest in expanding it out.

Here is our short list:

- Bad Request
- Not Found
- Forbidden
- Unauthorized
- OK
- Created
- Internal Server Error

Like the `HttpResponse` objects, our custom class would inherit the original response class and build a base class. The base class contains several immutable characteristics (status, message) and the ability to append any additonal arguments the developer sees fit.

```python
class JsonResponseBase(JsonResponse):
        def __init__(self, message: str, status_code: int, **kwargs):
            http_status = HTTPStatus(status_code)
            response_data = {
                'status': http_status.value,
                'title': http_status.phrase,
                'message': message,
            }
            response_data.update(kwargs)
            super().__init__(data=response_data, status=status_code)
```

From the base class spawns all the specific response calls:

```python
class JsonResponseForbidden(JsonResponseBase):
        def __init__(self, message: str, **kwargs):
            super().__init__(message, status_code=403, **kwargs)

    ...
```

This short library would then be put to use throughout the test blocks like so:

```python
return JsonResponseForbidden("User is unauthorized to access this url")
```

We recognize this is a minor change, but when expanded out to hundreds of response calls, it made
for a much cleaner and readable code base.

In the project this was created in, we stored the custom `JsonResponse` in a specifc helpers
directory inside the app. This is not efficient if the intent is to use these new methods to
clean up code across dozens of projects.

Let’s detail the process of creating and implementing a repository of custom helper methods
which will become part of the natural work flow and syntax at ARCsoft.

## Initial setup

Prior to having a project skeleton we would need to add or create the required files and compose the whole project from the ground up. This often led to discrepencies from project to project. To remedy this, ARCsoft created [a template](https://gitlab.com/uvic-arcsoft/misc/skeleton) so that there would be linearity to how each project is structured. Aside from the custom needs of a specific project this makes the inital setup quite simple!

1. Clone the project skeleton into your local environment.

2. Read the skeleton `README` thoroughly, it includes important info regarding versioning, tagging, and
   maintaining the published package.

3. Re-write the `README`. Even something as simple as the project title and stated goals is a
   good start.

Next, let’s briefly summarize the various files that come packaged in the project skeleton:

### Directories

- `deployment`: This directory wil be home to all build dependencies such as Docker.

- `src`: This is the heart of your project. If it is a Django project, it will contain your app files and your api files. For the Helpers project, it contains directories for each helper library. This is where we store our `custom_json_response` package.

- `tests`: This is where all things test related are kept. When you write unit/ui tests they will be kept in this directory. When you lint your project you will be referencing the `linting` submodule which is referenced from a separate repo, but is included in the tests package. The `requirements.txt` is for downloading your test dependencies when making the project. Add custom requirements to this file.

### Files

- `.gitignore`: Creating, setting up, publishing, and deploying projects comes with all types of artifacts, environment specific setups, and databases (among other things). We do not want these being tracked by Git and available for the world to access. In some cases, there will be security issues if they ARE tracked, so be careful. Include all files you don’t want tracked into this file.

- `.gitlab-ci.yml`: This is a set of tests/rules written by ARCsoft. Every time a package is pushed to Git the pipeline in Git will run these tests and return a pass or fail. You will likely add to these rules as you develop your project.

- `.pylintrc` and `.yamllint`: These are custom rules for linting Python files and `YAML` files.

- `CHANGELOG.md`: This will track the various changes in the published package, the current version, and the tags for the package. Read about it in the project skeleton `README`.

- `LICENSE`: The license for the project. You won’t touch this.

- `Makefile`: This very important file is the set of commands for your project. When you want to set up your project you will use the `make dev-setup` command. The same goes for when you want to test it, lint it, clean up the cache, build, and publish it. Get familiar with the commands.

- `README.md`: Work on the `README` throughout the project lifespan. Make sure a developer who is new to the project can navigate it by including the up-to-date instructions.

- `pyproject.toml`: A configuration file used when publishing your library as a package. It defines essential project settings such as the project name, version, dependencies, build system requirements, import paths, project URLs, and more.

- `requirements.txt`: This lists your project dependencies. As you use install new frameworks and libraries make sure to include them here. If you are ever unsure of which you already have installed in your virtual environment, run `pip freeze` to get a list of all installed packages and their current versions.

## Storing a Library

We have the beginnings of a [collection of libraries](https://gitlab.com/uvic-arcsoft/libraries) in our code repository so it makes sense to add [this library](https://gitlab.com/uvic-arcsoft/libraries/django-helpers) to it. The ‘django-helpers’ repo works like other repositories and needs to be configured when first cloned. So far it allows for lint checking and publishes packages when pushed to Gitlab. Follow the `README`.

When creating new modules for the ‘django-helpers’, create a new branch, then add a directory to the src folder and keep your program in this new directory. If you need to write tests for the new program, keep the tests in the tests directory and
make sure they start with the term test. For example: `test_custom_json_response.py`. When complete, create an MR towards main
and our dedicated team members will have a review. Once everything looks good, we’ll merge the branch and your changes will be live.

### Importing the Helpers Submodules

NOTE: This will be updated once the helpers packages is offcially published

Likely, this will be included in the dependencies for ARCsoft’s Django projects going forward. However, if it is not included, importing it is a simple process.

1. Make sure you are in the virtual environment for your project.

2. In the root of your project, enter:

```
pip install uvic-django-helpers --index-url https://__token__:<your_personal_token>@gitlab.com/api/v4/projects/65317917/packages/pypi/simple
```

You will need your [personal access token](https://docs.gitlab.com/user/profile/personal_access_tokens/).

3. Make sure you import the module as needed into your project files.

Bingo!

---
