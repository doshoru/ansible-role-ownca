#######################################################################################
# Overview
#######################################################################################

This Ansible role:
(1) Installs and configures a single Root Certificate Authority server
(2) Installs and configures one or more Subordinate Certificate Authority servers signed by the Root
(3) Create concatenated Root CA + Subordinate CA chain certificate files for each Subordinate
(4) Provides a reusable task that can be leveraged by (not-included) playbooks in the environment to:
    - Create a Private Key and CSR for the client
    - Copy the CSR to a Subordinate CA for signing
    - Copy the Signed Certificate back to the specified directory of the client
    - Copy all supporting Root, Subordinate, and chain certificate files from the Subordinate CA to the client's trust store and update it


#######################################################################################
# Directory Structure
#######################################################################################

    roles/
        ROLENAME/
            playbooks/
                install-ROLENAME.yml
            tasks/
                main.yml
                create-rootca.yml
                create-subca.yml
                request-certificate.yml
                request-chain.yml
            handlers/
                update-ca-trust.yml
            vars/
                group-vars.yml
                global-vars.yml
        README.md


#######################################################################################
# Installing the PKI
#######################################################################################

Step 1 -- Download this role from github.

    Example:
    git clone https://github.com/doshoru/ansible-role-ownca.git /etc/ansible/roles/ownca

Step 2 -- Add your intended Root and Subordinate servers to the Ansible inventory.

Create a group named 'ownca' in your Ansible inventory.  You can create sub-groups if choose (shown in the example below), but make sure that you specify at least two servers in total.  You will need one server to function as the Root CA, and at least one other server to function as a Subordinate CA.

    Example:
    cat <<EOF>> /etc/ansible/hosts
    ### OWNCA INTERNAL CERTIFICATE AUTHORITY ###
        ownca:
          children:
            root_ca:
            sub_ca:
        root_ca:
          hosts:
            rootca.example.org
        sub_ca:
          hosts:
            subca01.example.org
            subca02.example.org
    EOF

Step 3 -- Define the variables.

Two variables template files have been included with this Ansible role:

(1) An 'ownca/vars/global_vars.yml' template containing the variables that must be accessible to ALL hosts in your Ansible inventory -- both the OWNCA servers and clients.  Copy the contents of the 'ownca/vars/global_vars.yml' template to your Ansible inventory's '/group_vars/all.yml' file.

(2) An 'ownca/vars/group_vars.yml' template containing variables unique to the OWNCA servers that will only be used in the OWNCA installation process.  Copy (or move) the 'ownca/vars/group_vars.yml' template to the 'group_vars' path of your OWNCA role in your Ansible inventory and rename it to 'vars'.


    Example:
    cp /etc/ansible/roles/ownca/group-vars.yml /etc/ansible/group_vars/ownca/vars

After copying the templates, modify the variable defaults in BOTH of the templates to suit your environment.  Note that the OWNCA installation process will also pull variables from the 'global' vars.  If you want your OWNCA servers to have different variables than the clients (e.g. 2048-bit keys for clients, but 4096-bit keys for the servers) you can copy those variables from 'global_vars' to 'group_vars' as an 'override'.

The templates and their more ambiguous values are described below.

    ### BEGIN global_vars.yml TEMPLATE #################################################
    ---
    ### OWNCA CERTIFICATE AUTHORITY VARIABLES


    # CA Servers for Clients to use when Requesting Certificates
    ownca_rootca:                     'rootca.example.org'
    ownca_signing_ca:                 'subca01.example.org'
    ownca_privkey_passphrase:         "{{ hostvars['subca01.example.org']['ownca_privkey_passphrase'] }}"


    # CA Server Install Paths
    ownca_csr_dir:                    '/pki/csr'          # Where CSRs are copied to on the CA
    ownca_privkey_dir:                '/pki/private'      # Where the CA's private keys are stored
    ownca_cert_dir:                   '/pki/issued'       # Where signed certs are created
    ownca_catrust_dir:                '/pki/ca-certs'     # Where CA certs and chains are stored


    # CA Key Standards
    ownca_key_format:                 'pkcs8'
    ownca_key_size:                   '4096'
    ownca_key_type:                   'RSA'
    ownca_key_usage_critical:         'yes'
    ownca_key_usage:
      - 'digitalSignature'
      - 'keyEncipherment'
    ownca_extended_key_usage:
      - 'clientAuth'
      - 'serverAuth'
    ownca_basic_constraints_critical: 'yes'
    ownca_basic_constraints:
      - 'CA:FALSE'


    # CA DN Information
    ownca_dn_country_name:            'US'
    ownca_dn_state_name:              'STATE'
    ownca_dn_locality_name:           'LOCALE'
    ownca_dn_org_name:                'ORGANIZATION'
    ownca_dn_org_unit_name:           'ORG UNIT'
    ownca_dn_email_address:           'hostmaster@example.org'


    ### END global_vars.yml TEMPLATE ###################################################


