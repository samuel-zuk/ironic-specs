..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Ironic Power Control Intermediary
=================================

https://storyboard.openstack.org/#!/story/2009184

Currently, if an Ironic node lessee wishes to access power management APIs
directly, they would require either admin privileges or the power credentials
to the BMC of the node in question. Considering that one of the primary
motivations for implementing node multi-tenancy was to allow for non-privileged
users to perform this sort of action, the absence of this functionality is a
potential roadblock for improving multi-tenant support. [0]_ [1]_

Since giving a node lessee direct BMC access is very unwise, we instead intend
to emulate a standards-based interface (preferably Redfish, although IPMI is
another valid option) directly within Ironic for use by node lessees. This
way, Ironic can handle authenticating this request directly and provisioning
workflows that require this type of API access can function as expected.


Problem description
===================

As it stands, the only way an unprivileged user, such as a lessee, can access
power management APIs would be directly through the BMC with the node's power
credentials. But since these credentials wouldn't necessarily change, the
lessee could re-use them to change power settings after they were supposed to
have lost access.

In a situation like this, the lessee would be forced to use Ironic's API and
provisioning tools, possibly requiring existing workflows to be completely
refactored. There are many potential cases in which this would be very
problematic to the end-user, and the alternative solution of providing the
lessee with power credentials would be a huge security risk unacceptable to
most administrators.


Proposed change
===============

We will create a new REST API endpoint at ``{IRONIC URI}/redfish/v1`` that
will act like a legitimate v1 Redfish endpoint to the end user. The
functionality of this interface will provide strictly the essentials for
Redfish-based provisioning tools to access power controls, as implementation
of a full Redfish proxy would be beyond the scope of what we intend to do.
(see `REST API Impact`_ below for more details). Minimally, it will allow for
a user to identify themselves to Ironic and get/set the power state of any
nodes that they have access to.

These pseudo-Redfish API calls will be be translated into corresponding Ironic
or Keystone API calls, and will allow for proper authentication as well as
access to power controls. Provisioning capabilities will be available through
Redfish's ``ComputerSystem`` interface and will provide equivalent
functionality to the following Ironic CLI commands:

* ``baremetal node list``
* ``baremetal node show``
* ``baremetal node power on``
* ``baremetal node power off``
* ``baremetal ndoe reboot``

The workflow for authentication will be as follows:

* User creates an application credential [2]_ for the project that has access
  to the node they wish to access.
* User uses the Redfish Sessions API to authenticate, specifying the
  application credential UUID as the username, and the credential's secret
  as the password.
* User receives a token that grants them access to provisioning actions that
  require authentication
* Session is deleted; any further provisioning actions will require
  re-authentication.

This intermediary will abide by version 1.0.0 [6]_ of the Redfish
specification for maximum backwards compatibility with existing tools.

Alternatives
------------

The type of BMC interface emulation we're looking to implement here does
already exist in sushy-tools [3]_ and VirtualBMC [4]_, which emulate
Redfish and IPMI respectively. A previous spec was submitted by Tzu-Mainn
Chen (tzumainn) which proposed the idea of a sushy-tools driver in Ironic to
enable this functionality, but concerns about security, along with the
potential value of this existing in Ironic proper have led to the proposal
of this spec. [5]_

As far as implementation goes, we currently plan on integrating this into the
Ironic API itself, however it could also exist separately as its own separate
WSGI service.

Data model impact
-----------------
None.

State Machine Impact
--------------------
None.

REST API impact
---------------

The following endpoints will be added, available beginning with a new Bare
Metal API microversion.

* GET /redfish

  * Returns the Redfish protocol version (v1). This will always return the same
    response shown below, as per the Redfish API spec. [6]_
  * Normal response code: 200 OK
  * Example response::

      {
          "v1": "/redfish/v1/"
      }

    +------+--------+----------------------------------------+
    | Name | Type   | Description                            |
    +======+========+========================================+
    | v1   | string | The URI of the Redfish v1 ServiceRoot. |
    +------+--------+----------------------------------------+

