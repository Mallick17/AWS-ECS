# Blue/Green Deployment
### What is a Blue/Green Deployment?

A **blue/green deployment** is a safe way to release updates to your app or service without downtime. It works by running **two identical environments**:

* **Blue**: The current live version.
* **Green**: The new version you want to release.

With this method, you test the green version first. If everything looks good, you switch traffic from blue to green. If something goes wrong, you can quickly switch back.

---

### Benefits of Blue/Green Deployment

* **Safer updates**: You test the new version before sending real user traffic.
* **No downtime**: Users won’t notice the switch — your app stays online.
* **Easy rollback**: If there’s a problem, you can go back to the old version (blue) right away.
* **Real-world testing**: Test new changes with real traffic without affecting users.
* **Consistent process**: Each deployment follows clear steps, making it more reliable.
* **Automated checks**: You can run automated tests during each step to make sure things work.

---

### Key Terms

* **Bake time**: The period when both versions (blue and green) are running after traffic is switched.
* **Blue deployment**: The current version running in production.
* **Green deployment**: The new version you’re releasing.
* **Lifecycle stage**: Steps in the deployment process, like testing before traffic is switched.
* **Lifecycle hook**: A special step (like a Lambda function) that checks if the deployment is working.
* **Listener**: Listens for incoming traffic (requests) and sends it to the right version.
* **Rule**: Tells the listener where to send traffic based on conditions.(ALB)
* **Target group**: A group of servers or containers that handle traffic.
* **Traffic shift**: The process of moving all traffic from blue to green.

---

### Things to Keep in Mind

* **More resources needed**: You’re running two versions at the same time, which uses more CPU/memory during deployment.
* **Auto scaling**: If your service auto-scales, it might still scale up/down during deployment — be careful as this could cause failures.
* **Better monitoring**: You get detailed updates on each step of the deployment.
* **Quick rollback**: Since the blue version is still running, you can go back fast if something breaks.

---

<img width="784" height="487" alt="image" src="https://github.com/user-attachments/assets/930e9346-c930-456d-803c-0e8c530e9f1d" />


### 6 Easy Steps in the Deployment Process

1. **Preparation**
   Set up the new version (**green**) alongside the current one (**blue**). ECS gets everything ready like new service versions and target groups.

2. **Deployment**
   ECS starts running the green version in the background. The blue version still handles all user traffic.

3. **Testing**
   ECS can send test traffic to the green version to make sure it works. Regular users still connect to the blue version.

4. **Traffic Switch**
   ECS slowly or instantly shifts real user traffic from blue to green — based on your settings.

5. **Monitoring**
   ECS watches the new green version closely. If there’s a problem, it can automatically roll back to blue.

6. **Completion**
   If everything looks good, ECS can remove the old blue version. Or you can keep it a bit longer just in case.

---

### Blue/Green Deployment Workflow (Simple Overview)

1. **Start**
   All users go to the blue version. The green version is set up but not getting traffic.

2. **Create Green**
   ECS launches the green tasks and adds them to a new target group (a group of servers).

3. **Health Checks**
   ECS checks if the green version is running properly and is healthy.

4. **Test Traffic (Optional)**
   ECS can send some test requests to green (like special test headers) to confirm it works.

5. **Switch Traffic**
   ECS moves real traffic to green — all at once or slowly (your choice).

6. **Watch Closely**
   ECS checks health, logs, and alarms. If there’s a problem, it switches back to blue automatically.

7. **Bake Time**
   Both blue and green versions run for a short while to make sure everything’s fine.

8. **Finish**
   Green version becomes the new live app. Blue version can be shut down or kept as a backup.

---


## Deployment Lifecycle Stages (Defined by AWS to Monitor our Deployments)

| Stage                                   | What's Happening                                                                                        | Use this stage for lifecycle hook? |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------- |
| **1. RECONCILE_SERVICE**               | ECS checks if you have multiple active versions. Only happens in special cases.                         | Yes               |
| **2. PRE_SCALE_UP**                   | Green version hasn't started yet. Blue is live and handling all user traffic.                           |  Yes               |
| **3. SCALE_UP**                        | Green version starts its containers (tasks), but it's not getting any traffic yet.                      |  No                |
| **4. POST_SCALE_UP**                  | Green is now running, but still no traffic. ECS checks if it's running fine.                            |  Yes               |
| **5. TEST_TRAFFIC_SHIFT**             | Green version receives **test traffic** (fake traffic just for testing). Blue still handles real users. |  Yes               |
| **6. POST_TEST_TRAFFIC_SHIFT**       | Test traffic has fully shifted to green. ECS checks if green passed the tests.                          | Yes               |
| **7. PRODUCTION_TRAFFIC_SHIFT**       | Real user traffic slowly moves from blue to green (like 10%, 50%, 100%).                                |  Yes               |
| **8. POST_PRODUCTION_TRAFFIC_SHIFT** | All real traffic is now going to green. Blue is still running for safety.                               |  Yes               |
| **9. BAKE_TIME**                       | Green version is live. ECS waits to monitor if it's working fine.                                       |  No                |
| **10. CLEAN_UP**                       | Blue version is removed. Green is now your new official version.                                        |  No                |


