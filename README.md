# aws-ec2-dynamic-dns

In order to eliminate the use of elastic IPs on EC2 instances, we will implement a dynamic DNS system that works at the EC2 level by utilizing an EventBridge event to fire when a machine is started. This event will trigger a Lambda function, which will grab the dynamic IP address assigned by AWS, then update a Route53 DNS record with the new IP address. 

# Create the Zone and appropriate records

First, we will assume you have a nice, simple domain you want to use as the root domain, but you probably don't have this hosted in Route53 already. That's ok, we can just use a subdomain. 

Create the zone first in Route53

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/02eb881e-7736-4d99-a1a4-64fa23434ff1)

Take note of the nameservers. You'll need to then add these nameservers to an NS record at the registrar, but only an NS record for your specific subdomain

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/e0d7d22d-171d-46c5-be27-9d833a36a086)

It would be slightly more efficient to migrate the root domain over to Route53 directly, but this is a nice solution if it's impractical to do so for the time being. 

# Create the EventBridge event

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/918ee837-672e-4793-8904-8c13a7370689)

![image](https://github.com/githubarooski/aws-ec2-dynamic-dns/assets/62593128/9d577439-4154-4b4e-8bb7-48d9d1ced918)

Event Pattern
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"]
}

