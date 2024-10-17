---
layout: post
title: From Outage to Advantage — Transforming a Real-Life Incident into a Learning Opportunity
date: 2024-10-17
excerpt: "In this article, I share the journey of how my team transformed a real-life service outage into learning experience. Following the incident, the team recognized the need to build confidence in troubleshooting and identified gaps in their knowledge. To address these challenges, I conducted an Outage Drill, simulating a service failure to provide hands-on experience in tackling complex issues."
---

{% include cover_image.html url="/assets/2024-10-17/poster.png" %}

In software engineering, being prepared for rare but critical incidents is essential to maintaining system reliability and minimizing downtime. For our team, ensuring the seamless operation of a data synchronization service was crucial, as it played a vital role in keeping various systems in sync and ensuring the availability of up-to-date information across the organization. When an unexpected outage occurred, it exposed gaps in our troubleshooting skills and incident response protocols, resulting in a prolonged four-hour recovery process.

This incident served as a wake-up call, highlighting the need for more comprehensive preparation. Rather than just fixing the immediate problem, we saw an opportunity to turn the setback into a learning experience that would strengthen the team’s resilience. In this article, I’ll share how we transformed a real-life outage into a proactive training exercise, detailing the steps we took to build a culture of preparedness, improve our problem-solving capabilities, and establish solid incident management practices.

## Part 1: The Problem – Understanding the Root Cause

The outage affected our data synchronization service, a crucial component that ensures consistency across multiple systems and enables information flow throughout the organization. The disruption not only delayed data updates but also impacted dependent processes, leading to cascading effects across teams.

To understand the underlying factors, I conducted a [5 Whys Analysis]. For those unfamiliar, the 5 Whys Analysis is a problem-solving technique used to identify the root cause of a problem by asking “Why?” repeatedly until the core issue is uncovered. During this process, it became clear that the main source of frustration stemmed from the team’s limited experience with debugging complex production issues. One participant’s comment highlighted the challenge:

<div class="box purple">
   💬 <i>We felt lost because we didn’t have someone with deep knowledge of the system to guide us during the incident.</i>
</div>


This revelation pointed to a critical gap in the team’s confidence and skills when facing unexpected challenges. Without the ability to quickly diagnose and resolve the issue, the team struggled to coordinate their efforts, resulting in a longer-than-necessary outage. It became evident that we needed to address these gaps to prepare the team for future incidents.

To tackle this problem, I decided to organize an Outage Drill, simulating service disruptions in a controlled environment to give the team hands-on experience with real-world troubleshooting scenarios. The goal was to strengthen problem-solving abilities, boost confidence, and improve readiness for handling rare but high-impact incidents.

## Part 2: Preparing for the Drill – Filling Knowledge Gaps

Before conducting the Outage Drill, we needed to address the team’s knowledge gaps that had contributed to the difficulty in resolving the previous incident. Two main areas where expertise was lacking stood out:

* **Kubernetes**: The majority of the team had limited experience with Kubernetes, which is critical for managing deployments and troubleshooting issues. Since Kubernetes is used to orchestrate our microservices, understanding how to access the cluster, inspect running services, and diagnose problems is essential for effective incident response. 
* **Production Debugging Skills**: The team faced challenges with debugging the service, as they were not accustomed to the tools and processes involved. This gap resulted in inefficient troubleshooting during the incident, further emphasizing the need for hands-on practice in a realistic setting.

To bridge these gaps, I created a dedicated Confluence page called “🤸Bootcamp,” serving as a central hub for all service-related documentation and training materials. The resources were designed to provide a comprehensive knowledge base that covered key concepts and skills, including:

