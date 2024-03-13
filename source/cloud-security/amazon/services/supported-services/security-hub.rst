.. Copyright (C) 2015, Wazuh, Inc.

.. meta::
  :description: Learn how to configure Amazon Security Hub findings and insights fetching.

.. _amazon_security_hub:

AWS Security Hub
================

.. versionadded:: 4.9.0

`AWS Security Hub <https://aws.amazon.com/security-hub/>`_ is a cloud security posture management (CSPM) service that automates security best practice checks, aggregates security alerts into a determined format, and understands the overall security posture across all of the user's AWS accounts.

Security Hub runs checks against security controls and generates control `findings <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-findings.html>`_ to help the user assess their compliance against security best practices. Related findings are also grouped in collections called `insights <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-insights.html>`_.

To get the most out of the Security Hub service, Wazuh combines its capabilities with `Amazon SQS <https://aws.amazon.com/sqs>`_ and `EventBridge <https://aws.amazon.com/eventbridge>`_ to process and centralize potential alerts from findings and insights in a single place.

AWS configuration
-----------------

Enabling Security Hub
^^^^^^^^^^^^^^^^^^^^^

   .. note::
      As a prerequisite, it is advised to `configure AWS Config <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-setup-prereqs.html#securityhub-prereq-config>`_ since AWS Security Hub uses service-linked rules from it to perform security checks for most controls.

The documentation establishes two ways to enable AWS Security Hub:
   - `By integrating with AWS Organizations <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html#securityhub-orgs-setup-overview>`_: strongly recommend for multi-account and multi-region environments.
   - `Manually <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-settingup.html#securityhub-manual-setup-overview>`_: for standalone accounts, or if the integration with AWS Organizations is unnecessary.

    .. thumbnail:: /images/aws/security-hub-set-up.png
      :align: center
      :width: 70%

The AWS managed policy called `AWSSecurityHubFullAccess <https://docs.aws.amazon.com/securityhub/latest/userguide/security-iam-awsmanpol.html#security-iam-awsmanpol-awssecurityhubfullaccess>`_ must be attached to the IAM identity to access the Security Hub console and API operations. The policy called `AWSSecurityHubOrganizationsAccess <https://docs.aws.amazon.com/securityhub/latest/userguide/security-iam-awsmanpol.html#security-iam-awsmanpol-awssecurityhuborganizationsaccess>`_ should also be attached to enable and manage the Security Hub through the Organizations integration.

    .. thumbnail:: /images/aws/security-hub-policies.png
      :align: center
      :width: 70%

When integrating Security Hub and Organizations, there is the option to use a feature called `central configuration <https://docs.aws.amazon.com/securityhub/latest/userguide/central-configuration-intro.html>`_ to set up and manage Security Hub for the organization. It is strongly recommended to use central configuration because it lets the administrator customize security coverage for the organization.

Types of Security Hub integration with EventBridge
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Security Hub events and insights can be stored in S3 buckets by integrating Security Hub with EventBridge as it is carried out, for example, for the :ref:`Amazon WAF integration <amazon_waf>`. 
The three available types of events are:

- *Security Hub Findings - Imported*: This event type is automatically sent by Security Hub to EventBridge. It includes all new findings and updates to existing findings. Each event contains a single finding.
- *Security Hub Findings - Custom Action*: This event type is sent by Security Hub to EventBridge when custom actions are triggered. The events are associated with the findings of the custom actions.
- *Security Hub Insight Results*: This event type is used to process the Security Hub Insights. Custom actions can be used to send sets of insight results to EventBridge. Insight results are the resources that match an insight.

   .. note::
      To send the last two types of events to EventBridge, you need to create a custom action in Security Hub. Please refer to the Security Hub `documentation <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cwe-custom-actions.html>`_ to achieve this.

Each type of event contains a `specific format <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cwe-event-formats.html>`_ from which the Wazuh integration takes every relevant ``detail`` field and value along with the ``detail-type`` value.

More information on how to configure each type in the `AWS Security Hub related section <https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-cwe-integration-types.html>`_.