The variables explained:

    ownca_rootca:                      'rootca.example.org'


The 'ownca_rootca' variable specifies which server should be configured as the root of the entire PKI.  It only supports one value, and must be resolvable by Ansible (e.g. an FQDN such as 'rootca.example.org').  It is used (1) to create the OWNCA, and (2) to establish the trust chain for clients.

    ownca_signing_ca:                 'subca01.example.org'


The 'ownca_signing_ca' variable specifies which Subordinate CA the clients should contact to fulfill Certificate Requests.  It only supports one value, and must be resolvable by Ansible (e.g. an FQDN such as 'subca01.example.org').  You can leverage multiple Subordinate CA's by setting this variable in more specific group_vars files to meet various environment, geographical, or D/R requirements.  You can also set the variable to a group for better redundancy and load-sharing (e.g. '{{ groups["sub_ca"] }}' ).

    ownca_privkey_passphrase:         "{{ hostvars['subca01.example.org']['ownca_privkey_passphrase'] }}"

The 'ownca_privkey_passphrase' is the password for the server's private key (the one specified in the 'ownca_signing_ca' variable).  It references the server's host_vars file where the password is stored.  In a production environment (or any environment, really) don't put your passwords in clear text.  Use a vault.

    ownca_basic_constraints:
      - 'CA:FALSE'

The 'CA' key must be set to 'FALSE' for clients.  Otherwise, they have the potential to be CA servers, themselves, which is bad from a security perspective.  This value will be overridden for the OWNCA servers in either the 'group_vars' or host_vars files.


    ### BEGIN group-vars.YML TEMPLATE #################################################
    ---
    ### OWNCA Certificate Authority variables
    # NOTE:  This vars file should ONLY be used when applying the pki_servers role -- e.g. used by the
    # CA servers themselves in the implementation of the pki_servers role.  Variables required by
    # PKI *CLIENTS* should be placed in a "global" group_vars file.
    
    # CA Servers
    ownca_root_privkey_passphrase:    "{{ hostvars['rootca.example.org']['ownca_privkey_passphrase'] }}"
    ownca_subca_servers:
      - subca01.example.org

    # CA Key Standards
    ownca_key_usage:
      - 'keyCertSign'
      - 'cRLSign'
    ownca_basic_constraints:
      - 'CA:TRUE'
    
    # CA DN Information
    ownca_dn_org_unit_name:           'OWNCA CERTIFICATE AUTHORITY'
    ownca_dn_email_address:           'hostmaster@example.org'

    ### END group-vars.YML TEMPLATE ###################################################


The variables explained:


    ownca_root_privkey_passphrase:     'my_password'


The 'ownca_root_privkey_passphrase' variable specifies the password for the Root CA's private key.  This variable is used to sign the Subordinate CA Certificate(s) which occurs during PKI installation.


    ownca_subca_servers:
      - subca01.example.org
      - subca02.example.org


The 'ownca_subca_servers' variable specifies which server(s) should be configured as Subordinate CA server(s).  This variable supports a list, where each value must be resolvable by Ansible.


    ownca_key_usage:
      - 'keyCertSign'
      - 'cRLSign'


The 'keyCertSign' key parameter is REQUIRED for a Certificate Server to sign Certificates for other servers, which is the sole purpose of a Certificate Authority.  In other words, this needs to be enabled for the CA to function.  Technically, you could leave this whole section blank, which defaults to all key_usage types being enabled, but that's generally not a good idea.


The 'cRLSign' key parameter is only required if you plan to issue a CRL.  I have it enabled by default to future-proof in case of adding that feature later, or manually.  Otherwise, it would require re-generating all CA certs.


    ownca_basic_constraints:
      - 'CA:TRUE'


Much like the 'keyCertSign' key, the 'CA' key MUST be set to 'TRUE' for the Certificate Authority to be valid.

