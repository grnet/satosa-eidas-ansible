# Ansible Role: SATOSA

Installs and configures SATOSA, has only been tested on Debian Stretch servers.

This role installs SATOSA and configures SATOSA. It offers the option to run SATOSA either as a systemd service or as a docker container. When running it as a systemd service SATOSA is installed using pip in a virtual environment and it is run using gunicorn.


## Role Variables
The variables that usually need to be modified can be found below (for seeing all the available variables and their default values look at [defaults/main.yml](defaults/main.yml))

### satosa__create_certificates
A list of certificates to be created using openssl. Each element must be a dict with the keys `name`, the name of the newly created certificate, and `dir`, the directory in which to store the certificate in. E.g.:
```
satosa__create_certificates:
  - name: "{{ satosa__frontends_cert_name }}"
    dir: "{{ satosa__data_dir }}/{{ satosa__satosa_certs_dir }}"
  - name: "{{ satosa__backends_cert_name }}"
    dir: "{{ satosa__data_dir }}/{{ satosa__satosa_certs_dir }}"
```

### satosa__certificates
This is a list which contains certificates to upload to the SATOSA server each element is a dictionary with the format:
```
   name: name
   content: content
```
the content will be stored in a file in: `{{ satosa__data_dir }}/{{ satosa__satosa_certs_dir }}/{{ name }}`.

### satosa__data_dir
The directory which will host the SATOSA configurations and certificates. Defaults to:
`/etc/satosa`

### satosa__port
The port to which SATOSA will listen to, defaults to:
`8080`

### satosa__install_path
Where to create the SATOSA virtualenv, only used if SATOSA is run as a service, defaults to:
`/opt/satosa`

### satosa__pip_packages
List of pip packages to install, defaults to:
```
  - git+https://github.com/IdentityPython/SATOSA.git@v4.4.0
```

### satosa__log_dir
The directory in which to store the SATOSA logs.

### satosa__run_docker
Run SATOSA on a docker container

### satosa__run_service
Run SATOSA as a systemd service

### satosa__base_url
The SATOSA base url:
```
https://{{ inventory_hostname }}
```

### satosa__cookie_state_name
The name of the cookie used by SATOSA, this must be common on all SATOSA instances in a distributed setup, defaults to:
`SATOSA_STATE`

### satosa__state_encryption_key
The key used to encrypt the state in the cookie, defaults to:
`asdASD123`

### satosa__logging_level
SATOSA's logging level, defaults to:
`DEBUG`

### satosa__oidc_client_db_path
Copy the oidc_client_db json from local to the remote server on dest. It is a dict with the keys `src`(where to find the find on the local server) and `dest`(the dest on the remove server).

### satosa__frontends
A list defining the frontends to be enabled. Each element must be a dict with the values `template_file`(the file template file) and `conf_file_name`(the name of the file on the target server), all extra variables are used by the jinja template. A more detailed overview of the templates provided on this role can be found [here](#frontend-templates).

### satosa__backends
A list defining the backends to be enabled. Each element must be a dict with the values `template_file`(the file template file) and `conf_file_name`(the name of the file on the target server), all extra variables are used by the jinja template. A more detailed overview of the templates provided on this role can be found [here](#backend-templates).

### satosa__micro_services
A list defining the micro services to be enabled. Each element must be a dict with the values `template_file`(the file template file) and `conf_file_name`(the name of the file on the target server), all extra variables are used by the jinja template. A more detailed overview of the templates provided on this role can be found [here](#micro-service-templates).

## Templates
There exists a generic template at `templates/etc/satosa/generic_plugin.yaml` which will convert any yaml to
### Frontend Templates

### Backend Templates

### Micro Service Templates

