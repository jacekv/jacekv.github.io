# AWS default security groups

When you create a new AWS account you get a default security group for each VPC. These security groups are named `default` and they are created in each VPC so that when you launch an instance, the instance is automatically associated with the default security group. The default security group allows all outbound traffic and no inbound traffic.

One AWS security best practice is to not use the default security group. Instead, create a new security group with the rules you need and associate it with your instances.
There is also a rule which can be activated in the AWS security hub, which checks if the default security group allows inbound or outbound traffic.

In order to remove all the default security groups or remove the security group rules, in order to satisfy the AWS security hub rule, you can use the following script:

```bash
#!/bin/sh

REGIONS=$(aws ec2 describe-regions)

echo $REGIONS | jq -r '.Regions[].RegionName' | while read region ; do
    security_groups=$(aws --region $region ec2 describe-security-groups)
    echo $security_groups | jq -r '.SecurityGroups[] | select(.GroupName == "default") | .GroupId' | while read sg_id ; do
        echo "Deleting default security group $sg_id in $region"
        aws --region $region ec2 delete-security-group --group-id $sg_id
        if (( $? > 0 )); then
            revoked_ingress=$(aws --region $region ec2 revoke-security-group-ingress --group-id $sg_id --protocol all)
            echo "Revoked ingress rules: $revoked_ingress"
            revoked_egress=$(aws --region $region ec2 revoke-security-group-egress --group-id $sg_id --ip-permissions 'IpProtocol="-1",IpRanges=[{CidrIp="0.0.0.0/0"}]')
            echo "Revoked egress rules: $revoked_egress"
        fi
    done
done
```

It will list all the default security groups in all the regions and delete them. If you do not have the required permissions to delete a security group, it will instead revoke the ingress and egress security group rule.

In order to run this script, you require the `jq` command line tool.