---

## Example Story: Simple Deployment

1. **Blue is live** (your current app).
2. You **deploy Green**.
3. ECS starts Green containers.
4. ECS sends **test traffic** to Green.
5. ✅ Green passed tests? Then go ahead.
6. ECS moves **real users** to Green **slowly**.
7. All good? Then Blue is deleted.

> * **Test Listener**: Special endpoint only for testing green.
> * **Bake Time**: Green runs for a while to ensure it's stable.
> * **Rollback**: If Green has problems, ECS switches back to Blue.

---

Here’s a **very simplified and crystal-clear version** of the **required resources and best practices** for **Amazon ECS Blue/Green Deployments**, written for beginners who want to grow to advanced levels:

---

## What You Need for ECS Blue/Green Deployment

To safely switch from your old app (blue) to your new version (green), **you must use at least one** of these:

### 1. **Load Balancer or Service Connect (Required for Traffic Shifting)**

You need one of the following so ECS can shift traffic smoothly:

* **Application Load Balancer (ALB)**
* **Network Load Balancer (NLB)**
* **Service Connect** (for ECS Services in a mesh-like setup)

> ❗ If you don’t use any of these, you can **still deploy**, but **you won’t get automatic traffic shifting** — it will be manual.

---

## 🛠️ High-Level Setup Checklist

| Step | What to Do                                                              |
| ---- | ----------------------------------------------------------------------- |
| 1  | Use **ALB**, **NLB**, or **Service Connect** in your ECS service        |
| 2  | In the service definition, set **Deployment Type = Blue/Green**         |
| 3  | Set **Deployment Controller = ECS**                                     |
| 4  | (Optional) Add extra features like:                                     |
| →    | **Bake Time** — time to monitor green before final switch               |
| →    | **CloudWatch Alarms** — auto rollback if problems                       |
| →    | **Lifecycle Hooks** — test green version during deployment using Lambda |

---

## Best Practices (Highly Recommended)

| Practice                                         | Why It's Important                                     |
| ------------------------------------------------ | ------------------------------------------------------ |
| Use **accurate health checks**                 | So ECS knows if green is healthy                       |
| Set a **bake time**                            | Wait a few minutes to catch hidden issues              |
| Use **CloudWatch alarms**                      | Auto rollback if CPU/memory/errors spike               |
| Add **lifecycle hooks**                        | Run test scripts or Lambda checks at each stage        |
| Make sure both blue and green can run together | So the switch doesn’t crash the app                    |
| Have **enough ECS capacity**                   | You need to run both versions side-by-side temporarily |
| Test your rollback process                     | Be ready in case something goes wrong                  |


---

<details>
   <summary>Application Load Balancer resources for blue/green deployments</summary>

Here is a **clean, detailed, and CLI-free documentation-style summary** of the [**official AWS ECS guide for Blue/Green deployments with Application Load Balancer**](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb-resources-for-blue-green.html), containing all the **important concepts, configurations, and components**, excluding CLI commands as requested.

---

# Amazon ECS Blue/Green Deployment with Application Load Balancer – Clean Documentation

## Overview

To enable **Amazon ECS Blue/Green deployments with Elastic Load Balancing**, you must configure specific resources that allow ECS to route traffic between the **blue (current)** and **green (new)** versions of your service.

These configurations enable **safe deployments**, **traffic shifting**, and **automatic rollback** if issues are detected.

---

## Key Components

### 1. **Target Groups**

You need **two target groups**:

* **Primary target group** → used by the **blue (current)** service version.
* **Alternate target group** → used by the **green (new)** service version.

#### Target Group Configuration:

* **Target type**: `IP` (for Fargate or EC2 with `awsvpc` network mode)
* **Protocol**: HTTP (or any protocol used by your app)
* **Port**: The port on which your app listens (commonly 80)
* **VPC**: Must be the same VPC where ECS tasks run
* **Health check settings**: Should match your app’s readiness (e.g., `/health`, HTTP 200, thresholds)

> ECS automatically registers blue and green tasks to the correct target groups during deployment.

---

### 2. **Application Load Balancer (ALB)**

You must create an ALB that handles routing to both target groups.

#### Load Balancer Configuration:

* **Scheme**: Internet-facing or internal (based on app requirement)
* **IP address type**: IPv4
* **VPC**: Same VPC as your ECS tasks
* **Subnets**: At least two in different Availability Zones
* **Security Groups**:

  * Inbound rule: Allow traffic on listener ports (e.g., 80/443)
  * Outbound rule: Allow traffic to ECS task security group

