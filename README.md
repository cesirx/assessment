# assessment_repo

## Project description

Automated deployment of a multilayer application: 
* 1x Load Balancer
* 2x Web servers
* 1x Database

The following tools are used along the project:
* Dockers: each application server will be deployed in its own container.
* Ansible: remote docker host setup and container orchestration is accomplished using Ansible playbooks.

## Project Structure

<pre>

.
|-- deployment.yml                               <> Playbook to (re)deploy the whole scenario
|-- group_vars
|   |-- docker_hosts                             <> Variables for "common" role
|   |-- httpd_servers                            <> Variables for "httpd" role
|   |-- mysql_servers                            <> Variables for "mysql" role
|   `-- nginx_servers                            <> Variables for "nginx" role
|-- hosts.yml                                    <> Inventory file
|-- README.md
|-- roles
|   |-- common                                   <> "common" role folder
|   |   `-- tasks
|   |       `-- main.yml                         <> Role to setup Docker Hosts
|   |-- httpd                                    <> "httpd" role folder
|   |   |-- files
|   |   |   `-- Dockerfile                       <> Dockerfile to assemble a custom httpd image build: get latest image and install/upgrade packages
|   |   |-- tasks
|   |   |   `-- main.yml                         <> Role to deploy web applications
|   |   `-- templates
|   |       `-- index.html.j2                    <> httpd index.html file (jinja2 template format)
|   |-- mysql                                    <> "mysql" role folder
|   |   |-- files
|   |   |   |-- Dockerfile                       <> Dockerfile to assemble a custom mysql image build: get latest image and install/upgrade packages
|   |   |   `-- my.cnf                           <> mysql config file
|   |   |-- tasks
|   |   |   `-- main.yml                         <> Role to deploy databse application
|   |   `-- templates
|   |       `-- sql_table.sql.j2                 <> mysql script to provision a sample database table (jinja2 template format)
|   `-- nginx                                    <> "nginx" role folder
|       |-- files
|       |   `-- Dockerfile                       <> Dockerfile to assemble a custom nginx image build: get latest image and install/upgrade packages
|       |-- tasks
|       |   `-- main.yml                         <> Role to deploy load balancer application
|       `-- templates
|           `-- nginx.conf.j2                    <> nginx configuration file (jinja2 template format)
`-- vault.yml                                    <> vault (not in use)

</pre>

## Project requirements/caveats:

* For CLI playbook style execution:
1. A functional Ansible Control Node from which playbooks will be executed.
2. A pool of docker hosts running Ubuntu or CentOS/RHEL to run the containers. 
3. For high availability in the webserver layer, at least 2 docker hosts would be required.
4. Network connectivity between the Ansible Control Node and the Docker Hosts ir required.

* For Github Actions (CI-CD Pipeline) to work:
1. Github Actions run on "runner" nodes, which can be provided by Git or self-provisioned.
2. Network connectivity between the Ansible Control Node and the Docker Hosts ir required.
3. "common" role includes a task (authorized_key) to copy ssh publick key to Docker Hosts. However, that only works when playbook (deployment.yml) is executed from CLI (fails when run from Github Actions ¿¿??). The workaround is to comment that task and manunally copy SSH public key from Ansible Control Node to each Docker Host (ssh-copy-id).
4. Tested in a self-provisioned runner.

For the Load Balancer to respond on http://ucpe.swisscom.com, the swisscom.com domain should have a type A registry to resolve "ucpe" to the ip address of the NGINX server.
For the purpose of this assessment, it should be enough to add a new entry to your /etc/hosts:
"ip_nginx ucpe.swisscom.com"

## Project setup

- Variables:
-- group_vars/docker_hosts --> configure docker hosts ssh user (ansible_user) and ssh password (ansible_ssh_pass) and adjust database variables

- Inventory:
-- hosts.yml --> Add/remove servers for each group. Adjust ip address of each docker host (ansible_ssh_host) according to your environment

## Project execution

* For CLI playbook style execution:
1. In Ansible Control Node clone repo: "git clone https://github.com/cesirx/assessment_repo.git"
2. For a full (re)deployment execute the command as follow: "ansible-playbook deployment.yml -i hosts.yml"
3. Specific applications can be redeployed/updated using tags: "ansible-playbook deployment.yml -i hosts.yml --tags=balancer"

* For Github Actions (CI-CD Pipeline) execution:
The configured Github action is automatically launched on "git push" 

## Missing parts

* "Write a REST call to READ any data from the SQL databse":

As for the given description, I understand that the goal is to run an API REST call from the Web servers towards the mysql application.
The problem is that mysql does not include a REST API.
I discovered a REST API plugin for mysql (https://labs.mysql.com/) but was not able to make it work...

## Future improvements

* Ansible secrets management: use Vault to keep secrets safe
* Web servers update: web application layer could be seamlessly (without enduser impact) updated. To do so web servers should be updated one by one following these steps. For each web server:
1. Remove web server from nginx proxy pass policy so that no more connections are routed to this webserver.
2. Wait for web server active connections to end.
3. Update web server using "deployment.yml" playbook pointed to that specifi webserver.
* Service continuity after updates: the playbook gets :latest available docker image of each application and upgrades its installed packets to the latest version (dockerfile)... but what if this makes the application to fail? Some kind of sanity checks should be run after each docker is deployed and revert to a previous version in case of failure

