.. _cloudformation-template-example:

CloudFormation template examples
================================

We are utilizing ``cfn-init`` to populate objects on the target instances. You
will need to ensure that the `CFN helper scripts <http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-helper-scripts-reference.html>`__ are installed on the servers.

Also, you will need to have Python 2.7 as well as ``cumulus-bundle-handler`` on
all servers.

If you are running on a Windows server, please make sure that the ``UserData``
is read on boot. You should take a look at `Bootstrapping AWS CloudFormation Windows Stacks <http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-windows-stacks-bootstrapping.html>`__ and `Configuring a Windows Instance Using the EC2Config Service <http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/UsingConfig_WinAMI.html>`__.

Linux host with an Auto Scaling Group
-------------------------------------

Here's an example CloudFormation JSON document for a webserver in an Auto Scaling Group with Cumulus configured.
::

    {
      "Description" : "Webservers for Cumulus test",

      "Parameters" : {

        "KeyName" : {
          "Description" : "AWS key to use",
          "Type" : "String",
          "Default": "cumulus-prod"
        },

        "InstanceType" : {
          "Description" : "EC2 instance type",
          "Type" : "String",
          "Default" : "t1.micro",
          "AllowedValues" : [ "t1.micro","m1.small","m1.medium","m1.large","m1.xlarge","m2.xlarge","m2.2xlarge","m2.4xlarge","c1.medium","c1.xlarge","cc1.4xlarge","cc2.8xlarge","cg1.4xlarge"],
          "ConstraintDescription" : "must be a valid EC2 instance type."
        },

        "CumulusEnvironment": {
          "Description" : "Cumulus environment name",
          "Type": "String"
        },

        "CumulusBundleBucket": {
          "Description" : "Cumulus bundle bucket name",
          "Type": "String"
        },

        "CumulusVersion": {
          "Description" : "Version of the software",
          "Type": "String"
        }
      },

      "Mappings" : {
        "AWSInstanceType2Arch" : {
          "t1.micro"    : { "Arch" : "64" }
        },

        "AWSRegionArch2AMI" : {
          "eu-west-1"      : { "32" : "NOT_YET_SUPPORTED", "64" : "ami-db595faf", "64HVM" : "NOT_YET_SUPPORTED" }
        }
      },

      "Resources" : {

        "WebServerLaunchConfiguration" : {
          "Type": "AWS::AutoScaling::LaunchConfiguration",
          "Metadata" : {
            "AWS::CloudFormation::Init" : {
              "configSets" : {
                "cumulus": [ "fileConfig", "commandConfig" ]
              },

              "fileConfig" : {
                "files" : {
                  "/etc/cumulus/metadata.conf" : {
                    "content" : { "Fn::Join" : ["", [
                      "[metadata]\n",
                      "access-key-id: ", { "Ref" : "WebServerKeys" }, "\n",
                      "secret-access-key: ", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]}, "\n",
                      "region: ", {"Ref" : "AWS::Region"}, "\n",
                      "bundle-bucket: ", { "Ref" : "CumulusBundleBucket"}, "\n",
                      "environment: ", { "Ref" : "CumulusEnvironment" }, "\n",
                      "bundle-types: generic, webserver\n",
                      "bundle-extraction-paths:\n",
                      "    generic -> /etc/example\n",
                      "    webserver -> /\n",
                      "version: ", { "Ref" : "CumulusVersion" }, "\n"
                    ]]},
                    "mode"  : "000644",
                    "owner" : "root",
                    "group" : "root"
                  },

                  "/etc/cfn/cfn-credentials" : {
                    "content" : { "Fn::Join" : ["", [
                      "AWSAccessKeyId=", { "Ref" : "WebServerKeys" }, "\n",
                      "AWSSecretKey=", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]}, "\n"
                    ]]},
                    "mode"    : "000400",
                    "owner"   : "root",
                    "group"   : "root"
                  },

                  "/etc/cfn/cfn-hup.conf" : {
                    "content" : { "Fn::Join" : ["", [
                      "[main]\n",
                      "stack=", { "Ref" : "AWS::StackName" }, "\n",
                      "credential-file=/etc/cfn/cfn-credentials\n",
                      "region=", { "Ref" : "AWS::Region" }, "\n",
                      "interval=1\n"
                    ]]},
                    "mode"    : "000400",
                    "owner"   : "root",
                    "group"   : "root"
                  },

                  "/etc/cfn/hooks.d/cfn-auto-reloader.conf" : {
                    "content": { "Fn::Join" : ["", [
                      "[cfn-auto-reloader-hook]\n",
                      "triggers=post.update\n",
                      "path=Resources.WebServerLaunchConfiguration.Metadata.AWS::CloudFormation::Init\n",
                      "action=/usr/local/bin/cfn-init -c cumulus -s ",
                          { "Ref" : "AWS::StackName" }, " -r WebServerLaunchConfiguration ",
                           " --credential-file /etc/cfn/cfn-credentials ",
                           " --region ", { "Ref" : "AWS::Region" }, "\n",
                      "runas=root\n"
                    ]]}
                  }
                }
              },

              "commandConfig" : {
                "commands" : {
                  "cumulus_bundle_handler" : {
                    "command" : "/usr/local/bin/cumulus_bundle_handler.py",
                    "ignoreErrors" : "false"
                  }
                }
              }
            }
          },

          "Properties": {
            "ImageId" : {
              "Fn::FindInMap" : [
                "AWSRegionArch2AMI",
                { "Ref" : "AWS::Region" },
                { "Fn::FindInMap" : [
                  "AWSInstanceType2Arch",
                  { "Ref" : "InstanceType" },
                  "Arch"
                ] }
              ]
            },
            "InstanceType"   : { "Ref" : "InstanceType" },
            "SecurityGroups" : [ {"Ref" : "WebServerSecurityGroup"} ],
            "KeyName"        : { "Ref" : "KeyName" },
            "UserData"       : { "Fn::Base64" : { "Fn::Join" : ["", [
              "#!/bin/bash -v\n",

              "# Install cfn bootstraping tools\n",
              "apt-get update\n",
              "apt-get -y install python-setuptools python-pip\n",
              "easy_install https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.tar.gz\n",

              "# Helper function\n",
              "function error_exit\n",
              "{\n",
              "  /usr/local/bin/cfn-signal -e 1 -r \"$1\" '", { "Ref" : "WaitHandle" }, "'\n",
              "  exit 1\n",
              "}\n",

              "# Make sure we have the latest cumulus-bundle-handler\n",
              "pip install --upgrade cumulus-bundle-handler || error_exit 'Failed upgrading cumulus-bundle-handler to the latest version'\n",

              "# Install software\n",
              "/usr/local/bin/cfn-init -v -c cumulus -s ", { "Ref" : "AWS::StackName" }, " -r WebServerLaunchConfiguration ",
              "    --access-key ",  { "Ref" : "WebServerKeys" },
              "    --secret-key ", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]},
              "    --region ", { "Ref" : "AWS::Region" }, " >> /var/log/cfn-init.log || error_exit 'Failed to run cfn-init'\n",

              "# Start up the cfn-hup daemon to listen for changes to the Web Server metadata\n",
              "/usr/local/bin/cfn-hup || error_exit 'Failed to start cfn-hup'\n",

              "# All is well so signal success\n",
              "/usr/local/bin/cfn-signal -e 0 -r \"Webserver setup complete\" '", { "Ref" : "WaitHandle" }, "'\n"

            ]]}}
          }
        },

        "WebServerAutoScalingGroup": {
          "Type": "AWS::AutoScaling::AutoScalingGroup",
          "Version": "2009-05-15",
          "Properties": {
            "AvailabilityZones": { "Fn::GetAZs": "" },
            "LaunchConfigurationName": { "Ref": "WebServerLaunchConfiguration" },
            "MinSize": "1",
            "MaxSize": "1",
            "Tags" : [{
              "Key"   : "Name",
              "Value" : { "Fn::Join"  : [ "-" , [ { "Ref" : "AWS::StackName" }, "webserver" ]]},
              "PropagateAtLaunch" : "true"
            }]
          }
        },

        "WebServerUser" : {
          "Type" : "AWS::IAM::User",
          "Properties" : {
            "Path": "/",
            "Policies": [
              {
                "PolicyName": "cloudformation",
                "PolicyDocument": { "Statement":[{
                  "Effect":"Allow",
                  "Action":[
                    "cloudformation:DescribeStackResource",
                    "s3:*"
                  ],
                  "Resource":"*"
                }]}
              }
            ]
          }
        },

        "WebServerKeys" : {
          "Type" : "AWS::IAM::AccessKey",
          "Properties" : {
            "UserName" : {"Ref": "WebServerUser"}
          }
        },

        "WaitHandle" : {
          "Type" : "AWS::CloudFormation::WaitConditionHandle"
        },

        "WaitCondition" : {
          "Type" : "AWS::CloudFormation::WaitCondition",
          "DependsOn" : "WebServerAutoScalingGroup",
          "Properties" : {
            "Handle" : {"Ref" : "WaitHandle"},
            "Timeout" : "600"
          }
        },

        "WebServerSecurityGroup" : {
          "Type" : "AWS::EC2::SecurityGroup",
          "Properties" : {
            "GroupDescription" : "Enable HTTP access via port 80/443 and SSH access",
            "SecurityGroupIngress" : [
              {"IpProtocol" : "tcp", "FromPort" : "80", "ToPort" : "80", "CidrIp" : "0.0.0.0/0"},
              {"IpProtocol" : "tcp", "FromPort" : "443", "ToPort" : "443", "CidrIp" : "0.0.0.0/0"},
              {"IpProtocol" : "tcp", "FromPort" : "22", "ToPort" : "22", "CidrIp" : "0.0.0.0/0"},
              {"IpProtocol" : "icmp", "FromPort" : "-1", "ToPort" : "-1", "CidrIp" : "0.0.0.0/0"}
            ]
          }
        }
      }
    }