* GET /redfish/v1/

  * The Redfish service root URL, will return a Redfish ServiceRoot object.
  * Normal response code: 200 OK
  * Example response::

      {
          "@odata.type": "#ServiceRoot.v1_0_0.ServiceRoot",
          "Id": "IronicProxy",
          "Name": "Ironic Redfish Proxy",
          "RedfishVersion": "1.0.0",
          "Links": {
              "Sessions": {
                  "@odata.id": "/redfish/v1/SessionService/Sessions"
              }
          },
          "Systems": {
              "@odata.id": "/redfish/v1/Systems"
          },
          "SessionService": {
              "@odata.id": "/redfish/v1/SessionService"
          },
          "@odata.id": "/redfish/v1/"
      }

    +------------------+--------+---------------------------------------------+
    | Name             | Type   | Description                                 |
    +==================+========+=============================================+
    | @odata.type      | string | The type of the emulated Redfish resource.  |
    +------------------+--------+---------------------------------------------+
    | @odata.id        | string | A resource link.                            |
    +------------------+--------+---------------------------------------------+
    | Id               | string | The identifier for this specific resource.  |
    +------------------+--------+---------------------------------------------+
    | Name             | string | The name of this specific ServiceRoot.      |
    +------------------+--------+---------------------------------------------+
    | Links            | object | Contains objects that contain links to      |
    |                  |        | relevant resource collections.              |
    +------------------+--------+---------------------------------------------+
    | Systems          | object | Contains a link to a collection of Systems  |
    |                  |        | resources.                                  |
    +------------------+--------+---------------------------------------------+
    | SessionService   | object | Contains a link to the SessionsService      |
    |                  |        | resource.                                   |
    +------------------+--------+---------------------------------------------+
    | Sessions         | object | Contains a link to a collection of Sessions |
    |                  |        | resources.                                  |
    +------------------+--------+---------------------------------------------+
    | RedfishVersion   | string | The version of this Redfish service.        |
    +------------------+--------+---------------------------------------------+

* GET /redfish/v1/SessionService

  * Returns a Redfish SessionService object, containing information about the
    authentication service interface.
  * Normal response code: 200 OK
  * Example response::

      {
          "@odata.type": "#SessionService.v1_0_0.SessionService",
          "Id": "IronicProxyAuth",
          "Name": "Ironic Proxy Authentication Service",
          "Status": {
              "State": "Enabled",
              "Health": "OK"
          },
          "ServiceEnabled": true,
          "SessionTimeout": 86400,
          "Sessions": {
              "@odata.id": "/redfish/v1/SessionService/Sessions"
          },
          "@odata.id": "/redfish/v1/SessionService"
      }

    +----------------+--------+----------------------------------------------+
    | Name           | Type   | Description                                  |
    +================+========+==============================================+
    | @odata.type    | string | The type of the emulated Redfish resource.   |
    +----------------+--------+----------------------------------------------+
    | @odata.id      | string | A resource link.                             |
    +----------------+--------+----------------------------------------------+
    | Id             | string | The identifier for this specific resource.   |
    +----------------+--------+----------------------------------------------+
    | Name           | string | The name of this specific resource.          |
    +----------------+--------+----------------------------------------------+
    | Status         | object | An object containing service status info.    |
    +----------------+--------+----------------------------------------------+
    | State          | string | The state of the service, one of either      |
    |                |        | "Enabled" or "Disabled".                     |
    +----------------+--------+----------------------------------------------+
    | Health         | string | The health of the service, typically "OK".   |
    |                |        | [*]_                                         |
    +----------------+--------+----------------------------------------------+
    | ServiceEnabled | bool   | Indicates whether the SessionService is      |
    |                |        | enabled or not.                              |
    +----------------+--------+----------------------------------------------+
    | SessionTimeout | number | The amount of time, in seconds, before a     |
    |                |        | session expires due to inactivity. [*]_      |
    +----------------+--------+----------------------------------------------+
    | Sessions       | object | Contains a link to a collection of Session   |
    |                |        | resources.                                   |
    +----------------+--------+----------------------------------------------+

