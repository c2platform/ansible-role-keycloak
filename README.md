# Ansible Role KeyCloak

Ansible role that installs, configures and manages [KeyCloak](https://www.keycloak.org/).

<!-- MarkdownTOC levels="2,3" autolink="true" -->

- [Requirements](#requirements)
- [Role Variables](#role-variables)
  - [Various files \( keycloak_files \)](#various-files--keycloak_files-)
- [Dependencies](#dependencies)
- [Example Play](#example-play)

<!-- /MarkdownTOC -->

## Requirements

<!-- Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required. -->

## Role Variables

<!--  A description of the settable variables for this role should go here, including any variables that are in defaults/main.yml, vars/main.yml, and any variables that can/should be set via parameters to the role. Any variables that are read from other roles and/or the global scope (ie. hostvars, group vars, etc.) should be mentioned here as well. -->

### Various files ( keycloak_files )

Using `keycloak_files` you can configure / download various files. See [Example Play](#example-play). This dict demonstrates three use cases:

1. Creation of `profile.properties` which allows features to be enabled / disabled. 
2. Creation / configuration of KeyCloak module `modules/system/layers/keycloak/org/postgresql` consisting of 1) download of JDBC client jar file, 2) creation of `module.xml` and configuration of `datasources` in `standalone.xml`.

## Dependencies

<!--   A list of other roles hosted on Galaxy should go here, plus any details in regards to parameters that may need to be set for other roles, or variables that are used from other roles. -->

Example below uses a number of roles from [C2Platform Core Collection](https://github.com/c2platform/ansible-collection-core). 

## Example Play

<!--   Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too: -->

Example below installs a KeyCloak standalone server which uses a PostgreSQL database. The database and user is created by the role by providing the `inventory_hostname` of the database server via `keycloak_database_inventory_hostname`. Using `keycloak_files` a number the `profile.properties` is created and database is configured. 

```yaml
---
- name: KeyCloak
  hosts: keycloak
  become: yes

  roles:
    - { role: c2platform.core.common,               tags: ["common"] }
    - { role: c2platform.core.java,                 tags: ["java"] }
    - { role: c2platform.core.postgresql_client,    tags: ["database"] }
    - { role: c2platform.keycloak,                  tags: ["keycloak"] }

  vars:
    postgresql_client_version: 10
    java_version: openjdk-11-jdk
    keycloak_jdbc_jar: postgresql-42.3.0.jar
    keycloak_database_host: "{{ c2_keycloak_database_host }}"
    keycloak_database_inventory_hostname: "{{ groups['database'][0] }}"
    c2_keycloak_jdbc_module: org.postgresql
    c2_keycloak_jdbc_xml:
      datasource-example: | # required
        <datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" enabled="true" use-java-context="true" statistics-enabled="${wildfly.datasources.statistics-enabled:${wildfly.statistics-enabled:false}}">
            <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE</connection-url>
            <driver>h2</driver>
            <security>
                <user-name>sa</user-name>
                <password>sa</password>
            </security>
        </datasource>
      datasource: |
        <datasource jndi-name="java:jboss/datasources/KeycloakDS" pool-name="KeycloakDS" enabled="true" use-java-context="true">
           <connection-url>{{ keycloak_database_jdbc_url }}</connection-url>
           <driver>postgresql</driver>
           <pool>
               <max-pool-size>20</max-pool-size>
           </pool>
           <security>
               <user-name>{{ keycloak_database_username }}</user-name>
               <password>{{ keycloak_database_password }}</password>
           </security>
        </datasource>
      drivers: |
        <drivers>
          <driver name="h2" module="com.h2database.h2">
              <xa-datasource-class>org.h2.jdbcx.JdbcDataSource</xa-datasource-class>
          </driver>
          <driver name="postgresql" module="{{ c2_keycloak_jdbc_module }}">
              <xa-datasource-class>org.postgresql.xa.PGXADataSource</xa-datasource-class>
          </driver>
        </drivers>

    keycloak_files:
      features:
        - dest: "{{ keycloak_home_version }}/standalone/configuration/profile.properties"
          content: |
            feature.account2=enabled
            feature.account_api=enabled
            feature.admin_fine_grained_authz=disabled
            feature.ciba=enabled
            feature.client_policies=enabled
            feature.par=enabled
            feature.declarative_user_profile=disabled
            feature.docker=disabled
            feature.impersonation=enabled
            feature.openshift_integration=disabled
            feature.scripts=disabled
            feature.token_exchange=disabled
            feature.upload_scripts=disabled
            feature.web_authn=enabled
          notify: keycloak-systemctl-restart
      jdbc-postgresql:
        - dest: "{{ keycloak_home_version }}/modules/system/layers/keycloak/org/postgresql/main/{{ keycloak_jdbc_jar }}"
          src: "https://jdbc.postgresql.org/download/{{ keycloak_jdbc_jar }}"
          notify: keycloak-systemctl-restart
        - dest: "{{ keycloak_home_version }}/modules/system/layers/keycloak/org/postgresql/main/module.xml"
          content: |
            <?xml version="1.0" ?>
            <module xmlns="urn:jboss:module:1.3" name="{{ c2_keycloak_jdbc_module }}">
                <resources>
                    <resource-root path="{{ keycloak_jdbc_jar }}"/>
                </resources>
                <dependencies>
                    <module name="javax.api"/>
                    <module name="javax.transaction.api"/>
                </dependencies>
            </module>
          notify: keycloak-systemctl-restart
        - dest: "{{ keycloak_home_version }}/standalone/configuration/standalone.xml"
          xpath: //*[name()='datasources']
          set_children:
            - "{{ c2_keycloak_jdbc_xml['datasource-example'] }}"
            - "{{ c2_keycloak_jdbc_xml['datasource'] }}"
            - "{{ c2_keycloak_jdbc_xml['drivers'] }}"
          notify: keycloak-systemctl-restart
```