Step 4 -- Create host_vars files for the Root CA and EACH of the Subordinate CA servers that you specified in your inventory in Step 2.

    ### Example host_vars file for the Root CA ##################
    ---
    ownca_common_name:                   'Root CA'
    ownca_privkey_passphrase:            'my_password'
    ownca_basic_constraints:
      - 'CA:TRUE'
      - 'pathlen:1'
    ### End Example #############################################


    ### Example host_vars file for Subordinate CA #1 ############
    ---
    ownca_common_name:                   'Subordinate CA 1'
    ownca_privkey_passphrase:            'my_other_password'
    ownca_basic_constraints:
      - 'CA:TRUE'
      - 'pathlen:0'
    ### End Example #############################################

The 'ownca_common_name' variable is the "human friendly" name that will be printed on the CA Certificate and MUST be unique between the Root and Subordinates.  The certificate process will fail if the Root and Subordinate for example have the same name.  It is recommended to place these variables into Ansible host_vars, as they are host-specific.

The 'ownca_privkey_passphrase' is the password for the server's private key.  In a production environment (or any environment, really) don't put your passwords in clear text.  Use a vault.  Passwords shown above just as an example to make testing this installation easier.

The 'ownca_basic_constraints' variables shown in the examples above will 'override' the 'ownca_basic_constraints' variable in the group_vars file from Step 5.  At a minimum, both the Root and the CA must have the 'CA' parameter set to 'TRUE'.  The 'pathlen' constraints are recommendations based on PKI best practices.

Step 5 -- Run the 'install-ownca.yml' playbook.

    ansible-playbook /etc/ansible/roles/ownca/playbooks/install-ownca.yml -i /etc/ansible/hosts

The playbook should create a fully-functional PKI, consisting of a Root CA and as many Subordinate CA's as you specified.

Step 6 -- Validate the PKI

You should check to ensure that the certificates look good, and that the certificate chain is valid.

Inspect the Root Certificate (from the Root CA server itself):

    openssl x509 -text -noout -in /pki/ca-certs/rootca.example.org.pem

Inspect the Subordinate Certificate(s):
    openssl x509 -text -noout -in /pki/ca-certs/subca01.example.org.pem

Validate the Certificate Chain:
    openssl verify -CAfile /pki/ca-certs/rootca.example.org.pem /pki/ca-certs/subca01.example.org.pem

The above command should output 'OK'.


#######################################################################################
# Requesting a Certificate from the PKI via the Reusable Task
#######################################################################################

Once built, you can request a certificate from one of the PKI's Subordinate CA servers via the included reusable task 'request-certificate.yml'.

Step 1 -- Create a playbook that calls the re-usable task 'request-certificate.yml'.

Below is an example of a playbook which will call the 'request-certificate.yml' task for the host 'client01.example.org'.

    ### Begin Example playbook ##############################################
    ---
    - hosts: client01.example.org
      vars:
        # client variables
        ownca_cert_name:                  'client01.example.org'
        ownca_client_privkey_dir:        /etc/pki/tls/private
        ownca_client_privkey_passphrase: 'my_password'
        ownca_client_csr_dir:            /etc/pki/tls/
        ownca_client_cert_dir:           /etc/pki/tls/certs
        ownca_client_catrust_dir:        /etc/pki/ca-trust/source/anchors
        ownca_dn_org_unit_name:          'Infrastructure'
        ownca_dn_email_address:          'hostmaster@example.org'
        ownca_common_name:               'client01.example.org'
        ownca_key_usage:
          - 'digitalSignature'
          - 'keyEncipherment'
        ownca_extended_key_usage:
          - 'clientAuth'
          - 'serverAuth'

      tasks:
      - name: Request Certificate
        include_tasks: /etc/ansible/roles/ownca/tasks/request-certificate.yml

      - name: Obtain Certificate Chain
        include_tasks: /etc/ansible/roles/ownca/tasks/request-chain.yml

      handlers:
      - name: Include Handlers
        import_tasks: /etc/ansible/roles/ownca/handlers/update-ca-trust.yml

    ### End Example playbook ##############################################

The 'request-certificate.yml' task requires the below variables which are client-specific.