* GET /redfish/v1/SessionService/Sessions

  * Returns a collection of Redfish Session interfaces.
  * Normal response code: 200 OK
  * Example response::

      {
          "@odata.type": "#SessionCollection.SessionCollection",
          "Name": "Ironic Proxy Session Collection",
          "Members@odata.count": 2,
          "Members": [
              {
                  "@odata.id": "/redfish/v1/SessionService/Sessions/ABC"
              },
              {
                  "@odata.id": "/redfish/v1/SessionService/Sessions/DEF"
              }
          ],
          "@odata.id": "/redfish/v1/SessionService/Sessions"
      }

    +---------------------+--------+------------------------------------------+
    | Name                | Type   | Description                              |
    +=====================+========+==========================================+
    | @odata.type         | string | The type of the emulated Redfish         |
    |                     |        | resource.                                |
    +---------------------+--------+------------------------------------------+
    | @odata.id           | string | A resource link.                         |
    +---------------------+--------+------------------------------------------+
    | Name                | string | The name of this specific resource.      |
    +---------------------+--------+------------------------------------------+
    | Members@odata.count | number | The number of Session interfaces present |
    |                     |        | in the collection.                       |
    +---------------------+--------+------------------------------------------+
    | Members             | array  | An array of objects that contain links   |
    |                     |        | to individual Session interfaces.        |
    +---------------------+--------+------------------------------------------+

* POST /redfish/v1/SessionService/Sessions

  * Requests Session authentication. A username and password is to be passed in
    the body, and upon success, the created Session object will be returned.
    Included in the headers of this response will be the authentication token
    in the ``X-Auth-Token`` header, and the link to the Session object in the
    ``Location`` header.
  * Normal response code: 201 Created
  * Error response codes: 400 Bad Request, 403 Forbidden, 500 Internal Server
    Error

    * 400 Bad Request will be returned if the username/password fields are not
      found in the message body.
    * 403 Forbidden will be returned if the credentials provided are invalid.
    * 500 Internal Server Error will be returned if the internal request to
      authenticate could not be fufilled.

  * Example Request::

      {
          "UserName": "85775665-c110-4b85-8989-e6162170b3ec",
          "Password": "its-a-secret-shhhhh"
      }

    +----------+--------+----------------------------------------------------+
    | Name     | Type   | Description                                        |
    +==========+========+====================================================+
    | UserName | string | The UUID of the Keystone application credential to |
    |          |        | be used for authentication.                        |
    +----------+--------+----------------------------------------------------+
    | Password | string | The secret of said application credential.         |
    +----------+--------+----------------------------------------------------+

  * Example Response::

      Location: /redfish/v1/SessionService/Sessions/identifier
      X-Auth-Token: super-duper-secret-aaaaaaaaaaaa

      {
          "@odata.id": "/redfish/v1/SessionService/Sessions/identifier",
          "@odata.type": "#Session.1.0.0.Session",
          "Id": "identifier",
          "Name": "user session",
          "UserName": "85775665-c110-4b85-8989-e6162170b3ec"
      }

    +-------------+--------+--------------------------------------------+
    | Name        | Type   | Description                                |
    +=============+========+============================================+
    | @odata.type | string | The type of the emulated Redfish resource. |
    +-------------+--------+--------------------------------------------+
    | @odata.id   | string | A resource link.                           |
    +-------------+--------+--------------------------------------------+
    | Id          | string | The identifier for this specific resource. |
    +-------------+--------+--------------------------------------------+
    | Name        | string | The name of this specific resource.        |
    +-------------+--------+--------------------------------------------+
    | UserName    | string | The application credential used for        |
    |             |        | authentication                             |
    +-------------+--------+--------------------------------------------+

