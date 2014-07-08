# -*- mode: ruby -*-
# vi: set ft=ruby :

# Assumes a box from https://github.com/jayjanssen/packer-percona

# This sets up 3 nodes with a common PXC, but you need to run bootstrap.sh to connect them.

# This is designed for aws
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'aws'

require File.dirname(__FILE__) + '/lib/vagrant-common.rb'

$mysql_version = "56"
$region = 'us-west-1'
$pxc_instance_size = 'm3.large'  
$tester_instance_size = 'c3.large'
$num_teams = 10

$puppet_config = {
	"percona_server_version"	=> $mysql_version,
	'innodb_buffer_pool_size' => '128M',
	'innodb_log_file_size' => '64M',
	'innodb_flush_log_at_trx_commit' => '1',
	'server_id' => 1,
  'extra_mysqld_config' => '
log-bin
log-slave-updates
sync_binlog=1  
gtid_mode = ON
enforce-gtid-consistency
'
}


# Node names
# aws_region is where to bring up the node
$pxc_nodes = {
	'node1' => {
		'aws_region' => $region,
		'security_groups' => ['default','pxc']
	},
	'node2' => {
		'aws_region' => $region,
		'security_groups' => ['default','pxc'] 
	},
	'node3' => {
		'aws_region' => $region,
		'security_groups' => ['default','pxc']
	}
}

def build_team(config, team_name )
	# Create all three nodes identically except for name and ip
	$pxc_nodes.each_pair { |name, node_params|
    node_name = team_name + name
		config.vm.define node_name do |node_config|
			node_config.vm.hostname = node_name
      
      # Provisioners
      provision_puppet( node_config, "base.pp" )
      provision_puppet( node_config, "pxc_server.pp" ) { |puppet|  
        puppet.facter = $puppet_config
      }
      provision_puppet( node_config, "pxc_client.pp" ) { |puppet|
        puppet.facter = $puppet_config
      }
      provision_puppet( node_config, "percona_toolkit.pp" )
      provision_puppet( node_config, "myq_gadgets.pp" )
			
      provision_puppet( node_config, "sysbench.pp" )
			
			# Setup a sysbench environment and test user on node1
			if name == 'node1'
	      provision_puppet( node_config, "sysbench_load.pp" ) { |puppet|
					puppet.facter = {
						'tables' => 20,
						'rows' => 1000000,
						'threads' => 4
					}
				}
				provision_puppet( node_config, "test_user.pp" )
			end
  
      # Provider -- aws only
    	provider_aws( "PXC training #{node_name}", node_config, $pxc_instance_size, node_params['aws_region'], node_params['security_groups']) { |aws, override|
    		aws.block_device_mapping = [
    			{
    				'DeviceName' => "/dev/sdb",
    				'VirtualName' => "ephemeral0"
    			}
    		]
        provision_puppet( override, "pxc_server.pp" ) {|puppet|
          puppet.facter = {"datadir_dev" => "xvdb"}
        }
    	}

    end
  }
  
  # Create a tester node for the team
  tester_name = team_name + 'test'
  config.vm.define tester_name do |node_config|
    provision_puppet( node_config, "base.pp" )
    provision_puppet( node_config, "pxc_client.pp" ) { |puppet|
      puppet.facter = $puppet_config
    }
    provision_puppet( node_config, "sysbench.pp" )
    
    # Provider -- aws only
  	provider_aws( "PXC training #{tester_name}", node_config, $tester_instance_size, $region, ['default','pxc']) { |aws, override|
  		aws.block_device_mapping = [
  			{
  				'DeviceName' => "/dev/sdb",
  				'VirtualName' => "ephemeral0"
  			}
  		]
  	}
  end
end


Vagrant.configure("2") do |config|
	config.vm.box = "perconajayj/centos-x86_64"
	config.ssh.username = "root"

  $i = 1;
  while $i <= $num_teams do
    build_team( config, "team#{$i}"  )
    $i += 1;
  end

end

