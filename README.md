varnish-dynbackend
==================

Aws ELB EC2 dynamic backends for varnish



composer.json
--
{
    "require": {
        "aws/aws-sdk-php": "2.*"
    }
}


varnish-dynbackend.php
--
<?php
define("ROOT_DIR",'c:\repos\update_frontal_cache'.DIRECTORY_SEPARATOR);
define("SSH_CLIENT", "C:\Program Files (x86)\Bitvise SSH Client\stermc.exe");
require ROOT_DIR.'vendor\autoload.php';
require ROOT_DIR.'awscredentials.php';

use Aws\Ec2\Ec2Client;
$client = Ec2Client::factory(array(
    'key'    => AWSKEY,
    'secret' => AWSSECRET,
    'region' => 'us-west-2'
));

if (getmyuid()!=0) die("You should be root\n");

if ( !@SSH_KEY )
        die("ssh key is it not defined in awscredentials.php");
if ( !file_exists(ROOT_DIR.SSH_KEY) )
        die("ssh key does not exist in ".ROOT_DIR.SSH_KEY);

if ( !file_exists(SSH_CLIENT) )
        die("Bitwise SSH client is not installed on ".SSH_CLIENT);

$result = $client->describeInstances(array(
        'DryRun' => false,
        'Filters' => array( array( 'Name' => 'group-name', 'Values' => array('SG-FRONTALCACHE'),),
        ),
))->toArray();

//print_r($result);die();

$aInstances = array();
foreach ($result["Reservations"] as $reservation) {
    $instances = $reservation['Instances'];
    foreach ($instances as $instance) {
        // echo 'Instance Name: ' . $instanceName . PHP_EOL.'---> State: ' . $instance['State']['Name'] . PHP_EOL.'--->Instance ID: ' . $instance['InstanceId'] . PHP_EOL. '---> Image ID: ' . $instance['ImageId'] . PHP_EOL. '---> Image ID: ' . $instance['ImageId'] . PHP_EOL. '---> Private Dns Name: ' . $instance['PrivateDnsName'] . PHP_EOL. '---> Intance Type: ' . $instance['InstanceType'] . PHP_EOL. '---> Security Group: ' . $instance['SecurityGroups'][0]['GroupName'] . PHP_EOL;
        // Only take in care RUNNING instances
        if ($instance["State"]["Code"] == 16 ) {
                $aInstances[] = $instance['PrivateDnsName'];
        }
    }
}


if (!count($aInstances) ) die("No backends available to refresh varnish config");

foreach ( $aInstances as $cInstanciaKey=>$cInstanciaValue )  {
        echo     "Updating $cInstanciaValue cache server...\n";
        $cCmd = "\"".SSH_CLIENT."\" -unat=y -keypairFile=".ROOT_DIR.SSH_KEY." root@".$cInstanciaValue."-cmd=/root/bin-utils/varnishupdate/updatebackends.php";
        echo $cCmd;
        //exec($cCmd,$o,$ret);
        //print_r($o);
}