* 📜 **How to Find Logs?**: This guide offered a step-by-step approach to accessing and interpreting service logs across different environments, helping team members quickly identify issues and troubleshoot effectively.
* 📈 **Monitoring**: Explained the monitoring setup, detailing how to interpret key metrics and access dashboards in Grafana for real-time insights into service health.
* 🚨 **Alerting & Synchronization Issues**: Provided guidance on understanding alert triggers, handling false positives, and responding to different alert types, as well as troubleshooting synchronization issues.
* ⚙️ **Configuration**: Covered the service’s configuration specifics, including Kubernetes deployment settings and best practices for making configuration changes.
* 🚛 **Deployment**: Offered detailed instructions for manual and automated deployments, along with strategies for troubleshooting deployment failures and performing rollbacks.
* ☁️ **How to Access the Service in Kubernetes?**: Focused on basic Kubernetes operations essential for incident response, such as connecting to the cluster, inspecting pods and services, checking logs, and restarting failing components. It also included an overview of fundamental Kubernetes concepts like pods, services, and namespaces to provide the necessary background.

To ensure the team became comfortable with Kubernetes, I launched a Jira epic called “Kubernetes Access Training” and assigned tickets to each team member. These tickets outlined practical exercises for learning core Kubernetes tasks, such as viewing logs, accessing the service, etc. The goal was to equip the team with hands-on skills that would allow them to effectively navigate the Kubernetes environment and diagnose problems during the drill.

Together with the updated documentation and targeted training, these preparations helped the team to get the knowledge needed to participate in the Outage Drill and tackle potential future incidents.

## Part 3: Planning the Drill – Setting Objectives and Creating a Framework

Setting clear objectives and expected outcomes is essential for effective training. The main goal of the drill was to strengthen the team’s resilience and preparedness for handling unexpected incidents. We aimed to achieve several key outcomes:

* **Identify Weaknesses**: The drill would help uncover gaps in response protocols, communication channels, and collective knowledge, allowing us to address these areas.
* **Improve Coordination**: It would provide an opportunity for the team to practice working together under pressure, improving collaboration during high-stress situations.
* **Boost Preparedness**: The experience would increase the team’s confidence in managing unexpected events, reducing stress and uncertainty during real-life incidents, and enabling quicker, more effective responses.
* **Update Protocols**: Feedback from the drill would be used to refine and improve response protocols, making them more robust.

To prepare for the drill, I documented all the steps involved, establishing a training framework that could be easily replicated for future sessions. I also created supporting materials, such as documentation updates and a briefing presentation to set expectations and provide guidelines for participants. With the groundwork laid, I coordinated the drill scenario, focusing on a critical component failure that would disrupt data synchronization.

To introduce a layer of complexity, the change was made in a less obvious location—a separate [Helm] repository where the deployment configuration was stored. This setup ensured that the simulated failure would challenge the team without overwhelming them, encouraging them to explore various debugging approaches.

## Part 4: The Training – Putting Preparedness to the Test

With all preparations complete, we scheduled the drill. The day before the drill, I disabled continuous deployment to production and introduced a breaking change in the service configuration in the Staging environment. To keep the change subtle and avoid making it too obvious, I asked a colleague from a different team to create the Pull Request.


```diff
SERVICE__ENV: staging
SERVICE__LOG_LEVEL: info
+SERVICE__REDIS__CONNECTION_POOL__SIZE: 0
```
<figure>
    <figcaption>Figure 4 - The breaking change</figcaption>
</figure>


On the day of the drill, an arbitrary Pull Request was merged, triggering the deployment of the new configuration to Staging. Shortly afterward, the service’s monitoring system detected a malfunction. Upon receiving the alert, I initiated a briefing meeting, where I explained the drill’s objectives and clarified that this was a training exercise. I invited the team to join a dedicated Slack channel where they would collaborate during the drill.

The briefing provided a high-level overview of the situation, emphasizing that the details of the failure were intentionally left vague to simulate the dynamics of a real incident. The team was encouraged to diagnose the problem independently, leveraging the knowledge and skills they had developed during the preparation phase.

The drill tested the team’s ability to coordinate under pressure and apply systematic problem-solving approaches. Here’s how the timeline unfolded:

<figure>
    <img src="/assets/2024-10-17/timeline.png"/>
    <figcaption>Figure 6 - Drill Timeline</figcaption>
</figure>


