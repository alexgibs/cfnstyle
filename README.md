# CloudFormation Style Guide - draft v0.0.1

This is an initial draft of a CloudFormation template style guide. The intent is to set some best practice guidelines for directly authoring CloudFormation templates. Please feel free to contribute be cloning this repo and submitting a pull request or starting a discussion by opening an issue.

## Table of Contents

  1. [Syntax](#syntax)
  1. [Comments](#comments)
  1. [Intrinsic Functions](#functions)
  1. [Conditions](#conditions)
  1. [Parameters](#parameters)
  1. [Resources](#resources)
  1. [Outputs](#outputs)
  1. [Stack Architecture](#architecture)

## Syntax

  <a name="syntax--yaml"></a><a name="1.1"></a>
  - [1.1](#syntax--yaml) **YAML**: Use YAML for authoring templates, avoid JSON.

    > Why?

    + YAML allows for comments, which can be helpful for documenting, or for commenting out entire resources while testing.
    + YAML is easier for a human to read, particularly if well commented.
    + YAML avoids any bracket {} scoping issues associated with JSON.

    This guide is focused primarily on YAML syntax however some of the rules will also apply to JSON.

## Comments

  <a name="comments--multiline"></a><a name="2.1"></a>
  - [2.1](#comments--multiline) Use ##...## for multiline comments.

    ```yaml
    # bad
    # S3 bucket for prod webapp assets, access granted to account 12345.
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        ...

    # good
    ##
    # S3 bucket for prod webapp assets,
    # access granted to account 12345.
    ##
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        ...
    ```

  <a name="comments--singleline"></a><a name="2.2"></a>
  - [2.2](#comments--singleline) Place single line comments on a newline above the subject. Put an empty line before the comment unless it is the first line of a resource block or property block.

    ```yaml
    # bad
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead
        BucketName: !Ref WebappSourceBucketName #Bucket needs to be explicitly named due to ...

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        AccessControl: PublicRead

        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName

    # also good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName
    ```

  <a name="comments--space"></a><a name="2.3"></a>
  - [2.3](#comments--space) Start all comments with a space after #. This makes it easier to read.

    ```yaml
    #bad
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        #Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        # Bucket needs to be explicitly named due to ...
        BucketName: !Ref WebappSourceBucketName


    # bad
    ##
    #S3 bucket for prod webapp assets,
    #access granted to account 12345.
    ##
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        ...

    # good
    ##
    # S3 bucket for prod webapp assets,
    # access granted to account 12345.
    ##
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        ...

    ```

## Intrinsic Functions

  <a name="functions--prefer-shorthand"></a><a name="3.1"></a>
  - [3.1](#functions--prefer-shorthand) Use shorthand for all intrinsic functions, where possible. See [below] for nesting intrinsic functions.

    > Why? Shorthand is easier to read and less error prone.

    ```yaml
    # bad
    - Fn::FindInMap:
        - AWSInstanceType2Arch
        - Ref: InstanceType
        - Arch

    # good
    !FindInMap [AWSInstanceType2Arch, !Ref InstanceType, Arch]
    ```

  <a name="syntax--prefer-sub"></a><a name="3.2"></a>
  - [3.2](#references--prefer-sub) Use !Sub (Fn::Sub) instead of !Join (Fn::Join) where possible.

    > Why? !Sub is much cleaner and easier to read. It is less error prone, particularly for UserData. !Sub allows for inline variable substitution, similar to ES2015 template literals.

    ```yaml
    # bad
    UserData:
      !Base64:
        Fn::Join:
        - ''
        - - "#!/bin/bash -xe\n"
          - "yum update -y aws-cfn-bootstrap\n"
          - "/opt/aws/bin/cfn-init -v "
          - "         --stack "
          - Ref: AWS::StackName
          - "         --resource EC2Instance "
          - "         --region "
          - Ref: AWS::Region
          - "\n"
          - "/opt/aws/bin/cfn-signal -e $? "
          - "         --stack "
          - Ref: AWS::StackName
          - "         --resource EC2Instance "
          - "         --region "
          - Ref: AWS::Region
          - "\n"

    # good
    UserData:
      Fn::Base64:
        !Sub |
          #!/bin/bash -xe
          yum update -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
    ```

    ```yaml
    # bad
    NotificationARN: { "Fn::Join": [ ":", [ "arn:aws:sns", { "Ref": "AWS::Region" },{ "Ref": "AWS::AccountId" }, "MySNSTopic" ] ] }

    # good
    NotificationARN: !Sub: arn:aws:sns:${AWS::Region}:${AWS::AccountId}:MySNSTopic
    ```

## Conditions

  <a name="conditions--avoid-complex"></a><a name="4.1"></a>
  - [4.1](#conditions--avoid-complex) Avoid complex conditions.

    > Why? YAML/JSON is not a scripting language, simple conditionals such as 'Should we create this resource based on a parameter value?' are ok however anything more complex becomes difficult to reason about and maintain. If you require complex conditions, look at either re-architecting your approach or look at consuming the CloudFormation Resource Specification <http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-resource-specification.html> and generating your templates programmatically.

## Parameters

  <a name="parameters--names"></a><a name="5.1"></a>
  - [5.1](#parameters--names) A parameter value type (e.g. ARN, name, tag, physical resource id), should be clear from the parameter name.

    > Why? When referencing a parameter from a resource in the template, it can become ambiguous what type of parameter value it contains.

    ```yaml
    # bad
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: !Ref WebappSourceS3Bucket

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: !Ref WebappSourceS3BucketName


    # bad
    InfrastructurePipelineCfnStack:
      Type: AWS::CloudFormation::Stack
      Properties:
        Parameters:
          ApprovalNotificationARN: !Ref CodePipelineSNSTopic

    # good
    InfrastructurePipelineCfnStack:
      Type: AWS::CloudFormation::Stack
      Properties:
        Parameters:
          ApprovalNotificationARN: !Ref CodePipelineSNSTopicARN


    ```

## Resources

  <a name="resources--logical-resource-id"></a><a name="6.1"></a>
  - [6.1](#resources--logical-resource-id) Use descriptive logical resource names with the resource type at the end.

    > Why? When referencing a resource elsewhere in the template, it makes it clear what type of resource you are referencing. Ambiguity can lead to errors.

    ```yaml
    # bad
    WebappSource:
      Type: AWS::S3::Bucket
      Properties:
        ...

    # good
    WebappSourceS3Bucket:
      Type: AWS::S3::Bucket
      Properties:
        ...

    ```

  <a name="resources--physical-resource-id"></a><a name="6.2"></a>
  - [6.2](#resources--physcial-resource-id) Avoid explicitly naming resources.

    > Why? Explicitly naming resources complicates stack updates where this resource needs to be replaced. This is because CloudFormation will create the new resources first, before removing the old one. Avoid becoming attached to names, where possible.

    ```yaml

    # bad
    WebappStatisticsDdbTable:
      Type: AWS::DynamoDB::Table
      Properties:
        AttributeDefinitions:
          ...
        KeySchema:
          ...
        TableName: TableName

    # good
    WebappStatisticsDdbTable:
      Type: AWS::DynamoDB::Table
      Properties:
        AttributeDefinitions:
          ...
        KeySchema:
          ...
    ```

    Note: There are some exceptions to this rule. There might be valid reasons for following a naming convention, or for knowing a resource name up-front. Also to avoid circular dependency issues within a template, a resource name may need to be specified (S3/Lambda for example).

  <a name="resources--securitygroupvpc"></a><a name="6.3"></a>
  - [6.3](#resources--securitygroupvpc) Always be explicit when referencing security group IDs.

    > Why? When referencing a security group using Fn::Ref, it will return a different value depending on whether the security group is in a custom VPC or a default VPC/EC2 Classic. Using Fn::GetAtt to reference the GroupId is more explicit and less error prone.

    ```yaml
    # bad
    HttpPortIngressRule:
      Type: AWS::EC2::SecurityGroupIngress
      Properties:
        GroupId:
          Ref: HttpHostSG
        SourceSecurityGroupId:
          Ref: PublicElbSG
        IpProtocol: tcp
        FromPort: 8080
        ToPort: 8080

    # good
    HttpPortIngressRule:
      Type: AWS::EC2::SecurityGroupIngress
      Properties:
        GroupId:
          !GetAtt: HttpHostSG.GroupId
        SourceSecurityGroupId:
          !GetAtt: PublicElbSG.GroupId
        IpProtocol: tcp
        FromPort: 8080
        ToPort: 8080
    ```

  <a name="resources--no-redundant-dependency"></a><a name="6.4"></a>
  - [6.4](#resources--no-redundant-dependency) Do not add explicitly dependencies when an intrinsic dependency already exists.

    > Why? CloudFormation handles dependency ordering. Adding 'DependsOn' when a reference already exists is unnecessary. It adds bloat to the template and can lead to confusion.

    ```yaml
    # bad
    VPCAttachGateway:
      DependsOn: VPCIGW
      Type: AWS::EC2::VPCGatewayAttachment
      Properties:
        VpcId:
          Ref: VPC
        InternetGatewayId:
          Ref: VPCIGW

    # good
    VPCAttachGateway:
      Type: AWS::EC2::VPCGatewayAttachment
      Properties:
        VpcId:
          Ref: VPC
        InternetGatewayId:
          Ref: VPCIGW
    ```

  <a name="resources--prefer-replacing-updates"></a><a name="6.5"></a>
  - [6.5](#resources--prefer-replacing-updates) Prefer AutoScaling Group Replacing Updates over Rolling Updates.

    > Why? Replacing updates follow the model of creating the new resource before deleting the old. Rolling updates delete the existing instances before creating new ones. This results in the need to perform the rolling update in reverse in order to roll back on failure. Replacing updates allow for quicker rollback and are safer, they should always be used unless there is a specific need for rolling updates.

    ```yaml
    # bad
    WebappAutoScalingGroup:
      Type: AWS::AutoScaling::AutoScalingGroup
      UpdatePolicy:
        AutoScalingRollingUpdate:
          MinInstancesInService: 1
          MaxBatchSize: 2
          WaitOnResourceSignals: true
          PauseTime: PT15M
      Properties:
        ...

    # good
    WebappAutoScalingGroup:
        Type: AWS::AutoScaling::AutoScalingGroup
        UpdatePolicy:
          AutoScalingReplacingUpdate:
            WillReplace: true
        Properties:
          ...
    ```

  <a name="resources--avoid-desired-capacity"></a><a name="6.6"></a>
  - [6.6](#resources--avoid-desired-capacity) Avoid setting a desired capacity on a load-based scaling AutoScaling Group.

    > Why? When an AutoScaling Group is set to dynamically scale based on load, scales the group by modifying the desired capacity. If you have specified a desired capacity in your CloudFormation template, a scaling action is essentially performing an out-of-band change to the resource.

    If you must use a desired capacity along with scaling metrics, and you have also set an UpdatePolicy, you should set IgnoreUnmodifiedGroupSizeProperties to true.

    ```yaml
    # bad
    WebappAutoScalingGroup:
      Type: AWS::AutoScaling::AutoScalingGroup
      UpdatePolicy:
        AutoScalingRollingUpdate:
          MinInstancesInService: 1
          MaxBatchSize: 2
          WaitOnResourceSignals: true
          PauseTime: PT15M
      Properties:
        ...

    # good
    WebappAutoScalingGroup:
        Type: AWS::AutoScaling::AutoScalingGroup
        UpdatePolicy:
          AutoScalingReplacingUpdate:
            WillReplace: true
        Properties:
          ...
    ```

## Outputs

  <a name="outputs--export-names"></a><a name="7.1"></a>
  - [7.1](#outputs--export-names) Use a dynamic value in stack export names.

    > Why? Export names are unique per account per region. In order to promote template re-use, add a dynamic value to your export name.

    ```yaml
    # bad
    PublicSubnet1:
      Description: Public Subnet 1
      Value: !Ref PublicSubnet1
      Export:
        Name: !Sub public-subnet1

    # good
    PublicSubnet1:
      Description: Public Subnet 1
      Value: !Ref PublicSubnet1
      Export:
        Name: !Sub ${Environment}-public-subnet1 # Environment refers to a parameter e.g. 'prod'

    # good
    PublicSubnet1:
      Description: Public Subnet 1
      Value: !Ref PublicSubnet1
      Export:
        Name: !Sub ${AWS::StackName}-public-subnet1
    ```

## Stack Architecture

  <a name="architecture--nesting"></a><a name="8.1"></a>
  - [8.1](#architecture--nesting) Nesting stacks should be avoided unless data needs to be passed both ways between the child and the parent stack.

    > Why? CloudFormation will honour dependency ordering when data is passed to a child stack via parameters and retrieved from the nested stack via Fn::GetAtt. This cannot be done with cross stack referencing.

  <a name="architecture--import-export"></a><a name="8.2"></a>
  - [8.2](#architecture--import-export) Cross stack referencing via import/export values should be used when referencing data is only required in one direction. Do not explicitly pass in a value created by one stack to another stack. E.g. Do not pass in the id of a security group created in Stack1 as a parameter to Stack2.

    > Why? When referencing a value from one stack to another (and not visa-versa), import/export values should always be preferred. CloudFormation handles the dependency ordering for you.

  <a name="architecture--cofig-stack-pattern"></a><a name="8.3"></a>
  - [8.3](#architecture--config-stack-pattern) The config stack pattern involves a single stack used as a configuration source. This stack is used to export configuration values that will then be imported by other stacks. This is particularly useful for resource name values or other external configuration items. A single AWS::CloudFormation::WaitConditionHandle resource can be created as a dummy resource to allow the template to validate and the stack to create.

    > Why? With large environments containing many stacks, it can become difficult to track where values are being set, particularly when parameters are being passed down through multiple levels of nested stacks. Exporting these values from a single stack allows you to use the cloudformation.ListExports() api call to see all values in a single place. The cloudformation.ListImports() api call allows you to see where the exports have been imported.

    ```yaml
    # config-stack example:
    Parameters:
      ResourcePrefix:
        Type: String
      Environment:
        Type: String
        Default: prod

    Resources:
      # CloudFormation requires minimum of one resource in order to launch a stack.
      NullResource:
        Type: AWS::CloudFormation::WaitConditionHandle

    Outputs:
      ResourcePrefix:
        Description: Prefix for all resources created in this environment.
        Value: !Sub ${Environment}-${ResourcePrefix}
        Export:
          Name: !Sub ${Environment}-resource-prefix

    ##############
    # Template importing config-stack values:
    Parameters:
      BucketName:
        Type: String

    Resources:
      S3Bucket:
        Type: AWS::S3::Bucket
        Properties:
          BucketName:
            Fn::ImportValue:
              !Sub ${Environment}-${BucketName}
    ```

## Attribution

The format for this guide has been influenced by the Airbnb Javascript style guide <https://github.com/airbnb/javascript>.

AWS and CloudFormation are trademarks of AWS <https://aws.amazon.com/trademark-guidelines/>.
