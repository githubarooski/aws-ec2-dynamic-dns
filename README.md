# aws-ec2-dynamic-dns

In order to eliminate the use of elastic IPs on EC2 instances, we will implement a dynamic DNS system that works at the EC2 level by utilizing an EventBridge event to fire when a machine is started. This event will trigger a Lambda function, which will grab the dynamic IP address assigned by AWS, then update a Route53 DNS record with the new IP address. 

# Create the Zone and appropriate records

First, we will assume you have a nice, simple domain you want to use as the root domain, but you probably don't have this hosted in Route53 already. That's ok, we can just use a subdomain. 

Create the zone first in Route53. The zone should be for a subdomain in this example. 

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/fbf43572-b430-448d-a68c-63feef4eb2f8)

Take note of the nameservers. You'll need to then add these nameservers to an NS record at the root domain's DNS configuration:

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/e0d7d22d-171d-46c5-be27-9d833a36a086)

It would be slightly more efficient to migrate the root domain over to Route53 directly, but this is a nice solution if it's impractical to do so for the time being. 

# Create a Lambda function
Make a new Lambda function with the following settings:

Name: get-instance-IP-assign-dynamic-DNS

Env: Python

Timeout: 3 minutes (very important)

Code (updated with your Route53 Zone ID on line 3)
```
import boto3
import json 
hostedZoneID="123456789"


def lambda_handler(event, context):
    
    ec2 = boto3.client('ec2')

    route53 = boto3.client('route53')
    
    instance_id = event['detail']['instance-id']

    print(instance_id)

    if not instance_id:
        return "Instance ID not provided in the event."

    response = ec2.describe_instances(InstanceIds=[instance_id])

    if len(response['Reservations']) == 0:
        print("Instance {instance_id} not found")
        return f"Instance {instance_id} not found"
    
    instance_state = response['Reservations'][0]['Instances'][0]['State']['Name']
    
    if instance_state == 'running':
        print("instance is running")
        instancePublicDNS = response['Reservations'][0]['Instances'][0].get('PublicDnsName', 'N/A')
        public_ip = response['Reservations'][0]['Instances'][0].get('PublicIpAddress', 'N/A')
        tags = response['Reservations'][0]['Instances'][0].get('Tags', [])
        for tag in tags:
            if tag['Key']=="DNS":
                dns=tag['Value']
                print("changing {dns} to {public_ip}")
                route53.change_resource_record_sets(
                    HostedZoneId= hostedZoneID,
                    ChangeBatch={
                        'Changes': [
                            {
                                'Action': 'UPSERT',
                                'ResourceRecordSet':
                                    {
                                        'Name': dns,
                                        'Type': 'A',
                                        'TTL': 5,
                                        'ResourceRecords': [
                                            {
                                                'Value': public_ip
                                            },
                                        ],
                                    }
                            }
                        ]
                    }
                )
                print("DNS for {dns} updated to {public_ip}") 
        return {'statusCode': 200}
    else:
        return f"The EC2 instance {instance_id} is not running. Current state: {instance_state}"
```


# Create the EventBridge event
Name: monitor-instances-when-starting
![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/7991575b-6fa4-43a5-bdcc-755574e975b8)

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/5ae93115-04ff-4910-9786-cb75fbf42233)

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/c95db0a1-edeb-423d-ad48-8956135a9412)

Event Pattern
```
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running"]
  }
}
```

Target will be our Lambda function
![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/3b9962b5-a42f-4730-bc78-620a2eac857a)

# Tag the EC2 instance

The Lambda code is written to parse a tag associated with the EC2 instance. 

Key = DNS

Value = full DNS hostname

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/bfa6f0a1-43ca-4921-9d12-13b88a0a759f)


If everything goes well, you'll see a new/updated A record in your Route53 zone once you start the machine

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/0f86b0e9-cf2d-4ee4-8acd-674d1d6b919a)

You can now use this hostname with remote desktop solutions like NICE DCV to create a persistent shortcut to the instance without paying for an elastic IP address. 

#Example NICE DCV shortcut code. Save as a .dcv file with Notepad.

```
[version]
format=1.0
[connect]
host=hk-contrib-01.prod.example.net
password="Password"
port=8443
weburlpath=
sessionid=
user=Administrator
proxyport=0
proxytype=system
```










