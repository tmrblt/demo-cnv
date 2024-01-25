```bash
userData: |
  #cloud-config
  system_info:
    default_user:
      name: "admin"
      plain_text_passwd: "redhat"
      lock_passwd: false
      groups: ["wheel"]
      ssh_authorized_keys: 
        - ${ssh_keys}
  packages:
    - "wget"
    - "bash-completion"
  runcmd:
    - wget https://yum.oracle.com/repo/OracleLinux/OL8/developer/x86_64/getPackage/oracle-database-preinstall-23c-1.0-0.5.el8.x86_64.rpm
    - dnf -y localinstall oracle-database-preinstall-23c-1.0-0.5.el8.x86_64.rpm
    - wget https://download.oracle.com/otn-pub/otn_software/db-free/oracle-database-free-23c-1.0-1.el8.x86_64.rpm
    - dnf -y localinstall oracle-database-free-23c-1.0-1.el8.x86_64.rpm
    - (echo "password"; echo "password";) | /etc/init.d/oracle-free-23c configure >> silentinstall.log 2>&1
    - echo ORACLE_HOME=/opt/oracle/product/23c/dbhomeFree >> /etc/environment
    - echo ORACLE_SID=FREE >> /etc/environment
    - echo ORAENV_ASK=NO >> /etc/environment
```