* GET /redfish/v1/SessionService/Sessions/{identifier}

  * Returns the Session with the identifier specified in the URL. Requires the
    user to have a valid ``X-Auth-Token`` in the request header for the session
    they are attempting to access.
  * Normal response code: 200 OK
  * Error response codes: 403 Forbidden, 404 Not Found, 500 Internal Server
    Error

    * 403 Forbidden will be returned if the ``X-Auth-Token`` in the header
      field is either absent or invalid for the Session being accessed.
    * 404 Not Found will be returned if the identifier specified does not
      correspond to a legitimate Session ID.
    * 500 Internal Server Error will be returned if the internal request to
      authenticate could not be fufilled.

  * Example Response::

      {
          "@odata.id": "/redfish/v1/SessionService/Sessions/identifier",
          "@odata.type": "#Session.1.0.0.Session",
          "Id": "identifier",
          "Name": "user session",
          "UserName": "85775665-c110-4b85-8989-e6162170b3ec"
      }

    +-------------+--------+--------------------------------------------+
    | Name        | Type   | Description                                |
    +=============+========+============================================+
    | @odata.type | string | The type of the emulated Redfish resource. |
    +-------------+--------+--------------------------------------------+
    | @odata.id   | string | A resource link.                           |
    +-------------+--------+--------------------------------------------+
    | Id          | string | The identifier for this specific resource. |
    +-------------+--------+--------------------------------------------+
    | Name        | string | The name of this specific resource.        |
    +-------------+--------+--------------------------------------------+
    | UserName    | string | The application credential used for        |
    |             |        | authentication                             |
    +-------------+--------+--------------------------------------------+

* DELETE /redfish/v1/SessionService/Sessions/{identifier}

  * Ends the session identified in the URL. Requires the user to have a valid
    ``X-Auth-Token`` in the request header for the session they are trying to
    end. Does *not* revoke the associated application credential.
  * Normal response code: 204 No Content
  * Error response codes: 403 Forbidden, 404 Not Found, 500 Internal Server
    Error

    * 403 Forbidden will be returned if the ``X-Auth-Token`` in the header
      field is either absent or invalid for the Session being accessed.
    * 404 Not Found will be returned if the identifier specified does not
      correspond to a legitimate Session ID.
    * 500 Internal Server Error will be returned if the internal request to
      authenticate could not be fufilled.

* GET /redfish/v1/Systems

  * Equivalent to ``baremetal node list``, will return a collection of Redfish
    ComputerSystem interfaces that correspond to Ironic nodes. Requires the
    user to have a valid ``X-Auth-Token`` in the request header for the
    resource they are trying to access.
  * Normal response code: 200 OK
  * Error response codes: 403 Forbidden, 500 Internal Server Error

    * 403 Forbidden will be returned if the ``X-Auth-Token`` in the header
      field is either absent or invalid.
    * 500 Internal Server Error will be returned if the internal request to the
      Bare Metal service could not be fufilled.

  * Example Response::

      {
          "@odata.type": "#ComputerSystemCollection.ComputerSystemCollection",
          "Name": "Ironic Node Collection",
          "Members@odata.count": 2,
          "Members": [
              {
                  "@odata.id": "/redfish/v1/Systems/ABCDEFG"
              },
              {
                  "@odata.id": "/redfish/v1/Systems/HIJKLMNOP"
              }
          ],
          "@odata.id": "/redfish/v1/Systems"
      }

    +---------------------+--------+------------------------------------------+
    | Name                | Type   | Description                              |
    +=====================+========+==========================================+
    | @odata.type         | string | The type of the emulated Redfish         |
    |                     |        | resource.                                |
    +---------------------+--------+------------------------------------------+
    | @odata.id           | string | A resource link.                         |
    +---------------------+--------+------------------------------------------+
    | Name                | string | The name of this specific resource.      |
    +---------------------+--------+------------------------------------------+
    | Members@odata.count | number | The number of System interfaces present  |
    |                     |        | in the collection.                       |
    +---------------------+--------+------------------------------------------+
    | Members             | array  | An array of objects that contain links   |
    |                     |        | to individual System interfaces.         |
    +---------------------+--------+------------------------------------------+

