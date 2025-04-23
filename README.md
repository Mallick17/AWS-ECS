# Rollback Strategies in Amazon ECS

In Amazon ECS (Elastic Container Service), a **rollback strategy** defines how ECS handles failed deployments by reverting to a previously stable state. Rollbacks are critical for maintaining application availability and reliability when a new deployment (e.g., updated task definition, new container image, or configuration change) fails. This response explains rollback strategies in ECS in a clear and detailed manner, including how they work, their configuration, and practical examples.

---

### Key Points
- **Rollback in ECS**: Automatically reverts to the last successful deployment if a new deployment fails, ensuring application stability.
- **Supported in**: ECS services using the **rolling update** deployment type with a **CODE_DEPLOY** or **ECS** deployment controller.
- **Configuration**: Uses parameters like `minimumHealthyPercent` and `maximumPercent` to determine when to trigger a rollback.
- **Examples**: Misconfigured task definitions or unhealthy containers can trigger rollbacks to restore the previous stable version.

---

### What is a Rollback Strategy?

A rollback strategy in ECS is a mechanism to undo a deployment that fails to meet health criteria, reverting the service to its previous stable configuration. This is particularly important in production environments where downtime or errors can impact users. Rollbacks are part of the deployment process for ECS services, ensuring that if a new task definition (e.g., updated application code, container image, or environment variables) causes issues, the system can automatically recover.

Rollbacks are supported for ECS services using:
- **Rolling Update Deployment**: Gradually replaces old tasks with new ones.
- **Deployment Controllers**:
  - **ECS Deployment Controller**: Managed by ECS natively.
  - **AWS CodeDeploy Deployment Controller**: For advanced deployment patterns like blue/green deployments.

---

### How Rollbacks Work in ECS

When you update an ECS service (e.g., by deploying a new task definition), ECS monitors the deployment to ensure it meets health criteria. If the deployment fails (e.g., tasks crash, health checks fail, or the service doesn't stabilize), ECS triggers a rollback to the previous successful task definition or configuration.

#### Key Parameters for Rollbacks
ECS uses the following parameters in the service definition to control deployments and rollbacks:
1. **minimumHealthyPercent**:
   - Defines the minimum percentage of tasks that must remain healthy (running and passing health checks) during a deployment.
   - Example: If set to 50% and you have 4 tasks, at least 2 tasks must be healthy at all times.
2. **maximumPercent**:
   - Defines the maximum percentage of tasks that can be running (including new and old tasks) during a deployment.
   - Example: If set to 200% and you have 4 tasks, up to 8 tasks (4 old + 4 new) can run temporarily during the deployment.
3. **Health Checks**:
   - ECS monitors container health via:
     - **Container Health Checks**: Defined in the task definition (e.g., HTTP endpoint checks).
     - **Elastic Load Balancer (ELB) Health Checks**: If the service is behind an Application Load Balancer or Network Load Balancer.
   - If tasks fail health checks, ECS considers the deployment unhealthy and may trigger a rollback.

#### Rollback Triggers
A rollback is triggered when:
- New tasks fail to start (e.g., due to misconfigured task definitions, invalid container images, or resource constraints).
- New tasks fail health checks (e.g., application errors or network issues).
- The service doesn't stabilize within the deployment timeout (e.g., tasks don't reach a `RUNNING` state).
- The `minimumHealthyPercent` threshold is violated (e.g., too many tasks fail, leaving fewer healthy tasks than required).

#### Rollback Process
1. ECS detects a deployment failure based on health checks or task status.
2. ECS stops launching new tasks from the failed task definition.
3. ECS reverts the service to the previous task definition (the last known stable configuration).
4. ECS restarts tasks using the previous task definition, ensuring the `minimumHealthyPercent` is maintained.
5. The service returns to its stable state, and the failed deployment is marked as unsuccessful.

---

### Rollback Strategies in Different Deployment Controllers

#### 1. ECS Deployment Controller (Rolling Update)
- **Description**: The default ECS deployment controller uses a rolling update strategy, where old tasks are gradually replaced with new ones.
- **Rollback Behavior**:
  - If the new tasks fail (e.g., crash or fail health checks), ECS stops the deployment and reverts to the previous task definition.
  - The rollback ensures the service maintains the `minimumHealthyPercent` of healthy tasks.
- **Configuration**: Set `minimumHealthyPercent` and `maximumPercent` in the service definition.

**Example**:
- **Scenario**: You have an ECS service with 4 tasks, using a task definition `web-app:v1`. You update the service to use `web-app:v2`, but `v2` has a bug causing tasks to crash.
- **Service Configuration**:
  ```json
  {
    "serviceName": "web-service",
    "taskDefinition": "web-app:v1",
    "desiredCount": 4,
    "deploymentConfiguration": {
      "minimumHealthyPercent": 50,
      "maximumPercent": 200
    }
  }
  ```
- **Deployment**:
  - ECS starts 4 new tasks (`web-app:v2`), allowing up to 8 tasks total (200% of 4).
  - Old tasks (`web-app:v1`) are gradually stopped as new tasks become healthy.
  - If `web-app:v2` tasks fail health checks, ECS detects the failure (e.g., fewer than 2 healthy tasks, violating 50% minimum).
- **Rollback**:
  - ECS stops launching `web-app:v2` tasks.
  - ECS reverts to `web-app:v1`, restarting 4 tasks using the previous task definition.
  - The service stabilizes with 4 healthy `web-app:v1` tasks.

#### 2. AWS CodeDeploy Deployment Controller (Blue/Green Deployment)
- **Description**: Uses CodeDeploy for advanced deployment strategies like blue/green, where a new task set (green) is deployed alongside the old task set (blue). Traffic is shifted to the green set only after validation.
- **Rollback Behavior**:
  - If the green deployment fails (e.g., tasks fail health checks or validation tests), CodeDeploy automatically shifts traffic back to the blue task set.
  - The green task set is terminated, and the blue task set continues serving traffic.