Amazon Simple Queue Service
^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Set up a *Standard* type SQS Queue with the default configurations.  You can apply an Access Policy similar to the following example, where ``<region>``, ``<account-id>``, and ``<s3-bucket>`` are the region, account ID, and the name you are going to provide to the S3 bucket.

   .. code-block:: json

     {
     "Version": "2012-10-17",
     "Id": "example-ID",
     "Statement": [  
       {
         "Sid": "example-access-policy",
         "Effect": "Allow",
         "Principal": {
           "Service": "s3.amazonaws.com"
         },
         "Action": "SQS:SendMessage",
         "Resource": "arn:aws:sqs:<region>:<account-id>:<s3-bucket>",
         "Condition": {
           "StringEquals": {
             "aws:SourceAccount": "<account-id>"
           },
           "ArnLike": {
             "aws:SourceArn": "arn:aws:s3:*:*:<s3-bucket>"
           }
         }
       }
     ]
     }
  
   You can make your access policy to accept S3 notifications from different account IDs and to apply different conditions. More information in `Managing access in Amazon SQS <https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-overview-of-managing-access.html>`_. 

Amazon S3 and Event Notifications
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To configure an S3 bucket that reports creation events, do the following.

#. Configure an S3 bucket as defined in the :doc:`Configuring an S3 Bucket <../prerequisites/S3-bucket>` section. Provide the name you decided in the previous section.
#. Once created, go to **Event notifications** inside the **Properties** tab. Select **Create event notification**. 
#. In **Event Types**, select **All object create events**. This generates notifications for any type of event that results in the creation of an object in the bucket.
#. In the **Destination** section, select the following options:

   -  **SQS queue**
   -  **Choose from your SQS queues**
#. Choose the queue you created previously.

Wazuh Configuration
-------------------

.. warning::
      
   Every message sent to the queue is read and deleted. Make sure you only use the queue for bucket notifications.

#. Edit the ``/var/ossec/etc/ossec.conf`` file. Add the SQS name and your `Configuration parameters`_ for the buckets service. Set this inside ``<subscriber type="security_hub">``. For example:

   .. code-block:: xml
      :emphasize-lines: 6,7

      <wodle name="aws-s3">
          <disabled>no</disabled>
          <interval>1h</interval>
          <run_on_start>yes</run_on_start>
          <subscriber type="security_hub">
              <sqs_name>sqs-queue</sqs_name>
              <aws_profile>default</aws_profile>
          </subscriber>
      </wodle>

   Check the :doc:`AWS S3 module </user-manual/reference/ossec-conf/wodle-s3>` reference manual to learn more about the available settings.

   .. note::
      
      The amount of notifications present in the queue affects the execution time of the AWS S3 module. If the ``<interval>`` value for the waiting time between executions is too short, the :ref:`Interval overtaken <interval_overtaken_message>` warning is logged into the ``ossec.log`` file.

#. Restart the Wazuh manager to apply the changes.

   .. include:: /_templates/common/restart_manager.rst

Configuration parameters
^^^^^^^^^^^^^^^^^^^^^^^^

Configure the following fields to set the queue and authentication configuration. For more information, check the :ref:`subscribers` reference.

Queue
~~~~~

-  ``<sqs_name>``: The name of the queue.
-  ``<service_endpoint>`` - Optional: The AWS S3 endpoint URL for data downloading from the bucket. Check :ref:`using_non-default_aws_endpoints` for more information about VPC and FIPS endpoints.

Authentication
~~~~~~~~~~~~~~

The available authentication methods are the following:

-  :ref:`IAM Roles <iam_roles>`
-  :ref:`Profiles <aws_profile>`

These authentication methods require using the ``/root/.aws/credentials`` file to provide credentials. You can find more information in :ref:`Configuring AWS credentials <amazon_credentials>`.

The available authentication configuration parameters are the following:

-  ``<aws_profile>``: A valid profile name from a Shared Credential File or AWS Config File with permission to read logs from the bucket.
-  ``<iam_role_arn>``: ARN for the corresponding IAM role to assume.
-  ``<iam_role_duration>`` - Optional: The session duration in seconds.
-  ``<sts_endpoint>`` - Optional: The URL of the VPC endpoint of the AWS Security Token Service.
