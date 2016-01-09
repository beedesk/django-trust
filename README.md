# Django Trusts

##### Django authorization add-on for multiple organizations and object-level permission settings

### Introduction
`django-trusts` is a add-on to Django's (>= 1.7) builtin<sup>[1](#footnote1)</sup> authorization. It strives to be a **minimal** implementation, adding only a single concept, `trust`, to enable maintainable per-object permission settings for a django project that hosts users of multiple organizations<sup>[2](#footnote2)</sup> with a single user namespace.

A `trust` is a relationship whereby content access is permitted by the creator [`settlor`] to specific user(s) [`trustee`(s)] or `group`(s). Content can be an instance of a ContentMixin subclass, or of an existing model via a junction table. Access to multiple content can be permitted by a single `trust` for maintainable permssion settings. Django's builtin model, `group`, is supported and can be used to define reusuable permissions for a `group` of `user`s.

`django-trusts` also strives to be a **scalable** solution. Permissions checking is offloaded to the database by design, and the implementation minimizes database hits. Permissions are cached per `trust` for the lifecycle of `request user`. If a project's request lifecycle resolves most checked content to one or few `trusts`, which should be very typically the case, this design should be a winner in term of performance. Currently, permissions checking is done against an individual content.

`django-trusts` supports Django's builtins User models `has_perms()` / `has_perms()` and does not provides any in-addition.


<sup id="footnote1">[1] See, [Django Object Permissions](https://github.com/djangoadvent/djangoadvent-articles/blob/master/1.2/06_object-permissions.rst)</sup>

<sup id="footnote2">[2] Even `django-trusts` is incepted to support multiple organizations in a single project, it does not define or restrict oraganization model design. One natural approach is to model an organization as a special user. With this arrangment, an organization can be the `settlor` of `trusts`. Alternative approach is to create another model for organization. With this arrangment, the `settlor` of `trust`s can simple be the creating user and one might or might not have all permissions of organization's content.</sup>

---

### Usages

#####  Installation
1. Put `django-trusts` in PYTHONPATH.

2. Replace `AUTHENTICATION_BACKENDS` in `settings.py`

```python
AUTHENTICATION_BACKENDS = (
    'trusts.backends.TrustModelBackend',
)
```

#####  Implementation

Alternative 1, use `ContentMixin`

```python
# app/models.py

from django.db import models
from trusts.models import ContentMixin

class Receipt(ContentMixin, models.Model):
    account = models.ForeignKey(Account, null=True)
    merchant = models.ForeignKey(Merchant, null=True)
    # ... other field

Trust.objects.register_content(Receipt)
```


Alternative 2, use `Junction`

```python
# app/models.py

from django.db import models
from django.contrib.auth.models import Group
from trusts.models import Junction

# New Junction to model that is not under your control
class GroupJunction(Junction, models.Model):
    # field name must be named as `content` and unique=True, null=False, blank=False
    content = models.ForeignKey(django.contrib.auth.models.Group, unique=True, null=False, blank=False)

Trust.objects.register_junction(Group, GroupJunction)
```

##### Permission Assignments

```python
from django.contrib.auth.models import User, Group, Permission
from trusts.models import Trust

# Helper function
def grant_user_permission_to_model(user, model_name, code='change', app='app'):
    # Django's auth permission mechanism, nothing specific to `django-trust`
    perm = Permission.objects.get_by_natural_key('change_%s' % model_name, app, model_name)
    user.user_permissions.add(perm)

    # user.has_perm('%s.change_%s' % (app, model_name)) ==> True
    # user.has_perm('%s.change_%s' % (app, model_name), obj) ==> False

# Helper function
def grant_user_group_permssion_to_model(user, group_name, model_name, code='change', app='app'):
    # Django's auth permission mechanism, nothing specific to `django-trust`

    # get perm by name
    perm = Permission.objects.get_by_natural_key('change_%s' % model_name, app, model_name)
    group = Group.objects.get(name=group_name)

    # connect them
    user.groups.add(group)
    perm.group_set.add(group)

    # user.has_perm('%s.change_%s' % (app, model_name)) ==> True
    # user.has_perm('%s.change_%s' % (app, model_name), obj) ==> False

# View
def create_receipt_object_for_user(request, title, details):
    grant_user_permssion_to_model(request.user, 'receipt', code='change')

    trust = Trust.objects.get_or_create_settlor_default(settlor=request.user)
    content = Receipt(trust=trust, title=title, details=details)
    content.save()

    # request.user.has_perm('app.change_receipt', content) ==> True

# View
def give_user_change_permission_on_existing_group(request, user, group_name):
    grant_user_permssion_to_model(request.user, group_name, code='change', app='auth')

    group = Group.objects.get(name=group_name)
    junction = GroupJunction(trust=trust, content=group)
    junction.save()

    # request.user.has_perm('auth.change_group', group) ==> True
```

##### Permissions Checking

```python
def check_permission_to_a_specific_receipt(request, receipt_id):
  return request.user.has_perm('app.change_receipt', Receipt.objects.get(id=receipt_id))

def check_permission_to_a_specific_group(request, group_id):
  return request.user.has_perm('app.change_group', Group.objects.get(id=group_id))
```

##### Use decorators (not implemented)

```python
from trusts.decorators import permission_required
from app.models import Xyz

def get_xyz(request, user_id, xyz_id):
  return Xyz.objects.get(id=xyz_id)

@permission_required('can_edit_xyz', obj_func=get_xyz)
def edit_xyz_view(request, user_id, xyz_id):
  # ...
  pass

@permission_required('can_read_xyz', model_obj=(Xyz, 'xyz_id'))
def read_xyz_view(request, user_id, xyz_id):
  # ...
  pass
```
