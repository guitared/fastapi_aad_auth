Using fastapi_aad_auth
**********************

``fastapi_aad_auth`` requires an Azure Active Directory App Registration (from the Azure Active Directory you want
the application to authenticate against), and these parameters should then be set in environment variables
(or a ``.env`` environment file) within the environment that fastapi is being served from.

.. _config-aad-appreg:

Configuring the Azure Active Directory App Registration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are several parts of the App Registration to Configure

In the "Authentication Tab" you need to configure the callback urls:

For the openapi/redoc authentication, configure a Single-page application eligible
for the Authorization Code Flow with PKCE for::

    https://<hostname>/docs/oauth2-redirect

e.g. for local development::

    http://localhost:8000/docs/oauth2-redirect

Configure a web platform for the UI based redirection (or whatever else is set in the config for the redirect path)::

    https://<hostname>/oauth/aad/redirect

e.g. for local development::

    http://localhost:8000/oauth/aad/redirect

.. warning::
    This is new behaviour that will be the default for version ``0.2.0``. To enable this now, you need to set the
    :class:`~fastapi_aad_auth.config.RoutingConfig` ``login_path`` and ``login_redirect_path`` variables to the empty string or ``None``


Youu also need to decide whether the application is multi-tenant or single-tenant

.. figure:: figures/App-Registration-Redirect-URIs.PNG
   :alt: Overview of redirect URI configuration for local testing

   An example configuration for redirect URIs for testing an application

On the "Expose an API tab", you need to set the Application ID URI

.. figure:: figures/App-Registration-App-ID.PNG
   :alt: Overview of app id URI

   An example configuration for api Scopes for testing an application

and add scopes as configured for the application (e.g. the default ``openid`` scope is needed)

.. figure:: figures/App-Registration-Scopes.PNG
   :alt: Overview of app scopes

   An example configuration for api Scopes for testing an application

.. warning::
    Some people have reported an Azure Active Directory issue - ``AADSTS90009`` (:issue:`59`). The scope given in ``acquire_token_interactive()`` has the prefix ``api://``,
    which seems to sometimes work, and sometimes not. It seems to always work when removing this prefix and using ``'<client_id>/.default'``.

.. _config-fastapi_aad_auth-env:

Configuring the fastapi environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The configuration is defined in ``src/fastapi_aad_auth/config.py``, and includes options for configuring
the AAD config, the login UI and the routes.

There are several key parameters:

 - ``AAD_CLIENT_ID``: The Azure Active Directory App Registration Client ID
 - ``AAD_TENANT_ID``: The Azure Active Directory App Registration Client ID

The ``AAD_CLIENT_SECRET`` parameter is needed if your application is not a public client (Generated
from the certificates and secrets section of the app registration)

.. figure:: figures/App-Registration-Client-Type.PNG
   :alt: Overview of client type on Azure AD App Registration Authentication Tab

These can be set in a ``.env`` file (e.g. for a private client)::

    AAD_CLIENT_ID="<enter-your-client-id-here>"
    AAD_TENANT_ID="<enter-your-tenant-id-here>"
    AAD_CLIENT_SECRET="<enter-your-client-secret-here>"


You can initialise it with::

    from fastapi_aad_auth import AADAuth, AuthenticationState, Config
    auth_provider = AADAuth()

    # If you had a config that wasn't set in the environment, you could use
    # auth_provider = AADAuth(Config(<my config kwargs>)

The full set of configuration options is documented in :doc:`config`


Including fastapi_aad_auth in your code
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


You can use it for fastapi routes::

    from fastapi import APIRouter, Depends


    # Use the auth_provider.api_auth_scheme for fastapi authentication

    router = APIRouter()

    @router.get('/hello')
    async def hello_world(auth_state: AuthenticationState = Depends(auth_provider.auth_backend.requires_auth(allow_session=True))):
        print(auth_state)
        return {'hello': 'world'}

For starlette routes (i.e. interactive/HTML pages), use the auth_provider.auth_required for authentication::

    from starlette.responses import PlainTextResponse

    @auth_provider.auth_required()
    async def test(request):
        if request.user.is_authenticated:
            return PlainTextResponse('Hello, ' + request.user.display_name)

This middleware will set the request.user object and request.credentials object::

    async def homepage(request):
        if request.user.is_authenticated:
            return PlainTextResponse('Hello, ' + request.user.display_name)
        return PlainTextResponse(f'Hello, you')


The :class:``fastapi.FastAPI`` ``swagger_ui_init_oauth`` variable is set automatically, along with the routing and required middleware using::

    auth_provider.configure_app(app)


Authenticating a client
~~~~~~~~~~~~~~~~~~~~~~~

If you are authenticating an e.g. console client, you need to get an access token via the Azure active directory configuration, there are examples of this (developed from the
`Azure Docs <https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-desktop-acquire-token?tabs=python#command-line-tool-without-a-web-browser>`_), e.g.::

    """Device Code authenticator for a target client"""
    import json
    import sys

    import msal
    import requests


    class AADDeviceCodeTokenRequester:
        """AAD Device Code requester"""

        def __init__(
                self,
                client_id,
                tenant_id,
                scopes=None):
            """Initialise AAD App for device code authentication."""
            self.client_id = client_id
            if scopes is None:
                scopes = []
            elif isinstance(scopes, str):
                scopes = [scopes]
            self._scopes = scopes
            self._authority = f'https://login.microsoftonline.com/{tenant_id}'
            self.msal_application = msal.PublicClientApplication(
                client_id,
                authority=self._authority)

        def get_token(self):
            """Authenticate via device code flow"""
            # From https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-desktop-acquire-token?tabs=python#command-line-tool-without-a-web-browser
            flow = self.msal_application.initiate_device_flow(scopes=self._scopes)
            if "user_code" not in flow:
                raise ValueError(
                    "Fail to create device flow. Err: %s" % json.dumps(flow, indent=4))

            print(flow["message"])
            sys.stdout.flush()  # Some terminal needs this to ensure the message is shown
            result = self.msal_application.acquire_token_by_device_flow(flow)
            return result

        def get_session(self):
            tokens = self.get_token()
            access_token = tokens['access_token']
            session = requests.sessions.Session()
            session.headers.update({'Authorization': f'Bearer {access_token}'})
            return session


There are alternative approaches for different languages, and note that this relies on the app being a Public Application when registered/configured.

For web apps/api's that are calling other web apis, the On-behalf-of flow is recomended - see the
`Azure Docs <https://docs.microsoft.com/en-us/azure/active-directory/develop/scenario-web-api-call-api-app-configuration?tabs=python>`_.

Alternatively you can request a permission using the ``Request API permissions`` part of the ``API permissions`` tab in the App Registration configuration, and adding that to the requested scope.

If you are authenticating from an app registration (e.g. a daemon application or other), you should use the client credentials flow - see the
`Azure Samples <https://github.com/Azure-Samples/ms-identity-python-daemon/blob/master/1-Call-MsGraph-WithSecret/confidential_client_secret_sample.py>`_.

Postman
-------

Tools like Postman allow you to configure authentication via oauth - this shows the example for the test server.

.. figure:: figures/Postman-Auth-Config.PNG
   :alt: Overview of authenticating for postman

   An example of how to configure client credentials (using another app registration) for postman - replace the {tenant} and {appid} info, along with the client id and client secret
