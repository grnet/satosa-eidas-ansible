# Deployment of eIDAS-enabled SATOSA
This guide describes how to deploy a SATOSA with an OIDC frontend and a SAML2 backend and how to connect it with an eIDAS node.

## Inventory
An example inventory exists in the `inventories/EIDAS` folder. We are going to use that as a base.
First thing you need to do is add your machine in the inventory, in the `proxy` group.

### User creation and management
This playbook will create a user user for you, so first of all you need to add your ssh public key to `files/{your_name}/id_rsa.pub`. Then you need to set the `core__root_ssh_key` varible in `inventories/EIDAS/group_vars/all/common` to `core__root_ssh_key=files/{your_name}/id_rsa.pub`.

You also need to add your user to `inventoried/EIDAS/group_vars/all/users`. It should look something like this:
```
user_{your_name}:
    name: {your_name}
    groups: ["{{ core__admin_group }}", "{{ ssh__allow_group }}"]
    shell: /bin/bash
    password: '*' # no password
    update_password: always
    sshkeys: "{{ lookup('file', 'users/{your_name}/id_rsa.pub') }}"
    sshkeys_exclusive: True

users_admins:
  - "{{ user_{your_name} }}"

users_devs:
  - "{{ user_{your_name} }}"

```

### Nginx configuration
In this example we are going to use certbot to create the SSL certificate, but we can also upload our certificates using this playbook.

If we want we can also upload our certificates by setting the variables:
```
nginx_ssl_crt_upload_src: "common/nginx/certs/*"
nginx_ssl_key_upload_src: "common/nginx/private/*"
```
And placing our certificates under `common/nginx/{certs,private}`

Also the nginx roles will upload any html and css files we want to use, this can be done simply by placing them in the folder pointed by `nginx_css_upload_src`(defaults to `nginx/www/css/*`)

If you are going to use certbot you should not have to alter the nginx file.

### SATOSA configuration
In the `inventories/EIDAS/group_vars/satosa` file we are going to add the SATOSA configurations. We run SATOSA using docker so the following variables must be set:
```
satosa__run_docker: True
satosa__run_service: False
satosa__docker_image_name: { image }
```

