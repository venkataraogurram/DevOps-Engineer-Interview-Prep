# 12. Behavioral and lead-level

[← Back to index](README.md)

**Q91. Tell me about yourself.**
Cloud and DevOps engineer with four-plus years across AWS and infrastructure automation. Currently at AWS as a Cloud Support Engineer in the Developer Tools space, where I own enterprise cases end to end on Elastic Beanstalk, EKS, CloudFormation, Control Tower and the CI/CD services: reproducing failures, root-causing them, and delivering validated fixes under SLA. That means I read and debug other people's Terraform, CloudFormation and Kubernetes every day. Outside work I've built production-style platforms end to end: an EKS microservice with Terraform, ALB ingress, Pod Identity, HPA and a GitHub Actions pipeline using OIDC. Before AWS I was at Infosys doing infrastructure and automation. I'm looking to move from fixing platforms to owning and building them, which is why this role interests me.

---

**Q92. Why IDFC FIRST Bank? Why leave AWS?**
I want to own a platform rather than support many. A bank modernising on AWS is where the problems I like, secure automation, compliance as code, reliable delivery, matter most. At AWS I've seen what goes wrong in hundreds of customer environments; I want to apply that to building one well. *[Add what you've learned about their tech direction.]*

---

**Q93. Tell me about a difficult production issue you solved.**
*[Use STAR story 1 from the prep guide. Situation: a failing deployment with a misleading error. Task: restore service under SLA. Action: read the actual logs instead of the obvious theory, reproduced in a sandbox, proved the root cause, delivered the fix. Result: resolved, customer confirmed, and the finding documented for the team. Learning: the first plausible theory is usually wrong; evidence first.]*

---

**Q94. Tell me about a time you disagreed with a stakeholder.**
*[Pick a case where a customer or colleague wanted a quick fix that would have created risk, for example opening a port to 0.0.0.0/0. Explain how you showed the risk concretely, proposed the secure alternative, and got agreement. Result and learning.]*

---

**Q95. How do you handle a change request from a developer that bypasses the standard pipeline?**
Understand the urgency, offer the fastest compliant path, usually an expedited pipeline run with approval, rather than a manual change. If it's a genuine emergency, break-glass with a documented role, logged, and a follow-up PR the same day. Then look at why the standard path felt too slow and fix that.

---

**Q96. How would you introduce Terraform standards to teams that each do their own thing?**
Start by listening: inventory what exists and what hurts. Publish a small set of opinionated modules and a reference repo layout, with a pipeline template that gives teams security scanning and plan review for free. Make the golden path the easiest path, not a mandate. Pilot with one willing team, measure, then expand. Policy as code catches the things that must never happen; everything else is guidance and review.

---

**Q97. How do you keep your skills current?**
Hands-on projects on my own AWS account, reading release notes for Terraform, EKS and the AWS services I work with, and the daily exposure at AWS to new failure modes. Recently: EKS Pod Identity, Terraform 1.10 S3 native locking, and agentic AI tooling for DevOps.

---

**Q98. What is your biggest weakness?**
*[Pick something real and bounded. Example: I've used Packer in my own lab rather than at enterprise scale, so my first month would include learning the bank's image lifecycle requirements in detail. Or: I can go deep on a root cause when a faster mitigation should come first; I've trained myself to roll back first and investigate after.]*

---

**Q99. Where do you see yourself in three years?**
Owning platform architecture for a line of business: the landing zone, the IaC standards, the delivery pipelines, and the reliability targets, and mentoring engineers into that work.

---

**Q100. Do you have questions for us?**
Yes. How is Terraform state and prod apply access managed today? Is Packer already in use for golden AMIs or is that greenfield? Which CI tool is the standard and how does CAB approval integrate? What does the on-call model look like, and what is the biggest reliability pain point right now? How much of the estate is EKS versus EC2 versus serverless, and where is it heading?

---

[← Back to index](README.md)
