# Tutorial: scheduled start/stop of EC2 instances managed by Auto Scaling Groups

If your EC2 instances in AWS are managed through Auto Scaling Groups, it is easy to schedule startup and shutdown of those instances, e.g. to save money.

This tutorial walks you through setting up an AWS Lambda function that is triggered by CloudWatch Events and automatically changes the min, max and desired instances in your Auto Scaling Group(s).  
The idea is to toggle between 0 (`stop`) and a specifed min, max and desired amount of instances (`start`), so you only need a single Lambda function.  
The premise is that you do not touch these Auto Scaling Group settings manually, or you might make your EC2 instances nocturnal. 


### Create new Lambda function and Start Event
1. Open AWS Lambda in the console and click `Create a Lambda function`.
2. On the `Select blueprint` page, select `Blank function`.
3. On the `Configure Triggers` page, click the grey square and select `CloudWatch Events`.
4. From the `Rule` dropdown, select `Create a new rule`.
5. Enter a `Rule name` and `Rule description`.
6. For `Rule type`, select `Schedule expression`.
7. For `Schedule expression`, enter a Cron expression to specify at which day(s) and time this event should trigger.  
For now only specify the event to start servers (E.g. at the beginning of the weekday). We will confire the second event later in this tutorial.  
E.g. `cron(00 06 ? * MON-FRI *)` fires every weekday (Monday to Friday) at 6:00 AM UTC. See [this](http://www.cronmaker.com) website for a handy Cron calculator.
8. Enable the `Enable trigger` checkbox.
9. Click `Next`.

### Configure function
10. On the `Configure function` page, enter a `Name` and `Description` for the function.
11. For `Runtime`, select `Python 2.7`.
12. For `Code entry type` make sure `Edit code inline` is selected.
13. Delete the existing code and paste the following code into the code field:
```
import os
import boto3

client = boto3.client('autoscaling')

def get_env_variable(var_name):
    msg = "Set the %s environment variable"
    try:
        return os.environ[var_name]
    except KeyError:
        error_msg = msg % var_name

def lambda_handler(event, context):
    auto_scaling_groups = get_env_variable('NAMES').split()

    for group in auto_scaling_groups:
        if servers_need_to_be_started(group):
            action = "Starting"
            min_size = int(get_env_variable('MIN_SIZE'))
            max_size = int(get_env_variable('MAX_SIZE'))
            desired_capacity = int(get_env_variable('DESIRED_CAPACITY'))
        else:
            action = "Stopping"
            min_size = 0
            max_size = 0
            desired_capacity = 0

        print action + ": " + group
        get_current_min_group_size(group)
        response = client.update_auto_scaling_group(
            AutoScalingGroupName=group,
            MinSize=min_size,
            MaxSize=max_size,
            DesiredCapacity=desired_capacity,
        )

        print response

def servers_need_to_be_started(group_name):
    min_group_size = get_current_min_group_size(group_name)
    if min_group_size == 0:
        return True
    else:
        return False
    

def get_current_min_group_size(group_name):
    response = client.describe_auto_scaling_groups(
        AutoScalingGroupNames=[ group_name ],
    )
    return response["AutoScalingGroups"][0]["MinSize"]
```    
    
14. For `Environment variables` add the following:  
  `NAMES` - Space separated list of the Auto Scaling Groups you want to manage with this function  
  `MIN_SIZE` - Minimum size of the Auto Scaling Group(s) when EC2 instances are started  
  `MAX_SIZE` - Maximum size of the Auto Scaling Group(s) when EC2 instances are started  
  `DESIRED_CAPACITY` - Desired capacity of the Auto Scaling Group(s) when EC2 instances are started  
15. For `Handler`, make sure the value is `lambda_function.lambda_handler`.
16. For `Role`, select `Create a custom role`. An IAM wizard will open in a new tab.

17. For `IAM Role`, select `Create a new IAM Role`.
18. Enter a `Role Name`.
19. Click on `View Policy Document` and click `Edit`. 
20. A warning popup will appear. Click `Ok`.
21. Add the following statement right after the closing } of `"Resource": "arn:aws:logs:*:*:*"
    }`
```
,
{
      "Effect": "Allow",
      "Action": "autoscaling:*",
      "Resource": "*"
    }
 ```
22. Click on `Allow`.

23. Back on the `Configure function` screen, make sure your new Role is selected under `Existing role`.
24. Click on `Next` and `Create Function`.

### Create Stop event
25. To configure the second event (to stop your servers at the end of the weekday), on the `Triggers` tab of your new Lambda function, click on `Add trigger`.
26. In the `Add trigger` popup, from the `Rule` dropdown, select `Create a new rule`.
27. Enter a `Rule name` and `Rule description`.
28. For `Rule type`, select `Schedule expression`.
29. For `Schedule expression`, enter a Cron expression to specify at which day(s) and time this event should trigger.  
E.g. `cron(00 20 ? * MON-FRI *)` fires every weekday (Monday to Friday) at 20:00 PM UTC. See [this](http://www.cronmaker.com) website for a handy Cron calculator.
30. Enable the `Enable trigger` checkbox.


That's it, all done!  
You can test the function by hitting the `Test` button. The first time an `Input test event` popup will appear.  
For `Sample event template` select `Scheduled event` and click `Save and test`.

# Acknowledgement
[This](http://blog.conygre.com/2016/11/18/starting-and-stopping-ec2-instances-using-a-lambda-and-cut-your-aws-bill-in-half/) tutorial helped setting up this tutorial.
