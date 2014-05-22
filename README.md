regaDBenvSetup-test
===================
reference-- https://github.com/akumria/puppet-postgresql
          --https://rega.kuleuven.be/cev/regadb/download/files/regadb-install-doc.pdf
          
    The vagrant file provided by Pascal
    Vagrant::Config.run do |config|
  # Box setup.
  config.vm.box = "precise32"
  config.vm.box_url = "http://files.vagrantup.com/precise32.box"
  config.vm.customize ["modifyvm", :id, "--memory", 768]

  # Network setup
  config.vm.network :bridged
  config.vm.forward_port 8080, 8084

  # Shared folder setup
  config.vm.share_folder "example", "/example", "./"

  # Ensure packages are up to date
  config.vm.provision :shell do |shell|
    shell.inline = "sudo apt-get update"
    end

  # Install required puppet modules
  config.vm.provision :shell do |shell|
      shell.inline = "mkdir -p /etc/puppet/modules;
             puppet module install puppetlabs/postgresql;
             puppet module install duritong/sysctl"
    end

  # Provisioning setup
  config.vm.provision :puppet do |puppet|
      puppet.manifests_path = "./manifests"
      puppet.options = ["--verbose"]
      puppet.manifest_file  = "example.pp"  
     end
end
------------------------------------------------------------------------
puppet file provided

# Variables
$postgresql_postgres_password = "example-postgres-password"
$postgresql_example_user = "example"          # These need to be same as the
$postgresql_example_password = "example-password"  # contents of hibernate.properties in
$postgresql_example_database_name = "example"    # the artifacts directory

# Defaults for exec
Exec {
  path => ["/bin", "/usr/bin", "/usr/local/bin", "/usr/local/sbin", "/sbin", "/usr/sbin"],
  user => "root"
}

# Kernel memory settings
$postgres_sysctl_settings = {
  "kernel.shmmax"  => { value => 268435456 },
  "net.core.rmem_max"  => { value =>  524288 },
    "net.core.wmem_max"  => { value =>  524288 }
}

# Make sure that we tweak the kernel before installing Postgresql
$postgres_sysctl_dependency = {
  before => Class["postgresql::server"]
}

# Generate the sysctl resources
create_resources(sysctl::value, $postgres_sysctl_settings, $postgres_sysctl_dependency)

# Install Postgresql server
class { "postgresql::server":
  postgres_password  => $postgresql_postgres_password
}

# Postgres configuration tweaks for the example
$postgres_server_conf ={
  "shared_buffers" => { value => "64MB" },
  "work_mem" => { value => "8MB" },
  "maintenance_work_mem" => { value => "32MB" },
  "effective_cache_size" => { value => "256MB" },
  "checkpoint_segments" => { value => "8" },
  "checkpoint_completion_target" => { value => "0.8"}
}

# Generate postgres configuration resources
create_resources(postgresql::server::config_entry, $postgres_server_conf)

# Extract the database
exec { "extract-example-database": 
  cwd  => "/example/artifacts",
  command  => "gzip -d example-20131120.sql.gz",
  creates => "/example/artifacts/example-20131120.sql"
}

# Create example database
postgresql::server::db { $postgresql_example_database_name:
  user     => $postgresql_example_user,
  password => $postgresql_example_password,
  before => Exec["import-example-dump"]
}

# Import example database dump
exec { "import-example-dump":
  cwd  => "/example/artifacts",
  environment => [ "PGPASSWORD=$postgresql_example_password" ],
  command  => "psql -U example -h localhost example < example-20131120.sql"
}

# Install Tomcat
package { "tomcat6": 
  ensure  => latest,
  require => Exec["import-example-dump"]
}

# Define Tomcat service
service { "tomcat6":
    ensure  => "running",
    enable  => "true",
    require => Package["tomcat6"]
}

# Configure Tomcat memory
file { "/usr/share/tomcat6/bin/setenv.sh":
  source  => "/example/artifacts/setenv.sh",
  owner  => "tomcat6",
  group   => "tomcat6",
  mode   => "a+x",
  require => Package["tomcat6"],
  notify   => Service["tomcat6"]
}

# Download example 2.13 from S3
exec { "download-example-2.13":
  cwd  => "/example/artifacts",
  command  => "wget https://s3-eu-west-1.amazonaws.com/moasis/shared-deploy-artifacts/example.war",
  creates => "/example/artifacts/example.war",
  timeout  => 0
}

# Create example2_HOME directory
file { "/opt/example2":
    ensure   => "directory",
    owner  => "tomcat6",
  group   => "tomcat6",
  require  => Package["tomcat6"]
}

# Copy database properties into place
file { "/opt/example2/hibernate.properties":
    source   => "/example/artifacts/hibernate.properties",
    owner  => "tomcat6",
  group   => "tomcat6",
  require  => File["/opt/example2"],
  before  => File["/var/lib/tomcat6/webapps/example.war"]
}

# Deploy example2 WAR file
file { "/var/lib/tomcat6/webapps/example.war":
  source  => "/example/artifacts/example.war",
  owner  => "tomcat6",
  group   => "tomcat6",
  require => Package["tomcat6"],
  notify  => Service["tomcat6"]
}