* GET /redfish/v1/Systems/{node_ident}

  * Equivalent to ``baremetal node show``, albeit with fewer details. Will
    return a Redfish System resource containing basic info, power info, and the
    location of the power control interface. Requires the user to have a valid
    ``X-Auth-Token`` for the resource they are trying to access.
  * Normal response code: 200 OK
  * Error reponse codes: 403 Forbidden, 404 Not Found, 500 Internal Server
    Error

    * 403 Forbidden will be returned if the ``X-Auth-Token`` in the header
      field is absent, invalid, or if the user has inadequate permissions.
    * 404 Not Found will be returned if the identifier specified does not
      correspond to a legitimate node UUID.
    * 500 Internal Server Error will be returned if the internal request to the
      Bare Metal service could not be fufilled.

  * Example Response::

      {
          "@odata.type": "#ComputerSystem.v1.0.0.ComputerSystem",
          "Id": "ABCDEFG",
          "Name": "Baremetal Host ABC",
          "Description": "It's a computer",
          "UUID": "ABCDEFG",
          "PowerState": "On",
          "Actions": {
              "#ComputerSystem.Reset": {
                  "target": "/redfish/v1/Systems/ABCDEFG/Actions/ComputerSystem.Reset",
                  "ResetType@Redfish.AllowableValues": [
                      "On",
                      "ForceOn",
                      "ForceOff",
                      "ForceRestart",
                      "GracefulRestart",
                      "GracefulShutdown"
                  ]
              }
          },
          "@odata.id": "/redfish/v1/Systems/ABCDEFG"
      }

    +--------------------+--------+-------------------------------------------+
    | Name               | Type   | Description                               |
    +====================+========+===========================================+
    | @odata.type        | string | The type of the emulated Redfish          |
    |                    |        | resource.                                 |
    +--------------------+--------+-------------------------------------------+
    | @odata.id          | string | A resource link.                          |
    +--------------------+--------+-------------------------------------------+
    | Id                 | string | The identifier for this specific          |
    |                    |        | resource. Equal to the corresponding      |
    |                    |        | Ironic node UUID.                         |
    +--------------------+--------+-------------------------------------------+
    | Name               | string | The name of this specific resource.       |
    |                    |        | Equal to the name of the corresponding    |
    |                    |        | Ironic node if set, otherwise equal to    |
    |                    |        | the node UUID.                            |
    +--------------------+--------+-------------------------------------------+
    | Description        | string | If the Ironic node has a description set, |
    |                    |        | it will be returned here. If not, this    |
    |                    |        | field will not be returned.               |
    +--------------------+--------+-------------------------------------------+
    | UUID               | string | The UUID of this resource.                |
    +--------------------+--------+-------------------------------------------+
    | PowerState         | string | The current state of the node/System in   |
    |                    |        | question, one of either "On", "Off",      |
    |                    |        | "Powering On", or "Powering Off". [*]_    |
    +--------------------+--------+-------------------------------------------+
    | Actions            | object | Contains the defined actions that can be  |
    |                    |        | executed on this system.                  |
    +--------------------+--------+-------------------------------------------+
    | #ComputerSystem.   | object | Contains information about the "Reset"    |
    | Reset              |        | action.                                   |
    +--------------------+--------+-------------------------------------------+
    | target             | string | The URI of the Reset action interface.    |
    +--------------------+--------+-------------------------------------------+
    | ResetType@Redfish. | array  | An array of strings containing all the    |
    | AllowableValues    |        | valid options this action provides [*]_   |
    +--------------------+--------+-------------------------------------------+

