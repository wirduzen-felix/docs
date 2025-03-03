# Assignment service

## Table of contents

* [The Pattern](#the-pattern)
* [The Repository](#the-repository)
* [The Service](#the-service)

## The Pattern

A repeating pattern used throughout the B2B-Suite are Assignment Services.
The B2B-Suite ships with many new entities, and therefore provides the means to connect them to each other.
This is done through M:N assignments in for that purpose alone created components.

The Diagram below shows the usually implemented objects with their outside dependencies.

![image](../../../../../../.gitbook/assets/assignment-service.svg)

## The Repository

Again the repository is the exclusive access layer to the storage engine.
Contrary to CRUD operations there is no object, but just plain integers (The primary keys).
The default repository will have these three methods relevant for assignment:

```php
<?php declare(strict_types=1);

namespace Shopware\B2B\RoleContact\Framework;

use Doctrine\DBAL\Connection;
use Shopware\B2B\Common\Repository\DbalHelper;

class RoleContactRepository
{
    public function removeRoleContactAssignment(int $roleId, int $contactId)
    {
        [...]
    }

    public function assignRoleContact(int $roleId, int $contactId)
    {
        [...]
    }

    public function isMatchingDebtorForBothEntities(int $roleId, int $contactId): bool
    {
        [...]
    }
```

## The Service

Services are even smaller. They contain the two relevant methods for assignment.
Internally they will check if the assignment is even allowed, and throw exceptions if not.

```php
<?php declare(strict_types=1);

namespace Shopware\B2B\RoleContact\Framework;

/**
 * Assigns roles to contacts M:N
 */
class RoleContactAssignmentService
{
    public function assign(int $roleId, int $contactId)
    {
        [...]
    }

    public function removeAssignment(int $roleId, int $contactId)
    {
        [...]
    }
}
```