If the image we want to use is in a private repo then we need to log in first, this can be done by setting:
```
satosa__docker_registry_login: true
satosa__docker_registry:
satosa__docker_username:
satosa__docker_password:
```
You should probably set the `satosa__docker_username` and `satosa__docker_password` in a vault(https://docs.ansible.com/ansible/latest/user_guide/vault.html).

If we want to alter were the configuration will be mounted we need to set:
```
satosa__data_dir: /path/to/somewhere

satosa__docker_env_variables:
  LC_ALL: C.UTF-8
  LANG: C.UTF-8
  DATA_DIR: "{{ satosa__docker_data_dir }}"
  METADATA_DIR: "{{ satosa__docker_data_dir }}/{{ satosa__docker_satosa_metadata }}"
  PROXY_PORT: "{{ satosa__docker_container_port }}"
  OPTION_DATA_DIR: "{{ satosa__data_dir }}"
```

We need to upload the html page which will be shown to the user when he has to choose a country:
```
satosa__upload_html_folder: true
satosa__html_folder_src: common/satosa/html/*
```

If we want to create our certificates using certbot then we must set the following variable:
```
satosa__create_certificates:
  - name: "{{ satosa__frontends_cert_name }}"
  - name: "{{ satosa__backends_cert_name }}"
  - name: frontend_oidc
  - name: backends_metadata
```

Also for our oidc frontend we will need to upload the file which contains the registered clients and their metadata:
```
satosa__oidc_client_db_path:
  src: common/satosa/oidc_client_db.json
  dest: oidc_client_db.json
```


#### OIDC/SAML2 Frontends
Next we have to properly configure our frontends. We are going to have 2 frontends a SAML2 one and an OIDC one. This can be done by using a configuration like this:
```
satosa__frontends:
  - template_file: etc/satosa/plugins/frontends/saml2_frontend.yaml.j2
    conf_file_name: saml2_frontend.yaml
    name: Saml2IDP
    idp_config:
      include_encryption_keypairs: true
      organization:
        display_name:
        name:
        url:
      contact_person:
        - contact_type: "technical"
          email_address:
          given_name: "Technical"
        - contact_type: "support"
          email_address:
          given_name: "Support"
      entityid: <base_url>/<name>/proxy.xml
      metadata:
        remote:
          - url: https://path/to/metadata.xml
            node_name: urn:oasis:names:tc:SAML:2.0:metadata:EntityDescriptor
      service:
        idp:
          name: Proxy IdP
          scopes:
            - "{{ inventory_hostname }}"
          policy:
            default:
              sign_assertion: true
    endpoints:
      single_sign_on_service:
        'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect': sso/redirect
        'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST': sso/post
  - template_file: etc/satosa/plugins/frontends/openid_connect_frontend.yaml.j2
    conf_file_name: openid_connect_frontend.yaml
    name: OIDC
    client_db_path: "{{ satosa__oidc_client_db_path }}"
    signing_key_path: "{{ satosa__data_dir }}/certs/frontend_oidc.key"
    provider:
      client_registration_supported: no
      response_types_supported: ["code"]
      subject_types_supported: ["public"]
      scopes_supported: ["openid", "profile", "legal_profile", "legal_address", "vat_registration"]
      extra_scopes:
        profile:
          - given_name
          - family_name
          - birthdate
          - person_identifier
        legal_profile:
          - legal_name
          - legal_person_identifier
        legal_address:
          - legal_address
        vat_registration:
          - vat_registration
```

Further information about these configurations can be found on the [official SATOSA repo](https://github.com/IdentityPython/SATOSA/blob/master/doc/README.md), which contains extensive documentation on the [SAML2 frontend](https://github.com/IdentityPython/SATOSA/blob/master/doc/README.md#frontend) and the [OIDC frontend](https://github.com/IdentityPython/SATOSA/blob/master/doc/README.md#frontend-1).

In this example we have configured the OIDC client to request natural person and legal person attributes depending on the scope.

Most commonly you won't have to alter this much (other than filling you contact information in and adding/removing clients).
To add new SPs/RPs you need to:
- To add a new SAML2 SP you simply have to add a URL in which it exposes its metadata under `metadata.remote`.
- To add a new RP you need to alter the file pointed by `satosa__oidc_client_db_path.src` and add an entry like:
```
  "client_id": {
    "redirect_uris": ["https://path.to/redirect/uri"],
    "response_types": ["code"],
    "client_secret": "client_secret"
    ...
  },
```
You can add any other metadata you want for each client.

#### eIDAS Backend
Then we need to configure the eIDAS backend:
```
satosa__backends:
  - template_file: etc/satosa/plugins/backends/saml2_eidas_backend.yaml.j2
    conf_file_name: saml2_eidas_backend.yaml
    name: Saml2
    metadata_key_file: backends_metadata.key
    metadata_cert_file: backends_metadata.crt
    html_form_spec:
      "": default.html
    requested_attributes:
      - friendly_name: PersonIdentifier
        required: true
      - friendly_name: DateOfBirth
        required: true
      - friendly_name: FamilyName
        required: true
      - friendly_name: FirstName
        required: true
      - friendly_name: LegalName
        required: true
      - friendly_name: LegalPersonIdentifier
        required: true
      - friendly_name: LegalAddress
        required: false
      - friendly_name: VATRegistration
        required: false
    sp_config:
      organization:
        display_name:
        name:
        url:
      contact_person:
        - contact_type: "technical"
          email_address:
          given_name: "Technical"
        - contact_type: "support"
          email_address:
          given_name: "Support"
      entityid: <base_url>/<name>/proxy_saml2_backend.xml
      name:
      metadata:
        remote:
          - url: https://eidas.gov.gr/EidasNode/ConnectorResponderMetadata
            node_name: urn:oasis:names:tc:SAML:2.0:metadata:EntityDescriptor
    encryption_keypairs:
      - key_file: "{{ satosa__backends_cert_name }}.key"
        cert_file: "{{ satosa__backends_cert_name }}.crt"
    preferred_bindings:
      single_sign_on_service:
        - urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST
    service:
      sp:
        requested_attributes:
          - friendly_name: PersonIdentifier
            required: true
          - friendly_name: FamilyName
            required: true
          - friendly_name: FirstName
            required: true
          - name: http://eidas.europa.eu/attributes/naturalperson/DateOfBirth
            required: true
        endpoints:
            assertion_consumer_service:
              - "[<base_url>/<name>/acs/post, 'urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST']"
    acr_mapping:
      "": http://eidas.europa.eu/LoA/low
        comparison: minimum
```
Further information about these configurations can be found [here](https://github.com/IdentityPython/SATOSA/blob/master/doc/README.md#backend).

Most commonly you would have to change the `html_form_spec` which is a dict which can be used to map SPs/RPs to different html pages for country selection, the `""` acts as default. To add a different html page you simply need to add the html under `satosa__html_folder_src` and add an entry to `html_form_spec` like: `{SP_entity_id}: {page.html}`.

If we want to request an attribute which is not listed under `requested_attributes` we simply have to add it.

You should also set the `metadata.remote[0].url` to your country's eidas node metadata URL.

Generally in SATOSA the requested acr is decided based on the IdP (this is configured using the `acr_mapping` variable). For the eIDAS node we have added the option to request an acr value based on the SP using the `requester_acr_mapping` option, so for example if we wanted to have high acr for an SP called `SPSP` we would add the following:
```
    requester_acr_mapping:
      "SPSP":
        class_ref: http://eidas.europa.eu/LoA/high
        comparison: minimum
```

In order to actually connect your SATOSA with an eIDAS node, the node first needs to trust your proxy. In order for this to happen the certificate used by your proxy's backend to sign its metadata must be trusted by the node. You should contact your node's operators and ask them to add you public key to their keystore.

#### Microservices
SATOSA offers a wide choice of [microservices](https://github.com/IdentityPython/SATOSA/blob/master/doc/README.md#micro-services).
Different use cases require different microservices.

In our example we use the `requested_attributes_adder`, `hasher`, `primary_identifier` and `attribute_processor` microservices.
- The `requested_attributes_adder` is used for dynamically deciding whether to request the attributes of a natural or a legal person.
- The `hasher` microservice hashes the person/legal person identifier.
- The `primary_identifier` sets the users identifier to `person_identifier`, `legal_person_identifier` depending on which one is present.
- The `attribute_processor` adds a scope to the previously selected primary identifier.