* POST /redfish/v1/Systems/{node_ident}/Actions/ComputerSystem.Reset

  * Invokes a Reset action to change the power state of the node/System. The
    type of Reset action to take should be specified in the request body.
    Requires the user to have a valid ``X-Auth-Token`` in the request header
    for the resource they are attempting to access.
  * Accepts the following values for ResetType in the body:

    * "On" (soft power on)
    * "ForceOn" (hard power on)
    * "GracefulShutdown" (soft power off)
    * "ForceOff" (hard power off)
    * "GracefulRestart" (soft reboot)
    * "ForceRestart" (hard reboot)

  * Normal response code: 202 Accepted
  * Error response codes: 400 Bad Request, 403 Forbidden, 404 Not Found,
    409 NodeLocked/ClientError, 500 Internal Server Error, 503
    NoFreeConductorWorkers:

    * 400 Bad Request will be returned if the "ResetType" field is not found in
      the message body, or if the field has an invalid value.
    * 403 Forbidden will be returned if the ``X-Auth-Token`` field in the
      header is either absent or invalid for the resource being accessed.
    * 404 Not Found will be returned if the identifier specified does not
      correspond to a legitimate node UUID.
    * 409 NodeLocked/ClientError is an error code specified in the Bare Metal
      API call this request is proxied to. [7]_ The body of a 409 response will
      be the same as that which was recieved from the Bare Metal API.
    * 500 Internal Server Error will be returned if the internal request to the
      Bare Metal service could not be fufilled.
    * 503 NoFreeConductorWorkers is an error code specified in the Bare Metal
      API call this request is proxied to. [7]_ The body of a 503 response will
      be the same as that which was recieved from the Bare Metal API.

  * Example Request::

      X-Auth-Token: super-duper-secret-aaaaaaaaaaaa

      {
          "ResetType": "ForceOff"
      }

  +-----------+--------+----------------------------------------------+
  | Name      | Type   | Description                                  |
  +===========+========+==============================================+
  | ResetType | string | The type of Reset action to take (see above) |
  +-----------+--------+----------------------------------------------+

.. [*] This is included for compatibility and should always be "OK", although
       the Redfish schema allows for "Warning" and "Critical" as well.
.. [*] This could be implemented, but it would come at the cost of running an
       expensive service to expire active Sessions. This spec currently calls
       for the max value (86400s) to be served to the user exclusively for the
       sake of compatibility.
.. [*] Five power states are possible for an Ironic node: "POWER_ON",
       "POWER_OFF", "SOFT_POWER_OFF", "REBOOT", and "SOFT_REBOOT". Three of
       these map cleanly onto defined Redfish power states (POWER_ON -> On,
       POWER_OFF -> Off, SOFT_POWER_OFF -> Powering Off), but the Ironic
       "REBOOT" power states do not. I see a few possible solutions here, one
       being to map both "reboot" states onto "Powering On", the other being
       to serve a custom schema that includes a "Rebooting" state. If the
       latter is used, we could either have both Ironic "REBOOT" states map on
       to this new Redfish state, or we could have one map onto the new state,
       and the other map onto "Powering On".
.. [*] The Redfish schema for ResetType also includes "Nmi" (diagnostic
       interrupt) and "PushPowerButton" (simulates a physical power button
       press event) but since these are not part of the Ironic spec, they are
       made unavailable here.

Client (CLI) impact
-------------------
None.

"openstack baremetal" CLI
~~~~~~~~~~~~~~~~~~~~~~~~~
None.

"openstacksdk"
~~~~~~~~~~~~~~
None.

RPC API impact
--------------
None.

Driver API impact
-----------------
None.

Nova driver impact
------------------
None.

Ramdisk impact
--------------
None.

Security impact
---------------

The main consideration when it comes to the security of this feature is the
addition of a new means of accessing Ironic hardware. Generally speaking, a
considerable amount of risk is mitigated by having the emulated Redfish
Session service use application tokens (which can be revoked at any time) as
the primary means of authentication, as opposed to being given user/password
credentials directly. Furthermore, generated application credentials can (and
probably should) be limited in scope to only allow access to the endpoints
required by this intermediary.