Update 2022-03-06: 'ownca_client_privkey_passphrase' is no longer required by the 'request-certificate.yml' task.  If you omit the variable, the private key will be created without a passphrase.  This change has been made to support use-cases where passphrases on private keys are not supported, such as with Kubernetes.

        ownca_client_privkey_dir:        /etc/pki/tls/private
        ownca_client_privkey_passphrase: 'my_password'
        ownca_client_csr_dir:            /etc/pki/tls/
        ownca_client_cert_dir:           /etc/pki/tls/certs
        ownca_client_catrust_dir:        /etc/pki/ca-trust/source/anchors

These variables basically instruct Ansible where to create the client's private key and CSR, and where to place the signed certificate and certificate chain files.  Rather than place these variables directly in the playbook, you can place them in group_vars specific to the clients.  For example, the above represents the default directory structure on Redhat-based Linux systems where certificates are stored.

        ownca_key_usage:
          - 'digitalSignature'
          - 'keyEncipherment'
        ownca_extended_key_usage:
          - 'clientAuth'
          - 'serverAuth'

These are common PKI variables that you may need to change based on the type of certificate you need.  You can override the default values in the 'global' vars by defining them here, or in a more specific group_vars file.

        ownca_common_name:               'client01.example.org'

The 'ownca_common_name' variable is of course, required and should be an FQDN.  It will automatically get added to the Subject Alternate Names (SAN) section of the Certificate.


      - name: Obtain Certificate Chain
        include_tasks: /etc/ansible/roles/pki/tasks/request-chain.yml

You may also notice that the above playbook references a second task 'request-chain.yml'.  This optional task downloads the entire Certificate Chain from the CA and installs it in the client's trust store.

      handlers:
      - name: Include Handlers
        import_tasks: /etc/ansible/roles/ownca/handlers/update-ca-trust.yml

A handler is also provided to instruct the OS to update the certificate store with the chain of the newly provided certificates.  Note that it is 'import_tasks' and not 'include_tasks' -- This is necessary due to how Ansible parses.

#######################################################################################
# Requesting a Wildcard Certificate from the PKI via the Reusable Task
#######################################################################################

Update 2022-03-17: Support for wildcard certificates has been added.  Wildcard certificates are typically deployed across multiple endpoints, hence why the common name needs to be a wildcard (vs host-specific).  To request a wildcard certificate, create a playbook that specifies all of the hosts which will require the wildcard certificate.  The playbook should then include the 'request-wildcard-cert.yml' reusable task.  This task will:

    1. Select a single host from the group of hosts to be the 'provisioner' of the certificate
    2. Run the 'request-certificate.yml' task on that host to obtain a signed certificate
    3. Copy the signed certificate to all hosts
    4. Fetch and copy the Private Key from the 'provisioner' host to all other hosts

    ### Begin Example playbook ##############################################
    ---
    - hosts: webservers
      vars:
        # client variables
        ownca_client_privkey_dir:        /etc/pki/tls/private
        ownca_client_privkey_passphrase: 'my_password'
        ownca_client_csr_dir:            /etc/pki/tls/
        ownca_client_cert_dir:           /etc/pki/tls/certs
        ownca_client_catrust_dir:        /etc/pki/ca-trust/source/anchors
        ownca_dn_org_unit_name:          'Infrastructure'
        ownca_dn_email_address:          'hostmaster@example.org'
        ownca_common_name:               'wildcard.example.org'
        ownca_cert_name:                 '{{ ownca_common_name }}'
        ownca_key_usage:
          - 'digitalSignature'
          - 'keyEncipherment'
        ownca_extended_key_usage:
          - 'clientAuth'
          - 'serverAuth'

      tasks:
      - name: Request Wildcard Certificate
        include_tasks: /etc/ansible/roles/ownca/tasks/request-wildcard-cert.yml

      - name: Obtain Certificate Chain
        include_tasks: /etc/ansible/roles/ownca/tasks/request-chain.yml

      handlers:
      - name: Include Handlers
        import_tasks: /etc/ansible/roles/ownca/handlers/update-ca-trust.yml

    ### End Example playbook ############################################

Update 2022-03-17: New wildcard functionality adds a variable named 'ownca_cert_name'.  Previously, certificate file names were extrapolated from the hostname.  Certificate file names will continue to default to the hostname-derived values, but can be overridden via the 'ownca_cert_name' variable.  Note that the .pem and .key extensions will be added to the public and private key files, respectively.  This is useful not only for wildcard certificates, but also for use-case where multiple certificates per server are required.

        ownca_cert_name:                 '{{ ownca_common_name }}'

In the example above, ownca_cert_name is set to the value of the common name, which in this example is 'wildcard.example.org'.

End of README