---

### 3. **Listeners and Rules**

Listeners define how traffic is routed to target groups.

#### a. **Production Listener** (Required):

* Typically listens on port 80 (HTTP) or 443 (HTTPS)
* Initially routes traffic to the **primary target group (blue)**
* During deployment, updates to point to **alternate target group (green)**

#### b. **Test Listener** (Optional):

* Used to send test traffic to the **green version** before full rollout
* Can listen on a different port (e.g., 8080, 8443)
* Routes traffic only to the **alternate target group (green)**

#### c. **Custom Listener Rules** (Optional):

* Rules can be added to forward traffic based on:

  * **Path patterns** (e.g., `/test/*`)
  * **HTTP headers** (e.g., `X-Environment: test`)
* Useful to route only specific traffic types to the green version during testing

---

### 4. **Service Configuration in ECS**

When configuring the ECS service for Blue/Green deployments, specify:

#### Load Balancer Details:

* **Primary target group ARN**: For blue version
* **Alternate target group ARN**: For green version
* **Container name** and **container port**
* **Production listener rule ARN**: To switch traffic from blue to green
* **Test listener rule ARN** (optional): For pre-deployment testing
* **IAM role ARN**: Allows ECS to manage ALB resources on your behalf

#### Deployment Configuration:

* **Strategy**: `BLUE_GREEN` (enables blue/green mode)
* **Bake time**: Time (in minutes) to wait after switching traffic to green, to monitor performance
* **Minimum Healthy Percent**: e.g., 100 (green must be fully healthy)
* **Maximum Percent**: e.g., 200 (green + blue can run at the same time)

---

### 5. **IAM Role for Load Balancer Management**

ECS needs permission to manage ELB resources during deployment. This is done using a dedicated **IAM role**, often called the **ECS service-linked role**.

Ensure that:

* The IAM role exists with appropriate ELB management permissions
* ECS is allowed to assume this role

Refer to the [ECS IAM Role documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-linked-roles.html) for exact setup.

---

## Traffic Flow During Deployment

Here’s how traffic flows across deployment stages:

1. **Initial State**:

   * All production traffic goes to the **blue** version via the **primary target group**.

2. **Green Deployment Starts**:

   * ECS launches green tasks and registers them with the **alternate target group**.

3. **Test Traffic (Optional)**:

   * If a **test listener** is set up, ECS routes test traffic to the green version.

4. **Production Traffic Shift**:

   * ECS updates the production listener to start routing real user traffic to the green version.

5. **Bake Time Period**:

   * Both blue and green services run. ECS waits for the configured bake time to ensure the green version is stable.

6. **Deployment Complete**:

   * ECS stops the blue tasks.
   * Green becomes the **new production version**.

7. **Rollback (if needed)**:

   * If ECS detects problems (via health checks or CloudWatch alarms), it automatically shifts traffic **back to blue**.

---

## Deployment Visual Diagram

```mermaid
graph TD
    ALB[Application Load Balancer]
    subgraph Before Deployment
        ALB --> BlueTG[Blue Target Group]
        BlueTG --> BlueTask[Blue ECS Tasks]
    end

    subgraph During Deployment
        ALB -->|Test Listener| GreenTG[Green Target Group]
        GreenTG --> GreenTask[Green ECS Tasks]
    end

    subgraph After Bake Time
        ALB -->|Production Listener| GreenTG
        BlueTG -.-> X[Blue Tasks Terminated]
    end

    style X fill=#ffdddd,stroke=#ff0000,stroke-width=2px
```

---

## Summary of Key Configuration Elements

| Component                     | Purpose                                                 |
| ----------------------------- | ------------------------------------------------------- |
| **Primary Target Group**      | Routes traffic to blue version                          |
| **Alternate Target Group**    | Routes traffic to green version                         |
| **Application Load Balancer** | Distributes traffic and manages routing                 |
| **Production Listener Rule**  | Updates during deployment to point to the green version |
| **Test Listener Rule**        | Sends test traffic to green before switching production |
| **IAM Role**                  | Gives ECS permission to manage ELB resources            |
| **Deployment Strategy**       | Set to `BLUE_GREEN`                                     |
| **Bake Time**                 | Wait period for green after traffic shift               |
| **Health Checks**             | Monitors application status                             |
| **CloudWatch Alarms**         | Optional monitoring for automated rollback              |

---

## Conclusion

Using Blue/Green deployments with ALB in ECS allows:

* Safe rollout of new application versions
* Automatic rollback in case of failure
* Controlled and testable deployments

By configuring the load balancer, target groups, listeners, and ECS service appropriately, you can achieve a highly reliable and automated deployment flow.

---

   
</details>
