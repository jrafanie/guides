# Development with an OIDC server

This document describes the steps needed to enable External Authentication (httpd),
running an OIDC server and Apache on a local development setup.

1. Ensure you have the [guides](https://github.com/ManageIQ/guides) repo cloned locally

2. `cd guides/external_auth`

3. Launch KeyCloak

   ```sh
      podman run --rm -it --name keycloak \
         -p 8443:8443 \
         -v $(pwd)/certs:/opt/keycloak/conf/certs \
         -v $(pwd)/realms:/opt/keycloak/data/import \
         -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
         -e KC_BOOTSTRAP_ADMIN_PASSWORD=smartvm \
         -e KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/certs/tls.crt \
         -e KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/certs/tls.key \
         quay.io/keycloak/keycloak:26.6.3 \
         start-dev --import-realm
   ```

   When it completes startup, go to `https://127.0.0.1.nip.io:8443` and login with `admin` / `smartvm` to verify it's working. You should see a realm for `ManageIQ`.

4. Launch the httpd container

   ```sh
   podman run --rm -it --name httpd \
     -p 8080:8080 \
     -v $(pwd)/external_auth/oidc-httpd-configs:/etc/httpd/conf.d \
     -e HTTPD_AUTH_OIDC_CLIENT_ID=manageiq-oidc-client \
     -e HTTPD_AUTH_OIDC_CLIENT_SECRET=3167ae6f-762d-49cd-b246-ef8856315957 \
     -e HTTPD_AUTH_HOST=127.0.0.1.nip.io \
     -e HTTPD_AUTH_PORT=8080 \
     --add-host=127.0.0.1.nip.io:192.168.127.254 \
     manageiq/httpd:latest
   ```
   Note: 192.168.65.2 / 192.168.127.254 is a hardcoded proxy for host.docker.internal / host.containers.internal on podman

5. Launch ManageIQ

   Run your Rails server as you normally would for development, however, instead of accessing via the browser at `https://localhost:3000`, use `http://127.0.0.1.nip.io:8080` (notice `http` as opposed to `https` and port `8080`).

   If ManageIQ is not yet configured for OIDC, do the following:

   1. Login as `admin` / `smartvm`
   2. Go to `Settings` -> `Application Settings` -> `Authentication`
   3. Change the following:

      | | |
      |-|-|
      | Mode                  | `External (httpd)` |
      | Enable Single Sign-On | checked |
      | Provider Type         | `Enable OpenID-Connect` |
      | Get User Groups from External Authentication (httpd) | checked |

   3. Save the changes
   4. Logout
   5. Click `Log In to Corporate System`

## Updating KeyCloak data

### Export data from a running KeyCloak

If you've made changes in KeyCloak that you'd like to save, leave KeyCloak running and in another terminal run:

```sh
podman exec -it keycloak /opt/keycloak/bin/kc.sh export \
  --file /opt/keycloak/data/import/ManageIQ-realm.json \
  --realm ManageIQ \
  --users realm_file
```

When it completes, `realms/ManageIQ-realm.json` will be updated on the host.

### Recreating KeyCloak setup from scratch

1. Ensure you have a `certs` directory and `realms` directory

   ```sh
   mkdir certs
   mkdir realms
   ```

2. Generate a cert and key

   ```sh
   openssl req -x509 -newkey rsa:4096 -keyout certs/tls.key -out certs/tls.crt -days 3650 -nodes
   ```

3. Launch KeyCloak

   ```sh
   podman run --rm -it --name keycloak \
      -p 8443:8443 \
      -v $(pwd)/certs:/opt/keycloak/conf/certs \
      -v $(pwd)/realms:/opt/keycloak/data/import \
      -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
      -e KC_BOOTSTRAP_ADMIN_PASSWORD=smartvm \
      -e KC_HTTPS_CERTIFICATE_FILE=/opt/keycloak/conf/certs/tls.crt \
      -e KC_HTTPS_CERTIFICATE_KEY_FILE=/opt/keycloak/conf/certs/tls.key \
      quay.io/keycloak/keycloak:26.6.3 \
      start-dev
   ```

4. Go to `https://127.0.0.1.nip.io:8443` and login with `admin` / `smartvm`

5. Create a Realm

   | | |
   |-|-|
   | Name | `ManageIQ` |

   **Note**: realm name is case-sensitive in URLs!

6. Create an OIDC Client

   | | |
   |-|-|
   | Client ID | `manageiq-oidc-client` |

   Once created, copy down the generated client secret.

7. Configure the OIDC Client

   1. Settings

      | | |
      |-|-|
      | Client authentication | `ON` |
      | Authorization         | `ON` |
      | Valid redirect URIs   | `http://127.0.0.1.nip.io:8080/*` |

      TODO: Confirm whether newer Keycloak also requires explicitly setting `Root URL`, `Home URL`, `Web origins`, or `Post logout redirect URIs` for this development flow.

   2. Client scopes / protocol mappers

      Create a mapper for groups so KeyCloak emits the `groups` claim used by the httpd config.

      | | |
      |-|-|
      | Name             | `groups` |
      | Mapper Type      | `Group Membership` |
      | Token Claim Name | `groups` |
      | Full group path  | `OFF` |

      TODO: Confirm the exact KeyCloak 26 UI path/name for creating this mapper, since protocol mappers moved in newer releases.

   3. Authorization

      The committed realm export no longer uses the older JavaScript-based default policy. In newer KeyCloak releases, the default policy is represented as a role-based policy that grants access to authenticated users via the built-in `offline_access` role.

      TODO: Document the exact KeyCloak 26 authorization UI steps needed to recreate the committed default policy:
      - policy type `role`
      - policy name `Default Policy`
      - role `offline_access`
      - behavior matching the committed `ManageIQ-realm.json`

8. Create a group

   | | |
   |-|-|
   | Name | `EvmGroup-super_administrator` |

9. Create a user

   | | |
   |-|-|
   | Username       | `user1` |
   | Email          | `user1@manageiq.org` |
   | First Name     | `User` |
   | Last Name      | `One` |
   | Email Verified | `ON` |

10. Configure the user

    1. Credentials

       | | |
       |-|-|
       | Password              | `smartvm` |
       | Password Confirmation | `smartvm` |
       | Temporary             | `OFF` |

    2. Groups

       Put the user into the `EvmGroup-super_administrator` group.

11. Verify the setup

    1. Set the OIDC client secret:

    ```sh
    client_secret="3167ae6f-762d-49cd-b246-ef8856315957"
    ```

    2. Fetch the configuration

       ```sh
       curl -s -k https://127.0.0.1.nip.io:8443/realms/ManageIQ/.well-known/openid-configuration | jq
       ```

    3. Get an access token

       ```sh
       token=$(curl -s -k -X POST https://127.0.0.1.nip.io:8443/realms/ManageIQ/protocol/openid-connect/token \
         -H "Content-Type: application/x-www-form-urlencoded" \
         -u manageiq-oidc-client:${client_secret} \
         -d username=user1 \
         -d password=smartvm \
         -d grant_type=password | jq -r ".access_token")
       ```

    4. Introspect the access token

       ```sh
       curl -s -k -X POST https://127.0.0.1.nip.io:8443/realms/ManageIQ/protocol/openid-connect/token/introspect \
         -H "Content-Type: application/x-www-form-urlencoded" \
         -u manageiq-oidc-client:${client_secret} \
         -d "token=${token}" | jq
       ```

12. Export the realm file. If you plan to commit this, also update the client secret values in this documentation.
