***Twoge BlogSite Application***

Description: Simple Flask-based blogging web application and “twitter killer”. Built using Flask, SQLAlchemy, and PostgreSQL. Cloud-based static web hosting configured utilizing AWS EC2, Amazon Load Balancing, AWS Auto Scaling, and Amazon Simple Notification Service.

***Twoge Architecture Diagram***

<img width="805" alt="image" src="https://github.com/perryb3693/twoge_app/assets/129805541/a9e1c429-c84b-49e2-a19f-7acbdbf7c30a">

**Overview**
- The Twoge VPC has public and private subnets in two availability zones
- The Twoge web servers run in the public subnets and receive traffic through a load balancer
- The web servers across both availability zones are both configured for auto scaling. 
- The database server runs in the private subnet and receives traffic from the web servers through the routing table. 
- The private subnets have access to the internet through the NAT gateway.

***Twoge Application Deployment using the AWS Web Interface***

**Building the Infrastructure**
First, configure a virtual private cloud network consisting of two public and two private subnets split between two availability zone for redundancy and availability. One NAT gateway will be provisioned for each private subnet for redundancy. Configure an Internet Gateway to route traffic out to the internet.

![image](https://github.com/perryb3693/twoge_app/assets/129805541/4fd77657-51d8-49b4-bd75-b40754c4af93)

An AWS S3 bucket will be used to store the Twoge app data files. Full access to S3 resources will be granted by configuring and assigning an IAM role.

![image](https://github.com/perryb3693/twoge_app/assets/129805541/72fb1a35-ef13-4738-9e5a-d3e7e0d8c693)

S3 IAM Role Configuration: Created to grant full access to the S3 storage to public users of Twoge
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "s3-object-lambda:*"
            ],
            "Resource": "*"
        }
    ]
}
```
Next, provision one EC2 Instance with all applicable settings. Once launched, create an image of the new EC2 instance to serve as the image for other required EC2 instances. 

- AMI: Amazon Linux 2 AMI
- Instance Type: t2.micro (free tier)
- Select existing or create a new key pair
- Select the VPC created in previous step
- Assign a subnet
- Select existing or create new security group for EC2 instance, allowing SSH and HTTP traffic
- Select storage size
- Adding the script below to the "user data" field allows for the EC2 instances to be configured with all neccessary services and dependencies needed to serve the web application: 
```
sudo yum update -y
sudo yum install git -y
git clone  https://github.com/chandradeoarya/twoge
cd twoge
sudo yum install python-pip -y
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
echo 'SQLALCHEMY_DATABASE_URI = "postgresql://postgres:password@twoge-privatedb.ctfbo8wbotzl.ca-central-1.rds.amazonaws.com"' > .env
echo '
Description=Gunicorn instance to serve twoge
Wants=network.target
After=syslog.target network-online.target
[Service]
Type=simple
WorkingDirectory=/home/ec2-user/twoge
Environment="PATH=/home/ec2-user/twoge/venv/bin"
ExecStart=/home/ec2-user/twoge/venv/bin/gunicorn app:app -c /home/ec2-user/twoge/gunicorn_config.py
Restart=always
RestartSec=10
[Install]
WantedBy=multi-user.target' > twoge.service
sudo cp twoge.service /etc/systemd/system/twoge.service
sudo systemctl daemon-reload
sudo systemctl enable twoge
sudo systemctl start twoge
sudo systemctl status twoge
```
An Amazon Load Balancer (ALB) is configured to listen for port 80 HTTP traffic going to the Twoge web servers and distribute the traffic between both of the public web servers.

Create an Auto Scaling group based on the AMI launch template of the configured Twoge Web server Instance

![image](https://github.com/perryb3693/twoge_app/assets/129805541/6e0cb8d0-92b7-4e96-853b-7232c089ebe2)

Configure Dynamic Scaling Policies configured to add an ec2 instance once CPU utilization has risen to over 80% for over 60 seconds. An alert will be sent to subscribers once parameters are met. 

![image](https://github.com/perryb3693/twoge_app/assets/129805541/ea927ff9-d2a7-4814-af6c-a3dcf038d484)

Conduct a stress test on EC2 instances in order to confirm proper function of the Auto Scaling Group with increase CPU utilization. SSH into EC2 command line interface and run the below script:
```
#!/usr/bin/env python
"""
Produces load on all available CPU cores
"""
from multiprocessing import Pool
from multiprocessing import cpu_count
def f(x):
while True:
x*x
if __name__ == '__main__':
processes = cpu_count()
print ('utilizing %d cores\n' % processes)
pool = Pool(processes)
pool.map(f, range(processes))
```

Use the DNS name of the configured load balance to test functionality of configuration and to publish your first Twoge Blog!

![image](https://github.com/perryb3693/twoge_app/assets/129805541/efbf5334-50fe-4a8f-99c7-fed601dc50ae)

**Future Improvements**
- Multi-AZ Database instance in private subnet 2 a failover for the primary database
- Removal of the NAT gateway in favor of the S3 gateway, depending on networking requirements for the private subnets
- Removal of the "user data" script and provisioning infrastructure through yml configurations through Ansible