Windows instance in a VPC
-------------------------

Below is an example of a Windows instance in a VPC.
::

    {
      "Description" : "Example with Windows instance and VPC",

      "AWSTemplateFormatVersion" : "2010-09-09",

      "Parameters" : {

        "InstanceType" : {
          "Description" : "Instance type to use",
          "Type" : "String",
          "AllowedValues" : [ "t1.micro","m1.small","m1.medium","m1.large","m1.xlarge","m2.xlarge","m2.2xlarge","m2.4xlarge","c1.medium","c1.xlarge","cc1.4xlarge","cc2.8xlarge","cg1.4xlarge"],
          "ConstraintDescription" : "must be a valid EC2 instance type."
        },

        "CumulusEnvironment": {
          "Description" : "Cumulus environment name",
          "Type": "String"
        },

        "CumulusBundleBucket": {
          "Description" : "Cumulus bundle bucket name",
          "Type": "String"
        },

        "CumulusVersion": {
          "Description" : "Version of the software",
          "Type": "String"
        }

      },

      "Mappings" : {

        "AWSInstanceType2Arch" : {
          "m1.small"    : { "Arch" : "64" },
          "m1.medium"   : { "Arch" : "64" },
          "m2.xlarge"   : { "Arch" : "64" },
          "m2.2xlarge"  : { "Arch" : "64" },
          "m2.4xlarge"  : { "Arch" : "64" },
          "m3.medium"   : { "Arch" : "64" },
          "m3.large"    : { "Arch" : "64" },
          "m3.xlarge"   : { "Arch" : "64" },
          "m3.2xlarge"  : { "Arch" : "64" },
          "m1.medium"   : { "Arch" : "64" }
        },

        "AWSRegionArch2AMI": {
          "eu-west-1": {
            "32" : "NOT_YET_SUPPORTED",
            "64" : "ami-12345678",
            "64HVM" : "NOT_YET_SUPPORTED"
          }
        }
      },

      "Resources" : {

        "WebServer" : {
          "Type" : "AWS::EC2::Instance",
          "Properties" : {
            "ImageId" : {
              "Fn::FindInMap" : [
                "AWSRegionArch2AMI",
                { "Ref" : "AWS::Region" },
                { "Fn::FindInMap" : [ "AWSInstanceType2Arch", { "Ref" : "InstanceType" }, "Arch" ] }
              ]
            },
            "KeyName": "sebdah-test",
            "InstanceType"   : { "Ref" : "InstanceType" },
            "NetworkInterfaces" : [{
              "GroupSet"                 : [{ "Ref" : "WebServerSecurityGroup" }],
              "AssociatePublicIpAddress" : "true",
              "DeviceIndex"              : "0",
              "DeleteOnTermination"      : "true",
              "SubnetId"                 : "subnet-12345678"
            }],
            "Tags" : [
              { "Key": "Name",    "Value" : { "Ref" : "AWS::StackName" } },
              { "Key": "Project", "Value" : { "Ref" : "Project" } }
            ],
            "UserData" : { "Fn::Base64" : { "Fn::Join" : ["", [
              "<powershell>\n",

              "pip install -U cumulus-bundle-handler\n",

              "cfn-init.exe -v -c cumulus ",
              "    -s ", { "Ref" : "AWS::StackName" },
              "    -r WebServer ",
              "    --access-key ",  { "Ref" : "WebServerKeys" },
              "    --secret-key ", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]},
              "    --region ", { "Ref" : "AWS::Region" }, "\n",

              "cfn-signal.exe -e $LASTEXITCODE ", { "Fn::Base64" : { "Ref" : "WaitHandle" }}, "\n",

              "</powershell>"

            ]]}}
          },
          "Metadata" : {

            "AWS::CloudFormation::Init" : {
              "configSets" : {
                "cumulus": [ "fileConfig", "commandConfig", "serviceConfig" ]
              },

              "fileConfig" : {
                "files" : {
                  "c:\\cumulus\\conf\\metadata.conf" : {
                    "content" : { "Fn::Join" : ["", [
                      "[metadata]\n",
                      "access-key-id: ", { "Ref" : "WebServerKeys" }, "\n",
                      "secret-access-key: ", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]}, "\n",
                      "region: ", {"Ref" : "AWS::Region"}, "\n",
                      "bundle-bucket: ", { "Ref" : "CumulusBundleBucket"}, "\n",
                      "environment: ", { "Ref" : "CumulusEnvironment" }, "\n",
                      "bundle-types: web\n",
                      "bundle-extraction-paths:\n",
                      "    web -> c:\\InetPub\\wwwroot\n",
                      "version: ", { "Ref" : "CumulusVersion" }, "\n"
                    ]]},
                    "mode"  : "000644",
                    "owner" : "root",
                    "group" : "root"
                  },

                  "c:\\cfn\\cfn-credentials" : {
                    "content" : { "Fn::Join" : ["", [
                      "AWSAccessKeyId=", { "Ref" : "WebServerKeys" }, "\n",
                      "AWSSecretKey=", {"Fn::GetAtt": ["WebServerKeys", "SecretAccessKey"]}, "\n"
                    ]]},
                    "mode"    : "000400",
                    "owner"   : "root",
                    "group"   : "root"
                  },

                  "c:\\cfn\\cfn-hup.conf" : {
                    "content" : { "Fn::Join" : ["", [
                      "[main]\n",
                      "stack=", { "Ref" : "AWS::StackName" }, "\n",
                      "credential-file=c:\\cfn\\cfn-credentials\n",
                      "region=", { "Ref" : "AWS::Region" }, "\n",
                      "interval=1\n"
                    ]]},
                    "mode"    : "000400",
                    "owner"   : "root",
                    "group"   : "root"
                  },

                  "c:\\cfn\\hooks.d\\cfn-auto-reloader.conf" : {
                    "content": { "Fn::Join" : ["", [
                      "[cfn-auto-reloader-hook]\n",
                      "triggers=post.update\n",
                      "path=Resources.WebServer.Metadata.AWS::CloudFormation::Init\n",
                      "action=cfn-init.exe -c cumulus -s ",
                          { "Ref" : "AWS::StackName" }, " -r WebServer ",
                           " --credential-file c:\\cfn\\cfn-credentials ",
                           " --region ", { "Ref" : "AWS::Region" }, "\n"
                    ]]}
                  }
                }
              },

              "commandConfig" : {
                "commands" : {
                  "cumulus-bundle-handler" : {
                    "command" : "python cumulus-bundle-handler",
                    "ignoreErrors" : false
                  },
                  "RecycleAppPool" : {
                    "command" : "C:\\windows\\System32\\inetsrv\\appcmd.exe recycle apppool DefaultAppPool",
                    "ignoreErrors" : false
                  }
                }
              },

              "serviceConfig" : {
                "services" : {
                  "windows" : {
                    "cfn-hup" : {
                      "enabled" : "true",
                      "ensureRunning" : "true",
                      "files" : ["c:\\cfn\\cfn-hup.conf", "c:\\cfn\\hooks.d\\cfn-auto-reloader.conf"]
                    }
                  }
                }
              }
            }
          }
        },

        "WebServerKeys" : {
          "Type" : "AWS::IAM::AccessKey",
          "Properties" : {
            "UserName" : {"Ref": "WebServerUser"}
          }
        },

        "WebServerUser" : {
          "Type" : "AWS::IAM::User",
          "Properties" : {
            "Path" : "/",
            "Policies" : [
              {
                "PolicyName" : "cloudformation",
                "PolicyDocument" : { "Statement":[{
                  "Effect" : "Allow",
                  "Action" : [
                    "cloudformation:DescribeStackResource",
                    "s3:*"
                  ],
                  "Resource" : "*"
                }]}
              }
            ]
          }
        },

        "WaitHandle" : {
          "Type" : "AWS::CloudFormation::WaitConditionHandle"
        },

        "WaitCondition" : {
          "Type" : "AWS::CloudFormation::WaitCondition",
          "DependsOn" : "WebServer",
          "Properties" : {
            "Handle"  : { "Ref" : "WaitHandle" },
            "Timeout" : "3600"
          }
        },

        "WebServerSecurityGroup" : {
          "Type": "AWS::EC2::SecurityGroup",
          "Properties" : {
            "VpcId": "vpc-12345678",
            "GroupDescription": "Allow all traffic",
            "SecurityGroupIngress": [
              {
                "IpProtocol": "tcp",
                "FromPort": "0",
                "ToPort": "65535",
                "CidrIp": "0.0.0.0/0"
              }
            ],
            "SecurityGroupEgress": [
              {
                "IpProtocol": "tcp",
                "FromPort": "0",
                "ToPort": "65535",
                "CidrIp": "0.0.0.0/0"
              }
            ],
            "Tags" : [
              { "Key": "Name",    "Value" : { "Ref" : "AWS::StackName" } }
            ]
          }
        }
      }
    }