- **Configuration**: Requires a CodeDeploy application, deployment group, and ECS service configuration with `deploymentController.type` set to `CODE_DEPLOY`.

**Example**:
- **Scenario**: You have an ECS service with 4 tasks (`web-app:v1`, blue). You deploy `web-app:v2` (green) using CodeDeploy blue/green deployment, but `v2` fails ELB health checks.
- **Service Configuration**:
  ```json
  {
    "serviceName": "web-service",
    "taskDefinition": "web-app:v1",
    "desiredCount": 4,
    "deploymentController": {
      "type": "CODE_DEPLOY"
    },
    "loadBalancers": [
      {
        "targetGroupArn": "arn:aws:elasticloadbalancing:region:account-id:targetgroup/my-targets/123456789",
        "containerName": "web-app",
        "containerPort": 80
      }
    ]
  }
  ```
- **CodeDeploy Configuration**:
  - Deployment group with a blue/green deployment strategy.
  - Validation tests (e.g., Lambda functions or manual approval) to check the green deployment.
- **Deployment**:
  - CodeDeploy creates a new task set (`web-app:v2`, green) with 4 tasks.
  - Traffic is initially routed to the blue task set (`web-app:v1`).
  - Once green tasks are running, CodeDeploy shifts traffic to the green task set after validation.
- **Rollback**:
  - If `web-app:v2` tasks fail health checks or validation tests, CodeDeploy shifts traffic back to the blue task set (`web-app:v1`).
  - The green task set is terminated, and the service continues with `web-app:v1`.

---

### Practical Examples

#### Example 1: Rolling Update Rollback Due to Misconfigured Task Definition
- **Setup**:
  - ECS service with 6 tasks, task definition `app:v1`, `minimumHealthyPercent=50`, `maximumPercent=200`.
  - You update to `app:v2`, but `v2` has an invalid environment variable, causing tasks to crash.
- **Deployment**:
  - ECS starts up to 6 new tasks (`app:v2`), allowing 12 tasks total (200% of 6).
  - As `app:v2` tasks crash, the number of healthy tasks drops below 3 (50% of 6).
- **Rollback**:
  - ECS detects the failure and stops deploying `app:v2`.
  - ECS reverts to `app:v1`, restarting 6 tasks.
  - The service stabilizes with 6 healthy `app:v1` tasks.
- **CLI Command to Update Service**:
  ```bash
  aws ecs update-service --cluster my-cluster --service web-service --task-definition app:v2
  ```

#### Example 2: Blue/Green Rollback with CodeDeploy
- **Setup**:
  - ECS service with 4 tasks (`app:v1`, blue), behind an Application Load Balancer.
  - CodeDeploy blue/green deployment for `app:v2` (green).
  - `app:v2` has a bug causing HTTP 500 errors, failing ELB health checks.
- **Deployment**:
  - CodeDeploy creates a green task set (`app:v2`) with 4 tasks.
  - After green tasks start, traffic is shifted to the green task set.
  - ELB health checks detect failures in `app:v2`.
- **Rollback**:
  - CodeDeploy shifts traffic back to the blue task set (`app:v1`).
  - The green task set is terminated.
  - The service continues with 4 healthy `app:v1` tasks.
- **CLI Command to Start Deployment**:
  ```bash
  aws deploy create-deployment \
    --application-name my-app \
    --deployment-group-name my-deployment-group \
    --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
    --description "Deploy app:v2"
  ```

---

### Best Practices for Rollback Strategies

1. **Set Appropriate Health Thresholds**:
   - Use `minimumHealthyPercent` to ensure enough healthy tasks are always running (e.g., 50% or 75% for critical services).
   - Use `maximumPercent` to control resource usage during deployments (e.g., 200% for gradual rollouts, 100% for constrained environments).

2. **Implement Robust Health Checks**:
   - Define container health checks in task definitions (e.g., HTTP `/health` endpoint).
   - Use ELB health checks for services behind load balancers to detect application-level issues.

3. **Test Deployments in Staging**:
   - Test new task definitions in a staging environment to catch issues before production deployments.
   - Use blue/green deployments with CodeDeploy to validate new versions without impacting users.

4. **Monitor and Log Rollbacks**:
   - Use Amazon CloudWatch to monitor deployment events and task health.
   - Enable ECS service events to log rollback triggers (e.g., "service web-service has **(1/4) has failed to deploy, rolling back to previous version").

5. **Use Blue/Green Deployments for Critical Services**:
   - For mission-critical applications, use CodeDeploy blue/green deployments to minimize downtime and validate new versions before shifting traffic.

6. **Automate Rollback Handling**:
   - Use AWS CloudFormation or Terraform to define ECS services with rollback configurations, ensuring consistency and reproducibility.

---

### Summary

Rollback strategies in Amazon ECS ensure that failed deployments do not disrupt application availability. By using rolling updates or blue/green deployments, ECS can automatically revert to a previous stable task definition when a new deployment fails. Key parameters like `minimumHealthyPercent` and `maximumPercent`, along with health checks, determine when a rollback is triggered. Practical examples, such as handling misconfigured task definitions or failed health checks, demonstrate how rollbacks work in both ECS and CodeDeploy controllers. By following best practices, you can configure robust rollback strategies to maintain reliability and minimize downtime in your ECS services.

---

### Key Citations
- [Updating a service - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/update-service.html)
- [Deployment configurations - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [Blue/Green deployments with AWS CodeDeploy - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [Monitoring Amazon ECS - Amazon Elastic Container Service](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/monitoring-ecs.html)
- [AWS CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html)
