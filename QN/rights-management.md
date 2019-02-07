

# How to handle Authorization in a ReSTful context ?

In a simple resources-access context, Access Control Lists is a mechanism that works pretty well, because objects and possible authorizations are defined and limited.

However, rights management using ACL can quickly become tricky.

At the end of the day, the question a system should address itself when it receives a request is always :

**can** {agent} **perform** {operation} **on** {target} ?



Which implies that the following concept have been defined and are identifiable within the request :

### Agent

Example: user, group, role

### Target

Example : package, class, field

### Operation

Example : CRUD operations, extended rights : manage, restore, ... or even custom operation 



Basic situation is easy to handle for a given user: 

* fetch user rights over the  target
* fetch rights granted to all groups and roles he's part of
* combine the permissions



To allow maximum flexibility, it is always beneficial to let the possibility to the DEV team to implement its own service, based on the App specific logic.

The nice thing being that, as long as it comply with the "**can** {agent} **perform** {operation} **on** {target}" logic, it will integrate seamlessly with the rest of the Application.



```php
public function isAllowed($operation, $target) {
    // check operation against default rights
    if(DEFAULT_RIGHTS & $operation) return true;        
    // retrieve current user identifier
    $agent = $this->container->get('auth')->userId();
    // build final user rights
    $user_rights = $this->getUserRights($user_id, $target);
    return (bool) ($user_rights & $operation);
}
```





