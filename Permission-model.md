New RBAC in Foreman
===================

Katello and Foreman RBAC core

Current state:
* Satelite 5
  * permissions (and roles) are hardcoded
  * channels can be individually assigned with permissions
* Katello
  * permissions are stored in database
  * users are able to assign permissions to all resources and particular resources
  * any particular object can be assigned with permissions
* Foreman
  * permissions are hardcoded
  * permissions can be only globally assigned

We are currently focusing on taking current Foreman model, slightly extend it
and apply it as a generic concept for the core module for both Katello and
Foreman. This plan assumes that Katello as rails engine spike is finished and
proven to be successful path.

Existing Foreman features:

 * all objects within the system can be assigned to specific organization or location (with plan to extend this with environment)
 * permissions and controllers/actions are hardcoded in the system (and engines)
 * permission checking code with filters
 * all UI screens for users, roles, user groups, permissions, filters
 * existing organizations and locations screens

Future Foreman features:

 * users (admins) can assign permissions and filters to individual roles (currently only users)
 * filters can be applied to:
 * organizations
 * environments
 * foreman plugin:
 * system groups (host groups?)
 * hosts
 * katello plugin
 * content
 * activation keys
 * new filtering system is developed based on more extensible scoped_search
 * new user interface is developed for filters
 * possibility to disable users is added
 * users settings is added (to store things like dasboard layout etc)

TBD