In theory, these Sessions are as secure as the authentication tokens that
they utilize. However, this process also calls for the generation of Session
tokens (separate from Keystone auth. tokens), which will require safe storage
and cryptographically secure algorithms for generating said tokens.

The Sessions interface is the preferred means of authentication for Redfish
operations. The alternative is Basic Authentication, which sends a username
and password along with each request. Redfish literature notes that Basic
Authentication is expensive, since it requires that credentials be checked
manually for every request, leading to the potential for numerous requests
to overload the system. [8]_ This is why we chose to use it as a means for
authenticating requests.

Other end user impact
---------------------

This will give end users an alternative way of accessing power controls, one
compatible with existing Redfish provisioning tools. This means in theory,
the majority of users won't be making API calls directly, instead utilizing
pre-existing Redfish-compatible software, such as
`<Redfishtool https://github.com/DMTF/Redfishtool>`_.

Scalability impact
------------------
None.

Performance Impact
------------------

The impact of this feature's addition should be light, as it shouldn't
require periodic tasks to be ran or extraneous database queries to be made.
If this is to be integrated into the Ironic API, any performance decrease
should be negligable. However, if this is to exist as its own separate WSGI
service, there will be some additional overhead required, although since this
is a simple service, the impact will be minor.

Other deployer impact
---------------------

This feature is not something that all Ironic users will want by default,
especially those who do not plan on making use of node multi-tenancy. It
should therefore be disabled by default, and should be enabled by setting
a configuration flag. Another configuration setting could also be implemented
to disable authentication for testing purposes only-- whether or not this is
useful or a good idea I will leave up for further discussion.

Developer impact
----------------

The Sessions feature does not exist in sushy-tools; since this spec proposes
an implementation of it, it is possible it could be a useful addition there.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Sam Zuk (sam_z / szuk) <szuk@redhat.com>

Other contributors:
    Tzu-Mainn Chen (tzumainn) <tzumainn@redhat.com>
    Lars Kellogg-Stedman (larsks) <lars@redhat.com>

Work Items
----------

* Create the necessary API endpoints
    * Implement the Redfish System -> Ironic Node proxy
    * Implement the Redfish Session -> Keystone authentication proxy
    * Write unit tests and functional tests to ensure proper functionality
* Write documentation for how to use and configure this functionality, for
  users, administrators, and developers.
* Test this feature on real hardware in a way that mimics expected use cases.

Dependencies
============

None.

Testing
=======

Functional testing will be required to ensure requests made to these new proxy
endpoints result in the correct behavior when ran on an actual Ironic setup.
Furthermore, rigorous test cases should be written to make extremely sure that
no unauthorized access to node APIs is possible.

Upgrades and Backwards Compatibility
====================================

N/A


Documentation Impact
====================

Documentation will need to be provided for the new API endpoints, along with
the necessary instructions for how to enable and configure this feature (for
operators), along with additional information end users may require, such as
how to work with authentication tokens.

References
==========

.. [0] https://storyboard.openstack.org/#!/story/2006506
.. [1] https://opendev.org/openstack/ironic-specs/src/commit/6699db48d78b7a42f90cb5c06ba18a72f94b6667/specs/approved/node-lessee.rst
.. [2] https://docs.openstack.org/keystone/latest/user/application_credentials.html
.. [3] https://docs.openstack.org/sushy-tools/latest/
.. [4] https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/environments/virtualbmc.html
.. [5] https://review.opendev.org/c/openstack/ironic-specs/+/764801/3/specs/approved/power-control-passthrough.rst
.. [6] https://www.dmtf.org/sites/default/files/standards/documents/DSP0266_1.0.0.pdf
.. [7] https://docs.openstack.org/api-ref/baremetal
.. [8] https://www.dmtf.org/sites/default/files/Redfish_School-Sessions.pdf