* **Within 5 minutes**: The team quickly identified the issue, finding the stack trace of the exception and sharing it in the Slack channel. This demonstrated an improvement in using the updated documentation and monitoring tools to detect problems.
* **25 minutes**: After investigating various potential causes, the team pinpointed a suspicious configuration change. This step required them to apply gained knowledge on Kubernetes deployment, configuration and logs to narrow down the possibilities.
* **20 minutes**: The issue was successfully reproduced in a local environment for further analysis. The ability to replicate the problem outside of Staging was a key aspect of the training, as it provided a safer setting for testing potential solutions.
* **3 minutes**: Once the root cause was confirmed, a fix was quickly introduced. The team applied what they had learned about configuration management and rollback strategies, ensuring a swift and effective resolution.
* **25 minutes**: The fix was deployed to the Staging environment, restoring the service and concluding the drill.

Throughout the drill, the team demonstrated significant growth in their approach to troubleshooting and incident management. The structured response helped them stay organized, delegate tasks efficiently, and utilize the documentation and tools provided. The entire process took 1 hour and 18 minutes, a significant improvement compared to the initial four-hour recovery time during the real outage.

## Part 5: Lessons Learned – Addressing Gaps and Building a Roadmap

Following the training, I gathered feedback from participants and held a discussion to review the results. This process allowed us to pinpoint areas for improvement and outline a roadmap for the team’s future growth:

* **Configuration**: Navigating and understanding the service configuration proved challenging for some team members.
* **Deployment Pipeline**: There were difficulties in grasping the deployment pipeline.

To address these gaps, we established the following action items.

### Additional Training

Conduct targeted training sessions focused on service configuration and deployment processes to improve the team’s understanding.

### Regular Drills

Schedule another Outage Drill every quarter to maintain and enhance the team’s skills, ensuring they stay prepared for future incidents.


### Document Guidelines – Incident Response & Recovery

After the drill, the team developed the “Incident Response & Recovery” guidelines to establish a structured approach for managing service incidents. These guidelines focused on practical techniques for quickly identifying and resolving outages, and included the following components:
 *	Response Playbooks: A detailed playbooks for common failure scenarios, such as configuration errors, deployment rollbacks, and synchronization issues. They outlined step-by-step instructions for debugging, places to check for potential problems, and methods for verifying system recovery. Additionally, they included guidelines on how to efficiently split the work among team members to ensure a coordinated response.
 *	Post-Incident Review Process: Defined a process for conducting thorough post-incident reviews, including steps for root cause analysis, documentation of lessons learned, and implementation of corrective actions to prevent similar incidents in the future.

Building the "Incident Response & Recovery" guidelines became a valuable experience, equipping the team with a clear approach for rare but critical incidents, helping ensure a quicker, more efficient, and less stressful recovery process when needed.

Feedback indicated that team members felt more confident during the drill compared to the original outage that prompted this initiative. The following quotes highlight the positive impact of the training:

<div class="box purple">
   💬 <i>"I think we were better prepared with documentation about service monitoring, logs, configuration, etc., so we were able to identify the issue and revert the offending Pull Request."</i>
</div>


<div class="box purple">
   💬 <i>"Our documentation was useful; we should keep in mind that we can use it as support."</i>
</div>

## Conclusion

The Outage Drill was more than just a training exercise—it was a transformative experience that turned a real-world outage into a learning opportunity. By identifying and addressing gaps in our collective knowledge, we significantly improved the team’s confidence and capabilities in handling unexpected challenges. The drill demonstrated that with proper preparation, even a newly formed team can effectively troubleshoot complex incidents under pressure.

Reflecting on the initial four-hour recovery during the actual outage, the team’s growth was evident. The reduced time to diagnose and resolve the simulated issue showed that the investments in training and documentation paid off, equipping the team with the skills needed to respond to incidents more quickly and efficiently. The creation of the "Incident Response & Recovery" guidelines, combined with hands-on experience, laid a strong foundation for future incident management.

[5 Whys Analysis]: https://www.atlassian.com/team-playbook/plays/5-whys
[Helm]: https://helm.sh

